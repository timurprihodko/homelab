# Windows Server Setup

## Environment

- OS: Windows Server 2019 (Evaluation, Server Core)
- RAM: 4096 MB
- Disk: 50 GB
- Hypervisor: VirtualBox
- Network: NAT (college) / Bridged (home)

## Goals

- [ ] Windows Server installation
- [ ] Active Directory Domain Services (AD DS)
- [ ] DNS
- [ ] DHCP
- [ ] GPO
- [ ] Domain join — Ubuntu

---

## Log

### 1. Windows Server Installation

*28.05.2026*

Downloaded Windows Server 2019 Evaluation ISO. Created VM in VirtualBox.
Server Core (no GUI). Hostname: DC01.

Server IP: 10.0.2.15 (NAT, college)

### 2. Active Directory Domain Services

*_.05.2026*

Installed AD DS role via Server Manager. Promoted server to Domain Controller.

- Domain name: `homelab.local`
- Forest/Domain functional level: Windows Server 2016
- DNS role installed automatically during promotion.

Result: DC promoted, server rebooted, domain `homelab.local` active.

### 3. DHCP Server

*_.05.2026*

Installed DHCP Server role. Created scope:

- Range: `192.168.0.100 – 192.168.0.200`
- Subnet mask: `255.255.255.0`
- Default gateway: `192.168.0.1`
- DNS: `192.168.0.x` (DC)

Authorized DHCP server in AD.

Result: clients receive addresses from scope.

### 4. GPO — Basic Policies

*_.05.2026*

Created GPOs via Group Policy Management Console (GPMC):

| GPO | Scope | Setting |
|-----|-------|---------|
| Password Policy | Default Domain Policy | Min length: 10, complexity: enabled |
| Desktop Restrictions | OU=Clients | Disable Control Panel |
| Drive Map | OU=Clients | Map \\server\share as Z: |

Result: policies applied, verified via `gpresult /r` on client.

### 5. Domain Join — Windows 10/11 Client

*_.05.2026*

Client machine: `192.168.0.x` (DHCP).
Joined domain `homelab.local` via Settings → System → Domain.

Result: login with domain credentials confirmed. GPO applied.
