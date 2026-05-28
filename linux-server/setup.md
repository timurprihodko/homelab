# Linux Server Setup

## Environment
- OS: Ubuntu Server 24.04 LTS
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

Downloaded Ubuntu Server 24.04 LTS ISO.
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
