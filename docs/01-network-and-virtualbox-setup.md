# Part 1 — Network Design & VirtualBox Setup

## Objective
Plan the lab architecture visually using draw.io, install VirtualBox, and create the shared NAT Network that all 4 virtual machines will communicate on.

---

## Lab Architecture Planning

Before touching any software, the network was planned on [draw.io](https://app.diagrams.net).

### Network Design Decisions
- All VMs placed on a single NAT Network (`192.168.10.0/24`) to allow inter-VM communication
- Static IPs assigned to all machines to prevent DHCP reassignment issues
- Kali Linux placed on `.250` to clearly distinguish attacker from victim machines
- Splunk server assigned `.10` for easy recall
- Domain Controller assigned `.20`
- Target machine assigned `.100`

### Final IP Scheme

| Machine | IP Address | Role |
|---|---|---|
| Windows Server 2022 | 192.168.10.20 | Domain Controller |
| Windows 11 Enterprise | 192.168.10.100 | Target Endpoint |
| Ubuntu (Splunk) | 192.168.10.10 | SIEM |
| Kali Linux | 192.168.10.250 | Attacker |
| Gateway | 192.168.10.1 | NAT Gateway |

---

## VirtualBox Installation

**Download:** https://www.virtualbox.org/wiki/Downloads

Steps:
1. Downloaded VirtualBox installer for Windows host
2. Ran installer with default settings
3. Accepted the network interface warning (brief internet disconnection is expected)
4. Completed installation and launched VirtualBox

---

## Creating the NAT Network

All VMs need to communicate with each other. This requires a shared NAT Network in VirtualBox (not the default NAT which isolates each VM).

**Steps:**
1. VirtualBox → File → Tools → Network Manager
2. Click the **NAT Networks** tab
3. Click **Create**
4. Settings:
   - Name: `AD-Project`
   - IPv4 Prefix: `192.168.10.0/24`
   - Enable DHCP: checked
5. Apply → OK

> All 4 VMs were later configured to use this same NAT Network (`AD-Project`) so they can communicate with each other.

---

## Hardware Constraints & RAM Management

**Host machine:** AMD Ryzen 7 6800H, 16 GB RAM, Windows 11

Since all 4 VMs together exceed comfortable RAM limits, a sequential boot strategy was used:

| Phase | VMs Running | RAM Used |
|---|---|---|
| Setup phases | 1 VM at a time | ~2–4 GB |
| Splunk config | Splunk + one Windows VM | ~6–8 GB |
| Attack phase | Kali + Windows 11 + Splunk | ~7–8 GB |
| Windows Server | Off during attack | Saves ~2 GB |

**Key insight:** Windows Server does not need to be running during the brute force attack. Once Windows 11 is already domain-joined, Kali can attack it independently.

---

## Next Step
[Part 2 → VM Installations](02-vm-installations.md)
