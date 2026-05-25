# 🚨 IR-Playbook-Lab

> **Incident Response Playbooks & Simulation Environment**  
> A hands-on cybersecurity project demonstrating incident detection, triage, containment, and documentation using open-source tools in an isolated lab environment.

---

## 📌 Project Overview

This repository contains a fully documented **Incident Response (IR) framework** built around the **NIST SP 800-61** standard. It includes:

- A simulated home lab for generating and detecting real security events
- Structured IR playbooks for 4 common incident types
- Evidence artifacts from simulated incidents
- A reusable incident report template

This project was built to demonstrate practical skills in **security operations, incident coordination, and stakeholder documentation** — aligned to real-world SOC analyst responsibilities.

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                  VirtualBox Host                     │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────────┐   │
│  │  Ubuntu Attacker  │    │   Windows 10 Victim  │   │
│  │  192.168.56.102   │───▶│   192.168.56.101     │   │
│  │  - Metasploit     │    │   - Wazuh Agent      │   │
│  │  - Hydra          │    │   - Sysmon           │   │
│  └──────────────────┘    └──────────┬───────────┘   │
│                                      │ Logs          │
│                           ┌──────────▼───────────┐   │
│                           │   Wazuh SIEM Server  │   │
│                           │   192.168.56.110     │   │
│                           │   Dashboard: :443    │   │
│                           └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
        Network: Host-Only Adapter (isolated)
```

---

## 🛠️ Tools & Technologies

| Tool | Purpose | Version |
|------|---------|---------|
| [Wazuh](https://wazuh.com) | SIEM, log collection, alerting | 4.7+ |
| [VirtualBox](https://virtualbox.org) | Lab virtualisation | 7.0+ |
| [Metasploit Framework](https://metasploit.com) | Attack simulation | 6.x |
| [Hydra](https://github.com/vanhauser-thc/thc-hydra) | Brute-force simulation | 9.x |
| [Nmap](https://nmap.org) | Network reconnaissance simulation | 7.x |
| [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Windows event telemetry | Latest |

---

## 📂 Repository Structure

```
IR-Playbook-Lab/
├── README.md                        ← You are here
├── lab_setup_guide.md               ← Step-by-step environment build
│
├── /playbooks
│   ├── 01_brute_force.md            ← SSH/RDP credential attacks
│   ├── 02_phishing.md               ← Phishing & malicious attachment
│   ├── 03_malware.md                ← Malware execution & persistence
│   └── 04_unauthorized_access.md   ← Insider threat / account misuse
│
├── /evidence
│   ├── incident_001_brute_force/
│   │   ├── wazuh_alert.png
│   │   ├── auth_log_snippet.txt
│   │   └── timeline.md
│   └── incident_002_recon/
│       └── ...
│
├── /reports
│   ├── incident_report_template.md
│   └── sample_completed_report.md
│
└── /diagrams
    ├── lab_architecture.png
    └── ir_process_flow.png
```

---

## 📋 IR Playbook Structure

Each playbook follows the **NIST SP 800-61 Rev. 2** lifecycle:

```
Detection → Analysis → Containment → Eradication → Recovery → Post-Incident
```

### Incident Types Covered

| # | Incident Type | Trigger | Severity |
|---|--------------|---------|---------|
| 01 | SSH/RDP Brute Force | >10 failed auth attempts in 60s | High |
| 02 | Phishing / Malicious Attachment | Suspicious email indicators | High |
| 03 | Malware Execution | Unexpected process spawn, C2 beacon | Critical |
| 04 | Unauthorised Access | Off-hours login, privilege escalation | High |

---

## 🔬 Simulated Incidents

### Incident 001 — SSH Brute Force
- **Tool used:** Hydra from attacker VM
- **Detection:** Wazuh Rule ID 5712 fired after 15 failed SSH attempts
- **Response:** Source IP blocked via firewall rule, affected account locked, credentials reset
- **See:** `/evidence/incident_001_brute_force/`

### Incident 002 — Network Reconnaissance
- **Tool used:** Nmap SYN scan from attacker VM
- **Detection:** Wazuh IDS alert on port scan signature
- **Response:** IP logged, added to watchlist, incident escalated for review
- **See:** `/evidence/incident_002_recon/`

---

## 📊 Frameworks & Standards Referenced

| Framework | Application in This Project |
|-----------|---------------------------|
| **NIST SP 800-61 Rev. 2** | Overall IR lifecycle structure |
| **MITRE ATT&CK** | Technique tagging in each playbook (e.g., T1110.001) |
| **SANS PICERL Model** | Supplementary IR phase reference |
| **ISO/IEC 27035** | Incident management policy alignment |

---

## 🚀 Getting Started

### Prerequisites
- VirtualBox 7.0+
- 16GB RAM recommended (runs 3 VMs simultaneously)
- Ubuntu 22.04 ISO and Windows 10 ISO

### Quick Setup
```bash
# Clone this repo
git clone https://github.com/YOUR_USERNAME/IR-Playbook-Lab.git
cd IR-Playbook-Lab

# Follow the full lab setup guide
cat lab_setup_guide.md
```

See **`lab_setup_guide.md`** for the complete step-by-step environment build, including VM configuration, Wazuh installation, and agent deployment.

---

## 📝 Incident Report Template

Each simulated incident produces a formal report using the template in `/reports/incident_report_template.md`. Fields include:

- Incident ID, date/time, classification
- Executive summary
- Technical timeline
- Root cause analysis
- Containment & remediation actions taken
- Lessons learned
- Recommendations

---

## 🎯 Skills Demonstrated

- ✅ Incident detection and triage using a SIEM
- ✅ Structured IR documentation following NIST 800-61
- ✅ MITRE ATT&CK technique identification
- ✅ Stakeholder-ready incident reporting
- ✅ Hands-on experience with Wazuh, Metasploit, Hydra
- ✅ Network isolation and lab environment management

---

## 📚 References

- [NIST SP 800-61 Rev. 2 — Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Wazuh Documentation](https://documentation.wazuh.com)
- [SANS Incident Handler's Handbook](https://www.sans.org/white-papers/33901/)

---

## ⚠️ Legal Disclaimer

All attacks and scans in this project were performed **exclusively within an isolated, self-owned virtual lab environment**. No external networks, systems, or third-party infrastructure were targeted. This project is for **educational purposes only**.

---
