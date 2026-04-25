# qBittorrent

BitTorrent client with web UI. Downloads are stored directly on the OMV NAS via a static NFS PersistentVolume.

## Access

| Property | Value |
|----------|-------|
| URL | https://qbittorrent.home |
| Default username | admin |
| Default password | adminadmin (set by qBittorrent on first run) |
| Namespace | media |
| Image | lscr.io/linuxserver/qbittorrent:latest |
| Managed by | ArgoCD (`qbittorrent` Application) |

> **Note:** On first run qBittorrent generates a random password and logs it. Check pod logs: `kubectl logs -n media -l app=qbittorrent`.

## Architecture

```
qbittorrent (Deployment, 1 replica)
  ├─ Image: lscr.io/linuxserver/qbittorrent:latest
  ├─ Timezone: Europe/Madrid
  ├─ PUID/PGID: 1000
  ├─ Ports:
  │    ├─ 8080 (WebUI) → Service port 80 → Ingress
  │    ├─ 6881/TCP (BitTorrent)
  │    └─ 6881/UDP (BitTorrent)
  ├─ /config    → qbittorrent-config-pvc (nfs-client, dynamic)
  └─ /downloads → qbittorrent-downloads-pvc (nfs-client-static, static PV on OMV)
```

## Persistent Storage

| PVC | Type | Mount | Size | Path on NAS |
|-----|------|-------|------|-------------|
| qbittorrent-config-pvc | Dynamic (nfs-client) | /config | default | auto-provisioned |
| qbittorrent-downloads-pvc | Static (nfs-client-static) | /downloads | 10Ti | 192.168.8.15:/share/qbittorent |

The downloads PVC uses a **static PersistentVolume** (`qbittorrent-downloads-pv`) pointing directly to the OMV NAS share `/share/qbittorent`. This means files are stored on the NAS, not in cluster storage — ensuring persistence across pod restarts and NFS mounts.

```yaml
# pv-downloads.yaml — the static NFS PV
spec:
  nfs:
    server: 192.168.8.15
    path: /share/qbittorent          # note: typo in NAS share name, keep as-is
  mountOptions:
    - nfsvers=4.2
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-client-static
```

## Common Operations

```bash
# Check pod is running
kubectl get pods -n media

# Get initial admin password (first run only)
kubectl logs -n media -l app=qbittorrent | grep -i password

# Check download disk usage on NAS
kubectl exec -n media deploy/qbittorrent -- df -h /downloads

# View torrent download logs
kubectl logs -n media -l app=qbittorrent -f

# Check ingress + TLS
kubectl get ingress -n media
kubectl get certificate qbittorrent-tls -n media

# Access downloads from NAS directly (NFS)
# Mount: 192.168.8.15:/share/qbittorent
```

## BitTorrent Port Access

Port 6881 (TCP+UDP) is exposed via the Kubernetes Service but requires the router/firewall to forward this port to vm01 (192.168.8.248) for proper seeding. Without port forwarding, qBittorrent works in passive mode only.

To check connectivity:
```bash
# From qBittorrent pod
kubectl exec -n media deploy/qbittorrent -- ss -tlnup | grep 6881
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Can't log in | Get auto-generated password from pod logs on first run |
| Downloads not appearing on NAS | Verify NFS mount: `kubectl exec -n media deploy/qbittorrent -- mount \| grep downloads` |
| NFS mount fails on pod start | Check NFS server (OMV) is up and `/share/qbittorent` share exists |
| WebUI slow / unresponsive | qBittorrent is CPU-intensive during heavy seeding; normal behaviour |
| Pod CrashLoopBackOff | Config PVC may have corrupted state — delete and recreate `qbittorrent-config-pvc` |
| TLS cert error in browser | Import homelab CA into browser (see cert-manager.md) |
