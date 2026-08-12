# 🏢 Integrated AD + Network Lab — VMware Workstation

> **Active Directory DS (DC01 + RODC01) + pfSense firewall — end-to-end design & deployment**

[![Status](https://img.shields.io/badge/Status-Validated-success?style=for-the-badge)]()
[![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation%20Pro-0078D6?style=for-the-badge&logo=vmware&logoColor=white)]()
[![OS](https://img.shields.io/badge/Windows%20Server-2022-0078D6?style=for-the-badge&logo=windows&logoColor=white)]()
[![Firewall](https://img.shields.io/badge/pfSense-CE-212121?style=for-the-badge&logo=pfsense&logoColor=white)]()
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)]()
[![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen?style=for-the-badge)]()
[![Made%20With](https://img.shields.io/badge/Made%20with-Markdown-1f425f?style=for-the-badge&logo=markdown&logoColor=white)]()
[![PRs%20Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=for-the-badge)]()

---

## 📑 Table of Contents

1. [🎯 Lab Objective](#-lab-objective)
2. [🧰 Software & Components](#-software--components)
3. [🌐 Network Architecture](#-network-architecture)
4. [⚙️ Detailed Configuration](#-detailed-configuration)
5. [✅ Validation](#-validation)
6. [🚀 Next Steps](#-next-steps)
7. [🖼️ Screenshots](#-screenshots)
8. [📂 Repository Structure](#-repository-structure)

---

## 🎯 Lab Objective

Build a fully functional identity infrastructure for a simulated SME:

- 🖥️ **Windows Server 2019** Domain Controller (DC01) hosting **AD DS**, **DNS** and **DHCP**.
- 🛡️ **RODC01** (Read-Only Domain Controller) in the user segment for branch-office style replication.
- 🔥 **pfSense CE** firewall/router isolating the **server** and **user** segments.
- 💻 Client workstations join the domain and apply GPOs in **< 90 seconds**.

> ⚠️ **Critical dependency (AD ↔ DNS):** A domain logon session is impossible if the DNS service is stopped.
> The AD SRV records (`_ldap._tcp.dc._msdcs`) are essential for authentication. The DC must reference **itself** as the preferred DNS server.

---

## 🧰 Software & Components

| Component | Software / Version |
|---|---|
| Hypervisor | VMware Workstation Pro |
| Firewall / Router | pfSense CE |
| Domain Controller | Windows Server 2019 |
| Read-Only DC | Windows Server 2019 (RODC) |
| ISO Images | Microsoft Volume Licensing Portal |
| Documentation | Markdown + Mermaid + Pandoc |

---

## 🌐 Network Architecture

### 🗺️ Topology (ASCII art)

```
                              ┌──────────────────────────┐
                              │   🌐  Internet / Host    │
                              │       (WAN – NAT)        │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │    🛡️  pfSense CE        │
                              │  ┌──────┐    ┌──────┐     │
                              │  │ WAN  │    │ NAT  │     │
                              │  └──────┘    └──────┘     │
                              └──┬────────────────────┬───┘
                  LAN 10.10.20.1/24│           │OPT1 10.10.10.1/24
                                 ▼             ▼
       ┌──────────────────────────┐   ┌──────────────────────────┐
       │  🏢  SEG_SRV (Servers)   │   │  👥  SEG_USR (Users)     │
       │       10.10.20.0/24      │   │       10.10.10.0/24      │
       │                          │   │                          │
       │  ┌────────────────────┐  │   │  ┌────────────────────┐  │
       │  │ 🖥️  DC01           │  │   │  │ 🔒  RODC01         │  │
       │  │ 10.10.20.10        │  │   │  │ 10.10.10.20        │  │
       │  │ AD DS / DNS / DHCP │  │   │  │ Read-Only DC       │  │
       │  └────────────────────┘  │   │  └────────────────────┘  │
       │                          │   │                          │
       │  ┌────────────────────┐  │   │  ┌────────────────────┐  │
       │  │ 🖥️  SRV02 (opt.)   │  │   │  │ 💻  CLI01..CLI10   │  │
       │  │ 10.10.20.20        │  │   │  │ DHCP from pfSense  │  │
       │  └────────────────────┘  │   │  └────────────────────┘  │
       └──────────────────────────┘   └──────────────────────────┘
```

### 📊 IP Addressing Table

| Device | Role | IP / Mask | Gateway | Preferred DNS |
|---|---|---|---|---|
| pfSense | WAN (NAT) | DHCP / auto | — | — |
| pfSense | LAN (Servers) | `10.10.20.1 / 24` | `10.10.20.1` | `127.0.0.1` |
| pfSense | OPT1 (Users) | `10.10.10.1 / 24` | `10.10.10.1` | `127.0.0.1` |
| DC01 | AD / DC / DNS / DHCP | `10.10.20.10 / 24` | `10.10.20.1` | **`10.10.20.10`** |
| RODC01 | Read-Only DC | `10.10.10.20 / 24` | `10.10.10.1` | `10.10.10.20` |

### 🔌 VMware Segmentation

| VMnet | Network | Mode | VMware DHCP |
|---|---|---|---|
| VMnet1 | `10.10.10.0/24` | Host-only | ❌ Disabled |
| VMnet2 | `10.10.20.0/24` | Host-only | ❌ Disabled |

> ℹ️ DHCP is delegated to **pfSense** on the LAN and OPT1 interfaces.

---

## ⚙️ Detailed Configuration

### 🛡️ pfSense CE

**Interfaces:**

| Interface | Network | Role |
|---|---|---|
| WAN | NAT (bridge) | Internet / Host access |
| LAN | `10.10.20.1/24` | Server segment |
| OPT1 | `10.10.10.1/24` | User segment |

**Active services:** DHCP Server (LAN/OPT1) · NAT · DNS Resolver → DC01
**Web UI:** https://10.10.20.1

### 🖥️ DC01 — Windows Server 2022 (Full DC)

**Static IP configuration:**

```
IPv4       : 10.10.20.10
Subnet mask: 255.255.255.0 (/24)
Gateway    : 10.10.20.1
DNS        : 10.10.20.10   ← SELF-REFERENCED (critical for AD)
```

**Installed roles:**

| Role | Description |
|---|---|
| AD DS | Active Directory Domain Services — central directory |
| DNS | DNS server with AD-integrated zones |
| DHCP | Automatic IP address assignment to clients |

### 🔒 RODC01 — Read-Only Domain Controller

A second Windows Server 2022 acts as a **Read-Only DC** placed in the **user segment** (`10.10.10.0/24`). It:

- Replicates the AD database from DC01 (one-way, read-only).
- Serves authentication requests for branch-office users without WAN round-trips.
- Is ideal for sites where physical security is reduced (no writable copy of AD held locally).

**Static IP configuration:**

```
IPv4       : 10.10.10.20
Subnet mask: 255.255.255.0 (/24)
Gateway    : 10.10.10.1
DNS        : 10.10.10.20   ← self-reference (consistent pattern)
```

**PowerShell — promote as RODC:**

```powershell
# 1. Rename and join to the domain (run on the future RODC)
Rename-Computer -NewName "RODC01" -Restart
Add-Computer -DomainName "corp.lan" -Credential (Get-Credential) -Restart

# 2. Install the AD DS role + management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# 3. Promote as a Read-Only Domain Controller
Install-ADDSDomainController `
    -DomainName "corp.lan" `
    -Credential (Get-Credential) `
    -SiteName "Default-First-Site-Name" `
    -InstallDns:$true `
    -ReadOnlyReplica:$true `
    -Force
```

### ✅ PowerShell — Validation

```powershell
# Confirm DC01 promotion
Get-ADDomainController -Identity DC01 | Format-List Name,IPv4Address,IsGlobalCatalog,OperatingSystem

# Confirm RODC01 promotion (IsGlobalCatalog must be True)
Get-ADDomainController -Identity RODC01 | Format-List Name,IPv4Address,IsGlobalCatalog,OperatingSystem

# Verify AD SRV records (DNS critical dependency)
nslookup -type=srv _ldap._tcp.dc._msdcs.corp.lan 10.10.20.10

# Replication health (DC01 ↔ RODC01)
repadmin /replsummary
repadmin /showrepl
```

### 💻 pfSense DHCP reservation (PowerShell-style management via GUI)

Add static mappings for `DC01` (MAC → `10.10.20.10`) and `RODC01` (MAC → `10.10.10.20`)
in **Status → DHCP Leases → Add Static Mapping**.

---

## ✅ Validation

- [x] VMware networks isolated (host-only)
- [x] pfSense: DHCP active on LAN/OPT1, WAN in NAT
- [x] DC01: static IP, DNS points to itself
- [x] RODC01: promoted as Read-Only DC, AD replication from DC01 verified
- [x] Bidirectional connectivity confirmed (ping, nslookup)
- [x] Dynamic resolution of AD SRV records
- [x] Experimentally validated: domain logon fails when DNS is stopped

---



---

## 🖼️ Screenshots

| # | Caption |
|---|---|
| Fig. 1 | VMware Network Editor (VMnet1 / VMnet2) |
| Fig. 2 | `ipconfig` on DC01 (static IP + self-referenced DNS) |
| Fig. 3 | pfSense web interface (`https://10.10.20.1`) |
| Fig. 4 | Network captures (`ping`, `nslookup`) |

> Screenshots are stored under `docs/screenshots/`.

---

## 📂 Repository Structure

```
.
├── README.md                          ← this file
├── docs/
│   ├── Documentation_Lab_AD_RODC_pfSense.docx   ← full Word documentation
│   ├── Documentation_Lab_AD_RODC_pfSense.pdf    ← exported PDF
│   ├── mermaid/
│   │   └── topology.mmd
│   └── screenshots/
│       ├── vmware-network-editor.png
│       ├── dc01-ipconfig.png
│       ├── pfsense-web.png
│       └── network-captures.png
├── scripts/
│   ├── powershell/
│   │   ├── 01-install-ad-ds.ps1
│   │   ├── 02-promote-dc01.ps1
│   │   ├── 03-promote-rodc01.ps1
│   │   ├── 04-validate-replication.ps1
│   │   └── 05-create-gpo-wallpaper.ps1
│   └── pfsense/
│       └── dhcp-static-mappings.xml
├── diagrams/
│   ├── topology.mermaid
│   └── topology.png
└── .github/
    └── ISSUE_TEMPLATE/
        └── bug_report.md
```



<p align="center">
  <sub>🛡️ Built with ❤️ — keep your DNS pointed at your DC.</sub>
</p>
