# Burpsuite Cheatsheet

Personal reference for penetration testing and OSCP lab work.

---

## Core Tabs Overview

| Tab | Purpose |
|-----|---------|
| Proxy | Intercept and inspect live HTTP traffic |
| Target | Site map of discovered endpoints |
| Intruder | Automated attack tool — brute force, fuzzing |
| Repeater | Manually resend and modify requests |
| Decoder | Encode/decode data (Base64, URL, HTML) |
| Comparer | Compare two requests/responses |
| Logger | Log of all HTTP traffic |

---

## Proxy Tab

```
Intercept on  → catches every request, holds it until you forward
Intercept off → passes traffic through without stopping
```

**Basic workflow:**
```
1. Turn Intercept on
2. Perform action in browser (login, submit form, click button)
3. Request appears in Proxy tab
4. Inspect/modify if needed
5. Click Forward to send to server
6. Or right-click → Send to Intruder/Repeater/etc
```

**Key right-click options on intercepted request:**
```
Send to Intruder  → automated attacks (brute force, fuzzing)
Send to Repeater  → manual request modification and resending
Send to Decoder   → encode/decode request data
Copy as curl      → get curl command equivalent
```

---

## Intruder Tab (Brute Force / Fuzzing)

### Attack Types

| Type | Use Case |
|------|----------|
| Sniper | One payload position, one wordlist — password brute force |
| Battering Ram | Same payload in all positions simultaneously |
| Pitchfork | Multiple positions, multiple wordlists in parallel |
| Cluster Bomb | Multiple positions, all combinations — credential stuffing |

### Password Brute Force Workflow

**Step 1 — Send login request to Intruder:**
```
Proxy → intercept login attempt → right-click → Send to Intruder
```

**Step 2 — Set attack position (Positions tab):**
```
1. Click Clear § — removes all auto-highlighted positions
2. Find password field in request:
   username=admin&password=test
3. Highlight password value (test)
4. Click Add § → becomes: username=admin&password=§test§
5. Attack type: Sniper
```

**Step 3 — Load wordlist (Payloads tab):**
```
Payload type: Simple list
Click Load → select passwords.txt
```

**Step 4 — Start attack:**
```
Click Start Attack
Results window opens — look for:
- Different status code from the rest (200 vs 302)
- Different response length from the rest
= successful login
```

### Identifying Successful Login
```
Failed login  → same status code + same response length for every attempt
Successful    → one response will be different — different length or status code
Sort by Length column to spot the outlier instantly
```

---

## Repeater Tab

Used for manually modifying and resending individual requests — useful for:
- Testing parameter manipulation
- SQL injection manual testing
- Checking how server responds to modified input

```
1. Send request to Repeater from Proxy
2. Modify any part of the request manually
3. Click Send
4. Compare Request vs Response side by side
5. Modify and resend as many times as needed
```

---

## Decoder Tab

Quickly encode/decode data:

```
Common encodings:
- URL encode/decode     → %20, %3D etc
- Base64 encode/decode  → dXNlcm5hbWU=
- HTML encode/decode    → &lt; &gt;
- Hex encode/decode
```

Paste data → select encoding type → result appears instantly.

---

## Useful Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Send to Repeater |
| `Ctrl+I` | Send to Intruder |
| `Ctrl+F` | Forward intercepted request |
| `Ctrl+Z` | Undo in editor |

---

## Common OSCP Workflows

```bash
# Brute force login page
1. Intercept login POST request
2. Send to Intruder
3. Mark password field as payload position
4. Load wordlist
5. Start attack → sort by Length → find outlier

# Manual parameter testing (SQLi, XSS)
1. Intercept request
2. Send to Repeater
3. Modify parameter value
4. Send and observe response
5. Iterate

# Decode suspicious parameter
1. Copy encoded value from request
2. Paste into Decoder tab
3. Try URL decode, Base64 decode until readable
```

---

## Intruder Throttling (Community Edition)

Burpsuite Community Edition **intentionally throttles Intruder** — adds delays between requests making brute force slow.

**Alternative for speed — use Hydra instead:**
```bash
# HTTP POST form brute force
hydra -l <username> -P passwords.txt <target IP> http-post-form \
"/login:username=^USER^&password=^PASS^:Invalid credentials"

# Adjust:
# /login          → actual login path
# username/password → actual POST parameter names
# Invalid credentials → text shown on failed login
```

Use Burpsuite for manual testing and inspection. Use Hydra for actual brute forcing under time pressure.

---

## Tips for OSCP

- Always use the built-in Burpsuite browser — saves setup time
- Turn intercept OFF when browsing normally, ON only when capturing specific requests
- Use Repeater for SQLi and parameter testing — faster than Intruder for manual work
- Sort Intruder results by Length column immediately — outlier = success
- Save interesting requests — right-click → Save item

---

*Part of Security-Shaman cybersecurity notes — github.com/Security-Shaman*
