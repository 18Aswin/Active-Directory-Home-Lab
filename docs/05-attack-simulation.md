# Part 5 — Attack Simulation & Detection

## Objective
Simulate multiple attacks from Kali Linux and via Atomic Red Team targeting the domain user EWhite on the Windows 11 machine, detect them in Splunk using Windows Event IDs, and analyze telemetry across the MITRE ATT&CK framework.

---

## Phase 1 — RDP Brute Force with Hydra

**Hydra**, which handles NLA more reliably than Crowbar:
```bash
sudo apt-get install hydra -y
```

### Create Password Wordlist

```bash
mkdir -p ~/Desktop/AD-Project
cd ~/Desktop/AD-Project
nano passwords.txt
```

Contents of `passwords.txt` (sample of common passwords + actual target password):
```
123456
12345
123456789
password
iloveyou
Password123!
princess
1234567
rockyou
12345678
abc123
...
```

> The actual password (`Password123!`) was included in the list to simulate a successful brute force.

### Run the Attack

```bash
hydra -l EWhite -P ~/Desktop/AD-Project/passwords.txt rdp://192.168.10.100
```

### Attack Output

```
[INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
[WARNING] the rdp module is experimental...
[DATA] attacking rdp://192.168.10.100:3389/
[ATTEMPT] target 192.168.10.100 - login "EWhite" - pass "123456"
[ATTEMPT] target 192.168.10.100 - login "EWhite" - pass "12345"
[ATTEMPT] target 192.168.10.100 - login "EWhite" - pass "123456789"
[ATTEMPT] target 192.168.10.100 - login "EWhite" - pass "password"
...
[3389][rdp] host: 192.168.10.100   login: EWhite   password: Password123!
1 of 1 target successfully completed, 1 valid password found
```

**Attack succeeded.** EWhite's credentials were brute forced via RDP.

---

## Phase 2 — Detecting the RDP Brute Force in Splunk

Opened Splunk at `http://192.168.10.10:8000` → Search & Reporting.

### Search 1 — Failed Login Attempts (Event ID 4625)

```
index=endpoint EventCode=4625
```

**Result:** Multiple events appeared with timestamps within seconds of each other — the classic signature of a brute force attack. Each failed password attempt in Hydra generated a 4625 event.

**Expanded event showed:**
- `TargetUserName`: EWhite
- `IpAddress`: 192.168.10.250 (Kali Linux)
- `LogonType`: 3 (Network logon via RDP)

### Search 2 — Successful Login (Event ID 4624)

```
index=endpoint EventCode=4624
```

**Result:** A 4624 event appeared after the flood of 4625s — confirming the exact moment the brute force succeeded and EWhite's account was accessed.

### Search 3 — Combined View

```
index=endpoint EventCode=4625 OR EventCode=4624
```

This showed the full attack timeline — failed attempts followed by the successful login — in a single view.

### Key Detection Indicators (IOCs)

| Indicator | Value |
|---|---|
| Attacker IP | 192.168.10.250 |
| Target user | EWhite |
| Failed logons (4625) | Multiple in rapid succession |
| Successful logon (4624) | Immediately after the flood |
| Logon type | 3 (Network/RDP) |
| Source workstation | Kali VM |

---

## Phase 3 — Atomic Red Team Setup

Atomic Red Team allows testing MITRE ATT&CK techniques to see which ones Splunk can detect.

### Prerequisites — Allow PowerShell Script Execution

On **Windows 11 (Target-PC)**, open PowerShell as Administrator:

```powershell
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
```

### Add Windows Defender Exclusion

To prevent Defender from blocking the installation:

1. Windows Security → Virus & threat protection → Manage Settings
2. Exclusions → Add or remove exclusions
3. Add an exclusion → Folder → select `C:\`

### Install Atomic Red Team

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing);
Install-AtomicRedTeam -getAtomics
```

> **Note:** Internet was required for this step. DNS was temporarily changed to `8.8.8.8` for the download, then switched back to `192.168.10.20` afterward.

Typed `Y` when prompted. Installation downloaded all atomic test modules.

**Verified installation:**

File Explorer → `C:\AtomicRedTeam\atomics\` — folder present with all technique folders

> **Important:** The Atomic Red Team module must be imported at the start of every new PowerShell session:
> ```powershell
> Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
> ```

---

## Phase 4 — T1136.001 (Create Local Account)

This technique simulates an attacker creating a local user account to maintain persistence on a compromised machine.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1136/001/

### Run the Test

```powershell
Invoke-AtomicTest T1136.001
```

Atomic Red Team created a new local user (`NewLocalUser`), then automatically cleaned it up after the test.

### Detecting T1136.001 in Splunk

```
index=endpoint NewLocalUser
```

**Result: 12 events detected**

### Event Codes Triggered

| EventCode | Count | Description |
|---|---|---|
| 4798 | 4 (40%) | User's local group membership was enumerated |
| 4738 | 2 (20%) | A user account was changed |
| 4720 | 1 (10%) | A new user account was **created** ← primary indicator |
| 4722 | 1 (10%) | A user account was enabled |
| 4724 | 1 (10%) | An attempt to reset account password |
| 4726 | 1 (10%) | A user account was **deleted** (ART cleanup) |

### Analysis

The complete lifecycle of the attack was captured:
1. **4720** — attacker created NewLocalUser
2. **4722** — account was enabled
3. **4724** — password was set
4. **4798** — group membership was queried
5. **4738** — account properties were modified
6. **4726** — ART cleanup deleted the account

> **Comparison with reference tutorial:** The MyDFIR video showed zero events for this test, presenting it as a detection gap. This lab detected all 12 events, demonstrating that the **SwiftOnSecurity Sysmon configuration provides complete coverage** for this technique — a meaningfully better security posture.

---

## Phase 5 — T1059.001 (PowerShell Execution)

This technique simulates an attacker executing malicious commands via PowerShell.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1059/001/

### Run the Test

```powershell
Invoke-AtomicTest T1059.001
```

### Detecting T1059.001 in Splunk

```
index=endpoint technique_id=T1059
```

**Result: 524 events detected in 15 minutes**

### Analysis

The high event count reflects the broad range of PowerShell activity that Sysmon and the Windows event log captured — script block logging, process creation, and command-line argument logging all contributed to the telemetry.

---

## Phase 6 — T1087.001 (Local Account Discovery)

This technique simulates an attacker enumerating local accounts and groups to understand the environment post-compromise.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1087/001/

### Run the Test

```powershell
Invoke-AtomicTest T1087.001
```

### Detecting T1087.001 in Splunk

```
index=endpoint technique_id=T1087
```

**Result: Detected — Sysmon Event ID 1 (Process Creation)**

### Key Detection Details

| Field | Value |
|---|---|
| Event ID | 1 (Sysmon — Process Creation) |
| Process | cmdkey.exe (PID 4128) |
| Parent Process | powershell.exe |
| User | ASWINAD\Administrator |
| Host | Target-PC.aswinAD.local |

### Commands Executed by Atomic Red Team

```
net user
get-localuser
get-localgroupmember -group Users
cmdkey.exe /list
ls C:/Users
get-childitem C:\Users\
dir C:\Users\
get-localgroup
net localgroup
```

These are all classic local account and group enumeration commands an attacker runs during the Discovery phase.

---

## Phase 7 — T1003.001 (LSASS Credential Dump)

This technique simulates an attacker dumping credentials from LSASS memory.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1003/001/

### Run the Test

```powershell
Invoke-AtomicTest T1003.001
```

### Detecting T1003.001 in Splunk

```
index=endpoint technique_id=T1003
```

**Result: 3 events detected**

### Event Codes Triggered

| Event ID | Description |
|---|---|
| 1 (Sysmon) | Process Creation — `xordump.exe` launched by `powershell.exe` |

### Key Detection Details

| Field | Value |
|---|---|
| Event ID | 1 (Sysmon — Process Creation) |
| Process | xordump.exe |
| Parent Process | powershell.exe |
| User | ASWINAD\Administrator |
| Host | Target-PC.aswinAD.local |
| Action | Dumped LSASS memory to `C:\Windows\Temp\lsass-xordump.t1003.001.dmp` |

### Analysis

Sysmon captured the full command line used to dump LSASS memory, including the output file path and process ID reference. The `xordump.exe` binary (an open-source LSASS dumping tool) was spawned directly by PowerShell, making parent-child process relationship a strong detection signal. The dump file path itself (`lsass-xordump.t1003.001.dmp`) is a high-fidelity indicator of credential dumping activity.

---

## Phase 8 — T1053.005 (Scheduled Task / Persistence)

This technique simulates an attacker creating a scheduled task to maintain persistence on a compromised machine.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1053/005/

### Run the Test

```powershell
Invoke-AtomicTest T1053.005
```

### Detecting T1053.005 in Splunk

```
index=endpoint technique_id=T1053
```

**Result: 81 events detected**

### Event Codes Triggered

| Event ID | Description |
|---|---|
| 7 (Sysmon) | Image Loaded — `schtasks.exe` loaded `taskschd.dll` (Task Scheduler COM API) |
| 1 (Sysmon) | Process Creation — `schtasks.exe` created with full command-line arguments |

### Key Detection Details

| Field | Value |
|---|---|
| Primary Process | schtasks.exe (PID 10784) |
| Parent Process | cmd.exe → powershell.exe |
| User | ASWINAD\Administrator |
| Host | Target-PC.aswinAD.local |
| Task Name | EventViewerBypass |
| Trigger | ONLOGON, RL HIGHEST |
| Image Loaded | `taskschd.dll` (Task Scheduler COM API) |

### Commands Executed by Atomic Red Team

```
schtasks /Run /TN "EventViewerBypass"
schtasks /Create /TN "EventViewerBypass" /TR "eventvwr.msc" /SC ONLOGON /RL HIGHEST
cmd.exe /c reg add "HKCU\...\mscfile\shell\open\command" /ve /t REG_EXPAND_SZ /d "c:\windows\system32\calc.exe"
```

### Analysis

The high event count (81) reflects the breadth of Task Scheduler activity captured by Sysmon — both `Image Load` (EID 7) and `Process Creation` (EID 1) events. The `EventViewerBypass` task name is a known UAC bypass technique. The combination of `schtasks.exe` spawned from `cmd.exe` under `powershell.exe`, with `ONLOGON` and `RL HIGHEST` flags, is a strong persistence indicator.

---

## Phase 9 — T1562.001 (Disable Windows Defender)

This technique simulates an attacker disabling Windows Defender to evade detection.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1562/001/

### Run the Test

```powershell
Invoke-AtomicTest T1562.001
```

### Result — Not Detected

```
index=endpoint technique_id=T1562
```

**Result: 0 events**

### Analysis

Sub-test T1562.001-58 ("Freeze PPL-protected process with EDR-Freeze") failed with Exit code 1 because it required downloading `EDR-Freeze_1.0.zip` from the internet, which was unavailable in the lab environment. As a result, no meaningful activity was executed and no Sysmon registry modification events (Event ID 13) were generated.

A secondary search also returned no results:
```
index=endpoint EventCode=13 TargetObject="*DisableRealtimeMonitoring*"
```

> **Finding — Detection Gap:** T1562.001 was not detected by the current Splunk/Sysmon configuration, indicating a detection gap for Defense Evasion techniques. Recommended improvement: deploy custom Sysmon rules targeting `DisableRealtimeMonitoring` registry key modifications to close this gap.

> **Note:** After running this test, re-enable Defender manually:
> ```powershell
> Set-MpPreference -DisableRealtimeMonitoring $false
> ```

---

## Summary of All Attacks

| # | Attack | Tactic | Tool | Result |
|---|---|---|---|---|
| 1 | RDP Brute Force | Initial Access | Hydra | Detected — EID 4625 + 4624 |
| 2 | T1136.001 Create Local Account | Persistence | Atomic Red Team | 12 events, 6 Event IDs |
| 3 | T1059.001 PowerShell Execution | Execution | Atomic Red Team | 524 events |
| 4 | T1087.001 Local Account Discovery | Discovery | Atomic Red Team | Detected — Sysmon EID 1 |
| 5 | T1003.001 LSASS Credential Dump | Credential Access | Atomic Red Team | 3 events — Sysmon EID 1 (xordump.exe) |
| 6 | T1053.005 Scheduled Task | Persistence | Atomic Red Team | 81 events — Sysmon EID 1 + EID 7 |
| 7 | T1562.001 Disable Defender | Defense Evasion | Atomic Red Team | Not Detected — detection gap identified |

---

## Key Findings

1. **Detection gap identified for T1562.001** — Defense Evasion techniques require additional Sysmon tuning.
2. **High-volume telemetry from T1059.001** — 524 events in 15 minutes highlights the need for alert tuning to reduce noise.

---
