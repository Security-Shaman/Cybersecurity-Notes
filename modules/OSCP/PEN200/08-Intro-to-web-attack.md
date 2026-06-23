# Module 8 : Introduction to Web Application Attacks

## 8.2.3 Directory Brute Force with Gobuster

Happens after you find out the IP is running http/https

```bash

#Reference
tldr gobuster

#brute force a web directory with gobuster
gobuster dir -u <ip.addr> -w /usr/share/dirb/wordlists/common.txt -t 80

#If faced with 301 error try: (forces redirect, if destination exists)
gobuster dir -u <ip.addr> -w /usr/share/wordlists/dirb/common.txt -r

#If faced with another error try: (ignores 301 requests)
gobuster dir -u <ip.addr> -w /usr/share/wordlists/dirb/common.txt -b 301

```