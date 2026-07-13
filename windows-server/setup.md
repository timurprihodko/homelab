# Windows Server Setup

## Environment

- OS: Windows Server 2019 (Evaluation, Server Core)
- RAM: 6144 MB
- Disk: 50 GB
- Hypervisor: VirtualBox
- Network: Bridged (home)

## Goals

- [X] Windows Server installation
- [X] Active Directory Domain Services (AD DS)
- [X] DNS
- [X] DHCP
- [X] GPO
- [X] Domain join — Ubuntu
- [X] Shared folders with AD-based access control

---

## Log

### 1. Windows Server Installation

*28.05.2026*

Downloaded Windows Server 2019 Evaluation ISO. Created VM in VirtualBox.
Server Core (no GUI). Hostname: DC01.

Server IP: 192.168.0.10

### 2. Active Directory Domain Services

*28.05.2026*

Installed AD DS role via PowerShell. Promoted server to Domain Controller.

- Domain name: `homelab.local`
- NetBIOS name: `HOMELAB`
- Forest/Domain functional level: Windows Server 2016 (WinThreshold)
- DNS installed on DC
- DC hostname: `DC01.homelab.local`

Result: DC promoted, server rebooted, domain `homelab.local` active.

### 3. DHCP Server
*30.05.2026*

Installed DHCP Server role. Created scope:
- Range: `192.168.0.100 – 192.168.0.200`
- Subnet mask: `255.255.255.0`
- Default gateway: `192.168.0.1`
- DNS: `192.168.0.10` (DC)
- DNS Domain: `homelab.local`
- Lease duration: 8 days

Authorized DHCP server in AD.

**Issue:** `Add-DhcpServerInDC` failed with `WIN32 5 / Access Denied`.
**Root cause:** user `admin` was not in `Domain Admins`. Fixed via
`Add-ADGroupMember`. Full logoff required for Kerberos token refresh.

Result: scope active, clients will receive addresses from scope.

### 4. GPO — Basic Policies
*30.05.2026*

Created OU=Clients for workstation policy scope.
Configured policies via PowerShell (Server Core, no GPMC GUI).

| GPO | Scope | Setting |
|-----|-------|---------|
| Password Policy | Default Domain Policy | Min length: 10, complexity: enabled, max age: 42d |
| Desktop Restrictions | OU=Clients | Disable Control Panel |
| Drive Map | OU=Clients | Map \\DC01\share as Z: |

Password Policy applied via `Set-ADDefaultDomainPasswordPolicy`.
Desktop Restrictions and Drive Map created via `New-GPO` + `New-GPLink`.

Drive Map configured via Group Policy Preferences (XML in SYSVOL).
No GUI available on Server Core — Drives.xml written manually
line by line using `Set-Content` / `Add-Content` directly into:
`SYSVOL\homelab.local\Policies\{GPO-GUID}\User\Preferences\Drives\`

Shared folder: `C:\share` → `\\DC01\share` (Domain Users: Full Access)

Result: GPOs created and linked to OU=Clients. 
Verification pending — requires domain-joined client.

### 5. Domain Join — Ubuntu
*07.06.2026*

Added PTR record to DC01 DNS for reverse lookup:
- Zone: `0.168.192.in-addr.arpa`
- Record: `10` → `DC01.homelab.local`

Created computer account in AD: `CN=LINUX-SERVER,CN=Computers,DC=homelab,DC=local`

Result: `linux-server` joined to `homelab.local` confirmed from both sides.

### 6. DNS Forwarder
*05.07.2026*
Added external DNS forwarders on DC01 so domain clients resolve public
names (previously only `homelab.local` resolved; external lookups failed).
- Forwarders: `8.8.8.8`, `1.1.1.1`
Configured via `Set-DnsServerForwarder`. Verified with `Get-DnsServerForwarder`.
Result: internal and external name resolution working from domain members.

### 7. Shared Folders with AD-Based Access Control
*05.07.2026*
Replaced the earlier flat share (`\\DC01\share`, Domain Users: Full) with
per-department shares isolated by AD security groups.

**AD structure:**
Created `OU=Departments`. Added three Global Security groups: HR, IT, Finance.
```powershell
New-ADOrganizationalUnit -Name "Departments" -Path "DC=homelab,DC=local"
# HR / IT / Finance
New-ADGroup -Name "HR" -GroupScope Global -GroupCategory Security -Path "OU=Departments,DC=homelab,DC=local"
```

**Folders:** `C:\Shares\{HR,IT,Finance}`.

**NTFS ACL:** inheritance removed, department group granted Modify,
admin access preserved. Applied via `icacls` (PowerShell
`FileSystemAccessRule` constructor failed to parse enum flags on Server Core):
```cmd
icacls C:\Shares\HR /inheritance:r /grant "HOMELAB\Domain Admins:(OI)(CI)F" "SYSTEM:(OI)(CI)F" "HOMELAB\HR:(OI)(CI)M"
```
Resulting ACL per folder — 3 ACEs: department group (Modify),
SYSTEM (Full), Domain Admins (Full).

**SMB shares:** created for each folder.
```powershell
New-SmbShare -Name "HR" -Path "C:\Shares\HR" -FullAccess "Everyone"
```
Share permission `Everyone: Full` — access restricted by NTFS ACL
(effective access = the more restrictive of share and NTFS).

**Test user:** `hr.test` created, added to HR group.

Result: per-department shares active, access enforced by AD group
membership. Old `\\DC01\share` removed via `Remove-SmbShare`.

**Note — GPO Drive Map:** the Drive Map GPO (section 4) still maps the

### 8. DHCP Reservation for linux-server
*09.07.2026*

Created a DHCP reservation on DC01 to pin `192.168.0.100` to the Ubuntu VM.
```powershell
Add-DhcpServerv4Reservation -ScopeId 192.168.0.0 -IPAddress 192.168.0.100 `
  -ClientId "08-00-27-84-EF-D8" -Name "linux-server"
```
Verified via `Get-DhcpServerv4Reservation`.

**Issue:** the reservation never took effect. `journalctl` on the Ubuntu VM
showed leases arriving `via 192.168.0.1` — the home router, not DC01.

**Root cause:** a second DHCP server (the home router) is active on the
bridged network. Both answer; the router wins the race. Its pool also
overlaps the DC01 scope (`.100–.200`), so IP conflicts are possible.

**Resolution:** DC01 remains the authorised DHCP server for the lab, but in
this bridged home network it cannot be the only one. Ubuntu was moved to a
static address instead (see Linux log §5). Reservation kept as an artefact.

### 9. Password Policy — First Real Verification
*09.07.2026*

Logging in to DC01 triggered a forced password change for `HOMELAB\admin`.
Cause: `max age: 42d` from the Password Policy set in §4 — the domain was
created 28.05.2026, so the password had expired.

Result: Password Policy confirmed working in practice. Previously only
configured, never observed enforcing.
removed `\\DC01\share` as `Z:`. Requires update to point at a valid
share (e.g. per-department mapping filtered by group). Pending.
