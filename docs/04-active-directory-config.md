# Part 4 — Active Directory Configuration

## Objective
Install Active Directory Domain Services on Windows Server 2022, promote it to a Domain Controller, create Organizational Units and domain users, and join the Windows 11 machine to the domain.

---

## Install Active Directory Domain Services

On **Windows Server 2022 (AD-SERVER01)**:

1. Open **Server Manager** → Add Roles and Features
2. Role-based installation → select local server
3. Check **Active Directory Domain Services**
4. Click **Add Features** in the popup
5. Next through all screens → **Install**
6. Waited for installation to complete

---

## Promote to Domain Controller

After AD DS installed, a warning flag appeared in Server Manager:

1. Clicked the flag → **Promote this server to a domain controller**
2. Selected: **Add a new forest**
3. Root domain name: `aswinAD.local`
4. Clicked Next → Set DSRM (Directory Services Restore Mode) password
5. Clicked Next through all remaining screens → **Install**
6. Server automatically restarted

After restart, the login screen showed: `ASWINAD\Administrator` — confirming the domain was live.

---

## Create Organizational Units

Opened: Server Manager → Tools → **Active Directory Users and Computers**

Right-clicked `aswinAD.local` → New → Organizational Unit:

| OU Name | Purpose |
|---|---|
| `Aswin_IT` | IT Department users |
| `Aswin_Infosec` | Information Security users |

---

## Create Domain Users

### User 1 — Emelyn White (IT Department)

Right-clicked **Aswin_IT** OU → New → User:

```
First name:   Emelyn
Last name:    White
Username:     EWhite
Password:     Password123!
```

Settings:
- Unchecked "User must change password at next logon"
- Checked "Password never expires" (lab environment only)

### User 2 — Aswin AN (Infosec Department)

Right-clicked **Aswin_Infosec** OU → New → User:

```
First name:   Aswin
Last name:    AN
Username:     AswinAN
Password:     Password123!
```

---

## Join Windows 11 to the Domain

### Step 1 — Change DNS to point to Domain Controller

On **Windows 11 (Target-PC)**:

Settings → Network & Internet → Ethernet → Edit IP:
```
Preferred DNS: 192.168.10.20   ← must point to DC for domain join
```

### Step 2 — Join the domain

1. Settings → System → About → **Domain or workgroup** (or search "This PC" → Properties)
2. Advanced system settings → **Computer Name** tab → **Change**
3. Selected: **Domain** → typed `aswinAD.local`
4. Clicked OK → entered credentials: `ASWINAD\Administrator` + password
5. Popup appeared: **"Welcome to the aswinAD.local domain"** 
6. Restarted

### Step 3 — Login as domain user

After restart, at login screen:
- Clicked **Other user**
- Logged in as: `aswinAD\EWhite` / `Password123!`

Domain login worked successfully.

---

## Enable Remote Desktop on Windows 11

Required for the brute force attack simulation.

### Via Settings
Settings → System → Remote Desktop → toggle **On**

### Add domain users to RDP access
1. Click **Select users**
2. Add → type `EWhite` → Check Names → OK
3. Also added `AswinAN`

### Via PowerShell (also run for reliability)
```powershell
# Allow RDP connections
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

# Disable NLA (required for Hydra/Crowbar compatibility with Windows 11)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0

# Open firewall port 3389
netsh advfirewall firewall add rule name="Allow RDP" protocol=TCP dir=in localport=3389 action=allow

# Add EWhite to Remote Desktop Users group
net localgroup "Remote Desktop Users" "aswinAD.local\EWhite" /add
```

### Verify RDP Port Open
Run from Kali:
```bash
nmap -p 3389 192.168.10.100
```

Output:
```
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
```

RDP confirmed open. ✅

---

## Active Directory Structure Summary

```
aswinAD.local
├── Aswin_IT
│   └── EWhite (Emelyn White)
├── Aswin_Infosec
│   └── AswinAN (Aswin AN)
├── Domain Controllers
│   └── AD-SERVER01
└── Computers
    └── TARGET-PC
```

---

## Next Step
[Part 5 → Attack Simulation & Detection](05-attack-simulation.md)
