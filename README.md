# Homelab Infrastructure

Declarative homelab infrastructure using:

- Arch Linux Hosts
- Ansible Automation
- BTRFS Snapshots & Incremental Backups
- Libvirt Virtualization
- Cloud-Init Ubuntu VMs
- Kubernetes (k3s) Active Cluster
- Prometheus & Grafana Monitoring
- NFS Persistent Storage

---

## Objective

Create a fully reproducible, automated, and resilient homelab environment where baremetal hosts and virtual machines can be destroyed and recreated at any time without data loss. The infrastructure provisions a fully functional Kubernetes (K3s) cluster on top of the underlying virtualization layer.

---

## Architecture Overview
```text
Mini PCs (Arch Linux)
├─ BTRFS Snapshots
├─ Ansible Deployment Agent
├─ Libvirt / KVM
│  ├─ Bridged Networking (br0)
│  └─ Cloud-Init Ubuntu VMs (vm01, vm02, vm03)
│     └─ Kubernetes (K3s) Cluster
└─ Backup & Storage → OpenMediaVault (NFS/BTRFS)
```

---

## Key Features

- Host backups with BTRFS incremental send to remote Target
- Fully declarative Ansible Infrastructure replacing fragmented manual steps.
- Automated VM creation mapped locally with Cloud-Init templates
- SSH key-based access only (No passwords)
- Zero-touch K3s High-Availability Deployment
- Internal DNS over OpenWrt (`*.home` domains)
- Centralized maintenance playbooks for updates and reboots
- Unified monitoring dashboard via Grafana
- Persistent storage for k8s workloads via NFS

---

## Quick Start
Ensure your inventory variables in `ansible/inventory/hosts.yml` match your network topology (DHCP reservations).

```bash
git clone https://github.com/badorius/homelab-infra.git 
cd homelab-infra/ansible

# Deploy bare-metal hypervisors, automated backups, provision VMs, and install the K3s cluster:
ansible-playbook -i inventory/hosts.yml site.yml
```

## Infrastructure Maintenance

Use the maintenance playbooks to update and reboot all non-network infrastructure:

```bash
cd ansible
ansible-playbook playbooks/maintenance/update_reboot.yml
```

---

## Service Endpoints

Once the infrastructure is deployed, the following services are available via Traefik:

| Service | Endpoint | Description |
| ------- | -------- | ----------- |
| Homepage | [http://homepage.home](http://homepage.home) | Infrastructure Overview |
| Grafana | [http://grafana.home](http://grafana.home) | Cluster Monitoring |

> [!NOTE]
> DNS resolution depends on the OpenWrt router configuration managed via Ansible.
