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