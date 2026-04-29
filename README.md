# 🛡️ SOC Home Lab — Full Pipeline

![SOC Lab](https://img.shields.io/badge/Status-Active-00e676?style=for-the-badge)
![Wazuh](https://img.shields.io/badge/Wazuh-v4.12.0-00d4ff?style=for-the-badge)
![n8n](https://img.shields.io/badge/n8n-SOAR-7c6af7?style=for-the-badge)
![Snort](https://img.shields.io/badge/Snort-IDS%2FIPS-ff5252?style=for-the-badge)

> A hands-on SOC Home Lab featuring **Wazuh SIEM**, **Snort IDS/IPS**, and **n8n automation**. Includes real attack simulations from Kali Linux, automated threat detection, IP enrichment via AbuseIPDB & VirusTotal, and instant alerting via Telegram, Gmail, and Google Sheets.

---

## 📋 Table of Contents

- [Lab Architecture](#-lab-architecture)
- [Tools & Stack](#-tools--stack)
- [Phase 1 — Wazuh SIEM](#-phase-1--wazuh-siem)
- [Phase 2 — Windows Agent](#-phase-2--windows-agent)
- [Phase 3 — Snort IDS/IPS](#-phase-3--snort-idsips)
- [Phase 4 — n8n SOAR Automation](#-phase-4--n8n-soar-automation)
- [Attack Simulations](#-attack-simulations)
- [Results](#-results)
- [Troubleshooting](#-troubleshooting)
- [Commands Reference](#-commands-reference)

---

## 🗺️ Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│           VMware Workstation — NAT 192.168.126.0/24  │
│                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────────────┐  │
│  │  Kali Linux  │    │  Windows 10  │    │   Ubuntu 22.04     │  │
│  │ .134 Attacker│───▶│ .135  Target │───▶│ .131 Wazuh Server  │  │
│  │ Snort IDS/IPS│    │ Wazuh Agent  │    │ n8n + ngrok        │  │
│  └──────────────┘    └──────────────┘    └────────────────────┘  │
└─────────────────────────────────────────────────────┘
                                                  │
                              ┌───────────────────┼───────────────────┐
                              ▼                   ▼                   ▼
                         Telegram             Gmail            Google Sheets
                         SOC Alerts         HTML Report       Incident Log
```

| VM | IP | Role | OS |
|----|----|------|----|
| Wazuh Server | 192.168.126.131 | SIEM + n8n + ngrok | Ubuntu 22.04 |
| Windows Target | 192.168.126.135 | Wazuh Agent | Windows 10 |
| Kali Attacker | 192.168.126.134 | Attack Tools + Snort | Kali Linux |

---

## 🧰 Tools & Stack

| Tool | Version | Purpose |
|------|---------|---------|
| Wazuh | v4.12.0 | SIEM — threat detection & log analysis |
| Snort | 3.11.1.0 | IDS/IPS — network intrusion detection |
| n8n | latest | SOAR — workflow automation |
| ngrok | latest | Tunnel — expose n8n webhook publicly |
| AbuseIPDB | API v2 | IP reputation check |
| VirusTotal | API v3 | IP/file analysis |
| Telegram Bot | — | Real-time SOC alerts |
| Gmail | OAuth2 | HTML email alerts |
| Google Sheets | — | Automated incident logging |

---

## 🔵 Phase 1 — Wazuh SIEM

### Installation
Wazuh All-in-One installation on Ubuntu 22.04 LTS.

```bash
# Download and run Wazuh installer
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
sudo bash wazuh-install.sh -a

# Access dashboard
https://192.168.126.131
```

### Key Configurations
- Fixed LVM disk partition issue during installation
- Dashboard accessible at `https://192.168.126.131:443`
- Default credentials reset using `wazuh-passwords-tool.sh`

### Screenshot — Wazuh Dashboard
![Wazuh Dashboard](screenshots/wazuh-dashboard.png)

### Alert Levels Reference

| Level | Severity | Description |
|-------|----------|-------------|
| 1–3 | Low | Informational events |
| 4–6 | Medium | Warning events |
| 7–10 | High | Suspicious activity |
| 11–15 | Critical | Active threat detected |

---

## 🟦 Phase 2 — Windows Agent

### Agent Installation
Wazuh Agent deployed on Windows 10 VM and enrolled with the Wazuh Manager.

```powershell
# Install agent (PowerShell as Admin)
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.126.131"

# Start agent service
NET START WazuhSvc
```

### Screenshot — Windows Agent Active
![Windows Agent](screenshots/wazuh-agent-windows.png)

---

## 🔴 Phase 3 — Snort IDS/IPS

### Installation on Kali Linux

```bash
# Install Snort
sudo apt install snort -y

# Test IDS mode
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0

# IPS mode with NFQueue (100% packet drop)
sudo iptables -I INPUT -j NFQUEUE --queue-num 0
sudo snort -Q --daq nfq --daq-var queue=0 -c /etc/snort/snort.conf
```

### Custom Rule Example
```
alert icmp any any -> 192.168.126.0/24 any (msg:"ICMP Ping Detected"; sid:1000001; rev:1;)
alert tcp any any -> 192.168.126.131 22 (msg:"SSH Connection Attempt"; sid:1000002; rev:1;)
```

### Results
- ✅ IDS mode — detected and logged all attacks
- ✅ IPS mode with NFQueue — **100% packet drop** confirmed

---

## ⚡ Phase 4 — n8n SOAR Automation

### Why n8n?
Replaced Shuffle SOAR due to stability issues after restarts. n8n provides native nodes for Telegram, Gmail, Google Sheets, and full HTTP Request flexibility.

### Infrastructure Setup

```bash
# Run n8n with Docker
docker run -d \
  --name n8n \
  --network host \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n

# Start ngrok tunnel
ngrok http 5678
```

### Workflow Architecture

```
Webhook (POST /wazuh-alerts)
    │
    ▼
AbuseIPDB — Check IP reputation score
    │
    ▼
VirusTotal — IP analysis & malware check
    │
    ▼
If (rule.level > 2)
    │
    ├── TRUE ──▶ Telegram SOC Alert
    │               │
    │               ▼
    │            Gmail HTML Email
    │               │
    │               ▼
    │            Auto Block IP (Wazuh API)
    │               │
    │               ▼
    │            Log Incident (Google Sheets)
    │
    └── FALSE ──▶ (ignored)
```

### Node Configuration

| Node | Type | Details |
|------|------|---------|
| **Webhook** | Trigger | POST /wazuh-alerts — receives Wazuh JSON |
| **AbuseIPDB** | HTTP Request | GET api.abuseipdb.com/api/v2/check |
| **VirusTotal** | HTTP Request | GET virustotal.com/api/v3/ip_addresses/{ip} |
| **If** | Conditional | rule.level > 2 |
| **Telegram SOC Alert** | Telegram | Instant formatted alert |
| **Email** | Gmail OAuth2 | HTML email with incident details |
| **Auto Block IP** | HTTP Request | POST Wazuh API /active-response |
| **Log Incident** | Google Sheets | Append row to Wazuh Incidents sheet |

### Screenshot — n8n Workflow Canvas
![n8n Workflow](screenshots/n8n-workflow.png)

### Wazuh Integration Script
File: `/var/ossec/integrations/custom-webhook`

```python
#!/usr/bin/env python3
import sys, json, urllib.request

alert_file = open(sys.argv[1])
alert = json.load(alert_file)
alert_file.close()

hook_url = sys.argv[3]
data = json.dumps(alert).encode('utf-8')
req = urllib.request.Request(
    hook_url, data=data,
    headers={'Content-Type': 'application/json'}
)
urllib.request.urlopen(req)
```

```bash
# Set permissions
sudo chmod +x /var/ossec/integrations/custom-webhook
sudo chown root:wazuh /var/ossec/integrations/custom-webhook
```

### ossec.conf Integration Block
```xml
<integration>
  <name>custom-webhook</name>
  <hook_url>https://YOUR-NGROK-URL/webhook/wazuh-alerts</hook_url>
  <api_key>none</api_key>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

### Telegram Alert Format
```
🚨 SOC ALERT 🚨
━━━━━━━━━━━━━━━━━━
📋 Rule: syslog: User missed the password more than one time
⚠️ Level: 10
🌐 Source IP: 192.168.126.134
🖥️ Agent: ubuntu-wazuh
🕐 Time: 2026-04-26T18:53:10.445+0000
━━━━━━━━━━━━━━━━━━
🤖 Powered by Wazuh + n8n
```

### Screenshot — Telegram Alerts
![Telegram Alerts](screenshots/telegram-alerts.png)

### Screenshot — Gmail Inbox
![Gmail Alerts](screenshots/gmail-alerts.png)

### Screenshot — Google Sheets Log
![Google Sheets](screenshots/google-sheets.png)

---

## 💥 Attack Simulations

### 1. SSH Brute Force — Kali → Ubuntu

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.126.131 -t 4
```

**Detection:** Rule 5760 — `syslog: User missed the password more than one time`
**Level:** 10 | **Alert:** ✅ Telegram + Gmail + Google Sheets

### 2. RDP Brute Force — Kali → Windows

```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.126.135 -t 4
```

**Detection:** `Multiple Windows Logon Failures`
**Level:** 10 | **Alert:** ✅ Telegram + Gmail + Google Sheets

### Screenshot — Wazuh Alerts Dashboard
![Wazuh Alerts](screenshots/wazuh-alerts.png)

---

## ✅ Results

| Attack | Source | Target | Level | Alert |
|--------|--------|--------|-------|-------|
| SSH Brute Force | Kali .134 | Ubuntu .131 | 🔴 10 | ✅ Telegram + Gmail + Sheets |
| RDP Brute Force | Kali .134 | Windows .135 | 🔴 10 | ✅ Telegram + Gmail + Sheets |
| Malware Detection | Windows Edge Update | Win-VM | 🔴 15 | ✅ Detected (False Positive) |

### Pipeline Status

| Component | Status |
|-----------|--------|
| Wazuh SIEM | ✅ Active |
| Snort IDS/IPS | ✅ Active |
| n8n Workflow | ✅ Active |
| Telegram Alerts | ✅ Working |
| Gmail Alerts | ✅ Working |
| Google Sheets Logging | ✅ Working |
| Auto Block IP | ⚠️ Pending — needs Wazuh JWT token |

---

## 🔧 Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Webhook 404 not registered | Workflow not active | Activate via n8n API: `POST /api/v1/workflows/{id}/activate` |
| AbuseIPDB fails on private IPs | Private IPs not in database | Set **On Error = Continue** on node |
| If condition not working | Type mismatch | Enable **Convert types where required** |
| Wazuh integration not sending | Missing custom-webhook script | Create script in `/var/ossec/integrations/` |
| SSL error on Wazuh API | Self-signed certificate | Disable SSL Certificates in n8n node settings |
| Telegram on wrong branch | Connected to False branch | Reconnect from True (+) branch |
| Google OAuth rejects redirect | Bare IP not allowed | Use ngrok domain as redirect URI |

---

## 📚 Commands Reference

### Wazuh
```bash
# Restart Wazuh
sudo systemctl restart wazuh-manager
sudo /var/ossec/bin/wazuh-control restart

# Check integration log
sudo tail -f /var/ossec/logs/integrations.log

# Check alerts
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Check errors
sudo tail -f /var/ossec/logs/ossec.log | grep -i "integrat"

# Reset password
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p "NewPassword1."

# Test log parsing
sudo /var/ossec/bin/wazuh-logtest
```

### n8n
```bash
# Start n8n
docker run -d --name n8n --network host \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n n8nio/n8n

# Activate workflow via API
curl -X POST http://localhost:5678/api/v1/workflows/{WORKFLOW_ID}/activate \
  -H "X-N8N-API-KEY: YOUR_API_KEY"
```

### ngrok
```bash
# Start ngrok
ngrok http 5678

# Check status
curl http://localhost:4040/api/tunnels
```

### Test Webhook Manually
```bash
curl -X POST https://YOUR-NGROK-URL/webhook/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{
    "src_ip": "185.220.101.1",
    "rule": {"description": "Test Alert", "level": 10},
    "agent": {"name": "ubuntu-wazuh"},
    "timestamp": "2026-04-26T18:53:10.445+0000"
  }'
```

### Kali Attack Tools
```bash
# SSH Brute Force
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.126.131 -t 4

# RDP Brute Force
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.126.135 -t 4

# Network Scan
nmap -sV -O 192.168.126.135

# SMB Vulnerability Scan
nmap --script smb-vuln* 192.168.126.135
```

---

## 👤 Author

**Amr Atef**
- GitHub: [@33mr](https://github.com/33mr)

---

*SOC Home Lab — Wazuh + Snort + n8n | Built for learning and documentation purposes*
