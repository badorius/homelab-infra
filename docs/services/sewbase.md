# Sewbase — Service Documentation & Deploy Runbook

Sewbase is a custom Next.js web application deployed via the full CI/CD pipeline (Gitea → Woodpecker → Harbor → ArgoCD → K3s).

---

## Overview

| Property | Value |
|----------|-------|
| **URL** | https://sewbase.home |
| **Tech stack** | Next.js 14 (App Router), Prisma ORM, PostgreSQL, NextAuth.js v5 |
| **Source repo** | `https://gitea.home/gitea_admin/sewbase_guitea` (private) |
| **Local mirror** | `/home/darthv/git/badorius/sewbase_guitea/` |
| **Container image** | `registry.home/badorius/sewbase:latest` |
| **Namespace** | `sewbase` |
| **Database** | PostgreSQL (StatefulSet in the same namespace) |

---

## How Deployment Works

```
git push → gitea.home/gitea_admin/sewbase_guitea (main branch)
       │
       ▼ (Gitea webhook)
Woodpecker CI pipeline (.woodpecker.yml):
  1. test    → npm ci + npm test (node:20-alpine)
  2. publish → docker buildx → registry.home/badorius/sewbase:latest
  3. deploy  → curl POST to ArgoCD API → hard refresh + sync sewbase app
       │
       ▼ (ArgoCD sync)
ArgoCD applies infra/k8s/sewbase/ from gitea.home repo:
  - Deployment (pulls new image from Harbor)
  - StatefulSet (postgres)
  - Service + Ingress (sewbase.home, TLS via cert-manager)
       │
       ▼
New version live at https://sewbase.home
```

---

## Normal Development Workflow

### Making a code change and deploying

```bash
cd /home/darthv/git/badorius/sewbase_guitea

# 1. Make your changes
vim src/app/page.tsx

# 2. Test locally (optional)
npm run dev
npm test

# 3. Commit and push
git add -A
git commit -m "feat: your change description"
git push origin main

# That's it. The CI/CD pipeline handles everything automatically.
```

The pipeline takes ~3-5 minutes (build is the slow part). Monitor at `https://ci.home`.

### Checking pipeline status

```bash
# Pipeline runs visible at:
# https://ci.home → sewbase_guitea repository → Pipelines

# ArgoCD sync status:
kubectl get application sewbase -n argocd -o jsonpath='{.status.health.status}'
# Expected: Healthy

# Pod status:
kubectl get pods -n sewbase
# Expected: sewbase pod Running, postgres pod Running
```

---

## First-Time Setup (after fresh deploy or namespace wipe)

If the sewbase namespace is empty (fresh cluster or after `kubectl delete namespace sewbase`), follow these steps:

### 1. Inject Kubernetes Secrets

These must exist before ArgoCD syncs the app:

```bash
# Create namespace
kubectl create namespace sewbase

# PostgreSQL password
kubectl create secret generic sewbase-db-secrets \
  --namespace sewbase \
  --from-literal=postgres-password='YOUR_DB_PASSWORD'

# App secrets
kubectl create secret generic sewbase-app-secrets \
  --namespace sewbase \
  --from-literal=DATABASE_URL='postgresql://sewuser:YOUR_DB_PASSWORD@postgres:5432/sewbase' \
  --from-literal=AUTH_SECRET='YOUR_AUTH_SECRET' \
  --from-literal=MINIO_ENDPOINT='http://openmediavault.home:9000' \
  --from-literal=MINIO_ACCESS_KEY='admin' \
  --from-literal=MINIO_SECRET_KEY='YOUR_MINIO_SECRET'

# Harbor pull secret (so K3s can pull the image)
kubectl create secret docker-registry harbor-pull-secret \
  --namespace sewbase \
  --docker-server=registry.home \
  --docker-username=badorius \
  --docker-password="$(pass-cli item view --vault-name homelab --item-title harbor-badorius --field password)"
```

### 2. Register Gitea Repo in ArgoCD

ArgoCD needs credentials to clone the private Gitea repo:

```bash
argocd repo add https://gitea.home/gitea_admin/sewbase_guitea.git \
  --username gitea_admin \
  --password 'YOUR_GITEA_PASSWORD' \
  --insecure-skip-server-verification \
  --grpc-web
```

### 3. Build the Image (first time only)

The pipeline builds the image on every push. But for the first deployment, the image doesn't exist yet. Build it manually:

```bash
# Ensure Docker trusts the homelab CA (one-time setup on workstation)
sudo mkdir -p /etc/docker/certs.d/registry.home
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/homelab-root-ca.crt
sudo cp /tmp/homelab-root-ca.crt /etc/docker/certs.d/registry.home/ca.crt
sudo systemctl restart docker

# Build and push
cd /home/darthv/git/badorius/sewbase_guitea
docker build -t registry.home/badorius/sewbase:latest .
pass-cli item view --vault-name homelab --item-title harbor-badorius --field password | docker login registry.home -u badorius --password-stdin
docker push registry.home/badorius/sewbase:latest
```

### 4. Trigger ArgoCD to Deploy

ArgoCD automatically syncs (it has `automated.selfHeal: true`). But you can force it:

```bash
argocd app sync sewbase --grpc-web
```

### 5. Configure Woodpecker Secrets (for future automated builds)

In Woodpecker UI (`https://ci.home`) or via CLI:

```bash
export WOODPECKER_SERVER=https://ci.home
export WOODPECKER_TOKEN=<your-token-from-ci.home-settings>

# Generate ArgoCD token
argocd login argocd.home --insecure --username admin \
  --password "$(kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath='{.data.password}' | base64 -d)" --grpc-web
ARGO_TOKEN=$(argocd account generate-token --account admin --grpc-web)

# Set secrets (global — available to all pipelines)
/tmp/woodpecker-cli secret add --global --name docker_password --value "$(pass-cli item view --vault-name homelab --item-title harbor-badorius --field password)"
/tmp/woodpecker-cli secret add --global --name argocd_server --value argocd.home
/tmp/woodpecker-cli secret add --global --name argocd_token --value "$ARGO_TOKEN"
```

Then activate the repo in Woodpecker UI:
1. Go to `https://ci.home`
2. Login with Gitea OAuth (`gitea_admin`)
3. Find `sewbase_guitea` → click **Activate**

### 6. Verify

```bash
kubectl get pods -n sewbase
# NAME                        READY   STATUS    RESTARTS
# sewbase-<hash>-<hash>       1/1     Running   0
# postgres-0                  1/1     Running   0

kubectl get certificate sewbase-tls -n sewbase
# READY=True

# Test the URL
curl -k https://sewbase.home
```

---

## Kubernetes Manifests (in sewbase_guitea repo)

The K8s manifests live in `infra/k8s/sewbase/` inside the sewbase_guitea repository. ArgoCD reads them from there.

| File | Contents |
|------|---------|
| `app.yaml` | Deployment (sewbase app), Service, Ingress (with TLS) |
| `postgres.yaml` | StatefulSet (postgres), Service, PVC (10Gi nfs-client) |
| `secrets.example.yaml` | Template showing what secrets are needed (no real values) |

### app.yaml structure

```yaml
# Deployment — pulls registry.home/badorius/sewbase:latest
# imagePullSecrets: harbor-pull-secret
# envFrom: sewbase-app-secrets (DATABASE_URL, AUTH_SECRET, etc.)

# Service — port 80 → containerPort 3000

# Ingress — sewbase.home, cert-manager homelab-ca TLS, ingressClassName: traefik
```

---

## Database

Sewbase uses PostgreSQL running as a StatefulSet in the same namespace.

| Property | Value |
|----------|-------|
| Image | `public.ecr.aws/bitnami/postgresql:16` |
| Database name | `sewbase` |
| Username | `sewuser` |
| Password | From `sewbase-db-secrets.postgres-password` |
| Storage | 10Gi NFS PVC |

### Access the Database

```bash
kubectl exec -it -n sewbase postgres-0 -- psql -U sewuser -d sewbase
```

### Run Prisma Migrations

Migrations run automatically during app startup (check Dockerfile CMD for details).

To run manually inside the pod:
```bash
kubectl exec -it -n sewbase deploy/sewbase -- npx prisma migrate deploy
```

---

## Troubleshooting

### Pod in ImagePullBackOff

```bash
kubectl describe pod -n sewbase -l app=sewbase | grep -A 10 Events
```

Common causes:
- `harbor-pull-secret` missing or wrong password → recreate it
- Image doesn't exist in Harbor → run the manual build (Step 3 above)
- Harbor unreachable → check `https://registry.home`

### App Crashes on Startup

```bash
kubectl logs -n sewbase -l app=sewbase --tail=50
```

Common causes:
- `DATABASE_URL` wrong in `sewbase-app-secrets` → update secret and restart pod
- PostgreSQL not ready → check `kubectl get pods -n sewbase`
- Missing env var → check `kubectl get secret sewbase-app-secrets -n sewbase -o yaml`

### Woodpecker Pipeline Fails

Check pipeline logs at `https://ci.home` → sewbase_guitea → Pipelines.

Common causes:
- `docker_password` secret wrong → update in Woodpecker settings
- Harbor push 401 → verify `badorius` user exists in Harbor with correct password
- ArgoCD token expired → regenerate and update `argocd_token` secret
- Tests failing → check npm test output in the `test` step

### ArgoCD Cannot Clone Gitea Repo

```bash
argocd app get sewbase --grpc-web | grep message
```

If "authentication required": re-add the repo credentials:
```bash
argocd repo add https://gitea.home/gitea_admin/sewbase_guitea.git \
  --username gitea_admin \
  --password 'YOUR_GITEA_PASSWORD' \
  --insecure-skip-server-verification \
  --grpc-web --upsert
```
