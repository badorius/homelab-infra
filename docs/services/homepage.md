# Homepage

Self-hosted start page / dashboard. Shows links to all homelab services in a clean web UI.

## Access

| Property | Value |
|----------|-------|
| URL | https://homepage.home |
| Auth | None (open access within LAN) |
| Namespace | homepage |
| Image | ghcr.io/gethomepage/homepage:latest |
| Managed by | ArgoCD (`homepage` Application) |

## Architecture

```
homepage (Deployment, 1 replica)
  ├─ Image: ghcr.io/gethomepage/homepage:latest
  ├─ Port: 3000 (container) → 80 (Service) → 443 (Ingress via Traefik)
  ├─ Config: ConfigMap `homepage-config` (mounted at /app/config)
  └─ Logs: emptyDir (ephemeral, not persisted)
```

No persistent storage — configuration lives in the `homepage-config` ConfigMap in Git.

## Configuration

All configuration is in `kubernetes/services/homepage/configmap.yaml`. Edit and commit to update the dashboard.

### Services shown

| Category | Services |
|----------|---------|
| Infrastructure | OpenMediaVault (192.168.8.15), MinIO (openmediavault.home:9001) |
| Downloads | qBittorrent (qbittorrent.home) |
| Cluster | K3s API (192.168.8.248:6443), Grafana (grafana.home) |

### Adding a new service

Edit `kubernetes/services/homepage/configmap.yaml`:

```yaml
data:
  services.yaml: |
    - Infrastructure:
        - MyNewService:
            icon: myicon          # from Simple Icons or local
            href: https://myservice.home
            description: What it does
```

Commit and push — ArgoCD picks it up within ~3 minutes.

## Common Operations

```bash
# Check homepage pod is running
kubectl get pods -n homepage

# View current configmap (live config)
kubectl get configmap homepage-config -n homepage -o yaml

# Force reload after configmap change (pod restarts automatically on ArgoCD sync)
kubectl rollout restart deployment homepage -n homepage

# Check ingress and TLS
kubectl get ingress -n homepage
kubectl get certificate -n homepage
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `homepage.home` shows blank page | Check pod logs: `kubectl logs -n homepage -l app=homepage` |
| Service link not working | The href in configmap uses `http://` not `https://` for some services — update to match actual protocol |
| "Not Allowed Host" error | `HOMEPAGE_ALLOWED_HOSTS` env var must include `homepage.home`; already set in deployment.yaml |
| Config changes not showing | ArgoCD may not have synced yet. Run `argocd app sync homepage --grpc-web` |
| TLS cert missing | Check cert-manager: `kubectl describe certificate homepage-tls -n homepage` |

## Adding New Service Links

> **Tip:** When you add a new service to the cluster, also add it to `homepage/configmap.yaml` so it appears on the dashboard.

Full list of supported icons: https://github.com/walkxcode/dashboard-icons
