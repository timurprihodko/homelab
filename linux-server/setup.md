# Linux Server Setup

## Environment
- OS: Ubuntu Server 22.04 LTS
- RAM: 2 GB
- Disk: 20 GB
- Hypervisor: VirtualBox
- Network: Bridged

## Goals
- [X] SSH key-based authentication
- [ ] Web server (Nginx)
- [ ] File server (Samba/NFS) with access control

## 1. Ubuntu Server Installation
*28.05.2026*

Downloaded Ubuntu Server 22.04 LTS ISO.
Created VM in VirtualBox with the above parameters.
OpenSSH Server enabled during installation.

Server IP address: `192.168.0.100`

Note: `192.168.0.100` falls inside the DHCP scope. Later replaced with a
static address — see §5.

## 2. SSH Key-Based Authentication
*28.05.2026*

Installed OpenSSH Server on the VM.
Generated Ed25519 key pair on Windows host.
Copied public key to server (~/.ssh/authorized_keys).
Disabled password authentication in /etc/ssh/sshd_config.

Result: Passwordless SSH login from Windows host confirmed.

## 3. Domain Join — homelab.local
*07.06.2026*

Joined Ubuntu to Active Directory domain `homelab.local`.

Packages installed: `realmd`, `sssd`, `sssd-tools`, `adcli`, `krb5-user`, `samba-common-bin`

DNS configured via netplan (`/etc/netplan/01-homelab.yaml`):
- Nameserver: `192.168.0.10` (DC01)
- Search domain: `homelab.local`
- DHCP DNS override disabled

Result: `linux-server` joined to `homelab.local`, verified via `realm list`.

## 4. Shared Folder Access — Verification
*05.07.2026*
Verified the AD-based ACL on DC01 department shares (see Windows log §7)
from the Ubuntu client using `smbclient`.
```bash
sudo apt install -y smbclient
smbclient //192.168.0.10/HR -U 'HOMELAB\hr.test'      # access granted
smbclient //192.168.0.10/Finance -U 'HOMELAB\hr.test' # NT_STATUS_ACCESS_DENIED
```
Result: `hr.test` (member of HR) could list `\\DC01\HR` but was denied on
`\\DC01\Finance`. AD group-based access control confirmed working.

## 5. Static IP Configuration
*09.07.2026*

**Problem:** the DHCP reservation on DC01 (Windows log §8) had no effect —
Ubuntu kept receiving addresses from the home router (`192.168.0.1`), which
runs its own DHCP server on the bridged network. The address drifted between
reboots (`.61`, `.101`), breaking SSH on a fixed IP.

**Fix:** switched `enp0s3` to a static address outside both DHCP pools.

`/etc/netplan/01-homelab.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.0.20/24]
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [192.168.0.10]
        search: [homelab.local]
```
Also moved `50-cloud-init.yaml` aside — netplan merges every `.yaml` in
`/etc/netplan/` alphabetically, and cloud-init's file re-enabled DHCP.

**Status: in progress.** `netplan get` reflects the static config, but the
old DHCP lease (`192.168.0.61`) persists on the interface alongside `.20`,
and `resolvectl` still prefers the router as DNS. Something outside netplan
is renewing the lease. To investigate: running `dhclient` processes,
cloud-init network management, drop-ins under `/etc/systemd/network/`.

### Side notes
- IPv6 is active again (`DHCPv6` in `journalctl`). The `sysctl -w
  net.ipv6.conf.all.disable_ipv6=1` from §3 was runtime-only and did not
  survive reboot — needs `/etc/sysctl.conf` to persist.

### Troubleshooting

| Error | Root cause | Fix |
|-------|-----------|-----|
| `Cannot contact any KDC` | `krb5-user` not installed | `apt install krb5-user` |
| `Server not found in Kerberos database` | No reverse DNS PTR record | Added PTR record on DC01 |
| `GSSAPI: Unspecified GSS failure` | IPv6 causing wrong SPN lookup | `sysctl -w net.ipv6.conf.all.disable_ipv6=1` |
| `Permissions denied` on netplan apply | Wrong file permissions | `chmod 600 /etc/netplan/01-homelab.yaml` |
| `apt` — `Could not resolve archive.ubuntu.com` | DC01 DNS had no external forwarder | Added forwarders `8.8.8.8`,`1.1.1.1` on DC01 (`Set-DnsServerForwarder`) |
