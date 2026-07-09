# Homelab — Windows Server & Linux Administration

A self-hosted lab environment for learning and practising IT infrastructure administration.
Built on VirtualBox VMs, documented step-by-step.

---

## Environment

| Component | Details |
|-----------|---------|
| Hypervisor | VirtualBox |
| Network | NAT (college) / Bridged (home) |

## Network Topology

![Network topology](topology.svg)

### Virtual Machines

| VM | OS | RAM | Disk | Role |
|----|----|-----|------|------|
| DC01 | Windows Server 2019 (Server Core) | 6 GB | 50 GB | Domain Controller |
| linux-srv | Ubuntu Server 22.04 LTS | 2 GB | 20 GB | Linux server |

---

> Note: the Linux VM's actual hostname is `linux-server`; `linux-srv` is
> used as a short label in this README and the topology diagram.

## Progress

### Windows Server

| Task | Status |
|------|--------|
| Windows Server 2019 installation (Server Core) | ✅ Done |
| Active Directory Domain Services (AD DS) | ✅ Done |
| DNS | ✅ Done |
| DHCP | ✅ Done |
| Group Policy Objects (GPO) | ✅ Done |
| Domain join — Ubuntu | ✅ Done |
| Shared folders with access control | ✅ Done |

### Linux Server

| Task | Status |
|------|--------|
| Ubuntu Server 22.04 LTS installation | ✅ Done |
| SSH key-based authentication | ✅ Done |
| Domain join (homelab.local) | ✅ Done |
| Static IP configuration | 🔄 In progress |
| Web server (Nginx) | 🔄 Planned |
| File server (Samba/NFS) | 🔄 Planned |

---

## Configuration Details

### Active Directory

- **Domain:** `homelab.local`
- **NetBIOS:** `HOMELAB`
- **DC hostname:** `DC01.homelab.local`
- **Functional level:** Windows Server 2016
- Configured entirely via **PowerShell** (Server Core — no GUI)
- **DNS forwarders:** `8.8.8.8`, `1.1.1.1` (external resolution for domain clients)

### DHCP Scope

- **Range:** `192.168.0.100 – 192.168.0.200`
- **Subnet mask:** `255.255.255.0`
- **Default gateway:** `192.168.0.1`
- **DNS:** `192.168.0.10` (DC)
- **Lease duration:** 8 days

### Group Policy Objects

| GPO | Scope | Setting |
|-----|-------|---------|
| Password Policy | Default Domain Policy | Min length: 10, complexity on, max age: 42d |
| Desktop Restrictions | OU=Clients | Disable Control Panel |
| Drive Map | OU=Clients | Map `\\DC01\share` as `Z:` |

### Shared Folders (AD-based Access Control)
- Structure: `OU=Departments` → Global Security groups HR, IT, Finance
- Folders: `C:\Shares\{HR,IT,Finance}` on DC01
- NTFS ACL: department group → Modify; SYSTEM, Domain Admins → Full; inheritance disabled
- SMB share permission `Everyone: Full` — real restriction via NTFS ACL
- Verified from Ubuntu (`smbclient`): HR member granted on HR share, denied on Finance
  
### SSH (Ubuntu Server)

- Key type: **Ed25519**
- Key generated on Windows host, public key copied to `~/.ssh/authorized_keys`
- Password authentication: **disabled**
- Result: passwordless SSH login from Windows host confirmed

---

## Troubleshooting Log

### DHCP — Access Denied on Authorization

**Problem:** `Add-DhcpServerInDC` failed with `WIN32 5 / Access Denied`  
**Root cause:** User `admin` was not a member of `Domain Admins`  
**Fix:**
```powershell
Add-ADGroupMember -Identity "Domain Admins" -Members admin
```
Full logoff required for Kerberos token refresh.

---

### Domain Join Ubuntu — GSSAPI Failure
**Problem:** `realm join` failed with `GSSAPI Error: Unspecified GSS failure`  
**Root cause:** Multiple issues in sequence:

| Step | Error | Fix |
|------|-------|-----|
| 1 | `Cannot contact any KDC` | `apt install krb5-user` |
| 2 | `Server not found in Kerberos database` | Added PTR record on DC01 for `192.168.0.10` |
| 3 | `GSSAPI: Unspecified GSS failure` | Disabled IPv6 via `sysctl` |
| 4 | Netplan not applying | `chmod 600 /etc/netplan/01-homelab.yaml` |

---

### Two DHCP Servers on the Bridged Network

**Problem:** DHCP reservation on DC01 ignored; Ubuntu's IP changed between reboots  
**Root cause:** the home router (`192.168.0.1`) runs a DHCP server whose pool overlaps the DC01 scope. Both answer; the router wins.  
**Fix:** static IP on the Ubuntu VM, outside both pools. DC01 stays the authorised DHCP server for the lab.

## Technologies Used

`Windows Server 2019` `Active Directory` `DNS` `DHCP` `GPO` `PowerShell`  
`Ubuntu Server 22.04` `OpenSSH` `realmd` `sssd` `Kerberos` `VirtualBox`

---

## Planned (Phase 2)

- pfSense/OPNsense — VLAN, NAT, Firewall
- Monitoring: Zabbix or Grafana + Prometheus
- Network topology diagram (Draw.io)
- PowerShell scripts for bulk AD user creation
