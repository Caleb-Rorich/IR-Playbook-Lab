# Playbook 01 — Brute Force Attack

**Playbook ID:** PB-001  
**Version:** 1.0  
**Owner:** IT Security  
**Last Updated:** 2024-01  
**MITRE ATT&CK:** T1110 (Brute Force), T1110.001 (Password Guessing), T1110.003 (Password Spraying)

---

## 1. Overview

A brute force attack involves an adversary systematically attempting large numbers of username/password combinations to gain unauthorised access to systems, services, or accounts. Common targets include SSH, RDP, web login portals, VPNs, and Active Directory.

### Detection Sources
- SIEM alert (Wazuh Rule IDs: 5551, 5712, 60204)
- Failed authentication log spike (Windows Event ID 4625, Linux `/var/log/auth.log`)
- IDS/IPS signature alert
- User report of account lockout

### Severity Classification

| Condition | Severity |
|-----------|---------|
| External IP, >50 attempts, account locked | **Critical** |
| External IP, >10 attempts, no lockout | **High** |
| Internal IP, any volume | **High** (possible insider or compromised host) |
| Single user, <10 attempts | **Medium** |

---

## 2. Detection Criteria

Trigger this playbook when **any** of the following are observed:

- [ ] 10+ failed authentication attempts from a single source IP within 60 seconds
- [ ] Authentication attempts against multiple accounts from one source (password spraying)
- [ ] Account lockout triggered on a privileged or service account
- [ ] Failed auth attempts originating from a foreign/unexpected geolocation
- [ ] SIEM alert matching brute force signature

---

## 3. Incident Response Steps

### Phase 1 — Identification (Target: <15 minutes)

**Step 1.1 — Acknowledge and log the alert**
```
- Record: Alert timestamp, source, initial severity
- Open an incident ticket
- Assign Incident ID: INC-[YYYY-MM-DD]-[###]
```

**Step 1.2 — Gather initial data from SIEM**
```bash
# Wazuh / Kibana KQL query — SSH brute force
agent.name: "Victim-Win10" AND rule.id: 5712

# Elastic — Windows failed logins (Event ID 4625)
event.code: 4625 AND winlog.event_data.IpAddress: [SOURCE_IP]

# Questions to answer:
# - What is the source IP? Internal or external?
# - What username(s) are being targeted?
# - What service/port is being attacked?
# - What time did attempts begin? Are they ongoing?
```

**Step 1.3 — Check threat intelligence on source IP**
```
- Query: https://www.virustotal.com/gui/ip-address/[SOURCE_IP]
- Query: https://www.abuseipdb.com/check/[SOURCE_IP]
- Query: https://threatintelligenceplatform.com/
- Record: Reputation score, known malicious activity, geolocation
```

**Step 1.4 — Determine if any attempt succeeded**
```bash
# Linux — check for successful login after failures
grep "Accepted" /var/log/auth.log | grep [SOURCE_IP]

# Windows — Event ID 4624 (successful logon) after 4625 failures
# Kibana: event.code: 4624 AND source.ip: [SOURCE_IP]

# CRITICAL: If a successful login is found → escalate to Critical
#           → Activate full IR response, not just brute force playbook
```

---

### Phase 2 — Containment (Target: <30 minutes from detection)

**Step 2.1 — Block source IP**
```bash
# Linux firewall (UFW)
sudo ufw deny from [SOURCE_IP] to any
sudo ufw reload

# Linux firewall (iptables)
sudo iptables -I INPUT -s [SOURCE_IP] -j DROP

# Windows Firewall (PowerShell as Admin)
New-NetFirewallRule -DisplayName "Block Brute Force [SOURCE_IP]" `
  -Direction Inbound -Action Block `
  -RemoteAddress [SOURCE_IP]
```

**Step 2.2 — Lock targeted account(s) if not already locked**
```bash
# Linux
sudo passwd -l [USERNAME]

# Windows (PowerShell)
Disable-ADAccount -Identity [USERNAME]
# OR
net user [USERNAME] /active:no
```

**Step 2.3 — Notify stakeholders**
```
Immediate notification required for:
- IT Manager / Security Lead: within 30 minutes
- Affected user(s): if their credentials were targeted
- If privileged account targeted: escalate to CISO

Communication template: See /reports/incident_report_template.md
```

---

### Phase 3 — Eradication

**Step 3.1 — Full log review**
```bash
# Review auth logs for past 24 hours from the source IP
# Determine full scope: how many accounts targeted?
grep [SOURCE_IP] /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Check for any successful authentications from same IP range
grep "Accepted" /var/log/auth.log | grep "192.168.X"
```

**Step 3.2 — Check for persistence (if breach occurred)**
```bash
# New user accounts created recently
lastlog | grep -v "Never"
cat /etc/passwd | grep -v "nologin\|false" | tail -10

# Scheduled tasks (Windows)
schtasks /query /fo LIST | findstr "Task Name\|Status"

# Unusual processes
ps aux | grep -v "^root\|^[a-z]" | head -20
```

**Step 3.3 — Reset credentials for targeted accounts**
```
- Force password reset for all accounts that were targeted
- Require new password to meet policy (12+ chars, complexity)
- Brief affected users on the incident
```

---

### Phase 4 — Recovery

**Step 4.1 — Re-enable accounts with controls**
```bash
# Linux — unlock account
sudo passwd -u [USERNAME]

# Windows
Enable-ADAccount -Identity [USERNAME]

# Ensure MFA is enforced before re-enabling remote access
```

**Step 4.2 — Monitor for 48 hours**
```
- Create SIEM watchlist for source IP subnet
- Set alert threshold lower: flag after 3 failures (not 10)
- Review auth logs morning and evening for 2 days
- Document any repeat activity
```

**Step 4.3 — Close the ticket**
```
- Update incident ticket with full timeline
- Complete incident report (template: /reports/incident_report_template.md)
- Confirm stakeholders notified of resolution
```

---

### Phase 5 — Post-Incident / Lessons Learned

Complete within **5 business days** of resolution.

**Discussion questions:**
1. Was the brute force detected promptly? What was the time from first attempt to alert?
2. Were account lockout policies effective? If not, why not?
3. Was MFA in place on the targeted service? If not, add to remediation backlog.
4. Did the blocking response take too long? Can it be automated?
5. Were communication processes smooth? Any gaps?

**Improvement actions (examples):**
- [ ] Implement fail2ban or equivalent for automatic IP blocking after N failures
- [ ] Enable MFA on all externally accessible services
- [ ] Tighten account lockout policy: lock after 5 failures (not 10)
- [ ] Add geographic IP blocking for countries with no business presence

---

## 4. Evidence Checklist

Collect and preserve before making changes:

- [ ] SIEM alert screenshot with timestamp
- [ ] Raw authentication log extract (source IP, timestamps, usernames)
- [ ] Firewall rule applied (screenshot or command output)
- [ ] Threat intel results for source IP
- [ ] Account lockout confirmation
- [ ] Timeline document

---

## 5. References

- NIST SP 800-61 Rev. 2 — Section 3.2 (Detection and Analysis)
- MITRE ATT&CK T1110 — https://attack.mitre.org/techniques/T1110/
- CIS Control 6 — Access Control Management
- PCI DSS Req 8.3.4 — Invalid authentication attempt lockout
