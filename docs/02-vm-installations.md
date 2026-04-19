# Part 2 — Virtual Machine Installations

## Objective
Install and configure all 4 virtual machines: Windows Server 2022, Windows 11 Enterprise, Ubuntu Server (Splunk), and Kali Linux.

---

## VM 1 — Windows Server 2022 (Domain Controller)

**Download:** https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022

### VirtualBox Settings

| Setting | Value |
|---|---|
| Name | AD-SERVER01 |
| Type | Microsoft Windows |
| Version | Windows 2022 (64-bit) |
| RAM | 4096 MB |
| Storage | 50 GB (VDI, dynamically allocated) |
| CPU | 2 cores |
| Network Adapter 1 | NAT Network → AD-Project |

**General → Advanced:**
- Shared Clipboard: Bidirectional
- Drag n Drop: Bidirectional

### Installation Steps
1. Booted from Windows Server 2022 ISO
2. Selected: **Windows Server 2022 Standard Evaluation (Desktop Experience)**
3. Chose Custom install → selected virtual disk
4. Set Administrator password after installation

### Post-Installation: Rename Server
1. Server Manager → Local Server → click computer name
2. Change → set name to `AD-SERVER01`
3. Restarted

### Static IP Configuration
Network icon (taskbar) → Open Network & Internet Settings → Change adapter options → right-click adapter → Properties → IPv4:

```
IP Address:       192.168.10.20
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.10.1
Preferred DNS:    8.8.8.8
```

> DNS was later changed to `127.0.0.1` (loopback) after Active Directory was installed, so the DC uses itself for DNS resolution.

---

## VM 2 — Windows 11 Enterprise Evaluation (Target Machine)

### Why Windows 11 Enterprise Instead of Windows 10

Used Windows 11 instead of Windows 10 for more realistic enterprise scanario.

**Download:** https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise

### VirtualBox Settings

| Setting | Value |
|---|---|
| Name | Target-PC |
| Type | Microsoft Windows |
| Version | Windows 11 (64-bit) |
| RAM | 4096 MB |
| Storage | 50 GB |
| CPU | 2 cores |
| Network Adapter 1 | NAT Network → AD-Project |
| System → Motherboard | Enable EFI: |
| System → TPM | TPM 2.0 (required for Windows 11) |

### Installation Steps
1. Booted from Windows 11 Enterprise ISO
2. Proceeded through setup — edition was auto-selected as Enterprise (single-edition ISO)
3. When asked for Microsoft account → clicked **"I don't have internet"** → **"Continue with limited setup"**
4. Created local user: `aswin`
5. Turned off all privacy settings

### Verified Edition
Settings → System → About:
```
Edition: Windows 11 Enterprise Evaluation
Version: 25H2
```

### Static IP Configuration
Settings → Network & Internet → Ethernet → Edit:

```
IP Address:       192.168.10.100
Subnet Mask:      255.255.255.0
Gateway:          192.168.10.1
Preferred DNS:    8.8.8.8   (temporary — changed to 192.168.10.20 for domain join)
```

> **Note:** DNS was set to `8.8.8.8` during tool installations to allow internet access, then changed to `192.168.10.20` (DC) when joining the domain.

### Connectivity Test
```powershell
ping 192.168.10.1    # Gateway — confirmed working
ping 192.168.10.20   # Domain Controller — confirmed working after firewall rule
```

**Firewall rule added on Windows Server to allow ICMP:**
```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

---

## VM 3 — Ubuntu Server 22.04 (Splunk Server)

**Download:** https://ubuntu.com/download/server

### VirtualBox Settings

| Setting | Value |
|---|---|
| Name | Splunk-Server |
| Type | Linux → Ubuntu (64-bit) |
| RAM | 8192 MB (Splunk requires this) |
| Storage | 100 GB |
| CPU | 2 cores |
| Network Adapter 1 | NAT Network → AD-Project |

### Installation Steps
1. Booted from Ubuntu Server 22.04 ISO
2. Selected: Ubuntu Server (not minimized)
3. Left network as default during install
4. Set username: `aswin` / server name: `splunk`
5. Checked  **Install OpenSSH server**
6. Skipped extra snaps → completed install

### Static IP Configuration
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.10.10/24]
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.10.1
  version: 2
```

```bash
sudo netplan apply
ip a    # verified 192.168.10.10 assigned
```

---

## VM 4 — Kali Linux (Attacker)

**Download:** https://www.kali.org/get-kali/#kali-virtual-machines (VirtualBox .ova image)

### Import Steps
1. VirtualBox → File → Import Appliance
2. Selected the `.ova` Kali file → Import
3. After import: Settings → Network → Adapter 1 → NAT Network → `AD-Project`

### Login
Default credentials: `kali` / `kali`

### Static IP Configuration
```bash
sudo nano /etc/network/interfaces
```

Added:
```
auto eth0
iface eth0 inet static
  address 192.168.10.250
  netmask 255.255.255.0
  gateway 192.168.10.1
  dns-nameservers 8.8.8.8
```

```bash
sudo systemctl restart networking
ip a    # verified 192.168.10.250 assigned
```

---

## Network Connectivity Verification

After all VMs were configured, tested inter-VM connectivity:

| From | To | IP | Result |
|---|---|---|---|
| Windows 11 | Gateway | 192.168.10.1 | 0% loss |
| Windows 11 | Windows Server | 192.168.10.20 | After ICMP firewall rule |
| Kali | Windows 11 | 192.168.10.100 | Confirmed |
| Kali | Splunk Server | 192.168.10.10 | Confirmed |

---

## Next Step
[Part 3 → Splunk & Sysmon Setup](03-splunk-sysmon-setup.md)
