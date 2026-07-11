# Default built-in librarians 

Need-to-know basis for default built-in functions.


---

## C libraries:
- `string.h` — `memset`, `memcpy`, `strcpy` (memory/string manipulation)
- `stdio.h` — `printf`, `fopen`, `fread` (input/output)
- `stdlib.h` — `malloc`, `free`, `exit` (memory allocation)
- `winsock2.h` — `socket`, `connect`, `send` (Windows networking)
- `sys/socket.h` — same but for Linux networking

---

## Python libraries (in exploit scripts):
- `socket` — TCP/UDP connections to targets
- `struct` — packing data into binary format (`struct.pack('<I', 0x10090c83)`)
- `requests` — HTTP requests (GET, POST)
- `sys` — command line arguments (`sys.argv[1]`)
- `os` — running system commands
- `subprocess` — running system commands (newer)
- `urllib` — URL handling and encoding
- `base64` — encoding/decoding

---

### Argparse (python)

**argparse** — handles command line arguments in Python scripts. It's how exploits take your target IP, port, and other options from the command line.

**Basic example:**
```python
import argparse

parser = argparse.ArgumentParser(description="My exploit")
parser.add_argument("-t", "--target", required=True, help="Target IP")
parser.add_argument("-p", "--port", type=int, default=80, help="Target port")
parser.add_argument("-c", "--command", default="id", help="Command to run")
args = parser.parse_args()

print(args.target)   # whatever you passed with -t
print(args.port)     # whatever you passed with -p
print(args.command)  # whatever you passed with -c
```

**Running it:**
```bash
python3 exploit.py -t 192.168.45.10 -p 8080 -c whoami
```

**What you'll see in exploit scripts:**
- `required=True` — must provide this argument
- `default=80` — uses this value if you don't specify
- `type=int` — converts input to integer
- `-t` is the short flag, `--target` is the long flag



---

### The ones you'll modify most often in exploits:**
- `socket` — changing target IP and port
- `struct.pack` — changing return addresses
- `requests` — changing URLs and payloads

---

Key parameters you'll see and need to modify in exploit scripts:

---

### requests:
- `verify=False` — skips SSL certificate verification (needed for self-signed HTTPS targets)
- `proxies={"http":"http://127.0.0.1:8080"}` — routes through Burp for debugging
- `timeout=5` — prevents hanging on dead targets
- `allow_redirects=False` — stops auto-following redirects

---

### socket: 
- `socket.AF_INET` — IPv4
- `socket.SOCK_STREAM` — TCP connection
- `s.connect(("ip", port))` — target address
- `s.settimeout(5)` — timeout

---

### struct.pack:
- `'<I'` — little-endian unsigned 32-bit integer (used for return addresses on x86)
- `'<Q'` — little-endian unsigned 64-bit integer (for x64)

---

### msfvenom (not Python but you'll use alongside):
- `-p` — payload type
- `-f` — output format (c, python, raw)
- `-b` — bad characters to avoid
- `LHOST` — your IP
- `LPORT` — your port
- `-e` — encoder
- `EXITFUNC=thread` — cleaner exit, avoids crashing the service

---

**The most common fix in OSCP exploits:**
Adding `verify=False` to `requests` calls when targeting HTTPS with self-signed certificates. Without it the script errors out immediately.