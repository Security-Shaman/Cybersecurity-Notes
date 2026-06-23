# SNMP Enumeration Cheatsheet

Personal reference for penetration testing and OSCP lab work.

---

## What is SNMP

Simple Network Management Protocol — used to monitor and manage network devices.
Runs on **UDP port 161.**

**Community strings** act as passwords:
```
public  → read-only access (default, often unchanged)
private → read-write access
```

SNMPv1/v2 transmit community strings in plaintext — major weakness.
SNMPv3 adds encryption and proper authentication.

---

## Workflow

```
1. Nmap    → confirm SNMP port 161 is open (UDP)
2. onesixtyone → brute force community string
3. snmpwalk    → extract data using valid community string
```

---

## Step 1 — Nmap Discovery

```bash
# Confirm SNMP is running
nmap -sU -p 161 <target>

# With NSE scripts
nmap -sU -p 161 --script snmp-info,snmp-sysdescr <target>

# Scan entire subnet for SNMP hosts
nmap -sU -p 161 192.168.1.0/24
```

---

## Step 2 — onesixtyone (Community String Brute Force)

```bash
# Single target
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <target>

# Multiple targets from file
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt -i targets.txt

```

---

## Step 3 — snmpwalk (Data Extraction)

```bash
# Full walk — dumps everything (verbose, use specific OIDs instead)
snmpwalk -c public -v2c <target>
snmpwalk -v1 -c public <target>

# Specific OID query
snmpwalk -c public -v2c <target> <OID>
```

---

## Windows SNMP OIDs

| OID | Returns |
|-----|---------|
| `1.3.6.1.2.1.25.1.6.0` | Total running processes count |
| `1.3.6.1.2.1.25.4.2.1.2` | Running process names |
| `1.3.6.1.2.1.25.4.2.1.4` | Process executable paths |
| `1.3.6.1.2.1.25.2.3.1.4` | Storage unit allocation |
| `1.3.6.1.2.1.25.6.3.1.2` | Installed software list |
| `1.3.6.1.4.1.77.1.2.25` | Windows user accounts |
| `1.3.6.1.2.1.6.13.1.3` | Open TCP ports |
| `1.3.6.1.2.1.1.5.0` | Hostname |
| `1.3.6.1.2.1.1.4.0` | Contact email |
| `1.3.6.1.2.1.1.6.0` | Location |

```bash
# Running processes (most useful for OSCP)
snmpwalk -v1 -c public <target> 1.3.6.1.2.1.25.4.2.1.2

# Windows user accounts
snmpwalk -v1 -c public <target> 1.3.6.1.4.1.77.1.2.25

# Installed software
snmpwalk -v1 -c public <target> 1.3.6.1.2.1.25.6.3.1.2

# Open TCP ports
snmpwalk -v1 -c public <target> 1.3.6.1.2.1.6.13.1.3
```

---

## SNMP Versions

| Version | Security | Notes |
|---------|----------|-------|
| v1 | Plaintext community string | Oldest, least secure |
| v2c | Plaintext community string | Most common in labs |
| v3 | Encrypted, authenticated | Secure, rare in CTF/OSCP |

Always try v1 and v2c first in lab environments.

---

## Why SNMP Matters for OSCP

- Default community string "public" is frequently left unchanged
- Exposes running processes, user accounts, installed software, open ports
- Can reveal credentials stored in process names or descriptions
- Common on network devices, Windows servers, printers
- **UDP 161 is often missed** — always scan UDP, not just TCP

---

## Common OSCP Workflow

```bash
# Step 1 — find SNMP hosts on subnet
nmap -sU -p 161 --open 192.168.x.0/24

# Step 2 — brute force community string
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <target>

# Step 3 — enumerate processes, users, software
snmpwalk -v1 -c public <target> 1.3.6.1.2.1.25.4.2.1.2   # processes
snmpwalk -v1 -c public <target> 1.3.6.1.4.1.77.1.2.25    # users
snmpwalk -v1 -c public <target> 1.3.6.1.2.1.25.6.3.1.2   # software

```

---

*Part of Security-Shaman cybersecurity notes — github.com/Security-Shaman*