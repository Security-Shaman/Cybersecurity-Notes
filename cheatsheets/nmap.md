# Nmap Cheatsheet

Personal reference for penetration testing and OSCP lab work.

---

## Basic Syntax
```bash
nmap [flags] [target]
```
Target can be an IP, range, or CIDR:
```bash
nmap 10.10.10.4
nmap 10.10.10.1-254
nmap 10.10.10.0/24
```

---

## Host Discovery

| Flag | Purpose |
|------|---------|
| `-sn` | Ping sweep — discover live hosts, no port scan |
| `-Pn` | Skip host discovery — treat all hosts as up (useful when ICMP is blocked) |
| `-PS` | TCP SYN ping to discover hosts |
| `-PA` | TCP ACK ping |
| `-PU` | UDP ping |
| `-PE` | ICMP echo ping (default) |

```bash
# Discover live hosts on subnet
nmap -sn 10.10.10.0/24

# Skip ping, assume host is up (common on OSCP — many hosts block ICMP)
nmap -Pn 10.10.10.4
```

---

## Scan Types

| Flag | Scan Type | Notes |
|------|-----------|-------|
| `-sS` | SYN scan (stealth) | Default with root — doesn't complete handshake |
| `-sT` | TCP Connect scan | Default without root — completes full handshake, louder |
| `-sU` | UDP scan | Slower, often missed — check for SNMP (161), DNS (53) |
| `-sF` | FIN scan | Bypasses simple firewalls — no response = open |
| `-sN` | NULL scan | No flags sent — no response = open |
| `-sX` | XMAS scan | FIN+URG+PSH — no response = open |
| `-sA` | ACK scan | Maps firewall rules — not for finding open ports |
| `-sV` | Version detection | Identifies service versions on open ports |
| `-sC` | Default scripts | Runs NSE default scripts |
| `-sO` | IP protocol scan | Scans for supported IP protocols |

```bash
# SYN scan (most common, requires root)
nmap -sS 10.10.10.4

# UDP scan (slow — target specific ports)
nmap -sU -p 161,53,67 10.10.10.4

# Full aggressive scan
nmap -A 10.10.10.4
```

---

## Port Specification

| Flag | Purpose |
|------|---------|
| `-p 80` | Scan specific port |
| `-p 80,443,8080` | Scan multiple ports |
| `-p 1-1000` | Scan port range |
| `-p-` | Scan ALL 65535 ports |
| `--top-ports 1000` | Scan top 1000 most common ports (default) |
| `-F` | Fast scan — top 100 ports only |

```bash
# Full port scan — critical for OSCP, don't miss high ports
nmap -p- 10.10.10.4

# Scan specific ports
nmap -p 21,22,80,443,445,3306,3389 10.10.10.4
```

---

## Output & Verbosity

| Flag | Purpose |
|------|---------|
| `-v` | Verbose — shows results as they come in |
| `-vv` | Very verbose |
| `-oN file.txt` | Save output in normal format |
| `-oX file.xml` | Save output in XML format |
| `-oG file.gnmap` | Save output in grepable format |
| `-oA basename` | Save in all formats simultaneously |
| `--open` | Show only open ports |

```bash
# Save output — always do this on OSCP for report writing
nmap -sV -p- -oA scan_results 10.10.10.4
```

---

## Timing & Performance

| Flag | Speed | Notes |
|------|-------|-------|
| `-T0` | Paranoid | IDS evasion, extremely slow |
| `-T1` | Sneaky | IDS evasion, very slow |
| `-T2` | Polite | Slower, less bandwidth |
| `-T3` | Normal | Default |
| `-T4` | Aggressive | Faster, assumes good network |
| `-T5` | Insane | Fastest, may miss results |

```bash
# T4 is standard for OSCP lab environments
nmap -T4 -p- 10.10.10.4
```

---

## OS & Version Detection

| Flag | Purpose |
|------|---------|
| `-O` | OS detection (requires root) |
| `-sV` | Service/version detection |
| `--version-intensity 0-9` | Aggressiveness of version detection (default 7) |
| `-A` | Aggressive — combines -O, -sV, -sC, traceroute |

```bash
# OS fingerprinting — often unreliable, use -A instead
nmap -O 10.10.10.4

# Aggressive scan — most useful single command
nmap -A 10.10.10.4
```

---

## NSE Scripts

| Flag | Purpose |
|------|---------|
| `-sC` | Run default scripts |
| `--script=<name>` | Run specific script |
| `--script=<category>` | Run all scripts in category |
| `--script-args` | Pass arguments to scripts |

**Script categories:**
```
auth, broadcast, brute, default, discovery, dos,
exploit, external, fuzzer, intrusive, malware,
safe, version, vuln
```

```bash
# Run default scripts + version detection (common combo)
nmap -sC -sV 10.10.10.4

# Vulnerability scanning
nmap --script vuln 10.10.10.4

# SMB vulnerability check
nmap --script smb-vuln* -p 445 10.10.10.4

# SMB enumeration
nmap --script smb-enum-shares,smb-enum-users -p 445 10.10.10.4

# HTTP enumeration
nmap --script http-enum -p 80 10.10.10.4

# DNS zone transfer
nmap --script dns-zone-transfer -p 53 10.10.10.4

# FTP anonymous login check
nmap --script ftp-anon -p 21 10.10.10.4

# Sniffer detection
nmap --script sniffer-detect 10.10.10.4
```

---

## Firewall & IDS Evasion

| Flag | Purpose |
|------|---------|
| `-f` | Fragment packets — splits into 8-byte fragments |
| `--mtu <size>` | Custom fragmentation size (multiple of 8) |
| `-D RND:10` | Decoy scan — spoof 10 random source IPs |
| `--source-port 53` | Spoof source port (firewalls often allow DNS/HTTP ports) |
| `--data-length <num>` | Append random data to packets |
| `-sF / -sN / -sX` | FIN/NULL/XMAS — bypass stateless firewalls |
| `--badsum` | Send packets with bad checksums — detect firewalls |

```bash
# Decoy scan
nmap -D RND:10 10.10.10.4

# Source port spoofing (appear as DNS traffic)
nmap --source-port 53 10.10.10.4

# Fragmented packets
nmap -f 10.10.10.4
```

---

## Common OSCP Workflows

```bash
# Step 1 — Quick initial scan to find open ports fast
nmap -T4 --open -p- 10.10.10.4

# Step 2 — Targeted version + script scan on discovered ports
nmap -sC -sV -p 22,80,445 10.10.10.4

# Step 3 — UDP scan on common ports
nmap -sU -p 161,53,67,123 10.10.10.4

# Full one-liner (slower but thorough)
nmap -A -p- -T4 -oA full_scan 10.10.10.4

# When host doesn't respond to ping
nmap -Pn -sV -p- 10.10.10.4
```

---

## Port Reference (Common Services)

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 110 | POP3 |
| 139/445 | SMB |
| 143 | IMAP |
| 161 | SNMP (UDP) |
| 389 | LDAP |
| 443 | HTTPS |
| 3306 | MySQL |
| 3389 | RDP |
| 5432 | PostgreSQL |
| 8080 | HTTP Alt |
| 8443 | HTTPS Alt |

---

*Part of Security-Shaman cybersecurity notes — github.com/Security-Shaman*
