# Homelab Architecture

This document explains the full homelab stack from hardware to deployed applications. Written for someone who has never seen this repository before.

---

## The Big Picture

The homelab is structured in 5 layers, each built on top of the previous one:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 5 — GitOps / CI-CD                                   │
│  ArgoCD · Gitea · Woodpecker CI · Harbor                    │
├─────────────────────────────────────────────────────────────┤
│  Layer 4 — Kubernetes (K3s)                                 │
│  Traefik · cert-manager · Prometheus · Applications         │
├─────────────────────────────────────────────────────────────┤
│  Layer 3 — Virtualization (KVM/libvirt)                     │
│  3 Ubuntu VMs · Cloud-Init · Bridged networking             │
├─────────────────────────────────────────────────────────────┤
│  Layer 2 — Backup                                           │
│  BTRFS snapshots → OMV NAS · K3s ETCD snapshots            │
├─────────────────────────────────────────────────────────────┤
│  Layer 1 — Physical                                         │
│  3× Arch Linux mini PCs · BTRFS filesystem · OpenWrt router│
└─────────────────────────────────────────────────────────────┘
```

Everything is managed as code. You can wipe any layer and rebuild it from this repository.

---

## Layer 1 — Physical Infrastructure

### Hardware

| Host | IP | OS | Role |
|------|----|----|------|
| arch01 | 192.168.8.11 | Arch Linux | KVM hypervisor → vm01 |
| arch02 | 192.168.8.12 | Arch Linux | KVM hypervisor → vm02 |
| arch03 | 192.168.8.13 | Arch Linux | KVM hypervisor → vm03 |
| OMV NAS | 192.168.8.15 | OpenMediaVault (Debian) | NFS storage + BTRFS backup target |
| Router | 192.168.8.253 | OpenWrt | DNS (dnsmasq) + DHCP + firewall |

All three mini PCs use **BTRFS** as their root filesystem, enabling efficient snapshots and incremental data transfer.

### Ansible Role Mapping

| Role | What it does |
|------|-------------|
| `base` | Arch system update, core packages (btrfs-progs, chrony, bind, etc.) |
| `common` | Hostname, timezone (Europe/Madrid), /etc/hosts, homelab CA trust |
| `kvm-host` | QEMU/libvirt install, tun module, br0 bridge via systemd-networkd |
| `nfs-client` | Mount OMV NFS at boot |
| `bridge-network` | br0 netdev + network config files |
| `libvirt-network` | Define the libvirt default network |
| `openwrt-dns` | Static DNS entries via UCI on OpenWrt dnsmasq |

### DNS: OpenWrt dnsmasq

All `*.home` domains resolve to `192.168.8.248` (K3s master / Traefik). Managed by the `openwrt-dns` role.

Current DNS entries:

| Domain | IP |
|--------|----|
| homepage.home | 192.168.8.248 |
| grafana.home | 192.168.8.248 |
| argocd.home | 192.168.8.248 |
| gitea.home | 192.168.8.248 |
| ci.home | 192.168.8.248 |
| registry.home | 192.168.8.248 |
| sewbase.home | 192.168.8.248 |
| traefik.home | 192.168.8.248 |
| calibre.home | 192.168.8.248 |
| qbittorrent.home | 192.168.8.248 |
| openmediavault.home | 192.168.8.15 |
| minio.home | 192.168.8.15 |

---

## Layer 2 — Backup System

### BTRFS Backup Chain

```
arch01/02/03 (BTRFS root)
  │
  ├─ 03:00 → btrfs-snapshot.timer  → creates /.snapshots/YYYY-MM-DD_HH-MM/
  ├─ 04:00 → btrfs-send.timer      → incremental send to OMV NAS
  ├─ 05:00 → btrfs-prune.timer     → keep 7 local snapshots
  └─ 06:00 → prune-remote.timer    → keep 14 snapshots on OMV
```

Scripts deployed: `/usr/local/bin/btrfs-snapshot.sh`, `btrfs-send.sh`, `btrfs-prune.sh`

Ansible roles: `btrfs-backup` (on arch hosts) + `btrfs-target-prune` (on OMV)

### K3s ETCD Backups

The K3s master (vm01) snapshots the ETCD database to `/mnt/nfs/backups/k3s/` on the NFS share.

Managed by: `ansible/roles/k3s-backup/`

---

## Layer 3 — Virtualization

### VM Creation Flow

```
ansible/roles/cloud-vms/ runs on each arch host:

1. Download Ubuntu 22.04 Jammy Cloud Image → /var/lib/libvirt/base/ubuntu.img
2. Create 20GB qcow2 disk (backed by cloud image) → /var/lib/libvirt/images/<vm>.qcow2
3. Render Cloud-Init user-data (SSH key, hostname, sudo NOPASSWD)
4. Render Cloud-Init meta-data (instance-id, hostname)
5. Build cloud-init ISO → /var/lib/libvirt/seed/<vm>.iso
6. Render libvirt XML from template → define VM with virsh
7. virsh autostart <vm>   ← starts on host boot
8. virsh start <vm>
```

VM definitions in `ansible/inventory/group_vars/all.yml`:
```yaml
vm_list:
  - { name: vm01, mac: "52:54:00:aa:bb:01", target_host: arch01 }
  - { name: vm02, mac: "52:54:00:aa:bb:02", target_host: arch02 }
  - { name: vm03, mac: "52:54:00:aa:bb:03", target_host: arch03 }
```

MAC addresses are fixed → OpenWrt DHCP assigns static IPs.

### Networking Architecture

```
arch01 (192.168.8.11)
  └─ br0 (Linux bridge, systemd-networkd)
       ├─ eth0 (physical NIC, member of bridge)
       └─ vm01 tap interface (192.168.8.248)

→ VM traffic goes through br0 to the same L2 segment as physical hosts
→ No NAT — VMs are directly routable on 192.168.8.0/24
```

---

## Layer 4 — Kubernetes (K3s)

### Cluster Layout

| Node | IP | Role |
|------|----|----|
| vm01 | 192.168.8.248 | Control plane (master) + Traefik entry point |
| vm02 | 192.168.8.249 | Worker |
| vm03 | 192.168.8.101 | Worker |

K3s is a lightweight Kubernetes distribution. It bundles Traefik (ingress), CoreDNS, and Flannel CNI.

### Key Cluster Components

#### Traefik — Ingress Controller
Routes all HTTP/HTTPS traffic entering on port 80/443 to the correct backend service based on the `Host` header. Configured via `HelmChartConfig` at `kubernetes/infra/traefik/`.

The dashboard is exposed at `https://traefik.home` (port 9000 internally, proxied via ingress).

#### cert-manager — Automatic TLS
Issues TLS certificates for every service using the homelab root CA:

```
selfsigned-issuer (ClusterIssuer)
  └─ issues → homelab-ca Certificate (stored in root-secret)
                └─ homelab-ca (ClusterIssuer)
                      └─ signs → service certificates
                                  (when ingress has cert-manager.io/cluster-issuer: homelab-ca)
```

The root CA cert must be trusted in your browser/OS. Export it:
```bash
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt
```

#### NFS Storage — nfs-client StorageClass
All PersistentVolumeClaims use `storageClassName: nfs-client`. The NFS subdir provisioner auto-creates subdirectories on `192.168.8.15:/share`.

#### kube-prometheus-stack — Monitoring
- **Prometheus**: scrapes K3s components + external node_exporter on arch01/02/03, OMV, and router
- **Grafana**: dashboards at `https://grafana.home`
- **AlertManager**: configured with 1Gi storage

External scrape targets (arch hosts + OMV + router) are in `kubernetes/services/monitoring/values.yaml`.

Upgrade via: `ansible-playbook playbooks/setup_monitoring_agents.yml`

### ArgoCD-Managed Services

These services are deployed and kept in sync by ArgoCD automatically:

| Service | Namespace | Helm Chart | Source |
|---------|-----------|------------|--------|
| Gitea | gitea | gitea 10.1.4 | dl.gitea.com/charts |
| Woodpecker | woodpecker | woodpecker 1.5.0 | woodpecker-ci.org |
| Harbor | harbor | harbor 1.14.0 | helm.goharbor.io |
| Sewbase | sewbase | (raw manifests) | gitea.home (private) |

All use the pattern: `kustomization.yaml` with `helmCharts:` block + `--enable-helm` in ArgoCD.

### Manually-Applied Services

These are applied once with `kubectl apply -k` and not managed by ArgoCD:

| Service | Namespace | Command |
|---------|-----------|---------|
| Homepage | homepage | `kubectl apply -k kubernetes/services/homepage/` |
| qBittorrent | media | `kubectl apply -k kubernetes/services/qbittorrent/` |
| Calibre-Web | calibre | `kubectl apply -k kubernetes/services/calibre/` |

---

## Layer 5 — GitOps / CI-CD

### Component Roles

| Component | Role | URL |
|-----------|------|-----|
| ArgoCD | Continuous Delivery — watches git, applies K8s manifests | https://argocd.home |
| Gitea | Private git server + OAuth provider | https://gitea.home |
| Woodpecker CI | Build pipelines (test → build → push → deploy) | https://ci.home |
| Harbor | Private OCI container registry | https://registry.home |

### ArgoCD App-of-Apps Pattern

```
kubernetes/infra/argocd/applications.yaml
  ├─ Application: gitea      → kubernetes/services/gitea/      (this repo)
  ├─ Application: woodpecker → kubernetes/services/woodpecker/ (this repo)
  ├─ Application: harbor     → kubernetes/services/harbor/     (this repo)
  └─ Application: sewbase    → infra/k8s/sewbase/              (gitea.home private repo)
```

ArgoCD has `automated.prune: true` and `automated.selfHeal: true` on all apps — it will revert manual changes and remove resources deleted from git.

### Sewbase CI/CD Pipeline

```
git push → gitea.home/gitea_admin/sewbase_guitea (main branch)
    │
    ├─ Woodpecker: npm ci + npm test
    ├─ Woodpecker: docker buildx → registry.home/badorius/sewbase:latest
    └─ Woodpecker: curl POST to ArgoCD API (sync sewbase app)
                        │
                        └─ ArgoCD applies infra/k8s/sewbase/ manifests
                             → K3s pulls new image from Harbor
                             → Pod restarts with new version
                             → https://sewbase.home updated
```

---

## Security Model

| Control | Implementation |
|---------|---------------|
| No secrets in git | Repo is public. All passwords via `kubectl create secret` |
| SSH key-only access | No password SSH on any host. Key in Cloud-Init user-data |
| Internal TLS | cert-manager homelab-ca signs all ingress certificates |
| Private registry | Harbor requires login. K8s pull secrets grant pod access |
| CI secrets | Stored in Woodpecker DB (not in git). Set via UI or CLI |
| CA trust | Root CA distributed to all hosts via Ansible `common` role |

---

## File Locations Quick Reference

| What | File |
|------|------|
| VM definitions | `ansible/inventory/group_vars/all.yml` |
| Host inventory | `ansible/inventory/hosts.yml` |
| DNS entries | `ansible/roles/openwrt-dns/tasks/main.yml` |
| Ansible master playbook | `ansible/site.yml` |
| ArgoCD App-of-Apps | `kubernetes/infra/argocd/applications.yaml` |
| cert-manager issuers | `kubernetes/infra/cert-manager/clusterissuers.yaml` |
| Traefik dashboard | `kubernetes/infra/traefik/` |
| Gitea Helm values | `kubernetes/services/gitea/kustomization.yaml` |
| Woodpecker Helm values | `kubernetes/services/woodpecker/kustomization.yaml` |
| Harbor Helm values | `kubernetes/services/harbor/kustomization.yaml` |
| Monitoring values | `kubernetes/services/monitoring/values.yaml` |
| Sewbase K8s manifests | `infra/k8s/sewbase/` (sewbase_guitea repo) |
| Sewbase pipeline | `.woodpecker.yml` (sewbase_guitea repo) |
| AI agent memory | `agents.md` |
