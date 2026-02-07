# Architecture

This project follows a layered architecture:

## Layer 1 – Physical Hosts
- Arch Linux
- Ansible managed
- BTRFS filesystem

## Layer 2 – Backup
- Local snapshots
- Remote incremental backups
- OpenMediaVault target

## Layer 3 – Virtualization
- Libvirt
- NAT internal network
- Cloud-Init VMs

## Layer 4 – Services (Future)
- k3s
- Reverse proxy
- Monitoring

