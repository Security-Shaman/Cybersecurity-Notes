# Important Port references

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

## RDP 3389 ms-wbt-server

```bash
#RDP connection
xfreerdp /u:username /p:password /v:<target_ip> /drive:shared,/home/wai/Desktop
```

---

## FTP 21

**Connect:**
```bash
ftp <target_ip>
```

**Anonymous login (try this first always):**
```bash
ftp <target_ip>
# Username: anonymous
# Password: (blank or any email)
```

**Basic navigation:**
```bash
ls              # list files
cd <directory>  # change directory
pwd             # current directory
```

**Download files:**
```bash
get filename.txt          # download single file
mget *.txt                # download multiple files
binary                    # switch to binary mode (for non-text files)
```

**Upload files:**
```bash
put exploit.php           # upload single file
mput *.php                # upload multiple files
```

**Other useful commands:**
```bash
passive                   # switch to passive mode (fixes connection issues)
type binary               # binary transfer mode
bye                       # disconnect
```

**Why FTP matters for OSCP:**
- Anonymous login often exposes sensitive files
- Writable FTP directories can be used to upload webshells if the FTP root overlaps with the web root
- Config files on FTP may contain credentials for other services

**The habit:**
When nmap shows port 21 open, always try anonymous login first before brute forcing. It works more often than you'd expect.

**How FTP actually works under the hood:**
It uses two connections:

Port 21 — control channel (sends commands like ls, get, put)
Port 20 (or random port in passive mode) — data channel (actual file content transfers)

This is why you sometimes need to type passive — firewalls often block the second data connection, and passive mode fixes that by letting the client initiate both connections.

--- 