## General Methodology

1. Enumerate everything first — don't touch exploits yet
   → What ports are open?
   → What services and versions?
   → What OS?

2. Research every service version found
   → Does it have a known vulnerability?
   → Is there a public exploit?

3. Access whatever is exposed
   → Web server? Browse it, run gobuster
   → SMB? Run enum4linux
   → FTP? Try anonymous login
   → SSH? Check for weak creds or key files

4. Find a foothold — get initial access

5. Enumerate again from inside
   → Who am I? (whoami, id)
   → What can I access?
   → What's running locally?
   → What are my privileges?

6. Escalate privileges
   → Run linPEAS or winPEAS
   → Look for misconfigurations, weak permissions, exploitable services

7. Complete objective — proof.txt, flag, full compromise

---

## Web Methodology 

Sure — the key insight is that **the feature tells you the attack**.

**File loader → LFI/RFI**
If the app loads, includes, or displays files based on your input (`?page=about`, `?file=report.pdf`, `?template=default`), the backend is probably doing `include($_GET['page'])` or similar. That's your LFI target.

**Search box → SQLi**
A search feature almost always queries a database. The backend runs something like `SELECT * FROM products WHERE name LIKE '%input%'`. Any input that goes into a database query is a SQLi candidate.

**System command feature → Command injection**
Features that feel like they're "doing something on the server" — ping a host, check DNS, convert a file, clone a git repo, create an archive, send an email. These often shell out to OS commands. That's your command injection target.

**File upload → Webshell upload**
Any upload functionality — check what file types are accepted and try uploading executable code.

**Login form → SQLi or default credentials**
Authentication forms query the database to check credentials. Also always try default passwords first.

**The practical habit:**
Before testing anything, ask yourself *"what does this feature do to my input?"* 

- Loads a file → LFI
- Queries a database → SQLi  
- Runs a system command → Command injection
- Saves a file → Upload vulnerability
- Checks credentials → SQLi or brute force
