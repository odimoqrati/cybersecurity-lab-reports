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


