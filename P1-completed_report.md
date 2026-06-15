# Incident Report — INC-2024-01-15-001

**INCIDENT REPORT**

| Field | Value |
|-------|-------|
| **Incident ID** | INC-2024-01-15-001 |
| **Report Date** | 2024-01-15 |
| **Report Author** | Caleb Rorich, IT Security Analyst |
| **Classification** | CONFIDENTIAL — IT SECURITY |
| **Incident Type** | Brute Force |
| **Severity** | High |
| **Status** | Closed |

---

## 1. Executive Summary

On 15 January 2024 at 14:17 UTC+2, the Wazuh SIEM generated a High-severity alert for repeated SSH authentication failures on the victim host (192.168.56.101). The attack originated from the simulated attacker VM (192.168.56.102) and involved 47 failed login attempts over 3 minutes using the Hydra tool. No successful authentication was achieved. The source IP was blocked via firewall rule within 8 minutes of detection, all targeted accounts were reviewed, and the incident was fully closed by 16:30 UTC+2.

---

## 2. Incident Timeline

| Date/Time (UTC+2) | Event |
|-------------------|-------|
| 2024-01-15 14:17:03 | Wazuh Rule 5712 fires — SSH brute force detected |
| 2024-01-15 14:17:03 | Alert visible on Wazuh dashboard, email notification sent |
| 2024-01-15 14:18:30 | Analyst acknowledges alert, begins triage |
| 2024-01-15 14:21:00 | Source IP 192.168.56.102 identified from auth logs |
| 2024-01-15 14:23:45 | AbuseIPDB / VirusTotal checked (lab IP, internal) |
| 2024-01-15 14:25:00 | UFW rule applied: DENY FROM 192.168.56.102 |
| 2024-01-15 14:27:00 | IT Manager notified via phone |
| 2024-01-15 14:30:00 | Log review — confirmed no successful authentication |
| 2024-01-15 14:45:00 | Targeted accounts reviewed, passwords flagged for reset |
| 2024-01-15 15:30:00 | Evidence collected and preserved |
| 2024-01-15 16:30:00 | Incident formally closed, report filed |

**Mean Time to Detect (MTTD):** ~0 minutes (automated alert)  
**Mean Time to Contain (MTTC):** 8 minutes

---

## 3. Technical Details

### 3.1 Affected Systems

| System | IP Address | Role | Impact |
|--------|-----------|------|--------|
| Victim-Win10 | 192.168.56.101 | Lab victim VM | SSH service targeted, no breach |

### 3.2 Attack Description

A dictionary-based SSH brute force attack was launched from the attacker VM (192.168.56.102) using Hydra (`hydra -l admin -P rockyou.txt ssh://192.168.56.101 -t 4`). The attack targeted the `admin` username exclusively with 47 password attempts. The attack generated a burst of failed authentication events that triggered Wazuh Rule ID 5712 (SSH brute force detection — threshold: 10 failures in 60 seconds).

No successful authentication was achieved. The SSH service on the target machine had a password configured that was not present in the wordlist used.

**Indicators of Compromise (IOCs)**

| IOC Type | Value | Description |
|----------|-------|-------------|
| IP Address | 192.168.56.102 | Source of brute force (attacker VM) |
| Username | admin | Targeted account |
| Tool | Hydra 9.4 | Attack tool identified from log pattern |

### 3.3 MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|--------|-----------|-----|
| Credential Access | Brute Force — Password Guessing | T1110.001 |
| Reconnaissance | Active Scanning | T1595 |

---

## 4. Impact Assessment

| Category | Impact | Details |
|----------|--------|---------|
| **Confidentiality** | No | No successful access, no data exposed |
| **Integrity** | No | No modifications made |
| **Availability** | No | SSH service remained available |
| **Regulatory** | No | No sensitive data involved |
| **Financial** | No | Lab environment only |
| **Reputational** | No | Internal lab, no external impact |

**PII/Sensitive Data Involvement:** No

---

## 5. Response Actions Taken

### Containment
- UFW deny rule applied blocking 192.168.56.102 inbound on all ports — 14:25 UTC+2
- SSH login monitoring threshold reduced from 10 to 5 failures for enhanced detection

### Eradication
- Full auth log review confirmed zero successful logins from source IP
- `admin` account password strength verified and noted for policy review

### Recovery
- No service disruption occurred; no recovery actions needed
- Enhanced monitoring left in place for 48 hours
- UFW rule retained as part of baseline hardening

---

## 6. Root Cause Analysis

**Root Cause:** SSH service accessible with password authentication enabled and no automatic IP blocking (fail2ban not installed).

**Contributing Factors:**
- No automatic lockout/block after N failed attempts
- Generic username `admin` used as target — username enumeration was trivial
- Alert threshold of 10 failures gave attacker 10 free attempts before detection

---

## 7. Lessons Learned

| Observation | Recommendation | Owner | Due Date | Priority |
|-------------|---------------|-------|---------|---------|
| No automatic brute force blocking | Install and configure fail2ban | IT Security | 2024-01-22 | High |
| SSH accepts password authentication | Disable password auth, require SSH keys | IT Security | 2024-01-22 | High |
| Alert threshold at 10 attempts | Reduce to 5 failures for privileged accounts | SIEM Admin | 2024-01-20 | Medium |
| Generic `admin` account exists | Rename or remove generic admin accounts | Systems Admin | 2024-01-31 | Medium |

---

## 8. Notifications

| Recipient | Role | Notified At | Method |
|-----------|------|------------|--------|
| [IT Manager Name] | IT Manager | 14:27 UTC+2 | Phone |
| N/A — no end-user affected | — | — | — |

**External Notification Required?** No

---

## 9. Evidence Log

| Item | Location | Collected By | Timestamp |
|------|---------|-------------|----------|
| Wazuh Rule 5712 alert screenshot | `/evidence/incident_001_brute_force/wazuh_alert.png` | [Your Name] | 14:18 UTC+2 |
| Auth log extract (47 failures) | `/evidence/incident_001_brute_force/auth_log_snippet.txt` | [Your Name] | 14:30 UTC+2 |
| UFW rule confirmation | `/evidence/incident_001_brute_force/ufw_rule.txt` | [Your Name] | 14:26 UTC+2 |
| Incident timeline | `/evidence/incident_001_brute_force/timeline.md` | [Your Name] | 16:00 UTC+2 |

---

## 10. Sign-off

| Role | Name | Date |
|------|------|------|
| Incident Handler | [Your Name] | 2024-01-15 |
| IT Security Lead | [Reviewer Name] | 2024-01-16 |

---

*This document is classified CONFIDENTIAL.*  
*Retention: 3 years from incident close date.*
