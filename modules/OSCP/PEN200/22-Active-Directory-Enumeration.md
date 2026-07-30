# Module 22 Active Directory Enumeration

The calm before the storm

---

## Slight tweaks as compared to windows privesc

```bash
#RDP Connection
xfreerdp3 /u:user /p:password /d:DomainName.com /v:192.168.179.75 /drive:shared,/home/wai

```

**Manual Enumeration**
```cmd
net user /domain
net user jeffadmin /domain #Specific User information

net group /domain
net group "Sales Department" /domain #Specific Group information
```

---

## Service Principal Names (SPNs)

**What an SPN is:**

An SPN is a unique identifier that links a **service** to the **account that runs it** in Active Directory. When a service (like a database, web server, or file share) runs under a domain account, it registers an SPN so Kerberos knows how to authenticate to it.

Think of it as: "This MSSQL service running on SERVER01 is operated by the account `svc_sql`."

**Why SPNs matter for attackers — Kerberoasting:**

Here's the exploitable part. In Kerberos, any authenticated domain user can **request a service ticket** for any SPN. That ticket is **encrypted with the service account's password hash**.

So the attack is:
1. Find accounts with SPNs (they're service accounts)
2. Request their service tickets
3. The tickets are encrypted with the account's password
4. Crack the tickets offline to recover the password

This is **Kerberoasting** — and it's one of the most reliable AD attacks because it only requires ANY valid domain user to start.

**Why service accounts are juicy targets:**
- They often have **high privileges** (service accounts frequently run as admins)
- They often have **weak or old passwords** (set once and forgotten)
- Cracking one can give you a privileged foothold

**Enumerating SPNs:**

**PowerView:**
```powershell
Get-DomainUser -SPN | select samaccountname,serviceprincipalname
```

**Built-in (setspn):**
```cmd
setspn -T corp.com -Q */*
```

**Impacket (from Kali):**
```bash
impacket-GetUserSPNs corp.com/user:password -dc-ip <DC_IP>
```

**The enumeration output tells you:**
Which accounts have SPNs = which accounts are Kerberoastable = your list of targets to request tickets for and crack.

**The connection to module 23:**
Enumerating SPNs (module 22) sets up **Kerberoasting** (module 23). In 22 you find the targets; in 23 you request their tickets and crack them:
```bash
# Module 23 - actually roast them
impacket-GetUserSPNs corp.com/user:password -dc-ip <DC_IP> -request
# Then crack the hash with hashcat mode 13100
hashcat -m 13100 hashes.txt rockyou.txt
```

**One-sentence summary:**
An SPN links a service to its domain account; enumerating SPNs finds service accounts whose Kerberos tickets you can request and crack offline (Kerberoasting) — and since service accounts are often privileged with weak passwords, this is a top AD attack path.

**Why it's worth understanding well:**
Kerberoasting appears frequently on OSCP AD sets. Any domain user can do it, service accounts are often privileged, and the passwords are often crackable. It's a go-to move once you have any domain foothold.

Does that give you a clear picture of SPNs and why you enumerate them?

---

## Domain Shares

Domain shares are network folders or files hosted on a server within a Windows Active Directory domain that can be accessed by authorized users across the network


---

