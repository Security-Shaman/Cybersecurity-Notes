# SSH Cheatsheet

> Focused on key-based authentication and common pitfalls encountered during pentesting.

---

## Basic Connection

```bash
# Standard SSH connection
ssh user@<ip>

# Specify port (always check nmap — SSH may not be on 22)
ssh -p 2222 user@<ip>

# Connect using a private key
ssh -i id_rsa user@<ip>

# Connect using a private key on a non-standard port
ssh -i id_rsa -p 2222 user@<ip>
```

---

## Key Permissions (chmod 400)

SSH will **refuse to use a private key** if it is too permissive.
Always set the correct permissions before connecting:

```bash
chmod 400 id_rsa
```

| Permission | Meaning |
|------------|---------|
| `400` | Owner can read only. No access for group or others. |
| `600` | Owner can read/write (too permissive for SSH keys) |
| `777` | Everyone can read/write/execute (SSH will reject) |

Verify permissions:
```bash
ls -la id_rsa
# Should show: -r-------- 1 user user ...
```

---

## Key Format Troubleshooting

A valid private key looks like this:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAA...
[multiple lines of base64]
-----END OPENSSH PRIVATE KEY-----
```

### Check for hidden characters (Windows line endings)

If you copy-paste a key from a browser, it may contain Windows-style `\r\n` (CRLF) line endings instead of Unix `\n` (LF). SSH cannot parse these.

```bash
# Check for hidden characters — look for ^M at end of lines
cat -A id_rsa | head -5
```

If you see `^M$` at the end of lines, the file has Windows line endings.

### Fix with dos2unix

```bash
dos2unix id_rsa
```

Verify the fix:
```bash
cat -A id_rsa | head -5
# Lines should now end with just $ (no ^M)
```

### Validate the key is readable

```bash
ssh-keygen -y -f id_rsa
# Should output the matching public key
# If it errors, the key file is still malformed
```

---

## Retrieving Keys During Pentesting

Avoid copy-pasting keys from the browser — use curl to save directly:

```bash
# Save remote file via directory traversal
curl "http://<ip>/vulnerable.php?page=../../../home/user/.ssh/id_rsa" -o id_rsa

# Then fix permissions and line endings
dos2unix id_rsa
chmod 400 id_rsa

# Connect
ssh -i id_rsa user@<ip>
```

---

## Common Private Key Locations

```
/home/<user>/.ssh/id_rsa
/root/.ssh/id_rsa
/home/<user>/.ssh/id_ed25519
```

> Tip: Read `/etc/passwd` first to enumerate users and their home directories,
> then check for keys at the default locations.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `error in libcrypto: unsupported` | CRLF line endings in key file | `dos2unix id_rsa` |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Permissions too open | `chmod 400 id_rsa` |
| `Identity file not accessible` | Wrong path or directory | Check `pwd`, use full or correct relative path |
| Prompted for password despite key | Wrong port / key not in `authorized_keys` / key still malformed | Check nmap for correct port, re-validate key |
| `No such file or directory` | Running command from wrong directory | `ls` to confirm file exists, or use full path |

---

## Getting Help

```bash
man ssh              # Full SSH manual
ssh --help           # Quick flag reference
ssh-keygen --help    # Key generation options
dos2unix --help      # Line ending conversion
```

Online references:
- [SSH man page](https://man.openbsd.org/ssh)
- [HackTricks - SSH](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ssh)

---

## More information

Private key (no extension, e.g. id_rsa) — stays with you, the person trying to authenticate. Never shared.

Public key (.pub extension, e.g. id_rsa.pub) — goes onto the target machine, specifically into ~/.ssh/authorized_keys.

So the flow for using your own generated key pair to gain persistent SSH access:

You generate the key pair on Kali
You take the public key and place it into the target's authorized_keys file
You connect using your private key — ssh -i id_rsa user@target

The server checks: does this private key match any public key in my authorized_keys? If yes, you're in — no password needed.

This is actually a really useful persistence technique once you have a shell — generate a key pair, drop your public key into authorized_keys, and you have guaranteed SSH access even if your original foothold gets patched.