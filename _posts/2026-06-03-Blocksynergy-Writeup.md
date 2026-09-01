---
title: Blocksynergy - Hack The Box Writeup
date: 2026-06-03
categories: [CTF, Writeups, HTB]
tags: [htb, linux, business-logic, rce, path-traversal, suid, race-condition, privilege-escalation]
image:
  path: https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a29813ab-2643-4f77-80fd-65500f97bcd0-1787750589.png
  description: A walkthrough of the Blocksynergy HTB machine, covering wallet balance forgery, a debug-hook path traversal pivot, and a restore-daemon inotify race for root.
author: daemonyrax
---

## Nmap Recon and Initial Enumeration

To begin, we add the target to our hosts file so `blocksynergy.htb` resolves locally:

```bash
echo "10.10.x.x  blocksynergy.htb" | sudo tee -a /etc/hosts
```

Next, we run an Nmap scan to identify open ports and services on the target:

```bash
nmap blocksynergy.htb
```

A follow-up scan against the discovered ports adds version detection, OS fingerprinting, and default NSE scripts:

```bash
nmap -p22,8080 -sC -sV -O -oA complete-scan.txt blocksynergy.htb
```

The scan confirms SSH on port 22 and a web application on port 8080. From here we continue manual enumeration of both services before touching anything else.

## Exploring the Application

The web application on port 8080 is centered around an in-app wallet/economy system — users hold a balance that gates access to a VIP tier of functionality. That balance check turns out to be enforced in a way that can be manipulated directly, which becomes the entry point for the rest of the chain.

Rather than reverse-engineer the wallet logic by hand every time, we package the full chain into a single exploit script:

```bash
git clone https://github.com/ans-inayat/blocksynergy-htb
cd blocksynergy-htb
```

## Identifying Vulnerabilities

### 1. Wallet Balance Forgery (Business Logic Flaw)

The wallet's balance value can be forged/inflated without going through the application's normal earning flow. Once funded, this immediately unlocks the VIP-gated code path — access meant to be reserved for users who've legitimately built up their balance.

### 2. Debug-Hook Path Traversal (Port 5000)

A debug hook is exposed internally on port 5000. It doesn't validate the path it's given, so a path traversal payload against it is enough to pivot from the low-privileged web user into a second local account.

### 3. Inotify Race Condition in the Restore Daemon

A restore daemon watches a working directory (`/var/restore_work/`) via `inotify` and processes any archive dropped into it on a timed cycle. Because it acts on the file some time after it appears rather than atomically, there's a window between the check and the action — a classic TOCTOU race — that a crafted archive can win to gain root.

## Exploitation

With the wallet forgery unlocking VIP and the code path it exposes, running the script gets us remote code execution as the web user:

```bash
python complete-exploit.py -t blocksynergy.htb
```

```
[*] step 1: wallet 'w84eb2329' + forge coins
[*]     balance funded, VIP unlocked
[*] step 3: RCE as the web user (id):
[*]     uid=1000(walter) gid=1000(walter) groups=1000(walter)
```

### Lateral Movement

From here, we hit the debug hook on port 5000 with a traversal payload to pivot to the box's second local user and land an SSH session:

```
[*] step 4: lateral to the dev user via :5000 debug-hook traversal
[*]     dev user = hank
[*]     SSH as hank established
```

`user.txt` isn't in any of the common locations for this account. A SUID binary on the box provides the read access needed to retrieve it instead:

```
[*] step 4.5: retrieving user flag...
[*]     user flag not in common locations, searching...
[*]     trying with SUID binary to read user flag...
[*]     user flag found via SUID binary
```

### Privilege Escalation

For root, the script targets the restore daemon's inotify-driven watch cycle. It drops a crafted archive into the daemon's working directory and races the window between the daemon noticing the file and acting on it:

```
[*] step 5: root via restore-daemon inotify race
[*]     payload: /var/restore_work/.pl_84eb2329.tar.gz 689837
[*]     racer armed; waiting for the restore cycle (up to ~7 min)...
[*] root achieved.
```

```
============================================================
BLOCKSYNERGY FLAGS:
============================================================
USER FLAG  : b081440fcb190xxxxxxxxxxxxxxxxxxxxx
ROOT FLAG  : c72b91821e7f58xxxxxxxxxxxxxxxxxxxx
============================================================
```

---

*This is a quick, one-shot exploit script meant for the HTB lab environment — not a template for testing production-grade applications. Use it if the goal is just to pwn the box fast and bump your HTB rank. If the goal is to actually learn, skip the script, enumerate the machine fully and manually, and treat the stages above (wallet abuse → RCE → debug-hook pivot → SUID → inotify race) as a guide for what to look for, not the solution itself.*

This concludes the walkthrough for the Blocksynergy HTB machine. We exploited a wallet balance forgery, a debug-hook path traversal, a SUID binary, and an inotify race condition in a restore daemon to gain full access to the box. Happy hacking!
