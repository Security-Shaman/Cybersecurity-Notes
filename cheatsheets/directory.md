# Important Directories


---

## Windows

```bash
#Key information (users most interacted)
C:\Users\<username>\Desktop
C:\Users\<username>\Documents
C:\Users\<username>\.ssh\id_rsa

#History
C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt  # PowerShell history

#Internal System + password (root equivilance)
C:\Windows\System32\drivers\etc\hosts
C:\Windows\System32\config\SAM  # Password hashes (needs SYSTEM)
C:\Windows\System32\config\SYSTEM

#Website
C:\xampp\htdocs\  # Web app files
C:\Inetpub\wwwroot\  # IIS web root

#General file (internal system)
C:\Program Files\
C:\Program Files (x86)\

```

### Windows Privesc

```bash
C:\Users\Public\          # Writable by everyone
C:\Windows\Temp\          # Writable by everyone
C:\Temp\                  # Often writable
C:\ProgramData\           # Sometimes writable
```


---


## Linux 

```bash
#Key information
/etc/passwd             # User accounts and home directories
/etc/shadow             # Hashed passwords (needs root)
/etc/hosts              # Internal hostnames and IPs

#ssh
/home/<user>/.ssh/    # SSH keys
/etc/ssh/sshd_config    # SSH config — shows allowed users, port
/home/user/.ssh/id_rsa  # Private SSH keys

#Web App
/var/log/apache2/access.log  # Web server logs — useful for log poisoning
/var/www/html/config.php     # Web app config — often has DB credentials

#Env
/proc/self/environ      # Environment variables — may contain secrets
/.env                   # App environment file — API keys, passwords

#History
/home/<user>/.bash_history  # Command history
/root/.bash_history   # Root command history
```


### Linux Privesc

```bash 

#FOR CRONJOBS
/etc/crontab          # System-wide crontab
/etc/cron.d/          # Additional cron jobs
/var/spool/cron/      # User-specific crontabs

#Writeable Directories
/tmp/                 # World writable
/var/tmp/             # World writable, persists reboots
/dev/shm/             # Memory-based, world writable
```

---