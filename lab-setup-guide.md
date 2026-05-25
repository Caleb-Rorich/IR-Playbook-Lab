# Lab Setup Guide — IR Playbook Lab

This guide walks through building the complete isolated lab environment from scratch. Estimated time: 2–3 hours. No cloud account required.

---

## Prerequisites

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| RAM | 12 GB | 16 GB |
| Disk space | 80 GB free | 120 GB free |
| CPU cores | 4 | 6+ |
| OS (host) | Windows 10 / Ubuntu 20.04+ / macOS 12+ | Any |

### Software to Download Before Starting

- [VirtualBox 7.0+](https://www.virtualbox.org/wiki/Downloads) + Extension Pack
- [Ubuntu Server 22.04 LTS ISO](https://ubuntu.com/download/server)
- [Windows 10 Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-10-enterprise)
- [Wazuh OVA (All-in-One)](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)

---

## Step 1 — Configure VirtualBox Host-Only Network

A Host-Only network isolates all VMs from the internet and each other in a controlled subnet.

1. Open VirtualBox → **File → Host Network Manager**
2. Click **Create**
3. Set the adapter:
   - IPv4 Address: `192.168.56.1`
   - IPv4 Mask: `255.255.255.0`
4. **Disable** DHCP server (we assign IPs manually)
5. Click **Apply**

> ⚠️ All VMs must use **Adapter 1: Host-Only Adapter** → `vboxnet0`. This ensures no traffic leaves your host machine.

---

## Step 2 — Deploy Wazuh All-in-One (SIEM)

Wazuh provides the SIEM, IDS alerting, and log dashboard.

### 2a. Import the OVA
1. VirtualBox → **File → Import Appliance**
2. Select the downloaded Wazuh `.ova` file
3. Set RAM to **4096 MB** minimum
4. Change Network Adapter to **Host-Only**
5. Click **Import**

### 2b. First Boot Configuration
```bash
# Default credentials (change immediately):
# Username: wazuh-user
# Password: wazuh

# Set static IP
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.56.110/24]
      gateway4: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
ip addr show eth0   # Confirm IP is 192.168.56.110
```

### 2c. Access the Dashboard
- From your **host machine** browser: `https://192.168.56.110`
- Default credentials: `admin` / `SecretPassword` (shown post-install)
- Accept the self-signed certificate warning

---

## Step 3 — Create Ubuntu Attacker VM

```
Name:    Attacker-Ubuntu
OS:      Ubuntu 22.04 LTS (64-bit)
RAM:     2048 MB
Disk:    25 GB (dynamically allocated)
Network: Adapter 1 → Host-Only (vboxnet0)
```

### 3a. Install Ubuntu Server
- During setup: select **Minimized** installation
- Set hostname: `attacker`
- Create user: `attacker` / password of your choice

### 3b. Set Static IP
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.56.102/24]
```
```bash
sudo netplan apply
```

### 3c. Install Attack Tools
```bash
sudo apt update && sudo apt install -y \
  nmap hydra metasploit-framework \
  curl wget git python3-pip

# Verify installations
nmap --version
hydra --version
msfconsole --version
```

---

## Step 4 — Create Windows 10 Victim VM

```
Name:    Victim-Windows10
OS:      Windows 10 (64-bit)
RAM:     4096 MB
Disk:    40 GB (dynamically allocated)
Network: Adapter 1 → Host-Only (vboxnet0)
```

### 4a. Set Static IP
- Settings → Network & Internet → Change adapter options
- Right-click → Properties → IPv4 → Use the following address:
  - IP: `192.168.56.101`
  - Subnet: `255.255.255.0`
  - Gateway: `192.168.56.1`

### 4b. Install Sysmon (Enhanced Logging)
```powershell
# Download Sysmon and config (run in PowerShell as Admin)
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath C:\Sysmon

# Download SwiftOnSecurity Sysmon config (best practice baseline)
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"

# Install Sysmon with config
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\sysmonconfig.xml
```

### 4c. Install Wazuh Agent
```powershell
# Download Wazuh agent MSI
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile "wazuh-agent.msi"

# Install and register with Wazuh manager
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.110" WAZUH_AGENT_NAME="Victim-Win10"

# Start the agent service
NET START WazuhSvc
```

---

## Step 5 — Verify Lab Connectivity

Run these checks before proceeding to incident simulations:

```bash
# From Attacker VM — confirm victim is reachable
ping 192.168.56.101
nmap -sn 192.168.56.0/24   # Should show all 3 IPs

# From Wazuh dashboard → Agents
# Confirm "Victim-Win10" appears as Active
```

Expected agent status in Wazuh: `Active` with a green indicator.

---

## Step 6 — Take VM Snapshots

Before running any simulations, snapshot all VMs. This lets you restore to a clean state instantly.

```
VirtualBox → Select each VM → Machine → Take Snapshot
Name: "Clean Baseline - [date]"
```

> 💡 **Always restore to this snapshot before starting a new simulation.** This keeps your evidence clean and reproducible.

---

## Lab IP Reference

| Host | IP | Role |
|------|----|------|
| VirtualBox Host | 192.168.56.1 | Host machine |
| Wazuh SIEM | 192.168.56.110 | Log aggregation, alerting |
| Windows Victim | 192.168.56.101 | Target, Wazuh agent installed |
| Ubuntu Attacker | 192.168.56.102 | Attack simulation platform |

---

## Troubleshooting

**VMs can't ping each other:**
- Confirm all are on the same Host-Only adapter (`vboxnet0`)
- Temporarily disable Windows Firewall on Victim VM for testing
- Check VirtualBox → Preferences → Network → vboxnet0 is enabled

**Wazuh agent shows Disconnected:**
- Check Wazuh manager IP is correct: `C:\Program Files (x86)\ossec-agent\ossec.conf`
- Restart agent: `NET STOP WazuhSvc && NET START WazuhSvc`
- Check firewall isn't blocking port 1514 (UDP) from Victim to SIEM

**Nmap from attacker returns nothing:**
- Confirm attacker IP is in the `192.168.56.x` subnet
- Run `ip addr show` to verify
