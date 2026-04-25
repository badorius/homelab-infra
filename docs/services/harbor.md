# Harbor Registry

Private OCI container registry. Woodpecker CI pushes built images here; ArgoCD pulls from here via Kubernetes imagePullSecrets.

## Access

| Property | Value |
|----------|-------|
| URL | https://registry.home |
| Admin username | admin |
| Admin password | `REDACTED` (Harbor enforces uppercase policy — cannot use `REDACTED`) |
| Namespace | harbor |
| Chart | goharbor/harbor 1.14.0 |
| Managed by | ArgoCD (`harbor` Application) |

> **Note:** Harbor's password policy requires at least one uppercase letter, one lowercase letter, and one digit. The homelab standard `REDACTED` is rejected. Use `REDACTED` for Harbor specifically.

## Architecture

```
harbor-core (Deployment)
  ├─ REST API + web UI
  ├─ Auth: local user database
  └─ Routes pushes/pulls to registry component

harbor-registry (Deployment)
  └─ Stores OCI images in 10Gi PVC (nfs-client)

harbor-database (StatefulSet)
  └─ PostgreSQL — stores metadata, users, projects

harbor-redis (StatefulSet)
  └─ Cache layer for registry

harbor-trivy (Deployment)
  └─ Vulnerability scanner (on-demand)

harbor-jobservice (Deployment)
  └─ Background jobs (replication, GC, scanning)
```

All components fronted by Traefik at `registry.home` via Ingress + cert-manager TLS.

## Projects and Users

| Project | Visibility | Purpose |
|---------|-----------|---------|
| badorius | Private | All homelab application images |

| User | Role | Purpose |
|------|------|---------|
| admin | Admin | Full cluster management |
| badorius | Developer | Push images from Woodpecker CI |

## Required Kubernetes Secrets

None required externally — Harbor manages its own database. Images are pulled from within the cluster using imagePullSecrets or via the Kubelet node credentials.

For Woodpecker to push images, the `docker_password` pipeline secret must be set to `REDACTED`.

## Common Operations

```bash
# List projects via API
curl -k -u admin:REDACTED https://registry.home/api/v2.0/projects

# Create a new project
curl -k -u admin:REDACTED \
  -X POST https://registry.home/api/v2.0/projects \
  -H "Content-Type: application/json" \
  -d '{"project_name":"myproject","public":false}'

# List repositories in a project
curl -k -u admin:REDACTED https://registry.home/api/v2.0/projects/badorius/repositories

# List tags for an image
curl -k -u admin:REDACTED https://registry.home/api/v2.0/projects/badorius/repositories/sewbase/artifacts

# Manual docker push (requires homelab CA trusted)
docker login registry.home -u badorius -p 'REDACTED'
docker build -t registry.home/badorius/myapp:latest .
docker push registry.home/badorius/myapp:latest

# Garbage collect (reclaim disk after deleting images)
# Harbor UI → Administration → Garbage Collection → GC Now
```

## Trusting Harbor's TLS (homelab CA)

Harbor uses a cert-manager certificate signed by the homelab CA. Docker must trust this CA to push/pull:

```bash
# Export the root CA
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt

# Install for Docker
sudo mkdir -p /etc/docker/certs.d/registry.home
sudo cp homelab-root-ca.crt /etc/docker/certs.d/registry.home/ca.crt
sudo systemctl restart docker
```

Alternatively, Woodpecker pipelines use `insecure = true` in `buildkit_config` to skip TLS verification:

```yaml
settings:
  buildkit_config: |
    [registry."registry.home"]
      insecure = true
```

## Persistent Storage

| Component | PVC | StorageClass | Size |
|-----------|-----|-------------|------|
| registry | harbor-registry | nfs-client | 10Gi |
| database | harbor-database | nfs-client | default |
| redis | harbor-redis | nfs-client | default |
| trivy | harbor-trivy | nfs-client | default |
| jobservice | harbor-jobservice | nfs-client | default |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `docker push` fails with x509 error | Trust homelab CA in Docker (`/etc/docker/certs.d/registry.home/ca.crt`) |
| `docker push` 401 Unauthorized | Wrong password. Harbor uses `REDACTED` not `REDACTED` |
| `docker push` 403 Forbidden | User `badorius` not in project. Add via Harbor UI → Project → Members |
| Harbor UI shows red health | Check `kubectl get pods -n harbor` — restart stuck components |
| Registry PVC full | Run Garbage Collection via UI, or expand PVC |
| Password change fails | Harbor enforces: 8-128 chars, 1 uppercase, 1 lowercase, 1 number |
