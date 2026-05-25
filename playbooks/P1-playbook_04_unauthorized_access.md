# Playbook 04 — Unauthorised Access

**Playbook ID:** PB-004  
**Version:** 1.0  
**MITRE ATT&CK:** T1078 (Valid Accounts), T1098 (Account Manipulation), T1136 (Create Account)

---

## 1. Overview

Unauthorised access occurs when an individual accesses systems, data, or resources without permission. This includes external attackers using stolen credentials, insiders abusing access rights, or privilege escalation by a legitimately authenticated user.

### Severity Classification

| Condition | Severity |
|-----------|---------|
| Admin/root access obtained by unauthorised party | **Critical** |
| Sensitive data (PII, financial) accessed | **Critical** |
| Unauthorised access to production systems | **High** |
| Unauthorised access to non-sensitive systems | **Medium** |
| Attempted access, no success confirmed | **Low** |

---

## 2. Detection Criteria

- [ ] Login outside of normal business hours (nights, weekends) for admin accounts
- [ ] Login from unexpected geographic location or IP range
- [ ] Privilege escalation event (standard user → admin)
- [ ] New account created outside of provisioning process
- [ ] Large volume of data accessed or downloaded by a single account
- [ ] Access to systems the user has no business reason to access
- [ ] Multiple accounts accessed from the same IP (credential stuffing)

---

## 3. Incident Response Steps

### Phase 1 — Identification

**Step 1.1 — Gather context on the access event**
```bash
# Windows — review logon event (Event ID 4624)
# Key fields: Account Name, Logon Type, Source IP, Workstation

# Logon Type reference:
# 2 = Interactive (physical keyboard)
# 3 = Network (mapped drive, file share)
# 10 = Remote Interactive (RDP)

# Kibana query for suspicious logons:
event.code: 4624 AND winlog.event_data.LogonType: 10
AND NOT source.ip: 192.168.56.0/24
```

**Step 1.2 — Establish what the account accessed**
```bash
# Windows — file access audit events (Event ID 4663)
# Requires file system auditing to be enabled

# Review:
# - Which files/folders were opened or modified?
# - Were any files copied to external storage or cloud?
# - Were any admin tools or sensitive directories accessed?

# Check for large data transfers:
# - Outbound data volume from this host (firewall/proxy logs)
# - Files copied to USB (Windows Event ID 4663 on removable media)
```

**Step 1.3 — Determine if the legitimate user was responsible**
```
Contact the account owner via a method OTHER than email (call or in-person):
- "Were you logged into [SYSTEM] at [TIME] from [LOCATION]?"
- If No: account is compromised → treat as Critical, escalate
- If Yes: document explanation, assess if access was authorised
- If Unavailable: treat as compromised until confirmed otherwise
```

---

### Phase 2 — Containment

**Step 2.1 — Disable the account immediately (if compromise confirmed)**
```bash
# Active Directory
Disable-ADAccount -Identity [USERNAME]

# Linux
sudo passwd -l [USERNAME]
sudo usermod -L [USERNAME]

# Force logout all active sessions:
# Windows: query session — logoff [SESSION_ID]
query session /server:[SERVERNAME]
logoff [SESSION_ID] /server:[SERVERNAME]
```

**Step 2.2 — Preserve session data before ending it**
```bash
# Before killing active RDP/SSH sessions, document:
# - Current running processes in that session
# - Opened files
# - Network connections

# Linux — who is logged in
who
w
last | head -20

# Windows — active sessions
qwinsta
```

**Step 2.3 — Revoke all active tokens and sessions**
```
- Invalidate all active authentication tokens
- Revoke VPN certificates if applicable
- Revoke OAuth tokens for cloud apps (M365, Google)
- Change service account passwords if the account is a service account
```

---

### Phase 3 — Eradication

**Step 3.1 — Check for backdoors left by attacker**
```bash
# New local accounts created
net user             # Windows
cat /etc/passwd | grep -v "nologin\|false"   # Linux

# New privileged group members
net localgroup administrators   # Windows
getent group sudo               # Linux

# Scheduled tasks created recently
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|status"

# New SSH authorised keys
cat ~/.ssh/authorized_keys
find /home -name "authorized_keys" 2>/dev/null
```

**Step 3.2 — Review and remove unauthorised access rights**
```
- Audit all permissions granted to the affected account
- Remove any access that was not formally provisioned
- Check for role assignments added since the last review
- Verify no new admin accounts were created
```

---

### Phase 4 — Recovery

- Re-enable account (if legitimate user, not insider threat) with new credentials + MFA
- Conduct access rights review for the account before re-enabling
- If insider threat is confirmed: follow HR/Legal process, do not re-enable
- Monitor account for 30 days with enhanced alerting after recovery

---

### Phase 5 — Post-Incident / Lessons Learned

1. How were the credentials obtained? (Phishing, dark web, brute force, insider?)
2. Was MFA in place? If not, this incident validates the need.
3. Were access rights appropriate (least privilege)?
4. How long did the unauthorised access go undetected? (MTTD)
5. Would User and Entity Behaviour Analytics (UEBA) have caught this earlier?

---

## 4. Evidence Checklist

- [ ] Logon event logs (Event ID 4624, 4625, 4634)
- [ ] Source IP and geolocation
- [ ] List of resources accessed during the unauthorised session
- [ ] Outbound data transfer volumes
- [ ] Account lockout/disable confirmation
- [ ] Communication record with account owner
- [ ] Timeline from first suspicious event to containment

---

## 5. References

- MITRE ATT&CK T1078 — Valid Accounts: https://attack.mitre.org/techniques/T1078/
- NIST SP 800-61 — Section 3.2.5 (Scoping the Incident)
- ISO/IEC 27035 — Information Security Incident Management
