# Homelab Infrastructure

Declarative homelab infrastructure using:

- Arch Linux Hosts
- Ansible Automation
- BTRFS Snapshots & Incremental Backups
- Libvirt Virtualization
- Cloud-Init VMs
- Future Kubernetes (k3s)

---

## Objective

Create a fully reproducible, automated, and resilient homelab
environment where hosts and virtual machines can be destroyed
and recreated at any time without data loss.

---

## Architecture Overview
```
Mini PCs (Arch Linux)
├─ BTRFS Snapshots
├─ Ansible
├─ Libvirt
│ └─ Cloud VMs
└─ Backup → OpenMediaVault
```

---

## Key Features

- Host backups with BTRFS incremental send
- Automated VM creation with Cloud-Init
- SSH key based access only
- No passwords.
- Fully declarative infrastructure

---

## Quick Start

```
git clone https://github.com/badorius/homelab-infra.git 
cd ansible
ansible-playbook playbooks/bootstrap.yml
```
.
