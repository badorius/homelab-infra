# Homelab Infrastructure

Declarative, reproducible homelab using Ansible + K3s + GitOps. Everything is code. Destroy and recreate at any time.

---

## Architecture at a glance

```
Physical Layer    3× Arch Linux mini PCs (arch01/02/03)
      │           BTRFS filesystem + systemd backup timers → OMV NAS
      ▼
Virtualization    KVM/libvirt, bridged networking (br0)
      │           3 Ubuntu 22.04 VMs (vm01/02/03), one per host
      ▼
Kubernetes        K3s cluster — vm01 master, vm02+vm03 workers
      │           Traefik ingress, cert-manager TLS, nfs-client storage
      ▼
GitOps            ArgoCD watches github.com/badorius/homelab-infra
      │           Gitea (private repos) + Woodpecker CI + Harbor registry
      ▼
Apps              Grafana · Homepage · Calibre-Web · qBittorrent · Sewbase
```

Full architecture: [`docs/architecture.md`](docs/architecture.md)

---

## Service Endpoints

All `*.home` domains resolve via OpenWrt dnsmasq to `192.168.8.248` (K3s master / Traefik).
TLS certificates are auto-issued by cert-manager using the homelab root CA.

| Service | URL | Notes |
|---------|-----|-------|
| **ArgoCD** | https://argocd.home | GitOps CD |
| **Gitea** | https://gitea.home | Private git server |
| **Woodpecker CI** | https://ci.home | CI pipelines (Gitea OAuth) |
| **Harbor Registry** | https://registry.home | OCI container registry |
| **Grafana** | https://grafana.home | Monitoring dashboards |
| **Traefik Dashboard** | https://traefik.home | Ingress routing UI |
| **Homepage** | https://homepage.home | Service dashboard |
| **Calibre-Web** | https://calibre.home | eBook library |
| **qBittorrent** | https://qbittorrent.home | Torrent downloader |
| **Sewbase** | https://sewbase.home | Custom app (CI/CD deployed) |
| **OpenMediaVault** | http://192.168.8.15 | NAS management (HTTP only) |

> **Browser TLS**: Import the homelab root CA into your browser.
> `kubectl get secret root-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt`

---

## Network Map

| Host | IP | Role |
|------|----|------|
| arch01 | 192.168.8.11 | KVM hypervisor → vm01 |
| arch02 | 192.168.8.12 | KVM hypervisor → vm02 |
| arch03 | 192.168.8.13 | KVM hypervisor → vm03 |
| vm01 | 192.168.8.248 | K3s master + Traefik entry point |
| vm02 | 192.168.8.249 | K3s worker |
| vm03 | 192.168.8.101 | K3s worker |
| OMV NAS | 192.168.8.15 | NFS storage + BTRFS backup target |
| Router | 192.168.8.253 | OpenWrt + dnsmasq |

---

## Quick Start — Fresh Deploy

Full procedure: [`docs/runbooks/fresh-deploy.md`](docs/runbooks/fresh-deploy.md)

```bash
# 1. Provision physical hosts, VMs, and K3s cluster
git clone https://github.com/badorius/homelab-infra.git
cd homelab-infra/ansible
ansible-playbook -i inventory/hosts.yml site.yml

# 2. Bootstrap Kubernetes infrastructure
kubectl apply -k kubernetes/infra/cert-manager/
kubectl apply -k kubernetes/infra/argocd/

# 3. Inject secrets (see fresh-deploy.md for full secret list)
kubectl create namespace gitea
kubectl create secret generic gitea-admin-secret -n gitea \
  --from-literal=username=gitea_admin --from-literal=password='YOUR_PASSWORD'

# 4. Deploy App-of-Apps (ArgoCD manages gitea, woodpecker, harbor, sewbase)
kubectl apply -f kubernetes/infra/argocd/applications.yaml

# 5. Manual services
kubectl apply -k kubernetes/services/homepage/
kubectl apply -k kubernetes/services/calibre/
kubectl apply -k kubernetes/infra/traefik/
```

---

## Repository Structure

```
homelab-infra/
├── agents.md                   # AI agent persistent memory (read first every session)
├── README.md                   # This file
├── ansible/
│   ├── site.yml                # Master playbook: network→hosts→backups→vms→k3s
│   ├── inventory/              # hosts.yml + group_vars/all.yml (VM definitions)
│   ├── playbooks/              # Individual playbooks
│   └── roles/                  # base, common, kvm-host, cloud-vms, k3s, btrfs-backup, ...
├── kubernetes/
│   ├── infra/                  # Bootstrap infrastructure (apply once manually)
│   │   ├── argocd/             # ArgoCD install + ingress + App-of-Apps
│   │   ├── cert-manager/       # cert-manager Helm + homelab-ca ClusterIssuer
│   │   └── traefik/            # Traefik dashboard (HelmChartConfig + ingress)
│   └── services/               # Application workloads
│       ├── gitea/              # ArgoCD-managed via Helm/Kustomize
│       ├── woodpecker/         # ArgoCD-managed via Helm/Kustomize
│       ├── harbor/             # ArgoCD-managed via Helm/Kustomize
│       ├── monitoring/         # kube-prometheus-stack (Helm, Ansible-upgraded)
│       ├── homepage/           # Manual kubectl apply -k
│       ├── calibre/            # Manual kubectl apply -k
│       └── qbittorrent/        # Manual kubectl apply -k
└── docs/
    ├── architecture.md         # Full stack explained for newcomers
    ├── services/               # Per-service documentation
    └── runbooks/               # Step-by-step operational procedures
```

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | Full stack explained from physical to GitOps |
| [Fresh Deploy](docs/runbooks/fresh-deploy.md) | Complete from-scratch deployment guide |
| [Add a New Service](docs/runbooks/new-service.md) | End-to-end: manifests → DNS → TLS → ArgoCD |
| [Add a CI/CD Pipeline](docs/runbooks/new-pipeline.md) | Woodpecker pipeline setup |
| [Secret Management](docs/runbooks/secrets.md) | Create, update and rotate secrets |
| [Backup & Restore](docs/runbooks/backup-restore.md) | BTRFS backups + K3s ETCD recovery |
| [K3s Operations](docs/runbooks/k3s-operations.md) | Day-2 cluster operations |
| [Sewbase Deploy](docs/services/sewbase.md) | Sewbase app CI/CD pipeline |

---

## Non-Negotiable Rules

1. **No secrets in this repo** — it is public. Use `kubectl create secret` at runtime.
2. **IaC first** — every change lives in Ansible or Kubernetes manifests, never ad-hoc.
3. **ArgoCD for managed services** — for gitea/woodpecker/harbor/sewbase: push to git, let ArgoCD sync. Never `kubectl apply` manually.
4. **TLS on every ingress** — add `cert-manager.io/cluster-issuer: homelab-ca` annotation + `spec.ingressClassName: traefik`.
5. **nfs-client StorageClass** — all PVCs use `storageClassName: nfs-client`.
6. **DNS for every new domain** — add entry in `ansible/roles/openwrt-dns/tasks/main.yml` and run `setup_network.yml`.

See [`agents.md`](agents.md) for AI agent context and current session state.
