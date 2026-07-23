
**Windows Enumeration — cmd vs PowerShell**

| Task | cmd | PowerShell |
|------|-----|-----------|
| Current user | `whoami` | `whoami` |
| Your privileges | `whoami /priv` | `whoami /priv` |
| Your groups | `whoami /groups` | `whoami /groups` |
| List all users | `net user` | `Get-LocalUser` |
| User details | `net user roy` | `Get-LocalUser roy` |
| List groups | `net localgroup` | `Get-LocalGroup` |
| Who's admin | `net localgroup administrators` | `Get-LocalGroupMember Administrators` |
| System info | `systeminfo` | `Get-ComputerInfo` |
| Running processes | `tasklist` | `Get-Process` |
| Services | `sc query` / `wmic service` | `Get-Service` |
| Network connections | `netstat -ano` | `Get-NetTCPConnection` |
| Scheduled tasks | `schtasks /query` | `Get-ScheduledTask` |
| Read a file | `type file.txt` | `Get-Content file.txt` |
| Find files | `dir /s /b C:\*.txt` | `Get-ChildItem -Recurse -Filter *.txt` |
| Registry query | `reg query <path>` | `Get-ItemProperty <path>` |

**The rule:** pick whichever you remember. They overlap. Don't waste mental energy deciding.

