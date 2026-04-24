# Active Directory Home Lab — Security Monitoring & Attack Simulation

> A fully functional enterprise-grade home lab built from scratch using VirtualBox, featuring Active Directory, Splunk SIEM, brute force attack simulation, and MITRE ATT&CK technique testing with Atomic Red Team.

---

## Project Overview

This project simulates a real-world enterprise network environment inside VirtualBox. It covers both **blue team (defensive)** and **red team (offensive)** security operations — making it a complete end-to-end cybersecurity lab.

The lab was built following the MyDFIR 5-part Active Directory series, with several real-world adaptations made due to hardware constraints and software compatibility challenges — all of which are documented in detail.

---

## Network Diagram

```
┌─────────────────────────────────────────────────────┐
│              NAT Network: 192.168.10.0/24           │
│                                                     │
│  ┌──────────────┐        ┌──────────────────────┐   │
│  │  Kali Linux  │        │   Windows 11 Ent.    │   │
│  │  (Attacker)  │──────▶│   (Target Machine)    │  |
│  │192.168.10.250│  RDP   │  192.168.10.100      │   │
│  └──────────────┘ Attack └──────────┬───────────┘   │
│                                     │ Logs          │
│  ┌──────────────┐        ┌──────────▼───────────┐   │
│  │ Win Server   │        │   Ubuntu Server      │   │
│  │ 2022 (DC)    │        │   Splunk SIEM        │   │
│  │192.168.10.20 │        │  192.168.10.10       │   │
│  │aswinAD.local │        │  Port: 8000 (Web)    │   │
│  └──────────────┘        └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

## Tools & Technologies

| Category | Tool | Purpose |
|---|---|---|
| Virtualization | VirtualBox | Hosts all 4 virtual machines |
| Domain Services | Windows Server 2022 | Active Directory Domain Controller |
| Target Endpoint | Windows 11 Enterprise Eval | Domain-joined target machine |
| Attack Machine | Kali Linux | RDP brute force + recon |
| SIEM | Splunk Enterprise 10.2.2 | Log collection and threat detection |
| Telemetry | Sysmon (SwiftOnSecurity config) | Advanced Windows endpoint logging |
| Log Forwarding | Splunk Universal Forwarder | Ships logs from Windows VMs to Splunk |
| Brute Force | Hydra | RDP credential brute force attack |
| Attack Simulation | Atomic Red Team | MITRE ATT&CK technique simulation |
| Network Recon | Nmap | Port scanning and service discovery |

---

## Lab Architecture

| Machine | OS | IP Address | Role |
|---|---|---|---|
| AD-SERVER01 | Windows Server 2022 | 192.168.10.20 | Domain Controller (aswinAD.local) |
| Target-PC | Windows 11 Enterprise Eval | 192.168.10.100 | Domain-joined Target Endpoint |
| Splunk-Server | Ubuntu Server 22.04 | 192.168.10.10 | SIEM (Splunk 10.2.2) |
| Kali | Kali Linux | 192.168.10.250 | Attacker Machine |

**Host Machine Specs:**
- OS: Windows 11
- CPU: AMD Ryzen 7 6800H
- RAM: 16 GB
- Storage: ~256 GB SSD

---

## Project Objectives

- Build a functional Active Directory domain (`aswinAD.local`) with Organizational Units and domain users
- Configure Splunk to ingest and index security events from Windows endpoints
- Install Sysmon for granular endpoint telemetry beyond default Windows logging
- Simulate a real RDP brute force attack using Hydra from Kali Linux
- Detect and analyze the attack in Splunk using Windows Event IDs 4625 and 4624
- Execute MITRE ATT&CK techniques using Atomic Red Team across multiple tactics
- Identify detection gaps and document findings

---

## Project Parts

| Part | Topic | Documentation |
|---|---|---|
| Part 1 | Network Design & VirtualBox Setup | [View →](docs/01-network-and-virtualbox-setup.md) |
| Part 2 | VM Installations | [View →](docs/02-vm-installations.md) |
| Part 3 | Splunk & Sysmon Configuration | [View →](docs/03-splunk-sysmon-setup.md) |
| Part 4 | Active Directory Configuration | [View →](docs/04-active-directory-config.md) |
| Part 5 | Attack Simulation & Detection | [View →](docs/05-attack-simulation.md) |

---

## Key Findings

### Brute Force Attack Detection (Hydra → RDP)
- Hydra successfully brute forced EWhite's RDP credentials using a custom wordlist
- Splunk detected multiple **Event ID 4625** (failed logon) events in rapid succession — a clear brute force signature
- **Event ID 4624** (successful logon) captured the moment the attack succeeded
- Source IP `192.168.10.250` (Kali Linux) was visible in expanded event details

### T1136.001 — Create Local Account (Persistence)
- Executed persistence technique via Atomic Red Team on the Target-PC
- Splunk detected **12 events** across **6 unique Event IDs**:

| EventCode | Description |
|---|---|
| 4720 | New user account created |
| 4722 | User account enabled |
| 4724 | Password reset attempted |
| 4726 | User account deleted (ART cleanup) |
| 4738 | User account changed |
| 4798 | Local group membership enumerated |

> **Note:** The reference tutorial showed zero detection for this technique. This lab's use of the SwiftOnSecurity Sysmon configuration provided complete detection coverage — a superior outcome demonstrating the value of proper Sysmon tuning.

### T1059.001 — PowerShell Execution (Execution)
- Splunk detected **524 events** in a 15-minute window
- Sysmon captured the full PowerShell execution chain including process hashes, parent-child process relationships, and DLL load activity
- `amsi.dll` (Anti-Malware Scan Interface) load was captured — demonstrating that even AMSI-aware techniques leave full forensic artifacts in Sysmon logs

| Detail | Value |
|---|---|
| Technique | T1059.001 — PowerShell |
| Tactic | Execution |
| Events Detected | 524 |
| Source | Target-PC.aswinAD.local |
| Log Source | XmlWinEventLog:Microsoft-Windows-Sysmon/Operational |
| Key Process | powershell.exe → SOAPHound.exe |
| Detection | Full visibility via Sysmon |

### T1087.001 — Local Account Discovery (Discovery)
- Atomic Red Team executed a series of local account and group enumeration commands
- Sysmon EID 1 (Process Creation) captured `cmdkey.exe` spawned by `powershell.exe`

| Detail | Value |
|---|---|
| Technique | T1087.001 — Local Account Discovery |
| Tactic | Discovery |
| Event ID | 1 (Sysmon — Process Creation) |
| Process | cmdkey.exe (PID 4128) |
| Parent Process | powershell.exe |
| User | ASWINAD\Administrator |

### T1003.001 — LSASS Credential Dump (Credential Access)
- Atomic Red Team executed `xordump.exe` to dump LSASS memory — a classic credential theft technique
- Splunk detected **3 events** via Sysmon EID 1 (Process Creation)
- The full dump command including output path (`lsass-xordump.t1003.001.dmp`) was captured in the command-line field

| Detail | Value |
|---|---|
| Technique | T1003.001 — LSASS Memory |
| Tactic | Credential Access |
| Events Detected | 3 |
| Event ID | 1 (Sysmon — Process Creation) |
| Process | xordump.exe |
| Parent Process | powershell.exe |
| User | ASWINAD\Administrator |

### T1053.005 — Scheduled Task (Persistence)
- Atomic Red Team created a scheduled task (`EventViewerBypass`) — a known UAC bypass persistence technique
- Splunk detected **81 events** across Sysmon EID 1 and EID 7

| Detail | Value |
|---|---|
| Technique | T1053.005 — Scheduled Task |
| Tactic | Persistence |
| Events Detected | 81 |
| Event IDs | 1 (Process Creation), 7 (Image Loaded) |
| Process | schtasks.exe (PID 10784) |
| Parent Process | cmd.exe → powershell.exe |
| Task Name | EventViewerBypass |
| Trigger | ONLOGON, RL HIGHEST |
| DLL Loaded | taskschd.dll (Task Scheduler COM API) |

### T1562.001 — Disable Windows Defender (Defense Evasion) — Detection Gap
- Sub-test T1562.001-58 failed with Exit code 1 due to a missing internet-hosted dependency (`EDR-Freeze_1.0.zip`)
- **0 events detected** — no registry modification events (Sysmon EID 13) were generated

> **Finding:** This represents a detection gap. Recommended improvement: deploy custom Sysmon rules targeting `DisableRealtimeMonitoring` registry key modifications to close this gap.

---

## Attack Summary

| # | Attack | Tactic | Tool | Result |
|---|---|---|---|---|
| 1 | RDP Brute Force | Initial Access | Hydra | Detected — EID 4625 + 4624 |
| 2 | T1136.001 Create Local Account | Persistence | Atomic Red Team | 12 events, 6 Event IDs |
| 3 | T1059.001 PowerShell Execution | Execution | Atomic Red Team | 524 events |
| 4 | T1087.001 Local Account Discovery | Discovery | Atomic Red Team | Sysmon EID 1 — cmdkey.exe |
| 5 | T1003.001 LSASS Credential Dump | Credential Access | Atomic Red Team | 3 events — Sysmon EID 1 (xordump.exe) |
| 6 | T1053.005 Scheduled Task | Persistence | Atomic Red Team | 81 events — Sysmon EID 1 + EID 7 |
| 7 | T1562.001 Disable Defender | Defense Evasion | Atomic Red Team | Not Detected — detection gap |

---

## 📸 Screenshots

All evidence screenshots are in the [`screenshots/`](screenshots/) folder.

| Screenshot | Description |
|---|---|
| `hydra_rdp1.png` | Showing RDP connection to host EWhite successful |
| `hydra_rdp2.png` | RDP details |
| `Splunk_Running.png` | Splunk running status |
| `Splunk_IP.png` | Splunk IP Address |
| `Login.png` | Windows Server 2022 AD Login |
| `Server_Manager_Dashboard.png` | Windows Server Manager Dashboard |
| `Splunk_Host.png` | Splunk Dashboard Host info |
| `Splunk_Source.png` | Splunk Dashboard Source info |
| `splunk-4624-success.png` | Event ID 4624 showing successful attacker login |
| `splunk-4625-fail.png` | Event ID 4625 showing failed attacker login |
| `Atomic_Red_Team-Local_Account_Creation.png` | Atomic Red Team Attack T1136.001 |
| `Local_Account_Event_Detection.png` | Events detected in Splunk from T1136.001 |
| `Atomic_Red_Team-Powershell_Command_and_Scripting_Interpreter.png` | Atomic Red Team Attack T1059.001 |
| `Attacker-IP.png` | Attacker machine IP details |
| `Brute_Force_Event_codes.png` | Brute force event codes from Splunk |
| `Local_Domain_Connection.png` | Connecting to domain aswinAD.local |
| `1087.001_Powershell.png` | 1087.001 attack Powershell |
| `1087.001_SplunkEvent.png` | 1087.001 Splunk Event |
| `1087.001_ProcessID.png` | 1087.001 attack Process IDs |
| `T1003.001_Powershell.png` | 1003.001 attack Powershell |
| `T1003.001_Splunk.png` | 1003.001 Splunk Event |
| `T1053.005_Powershell.png` | 1053.005 attack Powershell |
| `T1053.005_Splunk.png` | 1053.005 Splunk Event |
| `T1562.001_Powershell.png` | 1053.005 attack Powershell |

---

## Skills Demonstrated

- Active Directory domain configuration and user management
- SIEM deployment and log ingestion (Splunk)
- Endpoint telemetry with Sysmon
- Network configuration in a virtualized environment
- RDP brute force attack execution and detection
- MITRE ATT&CK framework application across 7 techniques
- Windows Event Log analysis (Event IDs 4624, 4625, 4720, 4722, 4724, 4726, 4738, 4798)
- Sysmon operational log analysis (EID 1, 7)
- Detection gap identification and remediation recommendations
- Real-world troubleshooting (OS edition issues, NLA compatibility, DNS configuration)
- Technical documentation and portfolio development

---

## Repository Structure

```
Active-Directory-Home-Lab/
├── README.md
├── network-diagram.png
── docs/
│   ├── 01-network-and-virtualbox-setup.md
│   ├── 02-vm-installations.md
│   ├── 03-splunk-sysmon-setup.md
│   ├── 04-active-directory-config.md
│   └── 05-attack-simulation.md
├── screenshots/
│   └── (all evidence screenshots)
└── configs/
    ├── inputs.conf
    └── sysmonconfig.xml
```

*Built by Aswin*
