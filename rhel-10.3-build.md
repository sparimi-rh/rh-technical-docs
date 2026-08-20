# RHEL-10.3 virt-tools Build For Downstream release

This document outlines the procedure for preparing downstream virt-tools builds for RHEL 10.3, serving as a reference guide.

## Table of Contents
- [1. Environment & Workspace Setup](#1-environment--workspace-setup)
  - [1.1 Development Host Provisioning](#11-development-host-provisioning)
  - [1.2 Directory Hierarchy Setup](#12-directory-hierarchy-setup)
- [2. Upstream Synchronization & Patch Backporting](#2-upstream-synchronization--patch-backporting)
  - [2.1 Repository Cloning](#21-repository-cloning)
  - [2.2 Feature & Fix Backporting](#22-feature--fix-backporting)
- [3. Local Compilation & Sanity Verification](#3-local-compilation--sanity-verification)
- [4. Downstream Synchronization Setup](#4-downstream-synchronization-setup)
  - [4.1 Brief Overview](#41-brief-overview)
  - [4.2 Installing centpkg, Kerberos setup](#42-installing-centpkg-kerberos-setup)
  - [4.3 Local Copies of Downstream GitLab-Dist Repos](#43-local-copies-of-downstream-gitlab-dist-repos)
  - [4.4 Request CentOS Stream Koji](#44-request-centos-stream-koji)
- [5. Downstream Build Verification](#5-downstream-build-verification)

---

## 1. Environment & Workspace Setup

### 1.1 Development Host Provisioning
* Deployed RHEL 10.2 development virtual machine on ESXi hypervisor.
* Installed standard development toolchain and compiler dependencies.<br>

### 1.2 Directory Hierarchy Setup
* Created primary release workspace: `rhel 10.3/`
* Further, created subfolder layout:
  * `rhel 10.3/upstream/` (Source repositories & backport folder)
  * `rhel 10.3/downstream/` (Package spec files & `centpkg` execution folder)

---

## 2. Upstream Synchronization & Patch Backporting

### 2.1 Repository Cloning
* Cloned `libguestfs` into `upstream/` directory.
* Cloned `virt-v2v` into `upstream/` directory.

### 2.2 Feature & Fix Backporting
* Checked out target branch `rhel-10.3`.
* Cherry-picked targeted commits from upstream `master`. Reference command `git cherry-pick -x <commit hash>`
* Synchronized and updated git submodule state (e.g., `common` submodule via `git update-index --cacheinfo`). These command were run as part of normal GIT process.

---

## 3. Local Compilation & Sanity Verification

* Local Compilation & Workspace Maintenance: Resolved dependencies, cleared stale environment caches (dnf clean all / supermin rebuilds), and successfully compiled and verified binaries for libguestfs and virt-v2v.

* Remote Branch Synchronization: Verified local commit history against the remote branch and safely updated the GitLab fork `(git push --force-with-lease origin rhel-10.3)` to safely update the remote branch.

These steps ensure that all pushed code changes are included.

---

## 4. Downstream Synchronization Setup

### 4.1 Brief Overview
* Downstream build depends on the following components
  - `Centpkg` - wrapper around `rpkg` which is python based library to manage RPM packages Fedora. Centpkg interacts with RPM dist-git repositories hosted on GitLab. Refer [centpkg](https://gitlab.com/CentOS/common/centpkg)
  - `koji build system` - Build system for CentOS stream which helps in spinning temporary build environments. Refer [koji](https://docs.pagure.org/koji/)
  - `GitLAB Account` - repository of RPM spec files etc. which are inputs to builds.

### 4.2 Installing centpkg, Kerberos setup
* Setup yum repos in the build VM - copy these repo configs to `/etc/yum.repos.d`
  - rcmtools repo [rcmtools](http://download.devel.redhat.com/rel-eng/RCMTOOLS/rcm-tools-rhel-10-baseos.repo)
  - Centos repo needed for AppStream [Centos Repo](centos-10-stream.repo)
* Install necessary tools for downstream operations
```
# Install centpkg 
sudo dnf install centpkg
# Tools like `centpkg` need kerberos utilities to connect to koji build system
sudo dnf install krb5-workstation 
# Install rhel-packager
sudo dnf install rhel-packager
```
* Since we are using an RHEL 10.2 VM and not Fedora CSB, certain certificates are needed to be installed in `/etc/pki/ca-trust/source/anchors/`. If these certs are already installed, skip this step
  - [2015-IT-Root-CA](2015-IT-Root-CA.pem)
  - [2022-IT-Root-CA](2022-IT-Root-CA.pem)
  (These files links may not be accessible directly - see below)
  - Access these certificate files here [rhel-developer-toolbox](https://gitlab.com/redhat/rhel/tools/rhel-developer-toolbox/-/tree/main/files?ref_type=heads)
  - Run the command `sudo update-ca-trust extract`


### 4.3 Local Copies of Downstream GitLab-Dist Repos
* Move into `rhel 10.3/downstream/` folder
```
# Create local copies of `libguestfs, virt-v2v` downstream repos inside
# the rhel
centpkg clone libguestfs
centpkg clone virt-v2v
```
### 4.4 Request CentOS Stream Koji
* Side-tag in CentOS Stream Koji is needed to build and gate multiple builds (libguestfs, virt-v2v).
* For reference [RHEL Development Guide](https://one.redhat.com/rhel-development-guide/)
``` 
# Fetch the Kerberos Ticket - enables centpkg to comnmunicate with koji
# build system
sparimi@dhcp-6-172-207 ~ $ kinit sparimi@IPA.REDHAT.COM
Password for sparimi@IPA.REDHAT.COM:  
sparimi@dhcp-6-172-207 ~ $ 

# Verify connectivity with koji build system.
sparimi@dhcp-6-172-207 ~ $ koji -p stream hello
bonjour, sparimi!

You are using the hub at https://kojihub.stream.rdu2.redhat.com/kojihub (Koji 1.36.1)
Authenticated via GSSAPI
sparimi@dhcp-6-172-207 ~ $

# Request gated side tag in CentOS Stream Koji - needed to build and 
# gate multiple builds - in this case libguestfs, virt-v2v

sparimi@dhcp-6-172-207 ~ $ centpkg request-gated-side-tag 
Side tag 'c10s-build-side-5010-stack-gate' (id 5010) created.                                  
Use 'centpkg build --target=c10s-build-side-5010-stack-gate' to use it.                        
Use 'koji -p stream wait-repo c10s-build-side-5010-stack-gate' to wait for the build repo to be
 generated.
```
* The tag returned in the above command is used in the builds. It needs to be generated ONLY once as multiple builds are included.

---

## 5. Downstream Build Verification
* The steps below are common across `libguestfs, virt-v2v` folders in the downstream workspace
  - Run the `copy-patches.sh` which generates patch file from rhel-10.3 upstream workspace based on version
  - Update the `patches` section in the `libguestfs.spec`
  - Update the `changelog` section based on the cherry-picked commits.
```
# Fork the repo in the personal GitLAB space.
centpkg fork

# Create a branch
git branch -c test-release-branch

# Add the modified/created files
git add < files modified >

# Commit the changes
centpkg commit -c

# push the changes
git push sriparim
```

---

