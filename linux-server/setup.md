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
Created VM in VMware with the above parameters.
OpenSSH Server enabled during installation.

Server IP address: `192.168.0.61`

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

### Troubleshooting

| Error | Root cause | Fix |
|-------|-----------|-----|
| `Cannot contact any KDC` | `krb5-user` not installed | `apt install krb5-user` |
| `Server not found in Kerberos database` | No reverse DNS PTR record | Added PTR record on DC01 |
| `GSSAPI: Unspecified GSS failure` | IPv6 causing wrong SPN lookup | `sysctl -w net.ipv6.conf.all.disable_ipv6=1` |
| `Permissions denied` on netplan apply | Wrong file permissions | `chmod 600 /etc/netplan/01-homelab.yaml` |
