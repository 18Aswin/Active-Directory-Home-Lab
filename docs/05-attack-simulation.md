# Part 5 — Attack Simulation & Detection

## Objective
Simulate an RDP brute force attack from Kali Linux targeting the domain user EWhite on the Windows 11 machine, detect it in Splunk using Windows Event IDs, then execute MITRE ATT&CK technique T1136.001 using Atomic Red Team and analyze the telemetry.

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

## Phase 2 — Detecting the Attack in Splunk

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

---

## Phase 4 — Execute T1136.001 (Create Local Account)

This technique simulates an attacker creating a local user account to maintain persistence on a compromised machine.

**MITRE ATT&CK Reference:** https://attack.mitre.org/techniques/T1136/001/

### Run the Test

```powershell
Invoke-AtomicTest T1136.001
```

Atomic Red Team created a new local user (`NewLocalUser`), then automatically cleaned it up after the test.

---

## Phase 5 — Detecting T1136.001 in Splunk

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

## Summary of Detections

| Attack | Tool Used | Splunk Result |
|---|---|---|
| RDP Brute Force | Hydra | Detected via Event IDs 4625 + 4624 |
| Create Local Account (T1136.001) | Atomic Red Team | 12 events, 6 event codes |

---

## Next Step
[Deviations from Tutorial →](06-deviations-from-tutorial.md)
