# Playbook 02 — Phishing & Malicious Email

**Playbook ID:** PB-002  
**Version:** 1.0  
**MITRE ATT&CK:** T1566 (Phishing), T1566.001 (Spearphishing Attachment), T1566.002 (Spearphishing Link)

---

## 1. Overview

Phishing attacks use deceptive emails to trick users into revealing credentials, downloading malware, or transferring funds. Phishing is the most common initial access vector in security incidents.

### Detection Sources
- User report ("I think I clicked something suspicious")
- Email gateway alert (blocked attachment or suspicious link)
- SIEM alert: suspicious process spawned by Outlook / mail client
- Endpoint detection: macro execution, PowerShell from Office app

### Severity Classification

| Condition | Severity |
|-----------|---------|
| User clicked link AND entered credentials | **Critical** |
| User opened attachment, malware confirmed running | **Critical** |
| User clicked link, no credential entry confirmed | **High** |
| Email received and reported, nothing clicked | **Medium** |
| Email blocked by gateway, never delivered | **Low** |

---

## 2. Detection Criteria

Trigger this playbook when:

- [ ] User reports a suspicious email or "I think I was phished"
- [ ] Email gateway flags a message with malicious attachment/URL
- [ ] SIEM detects Office application spawning PowerShell or cmd.exe
- [ ] Credential stuffing alert follows email campaign
- [ ] Multiple users receive identical suspicious email (campaign)

---

## 3. Incident Response Steps

### Phase 1 — Identification

**Step 1.1 — Take the report seriously, gather details**
```
Ask the reporting user:
- When did you receive the email?
- Did you click any links? (Yes/No — don't lead them)
- Did you open any attachments?
- Did you enter any passwords or personal info?
- Is your computer behaving unusually?

DO NOT: Shame or blame the user. Encourage future reporting.
```

**Step 1.2 — Obtain the email for analysis**
```
Request the user forward as an attachment (not inline forward):
- Outlook: File → Save As → .msg format
- Gmail: Show original → Download

Alternatively: pull from email gateway/quarantine
```

**Step 1.3 — Analyse the email headers**
```bash
# Key header fields to check:
# - From: vs Reply-To: (do they match?)
# - Received: chain — does the origin make sense?
# - SPF / DKIM / DMARC results (look for "fail" or "none")
# - X-Originating-IP

# Free header analysis tools:
# https://mxtoolbox.com/EmailHeaders.aspx
# https://toolbox.googleapps.com/apps/messageheader/

# Check sending domain age:
whois [SENDER_DOMAIN] | grep "Creation Date"
```

**Step 1.4 — Analyse links and attachments (safely)**
```bash
# NEVER click links directly — use sandboxes:
# - https://urlscan.io  (paste URL, see screenshot + network activity)
# - https://www.virustotal.com  (scan URL or file hash)
# - https://app.any.run  (interactive sandbox for attachments)

# For attachments — check hash before opening:
sha256sum suspicious_attachment.docx
# Submit hash to VirusTotal
```

---

### Phase 2 — Containment

**Step 2.1 — Isolate affected machine (if clicked/opened)**
```
If user clicked a link or opened an attachment:
→ Disconnect machine from network (physical cable AND Wi-Fi)
→ Do NOT power off (preserves volatile memory/evidence)
→ Label machine: "EVIDENCE — DO NOT USE"
→ Notify user their machine will be unavailable temporarily
```

**Step 2.2 — Revoke credentials if entered on phishing site**
```bash
# Immediately reset password for affected account(s)
# Force logout all active sessions

# Active Directory (PowerShell)
Set-ADAccountPassword -Identity [USERNAME] -Reset -NewPassword (ConvertTo-SecureString "[NEWPASSWORD]" -AsPlainText -Force)
Set-ADUser -Identity [USERNAME] -ChangePasswordAtLogon $true

# Also revoke OAuth tokens / app sessions if cloud account:
# Microsoft 365: Revoke-AzureADUserAllRefreshToken
# Google: Admin Console → User → Security → Sign out all sessions
```

**Step 2.3 — Block the malicious sender and domain**
```
Email gateway:
- Block sender address
- Block sender domain
- Block URLs from the email body
- Add attachment hash to blocklist (if applicable)

Communicate to email team: "Please purge all instances of email from [SENDER] from all mailboxes"
```

**Step 2.4 — Check for campaign — other recipients**
```bash
# Search mail gateway logs for same sender / same subject
# Identify all users who received the email
# Did anyone else click or open?

# Communicate to all recipients:
# "If you received an email from [SENDER] with subject [SUBJECT],
#  do not click any links. Delete it immediately and report to IT Security."
```

---

### Phase 3 — Eradication

**Step 3.1 — Forensic review of affected machine**
```bash
# Check recently created/modified files
find / -newer /tmp/reference_file -type f 2>/dev/null | head -30

# Check running processes for anomalies
ps aux --sort=-%cpu | head -20
netstat -anp | grep ESTABLISHED

# Check scheduled tasks and startup items
crontab -l
ls -la ~/.config/autostart/
```

**Step 3.2 — Check for credential reuse across other systems**
```
If credentials were entered on a phishing site:
- Identify all systems where affected user has the same/similar password
- Force reset on all
- Check logs on critical systems for access from affected account in past 24h
```

**Step 3.3 — Reimage if malware confirmed**
```
If malware is confirmed on the affected machine:
- Do not attempt to clean — reimage from known-good baseline
- Restore user data from backup (verify backup predates incident)
- Re-enrol in endpoint management after reimaging
```

---

### Phase 4 — Recovery

- Re-enable user account with new credentials and MFA enforced
- Return cleaned/reimaged machine to user
- Brief user: what happened, what was done, what to watch for

---

### Phase 5 — Post-Incident / Lessons Learned

1. How did the phishing email bypass the gateway? Update rules?
2. Was the user's security awareness training current?
3. Was MFA in place? (MFA prevents credential theft being useful even if entered)
4. How quickly was the incident reported? Recognise the reporting user.
5. Update phishing indicators in email gateway blocklists.

---

## 4. Evidence Checklist

- [ ] Original email file (.msg or .eml) with full headers
- [ ] Screenshot of email body
- [ ] VirusTotal / urlscan.io results for links and attachments
- [ ] SIEM alert screenshots
- [ ] List of all recipients (from mail gateway logs)
- [ ] Timeline of user actions (clicked? when? what entered?)
- [ ] Screenshots of credential reset confirmations

---

## 5. References

- MITRE ATT&CK T1566 — https://attack.mitre.org/techniques/T1566/
- NIST SP 800-61 Rev. 2 — Phishing handling guidance
- Anti-Phishing Working Group (APWG) — https://apwg.org
