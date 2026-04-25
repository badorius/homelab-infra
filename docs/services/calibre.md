# Calibre-Web

Web-based ebook library manager. Browse, read, and download ebooks from the browser.

## Access

| Property | Value |
|----------|-------|
| URL | https://calibre.home |
| Default username | admin |
| Default password | admin123 (set on first login) |
| Namespace | calibre |
| Image | lscr.io/linuxserver/calibre-web:latest |
| Managed by | ArgoCD (`calibre` Application) |

## Architecture

```
calibre-web (Deployment, 1 replica)
  ├─ Image: lscr.io/linuxserver/calibre-web:latest
  ├─ Port: 8083 (container) → 80 (Service) → 443 (Ingress via Traefik)
  ├─ Timezone: Europe/Madrid
  ├─ PUID/PGID: 1000
  ├─ /config → calibre-config-pvc (1Gi, nfs-client)  — app database + settings
  └─ /books  → calibre-books-pvc  (50Gi, nfs-client) — Calibre library files
```

## Persistent Storage

| PVC | Mount | Size | StorageClass | Purpose |
|-----|-------|------|-------------|---------|
| calibre-config-pvc | /config | 1Gi | nfs-client | Calibre-Web database (SQLite), settings |
| calibre-books-pvc | /books | 50Gi | nfs-client | Calibre library (metadata.db + ebook files) |

The 50Gi books PVC is dynamically provisioned by the NFS provisioner on OMV (192.168.8.15).

## First-Time Setup

On first access to https://calibre.home, Calibre-Web will ask for the Calibre library location:

1. Set **Database Configuration** → Location of Calibre database: `/books`
2. Log in with default credentials: `admin` / `admin123`
3. Change the admin password via Admin → Edit User → admin

## Uploading Books

### Option 1: Upload via Web UI

In Calibre-Web UI: Books → Upload Book (top right corner). Accepts EPUB, PDF, MOBI, etc.

### Option 2: Copy to the PVC directly

Find the NFS path for the books PVC:

```bash
# Find the NFS path that was provisioned
kubectl get pv -o wide | grep calibre-books

# The NFS provisioner creates a subdir on OMV like:
# 192.168.8.15:/share/<namespace>-<pvcname>-<pv-uid>

# Copy books directly to the NFS share (from a machine with NFS access)
rsync -av /local/books/ 192.168.8.15:/share/calibre-calibre-books-pvc-<uid>/
```

After copying, trigger a library scan in Calibre-Web: Admin → Tasks → Reconnect Database.

### Option 3: Use Calibre desktop app

1. Point Calibre desktop to the same NFS share as a library
2. Add books in the desktop app
3. Calibre-Web reads from the same library path

## Common Operations

```bash
# Check pod is running
kubectl get pods -n calibre

# View logs (useful for debugging book import issues)
kubectl logs -n calibre -l app=calibre-web -f

# Check ingress and TLS
kubectl get ingress -n calibre
kubectl get certificate calibre-tls -n calibre

# Restart the pod (e.g., after adding books via NFS)
kubectl rollout restart deployment calibre-web -n calibre

# Check PVC usage
kubectl exec -n calibre deploy/calibre-web -- df -h /books /config
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Books not appearing after upload | Trigger reconnect: Admin → Tasks → Reconnect Database |
| Calibre-Web not starting | Check pod logs — likely the `/books` directory has no `metadata.db`. Create an empty Calibre library first via the desktop app |
| "Unable to connect to server" | Pod not running: `kubectl get pods -n calibre` |
| TLS cert error in browser | Import homelab CA into browser (see cert-manager.md) |
| Books PVC full | Expand PVC or add another volume; current limit: 50Gi |
| Config PVC full (unlikely) | Config PVC is 1Gi — only fills if book covers cache grows very large |
