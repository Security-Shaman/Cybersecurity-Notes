# MODULE 6 : INFORMATION GATHERING

fingerprinting and enumeration


## 6.2.1 Whois Enumeration

- whois : Look up ownership, registration and contact information for a domain name

- -h : The flag used to specify custom WHOIS server host instead of public regional registries


## 6.4.1 DNS Enumeration

- host : DNS lookup utility, used for quick look ups

- dnsenum : DNS enumeration


## 6.4.2 + 6.4.3 netcat + nmap

inside the cheatsheet


## 6.4.4 SMB Enumeration

Services: NetBIOS
Port number : 445, 139

- enum4linux : enumerates linux IPs
- -a : aggressively enumerates through the IP


## 6.4.5 SMTP enumeration

Port number : 25

```bash
# Connection to via SMTP
nc -nv ip.addr 25 #port number

# Once connected, two useful commands:
VRFY username        # verify if user exists
EXPN username        # expand mailing list

# Automated user enumeration
smtp-user-enum -M VRFY -U /usr/share/wordlists/metasploit/unix_users.txt -t ip.addr
```


## 6.4.6 SNMP enumeration

Port number : 161
 
SNMPv1, SNMPv2 does not have authentication and encryption

```bash
#SNMP port is open
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt ip.addr


```

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