---
title: "Write-Up: CAP Machine Walkthrough Practice OSP/CEH"
layout: post
date: 2025-02-12
tags: [penetration-testing, IDOR, privilege-escalation, Linux, cybersecurity]
categories: [Write-Ups]
filename: "2025-02-11-cat-htb-writeup.md"
---
![1_ZI68WzEQraOMcF1Y6tcElA](https://github.com/user-attachments/assets/524fbad7-de22-48e7-9fa7-5835e385f747)

## Introduction
The CAP machine is an easy Linux-based challenge focused on web enumeration, packet analysis, and exploiting Insecure Direct Object Reference (IDOR) vulnerabilities. This walkthrough demonstrates how improper controls can expose sensitive data and explores how Linux capabilities can be exploited for privilege escalation.

## Skills Required
- Web enumeration  
- Packet capture analysis  

## Skills Learned
- Understanding and exploiting IDOR  
- Leveraging Linux capabilities for privilege escalation  

## Step 1: Enumeration with Nmap
We begin by scanning all TCP ports on the target machine (`10.10.10.245`) to identify open services. Here's the command used:

```bash
ports=$(nmap -p- --min-rate=1000 -Pn -T4 10.10.10.245 | grep '^[0-9]' | cut -d'/' -f1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -Pn -sC -sV 10.10.10.245
```

### Result: Three open ports
- **21**: FTP  
- **22**: SSH  
- **80**: HTTP  
![1_LaRSiUkfz2Hz9nFavtrXdQ](https://github.com/user-attachments/assets/9f59ae46-fd41-4dae-a148-7fb4e3979246)

## Step 2: FTP Enumeration
We tested FTP for anonymous access, but it was disabled. So, we moved on to exploring the HTTP service.
![1_38m5Uq3FaJbcqy8IWM-W7A](https://github.com/user-attachments/assets/883f3ce8-5d7a-496f-8085-daf901e66c55)

## Step 3: HTTP Service Exploration
Port 80 is running Gunicorn, a Python-based HTTP server. The website displayed a dashboard with system and network information, hinting at system commands being executed in the backend.

- **IP Config page**: Displays `ifconfig` output.
- **Network Status page**: Displays `netstat` output.
![1_Sxa7CFB5CUNf1d4-jqkYIg](https://github.com/user-attachments/assets/7f04c7fd-e9f4-4535-8ce9-9f9e065992af)

Clicking on the Security Snapshot menu item pauses the page for a few seconds and returns a downloadable packet capture file. 
![1_DlE48pR6EpqKRq8dDeS2tA](https://github.com/user-attachments/assets/c031c9c0-2008-4a99-928c-24c13a773ad4)
![1_j121MnkzIGuS_ATEnsm33w](https://github.com/user-attachments/assets/5fe689ac-54c4-4d1b-820d-ab58538b8d37)
![1_A1YEcjsdHa7h6YtxMr8Xlg](https://github.com/user-attachments/assets/81869e91-dda2-4057-8833-38527f1f948f)

## Step 4: Exploiting IDOR
While browsing, we noticed the application generating packet captures at `/data/<id>` with incremental IDs. By manually navigating to `/data/0`, we accessed a previous user's capture file.
![1_XACOGZN1jHQfgA7XKneqWg](https://github.com/user-attachments/assets/75eeee6f-0362-4e98-8fae-d957f42629d9)

This vulnerability is known as **Insecure Direct Object Reference (IDOR)**, where insufficient access controls allow users to access others' data.

## Step 5: Foothold
Downloading and analyzing the packet capture with Wireshark revealed FTP credentials transmitted in plaintext:

```
Username: nathan
Password: Buck3tH4TF0RM3!
```
![1_ap2ySbSpRPeXgQTQHabdxw](https://github.com/user-attachments/assets/89636320-c140-4182-8226-78f50c7de5cd)
![1_NrRyvC-Lkth8M-dJSWY9Uw](https://github.com/user-attachments/assets/57a4fe81-db99-41f4-8770-8e9c8a4e6836)

These credentials worked for both FTP and SSH. We used SSH to gain access.

```bash
python -m http.server
```
![1_hckR8k0HpZvYRuH8-czz4w](https://github.com/user-attachments/assets/fef4dcd4-a2f2-40a9-a40e-7998e6851589)

## Step 6: Privilege Escalation
Using the **linPEAS** script, we identified an unusual Linux capability set on `/usr/bin/python3.8`:
![1_PhMlsVSD_ULSK1wYgyUwJg](https://github.com/user-attachments/assets/38d98717-fef8-4d54-946d-dc228120dcc3)
![1_PhMlsVSD_ULSK1wYgyUwJg](https://github.com/user-attachments/assets/fb9623b0-7599-4292-9d93-f9382fb4bbfb)
![1_5HG4GQFIlyO6oCDE2yvIUA](https://github.com/user-attachments/assets/2e3ec508-c7b4-40e2-ab1c-b0750e79d396)

```
CAP_SETUID: Allows switching to UID 0 (root).
```

Python was likely granted this capability for non-root users to capture traffic. Using the following script, we escalated privileges to root:

```python
import os
os.setuid(0)
os.system("/bin/bash")
```
![1_AyWLqGfMnPwIjiQZzW_H_Q](https://github.com/user-attachments/assets/a3f2428c-9107-4ade-a6ce-b93c9f0703c6)

The `os.setuid(0)` function modifies the process user identifier (UID), granting root access.

## Conclusion
This machine reinforced the importance of proper access controls (to prevent IDOR) and secure Linux capability settings. The challenge highlights common oversights in web application and system configurations.

### Key Takeaways:
- Test all potential vulnerabilities during web enumeration.
- Be vigilant about sensitive data exposed in network traffic.
- Monitor and restrict Linux capabilities effectively.
