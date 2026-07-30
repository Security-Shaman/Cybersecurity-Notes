# Module 19 Port Redirection and SSH Tunneling

The great pivoting begins.
Just use ligolo-ng

---

## socat

> Refer to socat-cheatsheet.md

---

## SSH Tunneling

SSH tunneling — the heart of module 19. Three types to learn, and getting them straight is what trips most people up. Let me anchor the mental model before you start the exercises.

**The three types and what each is for:**

**1. Local Port Forwarding (`-L`)**
Brings a remote/internal service *to your machine*. You access it as if it were local.

```bash
ssh -L <local_port>:<target>:<target_port> user@pivot
```
Use when: you want to reach ONE specific internal service from Kali.

**2. Remote Port Forwarding (`-R`)**
Pushes a service from your side *to the remote machine*. The reverse direction.
```bash
ssh -R <remote_port>:<target>:<target_port> user@pivot
```
Use when: the pivot can't reach you directly, so you push access to it.

**3. Dynamic Port Forwarding (`-D`)**
Turns SSH into a SOCKS proxy — reach an *entire network*, not just one port.
```bash
ssh -D <local_port> user@pivot
```
Use when: you want to reach many hosts/ports on an internal network (this is the big one for AD).

**The mental hooks:**
- `-L` = **L**ocal = pull a remote service TO you
- `-R` = **R**emote = push your access TO the remote side
- `-D` = **D**ynamic = proxy into a whole network

**The direction confusion (what trips everyone):**
- `-L` — you initiate, traffic flows: you → pivot → target
- `-R` — traffic flows back: target → pivot → you
- `-D` — you get a proxy; point tools at it via proxychains

---

## Local Port Forwarding (`-L`)

**Local port forwarding** — breaking it down piece by piece:

```bash
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215

ssh -N -L 0.0.0.0:4455:172.16.187.217:445 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null database_admin@10.4.187.215
```

**`ssh`** — you're establishing an SSH connection

**`-N`** — don't execute a command or open a shell. You just want the tunnel, not an interactive session. SSH connects and sits there forwarding traffic.

**`-L 0.0.0.0:4455:172.16.50.217:445`** — this is the local forwarding rule, read as four parts:
- `0.0.0.0` — bind address on YOUR machine. `0.0.0.0` means listen on all your interfaces (so other machines can use this tunnel too, not just localhost)
- `4455` — the local port YOU open on your machine
- `172.16.50.217` — the final destination (an internal machine you can't reach directly)
- `445` — the port on that destination (SMB)

**`database_admin@10.4.50.215`** — you SSH into this pivot machine (`10.4.50.215`) as user `database_admin`. This is the machine that CAN reach `172.16.50.217`.

**What this accomplishes:**

```
Your Kali:4455  ──SSH tunnel──>  Pivot (10.4.50.215)  ──>  172.16.50.217:445
```

You open port 4455 on your own machine. Anything you send to `localhost:4455` gets tunneled through SSH to the pivot (`10.4.50.215`), which then forwards it to `172.16.50.217:445`.

**In practice:**
Now you can run SMB tools against your OWN machine's port 4455, and they'll actually hit the internal `172.16.50.217:445` that you couldn't reach before:

```bash
smbclient -L //127.0.0.1:4455 -U username
# Actually talks to 172.16.50.217:445 through the tunnel
```

**Why `0.0.0.0` instead of `127.0.0.1`:**
Binding to `0.0.0.0` lets other machines (or tools that don't like localhost) use the tunnel. `127.0.0.1` would restrict it to only your local machine. For OSCP, `0.0.0.0` is more flexible but `127.0.0.1` is more secure — either works.

**The one-line summary:**
This opens port 4455 on your Kali, and any traffic to it tunnels through the SSH pivot `10.4.50.215` to reach the internal SMB service on `172.16.50.217:445` — a machine you couldn't otherwise reach.

**The condition for this to work:**
You need valid SSH credentials (`database_admin`) on the pivot machine, and the pivot must be able to reach `172.16.50.217`. You confirmed the pivot's reach earlier with `ip addr`/`ip route`.

---

## Dynamic Port Forwarding (`-D`)

Yes — dynamic port forwarding is arguably the **most important** of the three for OSCP, especially for the AD set.

**Why it matters more than local/remote forwarding:**

Local forwarding (`-L`) reaches ONE port on ONE internal machine. But in the AD set, you need to reach **many machines and many ports** on an internal network — SMB, WinRM, RDP, LDAP, Kerberos across multiple hosts. Setting up a separate `-L` tunnel for every port on every machine is impractical.

Dynamic forwarding (`-D`) solves this — it turns SSH into a SOCKS proxy that lets you reach the **entire internal network** through one tunnel.

**The basic setup:**
```bash
# Create the SOCKS proxy through the pivot
ssh -N -D 9050 user@pivot
```

Then configure **proxychains** to route your tools through it:
```bash
# /etc/proxychains4.conf - add at the bottom:
socks5 127.0.0.1 9050
```

Now prefix any tool with `proxychains` to route it through the tunnel:
```bash
proxychains nmap -sT 10.4.187.0/24
proxychains smbclient -L //10.4.187.20
proxychains evil-winrm -i 10.4.187.20 -u user -p pass

sudo proxychains nmap -vvv -sT --top-ports=20 -Pn 172.16.50.217
```

All of these reach the internal network through the single dynamic tunnel.

**The honest OSCP reality:**

For the exam, most people now use **Ligolo-ng** instead of SSH dynamic forwarding + proxychains, because:
- Ligolo is faster and more stable
- No proxychains overhead (proxychains can be slow and breaks some tools)
- It gives you a real network route, so tools work natively without the `proxychains` prefix

**So my recommendation:**
- **Understand** dynamic forwarding conceptually — know that `-D` + proxychains reaches a whole network. It's the fallback and the foundation.
- **Use Ligolo-ng** as your primary tool for the actual exam AD pivoting.

Both do the same job (route your tools into an internal network). Ligolo just does it more cleanly.

---

## SSH Remote Port Forwarding (`-R`)

**The command:**
```bash
ssh -N -R <remote_bind_port>:<final_target>:<final_port> user@<your_kali>
```

**Simple explanation:**

Remote forwarding is the **reverse** of local forwarding. Instead of pulling a service TO you, you push a path FROM the pivot back to you.

You run this **on the compromised pivot machine**, connecting back to your Kali. It opens a port **on your Kali** that forwards through the pivot to an internal target.

**Concrete example:**
```bash
# Run this ON the pivot machine
ssh -N -R 9999:172.16.187.217:445 kali@192.168.187.50
```

This means:
- The pivot connects back to your Kali (`192.168.187.50`)
- Opens port `9999` on your Kali
- Anything hitting `localhost:9999` on Kali gets forwarded through the pivot to `172.16.187.217:445`

Then from Kali:
```bash
smbclient -L //127.0.0.1:9999
# Actually reaches the internal 172.16.187.217:445
```

**When you use remote (`-R`) instead of local (`-L`):**

Use remote forwarding when the pivot **can't accept incoming SSH connections** (no SSH server on it), but it **can reach out to you**. You flip the direction — the pivot initiates the SSH connection back to your Kali.

**The key difference:**

| | Local (`-L`) | Remote (`-R`) |
|---|-------------|---------------|
| Who runs it | You, from Kali | The pivot, connecting back to Kali |
| Who needs SSH server | The pivot | Your Kali |
| Direction | Kali → pivot → target | pivot → back to Kali → target |
| Use when | Pivot has SSH server you can log into | Pivot can't be SSH'd into, but can reach out |

**The condition that decides which to use:**
- Pivot runs an SSH server you have creds for → use **local (`-L`)**
- Pivot has no SSH server, but you can run SSH client on it and it can reach you → use **remote (`-R`)**, and you need an SSH server running on your Kali

**For remote forwarding, you need SSH running on Kali:**
```bash
sudo systemctl start ssh
```

**One-sentence summary:**
Remote forwarding (`-R`) is run from the pivot, connects back to your Kali, and opens a port on Kali that tunnels through the pivot to an internal target — used when you can't SSH *into* the pivot but it can reach *out* to you.

---
