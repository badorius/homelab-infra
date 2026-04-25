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

Calibre-Web is a **frontend** for an existing Calibre library — it does not create the library itself. The `/books` PVC must contain a valid `metadata.db` before the web UI accepts the path.

### Step 1 — Initialize the Calibre library (one-time)

Run the provided Kubernetes Job to create an empty library in the books PVC:

```bash
kubectl apply -f kubernetes/services/calibre/init-library-job.yaml
kubectl logs -n calibre -l job-name=calibre-init-library -f
```

Wait until you see `Library initialized successfully`. Verify:

```bash
kubectl exec -n calibre deploy/calibre-web -- ls -la /books/
# Should show metadata.db
```

The Job auto-deletes itself after 5 minutes (`ttlSecondsAfterFinished: 300`).

### Step 2 — Configure the web UI

1. Open https://calibre.home
2. Set **Database Configuration** → Location of Calibre database: `/books`
3. Log in with default credentials: `admin` / `admin123`
4. Change the admin password via Admin → Edit User → admin

## Uploading Books

### Option 1: Upload via Web UI (pocos libros)

En https://calibre.home: icono de libro con `+` arriba a la derecha → Upload Book. Acepta EPUB, PDF, MOBI, CBZ, etc. Solo permite subir de uno en uno.

### Option 2: Bulk import via calibredb (recomendado para colecciones)

Este es el método usado para la importación inicial de la colección HumbleBundle (342 libros).
Requiere `calibre` instalado en la máquina local (`pacman -S calibre` en Arch).

**NFS path del books PVC:**
```
192.168.8.15:/share/calibre-calibre-books-pvc-pvc-48c38642-2271-41dc-8e59-ecb2b490566f
```

**Proceso completo:**

```bash
# 1. Montar el NFS (solo la primera vez, o si no está montado)
sudo mkdir -p /mnt/calibre-books
sudo mount -t nfs 192.168.8.15:/share/calibre-calibre-books-pvc-pvc-48c38642-2271-41dc-8e59-ecb2b490566f /mnt/calibre-books

# 2. Parar calibre-web para liberar el lock de metadata.db
kubectl scale deployment calibre-web -n calibre --replicas=0
kubectl wait --for=delete pod -l app=calibre-web -n calibre --timeout=60s

# 3. Importar los libros (--recurse escanea subdirectorios, los zips sin ebooks se ignoran)
sudo calibredb add --recurse --library-path /mnt/calibre-books /ruta/a/los/libros/

# 4. Levantar calibre-web
kubectl scale deployment calibre-web -n calibre --replicas=1
kubectl rollout status deployment calibre-web -n calibre

# 5. Desmontar el NFS
sudo umount /mnt/calibre-books
```

**Notas:**
- Los archivos `.zip` que no contienen ebooks (code supplements) se saltan automáticamente — los errores `No ebook found in ZIP archive` son normales.
- Los duplicados se detectan y saltan — usa `--duplicates` para añadirlos igualmente.
- `calibredb` organiza los libros en subdirectorios por autor dentro de `/books` automáticamente.

### Option 3: Calibre desktop app

1. Montar el NFS en local (ver paso 1 de Option 2)
2. Parar el pod (ver paso 2 de Option 2)
3. Abrir Calibre desktop → Switch/create library → apuntar a `/mnt/calibre-books`
4. Añadir, editar metadatos y convertir formatos desde el desktop
5. Levantar el pod de nuevo (ver paso 4 de Option 2)

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
| "New db location is invalid" en setup | `/books` no tiene `metadata.db` todavía. Ejecuta `init-library-job.yaml` primero (ver First-Time Setup) |
| Calibre-Web not starting | Check pod logs — likely the `/books` directory has no `metadata.db`. Run `init-library-job.yaml` to initialize. |
| "Unable to connect to server" | Pod not running: `kubectl get pods -n calibre` |
| TLS cert error in browser | Import homelab CA into browser (see cert-manager.md) |
| Books PVC full | Expand PVC or add another volume; current limit: 50Gi |
| Config PVC full (unlikely) | Config PVC is 1Gi — only fills if book covers cache grows very large |
