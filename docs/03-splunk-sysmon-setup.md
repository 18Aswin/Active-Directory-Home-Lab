# Part 3 — Splunk & Sysmon Configuration

## Objective
Install Splunk Enterprise on Ubuntu Server, create the endpoint index, enable log receiving, install Sysmon on Windows endpoints, and configure Splunk Universal Forwarder to ship Windows logs to Splunk.

---

## Splunk Installation on Ubuntu Server

### Download & Install

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Download Splunk Enterprise 10.2.2
wget -O splunk-10.2.2-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.2.2/linux/splunk-10.2.2-linux-2.6-amd64.deb"

# Install
sudo dpkg -i splunk-10.2.2-linux-amd64.deb
```

### Start Splunk

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

When prompted:
- Username: `admin`
- Password: (set a strong password)

### Enable Boot Start

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

Splunk now starts automatically whenever the Ubuntu VM boots.

### Access Splunk Web UI

From any browser on the network:
```
http://192.168.10.10:8000
```

Login with `admin` and the password set above.

> **Troubleshooting:** If login fails with "Login failed", the username must be `admin` — not the Ubuntu system username.

---

## Splunk Configuration

### Create the Endpoint Index

In Splunk Web UI:
1. Settings → Indexes → New Index
2. Name: `endpoint`
3. All other settings: default
4. Save

All Windows event logs will be stored in this index.

### Enable Splunk to Receive Data

1. Settings → Forwarding and Receiving
2. "Receive Data" → Configure Receiving → New Receiving Port
3. Port: `9997`
4. Save

Splunk is now listening for forwarded logs on port 9997.

---

## Sysmon Installation — Windows 11 (Target-PC)

Sysmon provides granular endpoint telemetry far beyond default Windows logging — covering process creation, network connections, file access, and more.

### Install Sysmon

Open PowerShell as Administrator:

```powershell
cd C:\Users\aswin\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i C:\sysmonconfig.xml
```

Output confirms: `Sysmon64 started`

### Verify

```powershell
Get-Service sysmon64
```

```
Status   Name
-------  ----
Running  Sysmon64
```

---

## Sysmon Installation — Windows Server 2022 (AD-SERVER01)

Repeated the same steps on the Domain Controller. Sysmon was installed using the same SwiftOnSecurity configuration file.

---

## Splunk Universal Forwarder — Windows 11

The Universal Forwarder is a lightweight agent that ships Windows event logs to the Splunk server.

### Download & Install

Downloaded from: https://www.splunk.com/en_us/download/universal-forwarder.html (Windows 64-bit MSI)

During installation:
- Selected: **"An on-premises Splunk Enterprise instance"**
- Set credentials: `admin` / (password)
- Receiving Indexer: `192.168.10.10` / Port: `9997`

### Configure inputs.conf

Created this file at:
```
C:\Program Files\SplunkUniversalForwarder\etc\apps\Splunk_TA_windows\local\inputs.conf
```

> The `local` folder was created manually as it did not exist by default.

```ini
[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

### Restart Forwarder

```powershell
Restart-Service SplunkForwarder
```

### Verify in Services

Start → `services.msc` → confirmed **SplunkForwarder** status = Running

---

## Splunk Universal Forwarder — Windows Server 2022

Repeated the same steps (download, install, inputs.conf, restart) on the Domain Controller.

---

## Verification — Logs Appearing in Splunk

In Splunk Web UI → Search & Reporting:

```
index=endpoint
```

Events from both Windows machines appeared within 1–2 minutes.

Hosts visible:
- `TARGET-PC`
- `AD-SERVER01`

Both machines confirmed sending logs to Splunk successfully.

---

## Next Step
[Part 4 → Active Directory Configuration](04-active-directory-config.md)
