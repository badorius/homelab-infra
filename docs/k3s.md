# Kubernetes (k3s)

Active lightweight Kubernetes cluster running inside cloud VMs (vm01, vm02, vm03).

**K3s API**: `https://192.168.8.248:6443`
**Ingress Controller**: Traefik (default)
**TLS**: cert-manager with `homelab-ca` ClusterIssuer
**DNS**: OpenWrt dnsmasq — all `*.home` → `192.168.8.248`

## Active Ingresses

| Domain | Service | Namespace | TLS |
|--------|---------|-----------|-----|
| `homepage.home` | Homepage Dashboard | `homepage` | Yes |
| `grafana.home` | Grafana Monitoring | `monitoring` | Yes |
| `argocd.home` | ArgoCD GitOps UI | `argocd` | Yes |
| `gitea.home` | Gitea Git Server | `gitea` | Yes |
| `ci.home` | Woodpecker CI | `woodpecker` | Yes |
| `registry.home` | Harbor Container Registry | `harbor` | Yes |
| `sewbase.home` | Sewbase Application | `sewbase` | No (pending) |
| `qbittorrent.home` | qBittorrent | `media` | Yes |

## Cluster Nodes

| Node | Role | IP |
|------|------|----|
| vm01 | Master | `192.168.8.248` |
| vm02 | Worker | `192.168.8.249` |
| vm03 | Worker | `192.168.8.101` |

## Features

- Immutable services via ArgoCD (GitOps)
- Automated host updates & reboots (Ansible)
- Unified DNS via OpenWrt dnsmasq
- Persistent storage via NFS Subdir Provisioner → OMV NAS
- Full cluster monitoring (Prometheus + Grafana)
- Private CI/CD: Gitea → Woodpecker → Harbor → ArgoCD

## GitOps CI/CD Flow

```
Push to gitea.home → Woodpecker CI builds & tests
                   → Docker image pushed to registry.home (Harbor)
                   → Woodpecker calls ArgoCD API to trigger sync
                   → ArgoCD pulls image and redeploys
```

See `agents.md` §6.10 for full Sewbase CI/CD runbook.
