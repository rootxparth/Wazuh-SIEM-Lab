# 🛡️ Wazuh SIEM Lab

> A hands-on Security Information and Event Management (SIEM) lab built with Wazuh, featuring real-world threat detection scenarios including File Integrity Monitoring, Brute Force Investigation, and Privilege Escalation detection.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Architecture](#lab-architecture)
- [Features & Use Cases](#features--use-cases)
  - [File Integrity Monitoring (FIM)](#-file-integrity-monitoring-fim)
  - [Brute Force Investigation](#-brute-force-investigation)
  - [Privilege Escalation Detection](#-privilege-escalation-detection)
- [Tech Stack](#tech-stack)
- [Setup & Installation](#setup--installation)
- [Screenshots / Results](#screenshots--results)
- [Key Takeaways](#key-takeaways)
- [Future Enhancements](#future-enhancements)
- [Author](#author)

---

## Overview

This lab simulates a real-world SOC (Security Operations Center) environment using **Wazuh** as the core SIEM platform. The goal was to build, configure, and validate detection capabilities for common attack vectors that security analysts encounter daily.

The lab covers:
- **Monitoring** file system changes in real time
- **Detecting and investigating** SSH/RDP brute force attacks
- **Alerting** on privilege escalation attempts via sudo abuse, SUID exploitation, and more

All scenarios were tested end-to-end — from attack simulation to alert triage in the Wazuh dashboard.

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Wazuh Manager                    │
│         (Indexer + Dashboard + Manager)             │
└────────────────────┬────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
   ┌──────▼──────┐       ┌──────▼──────┐
   │ Linux Agent │       │Windows Agent│
   │  (Victim-1) │       │  (Victim-2) │
   └─────────────┘       └─────────────┘
```

- **Wazuh Manager** — Central brain; collects logs, correlates events, generates alerts
- **Wazuh Agents** — Deployed on monitored endpoints; forward logs and events
- **Wazuh Dashboard** — Kibana-based UI for alert visualization and investigation

---

## Features & Use Cases

### 📁 File Integrity Monitoring (FIM)

**Objective:** Detect unauthorized changes to critical system files and directories in real time.

**How it works:**
Wazuh's FIM module (`syscheck`) hashes monitored files at regular intervals and on real-time events. Any creation, modification, or deletion triggers an alert with a full diff of what changed, who changed it, and when.

**Monitored Paths Configured:**
```xml
<syscheck>
  <directories realtime="yes" report_changes="yes" check_all="yes">/etc</directories>
  <directories realtime="yes" report_changes="yes" check_all="yes">/var/www/html</directories>
  <directories realtime="yes" check_all="yes">/usr/bin,/usr/sbin</directories>
</syscheck>
```

**Simulated Attack:**
```bash
# Attacker modifies a sensitive file
echo "root:0:0:root:/root:/bin/bash" >> /etc/passwd
```

**Detected Alert:**
- Rule ID: `550` — Integrity checksum changed
- Rule ID: `554` — File added to the system
- MITRE ATT&CK: `T1565.001` — Stored Data Manipulation

**Results:**
- ✅ Real-time alert triggered within seconds of file modification
- ✅ Captured old/new hash, file owner, permissions, and inode info
- ✅ Full audit trail logged in Wazuh Dashboard

---

### 🔐 Brute Force Investigation

**Objective:** Detect, correlate, and investigate SSH brute force login attempts.

**How it works:**
Wazuh ingests `/var/log/auth.log` (Linux) and uses built-in rules to detect repeated failed authentication attempts. When a threshold is crossed, a high-severity alert is raised along with the attacker's IP, targeted user, and timestamp.

**Attack Simulation:**
```bash
# Simulated brute force using Hydra
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.105
```

**Key Rules Triggered:**

| Rule ID | Description | Level |
|---------|-------------|-------|
| `5710`  | SSH authentication failed | 5 |
| `5712`  | SSHD Brute Force (multiple failures) | 10 |
| `5763`  | Valid user logged in after brute force | 12 |

**Investigation Steps Performed:**
1. Identified source IP and attack timeline in the Wazuh dashboard
2. Correlated failed attempts with eventual successful login
3. Verified MITRE mapping: `T1110.001` — Brute Force: Password Guessing
4. Simulated active response to auto-block attacker IP via `iptables`

**Active Response Config:**
```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>
  <timeout>180</timeout>
</active-response>
```

**Results:**
- ✅ Detected brute force within seconds across multiple login attempts
- ✅ Automatically blocked attacking IP using active response
- ✅ Alert correlated across both agent and manager logs

---

### ⚡ Privilege Escalation Detection

**Objective:** Identify attempts by attackers (or malicious insiders) to gain elevated system privileges.

**How it works:**
Wazuh monitors `sudo` usage, SUID binary abuse, `/etc/sudoers` modifications, and audit logs to catch privilege escalation attempts. Linux Audit daemon (`auditd`) rules feed enriched syscall-level data into Wazuh for deeper analysis.

**Scenarios Tested:**

#### 1. Sudo Abuse
```bash
# Unauthorized user attempts sudo
sudo su -
# Wazuh Rule 5401: Attempted to run sudo without privilege
```

#### 2. SUID Binary Exploitation
```bash
# Attacker abuses SUID bit on a binary (GTFOBins technique)
find / -perm -u=s -type f 2>/dev/null
/usr/bin/find . -exec /bin/sh -p \; -quit
```

#### 3. /etc/sudoers Modification
```bash
# Attacker adds themselves to sudoers
echo "attacker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
# Triggers FIM alert + privilege escalation alert simultaneously
```

**Key Rules Triggered:**

| Rule ID | Description | Level |
|---------|-------------|-------|
| `5401`  | Unauthorized sudo attempt | 5 |
| `5402`  | Successful sudo execution | 7 |
| `5403`  | sudoers file changed | 10 |
| `80792` | Privilege escalation via auditd | 12 |

**MITRE ATT&CK Mapping:**
- `T1548.003` — Abuse Elevation Control Mechanism: Sudo and Sudo Caching
- `T1548.001` — Setuid and Setgid

**Results:**
- ✅ All three escalation techniques generated high-severity Wazuh alerts
- ✅ `auditd` integration provided syscall-level visibility (uid, euid, command, args)
- ✅ `/etc/sudoers` modification detected via both FIM and audit rule correlation

---

## Tech Stack

| Component | Details |
|-----------|---------|
| **SIEM** | Wazuh 4.x (Manager + Indexer + Dashboard) |
| **OS** | Ubuntu 22.04 LTS (Manager & Agents) |
| **Virtualization** | VirtualBox / VMware |
| **Attack Tools** | Hydra, Nmap, custom bash scripts |
| **Log Sources** | `auth.log`, `syslog`, `auditd`, `syscheck` |
| **MITRE ATT&CK** | Mapped to relevant TTPs per scenario |

---

## Setup & Installation

### Prerequisites
- 8GB+ RAM (for Manager VM)
- VirtualBox or VMware
- Ubuntu 22.04 ISO

### 1. Deploy Wazuh Manager (All-in-One)
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

### 2. Deploy Wazuh Agent on Endpoint
```bash
# On the agent machine
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.0-1_amd64.deb
sudo WAZUH_MANAGER='<MANAGER_IP>' dpkg -i ./wazuh-agent_4.7.0-1_amd64.deb
sudo systemctl enable --now wazuh-agent
```

### 3. Configure FIM Rules
Edit `/var/ossec/etc/ossec.conf` on each agent to add directories for monitoring (see FIM section above).

### 4. Enable Auditd Integration
```bash
sudo apt install auditd -y
sudo auditctl -w /etc/passwd -p wa -k identity
sudo auditctl -w /etc/sudoers -p wa -k sudoers_change
```

### 5. Access Dashboard
```
https://<MANAGER_IP>:443
Default credentials set during installation
```

---

## Screenshots / Results

> _Add your Wazuh dashboard screenshots here._

| Scenario | Alert Level | Screenshot |
|----------|-------------|------------|
| FIM — `/etc/passwd` modified | Critical (12) | `screenshots/fim-passwd.png` |
| SSH Brute Force detected | High (10) | `screenshots/bruteforce-alert.png` |
| Privilege Escalation via sudoers | Critical (12) | `screenshots/privesc-sudoers.png` |

---

## Key Takeaways

- Wazuh's **real-time FIM** is highly effective for detecting tampering with sensitive files — especially when combined with `auditd` for user-level attribution.
- **Brute force detection** works well out of the box, and **active response** dramatically reduces the attack window by auto-blocking IPs.
- **Privilege escalation** requires layered detection: FIM for sudoers changes, audit rules for syscall monitoring, and Wazuh rules for behavioral correlation.
- MITRE ATT&CK mapping in Wazuh makes it easy to contextualize alerts within a broader threat model.

---

## Author

**Your Name**
- GitHub: [RootxParth](https://github.com/rootxparth)
- LinkedIn: [linkedin.com/in/Parth-Singh](https://linkedin.com/in/parth-singh-smasher)

---

> ⭐ If you found this lab useful or learned something from it, give it a star!

---

*Built for learning. All attacks were performed in an isolated lab environment.*
