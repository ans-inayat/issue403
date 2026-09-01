---
title: "HackTheBox — Blocksynergy Writeup"
author: daemonyrax
pubDatetime: 2026-06-03T00:00:00Z
customSlug: htb-blocksynergy-writeup
featured: false
draft: false
tags:
  - hackthebox
  - linux
  - business-logic-abuse
  - rce
  - path-traversal
  - suid
  - race-condition
  - inotify
  - privilege-escalation
  - lateral-movement
description: >
  Quick exploit notes for HackTheBox Blocksynergy — a Linux box built around
  an in-app wallet/economy system. Chain includes forging a wallet balance to
  unlock a VIP-gated code path for RCE as a web user, a debug-endpoint path
  traversal pivot to a second local user, user-flag recovery via a SUID
  binary, and root via an inotify race in a restore daemon.
---

Blocksynergy centers on a web application with an in-app wallet/economy
system. The chain starts by forging a wallet balance to unlock a VIP-gated
code path, which leads to remote code execution as a low-privileged web
user. From there, a debug endpoint on port 5000 allows a path-traversal
pivot to a second local user, and privilege escalation to root abuses a race
condition in a restore daemon that watches a directory via inotify.

> These are condensed usage notes built around a bundled exploit script
> rather than a full technique-by-technique breakdown — see the [exploit
> repo](https://github.com/ans-inayat/blocksynergy-htb) for the complete
> implementation.

> **Disclaimer:** This is a quick, one-shot exploit script meant for the
> HTB lab environment — it is **not** a template for testing or attacking
> production-grade applications. Use it if your goal is just to pwn the box
> fast and bump your HTB profile rank. If your goal is to actually learn,
> skip the quick exploit, enumerate the machine fully and manually, and
> treat this write-up as a reference for how each stage works rather than
> a shortcut to root.

## Table of contents

## Machine Info

| Field        | Detail                                              |
| ------------- | ------------------------------------------------------ |
| OS            | Linux                                                  |
| Target        | `blocksynergy.htb`                                     |
| Target IP     | `[TARGET_IP]`                                          |
| Exploit repo  | `https://github.com/ans-inayat/blocksynergy-htb`        |

## Attack Chain Overview

```
Wallet balance forgery → VIP-gated path unlocked
  → RCE as web user (walter)
  → :5000 debug-hook path traversal → lateral to dev user (hank) via SSH
  → user flag recovered via SUID binary
  → restore-daemon inotify race → root
```

## Step 1 — Add Target to /etc/hosts

```text
[TARGET_IP]  blocksynergy.htb
```

Replace `[TARGET_IP]` with the actual IP address of the target in your lab
environment.

## Step 2 — Enumeration

Start with a broad scan, then a targeted one against the discovered ports:

```bash
# basic host discovery / service scan
nmap blocksynergy.htb

# targeted full scan: common services, NSE scripts, version and OS detection
nmap -p22,8080 -sC -sV -O -oA complete-scan.txt blocksynergy.htb
```

Continue manual enumeration of whatever services the scan turns up (web,
SSH, etc.) before moving on — the bundled exploit assumes some of that
groundwork is already done.

## Step 3 — Run the Bundled Exploit

If the exploit needs a particular profile, rank, or configuration and you
don't have it yet, keep enumerating manually until you've obtained the
required access. Once ready:

```bash
git clone https://github.com/ans-inayat/blocksynergy-htb
cd blocksynergy-htb
python complete-exploit.py -t blocksynergy.htb
```

## Step 4 — Exploit Walkthrough

Example run, with sensitive values censored — this shows the high-level
stages and outcome without exposing full flags or secrets:

```
[*] step 1: wallet 'w84eb2329' + forge coins
[*]     balance funded, VIP unlocked
[*] step 3: RCE as the web user (id):
[*]     uid=1000(walter) gid=1000(walter) groups=1000(walter)
[*] step 4: lateral to the dev user via :5000 debug-hook traversal
[*]     dev user = hank
[*]     SSH as hank established
[*] step 4.5: retrieving user flag...
[*]     user flag not in common locations, searching...
[*]     trying with SUID binary to read user flag...
[*]     user flag found via SUID binary
[*] step 5: root via restore-daemon inotify race
[*]     payload: /var/restore_work/.pl_84eb2329.tar.gz 689837
[*]     racer armed; waiting for the restore cycle (up to ~7 min)...
[*] root achieved.
============================================================
BLOCKSYNERGY FLAGS:
============================================================
USER FLAG  : b081440fcb190xxxxxxxxxxxxxxxxxxxxx
ROOT FLAG  : c72b91821e7f58xxxxxxxxxxxxxxxxxxxx
============================================================
```

**User and root flags captured.**

## Key Takeaways

**Business/economy logic is an attack surface, not just a feature.** Forging
a wallet balance to unlock a VIP-gated code path shows how in-app currency
or rank checks can become a straight line to RCE if they're trusted
client-side or under-validated server-side.

**Secondary ports are common lateral-movement vectors.** A debug hook
exposed on `:5000` allowed a path traversal that pivoted from the low-priv
web user straight to a second local account over SSH.

**Check SUID binaries when a flag isn't where you expect.** When `user.txt`
wasn't in a common location, a SUID binary provided the read access needed
to retrieve it.

**inotify-driven restore/watch daemons create TOCTOU race windows.** A
daemon that reacts to files dropped into a watched directory can often be
raced between the check and the action — here, arming a payload and waiting
for the restore cycle was enough for root.

## Notes

- Some `nmap` options (OS detection, certain NSE scripts) require `sudo`.
- Only run this exploit against machines you own or have explicit permission
  to test.
- Adjust IPs, paths, and filenames to match your own environment.
- This exploit is a shortcut for the lab, not a demonstration of a technique
  safe to run against real, production-grade applications.
- Ranking up on HTB: the one-shot script gets you to root fast. Learning the
  box: skip it, enumerate everything yourself, and use the exploit's stages
  (wallet abuse → RCE → debug-hook pivot → SUID → inotify race) as a guide
  for what to look for at each step, not as the solution itself.
