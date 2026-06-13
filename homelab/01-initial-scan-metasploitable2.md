# Metasploitable2 — Initial Scan

## Environment
- Attacker: Kali Linux (10.10.10.x)
- Target: Metasploitable2 (10.10.10.4)
- Network: VirtualBox NAT Network "SecurityLab"

## Reconnaissance
\`\`\`
nmap -sV 10.10.10.4
\`\`\`

## Findings
23 open ports identified. Notable services:
- vsftpd 2.3.4 (port 21) — known backdoor (CVE-2011-2523)
- UnrealIRCd (port 6667) — known backdoor
- Java RMI (port 1099) — known RCE
- bindshell (port 1524) — open root shell

## Next Steps
Exploit vsftpd backdoor as first practice target.

