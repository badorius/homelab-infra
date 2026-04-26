# Runbook: Add a New CI/CD Pipeline

This runbook covers setting up a full CI/CD pipeline for a new application using Woodpecker CI + Harbor + ArgoCD.

---

## Architecture of a Pipeline

```
Developer push to Gitea
        │
        ▼
Woodpecker CI (.woodpecker.yml)
  ├─ test     → run unit tests
  ├─ publish  → docker buildx → Harbor registry
  └─ deploy   → ArgoCD sync (HTTP POST to ArgoCD API)
                    │
                    ▼
               ArgoCD applies K8s manifests
               → K3s pulls image from Harbor
               → App updated
```

---

## Step 1 — Create the App Repo in Gitea

If the repo doesn't exist yet:

```bash
# Via Gitea API
curl -k -u gitea_admin:YOUR_PASSWORD \
  -X POST https://gitea.home/api/v1/user/repos \
  -H "Content-Type: application/json" \
  -d '{"name":"myapp","private":true,"description":"My app"}'
```

Or use the Gitea UI at `https://gitea.home`.

---

## Step 2 — Add Kubernetes Manifests to the App Repo

Create an `infra/k8s/myapp/` directory in the app repo. Minimum files:

**`infra/k8s/myapp/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      imagePullSecrets:
        - name: harbor-pull-secret
      containers:
        - name: myapp
          image: registry.home/badorius/myapp:latest
          ports:
            - containerPort: 3000
          envFrom:
            - secretRef:
                name: myapp-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myapp.home
      secretName: myapp-tls
  rules:
    - host: myapp.home
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

---

## Step 3 — Create the Woodpecker Pipeline File

Create `.woodpecker.yml` in the root of the app repo:

```yaml
steps:
  test:
    image: node:20-alpine          # adjust for your stack
    commands:
      - npm ci
      - npm run test -- --passWithNoTests

  publish:
    image: woodpeckerci/plugin-docker-buildx
    privileged: true
    settings:
      registry: registry.home
      repo: registry.home/badorius/myapp
      tags: latest
      username: badorius
      password:
        from_secret: docker_password
      buildkit_config: |
        [registry."registry.home"]
          insecure = true
    when:
      branch: main
      event: push

  deploy:
    image: curlimages/curl:latest
    environment:
      ARGOCD_SERVER:
        from_secret: argocd_server
      ARGOCD_TOKEN:
        from_secret: argocd_token
    commands:
      - |
        curl -k -s -f -X POST \
          "https://${ARGOCD_SERVER}/api/v1/applications/myapp/sync" \
          -H "Authorization: Bearer ${ARGOCD_TOKEN}" \
          -H "Content-Type: application/json" \
          -d '{"hardRefresh":true}' \
          && echo "ArgoCD sync triggered" \
          || (echo "ArgoCD sync failed" && exit 1)
    when:
      branch: main
      event: push
```

> The `buildkit_config` block is required for Harbor's self-signed TLS certificate.
> `insecure = true` in buildkitd context means "skip TLS verification" (not HTTP-only).

---

## Step 4 — Set Up Harbor

Create a Harbor project for your app (if it doesn't exist):

```bash
# Via API (Harbor admin)
curl -k -u admin:$(pass-cli item view --vault-name homelab --item-title harbor-admin --field password) \
  -X POST https://registry.home/api/v2.0/projects \
  -H "Content-Type: application/json" \
  -d '{"project_name":"myapp-project","public":false}'

# Add badorius user to the project as Developer
PROJ_ID=<id from above response>
curl -k -u admin:$(pass-cli item view --vault-name homelab --item-title harbor-admin --field password) \
  -X POST "https://registry.home/api/v2.0/projects/${PROJ_ID}/members" \
  -H "Content-Type: application/json" \
  -d '{"role_id":2,"member_user":{"username":"badorius"}}'
```

---

## Step 5 — Inject Required Kubernetes Secrets

```bash
# Create the app namespace
kubectl create namespace myapp

# Harbor pull secret (so K3s can pull the image)
kubectl create secret docker-registry harbor-pull-secret \
  --namespace myapp \
  --docker-server=registry.home \
  --docker-username=badorius \
  --docker-password="$(pass-cli item view --vault-name homelab --item-title harbor-badorius --field password)"

# App-specific secrets
kubectl create secret generic myapp-secrets \
  --namespace myapp \
  --from-literal=DATABASE_URL='postgresql://user:password@host/db' \
  --from-literal=SECRET_KEY='your-secret-key'
```

---

## Step 6 — Register the App in ArgoCD

ArgoCD must be able to clone the Gitea repo (it's private):

```bash
argocd repo add https://gitea.home/gitea_admin/myapp.git \
  --username gitea_admin \
  --password 'YOUR_GITEA_PASSWORD' \
  --insecure-skip-server-verification \
  --grpc-web
```

Then add the ArgoCD Application to `kubernetes/infra/argocd/applications.yaml` in this repo:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://gitea.home/gitea_admin/myapp.git'
    targetRevision: HEAD
    path: infra/k8s/myapp
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Commit and push so ArgoCD picks it up.

---

## Step 7 — Configure Woodpecker Secrets

These secrets are global (shared across all pipelines) and must be set once:

| Secret | Value | Description |
|--------|-------|-------------|
| `docker_password` | Harbor password for badorius | Push images to Harbor |
| `argocd_server` | `argocd.home` | ArgoCD API host |
| `argocd_token` | ArgoCD API token | Trigger syncs |

If not already set:

```bash
# Get Woodpecker personal token from https://ci.home → Settings → Token
export WOODPECKER_SERVER=https://ci.home
export WOODPECKER_TOKEN=<your-token>

# Generate ArgoCD token
argocd login argocd.home --insecure --username admin --password '<pass>' --grpc-web
ARGOCD_TOKEN=$(argocd account generate-token --account admin --grpc-web)

# Set global secrets via woodpecker-cli
/tmp/woodpecker-cli secret add --global --name docker_password --value "$(pass-cli item view --vault-name homelab --item-title harbor-badorius --field password)"
/tmp/woodpecker-cli secret add --global --name argocd_server --value argocd.home
/tmp/woodpecker-cli secret add --global --name argocd_token --value "$ARGOCD_TOKEN"
```

---

## Step 8 — Add DNS Entry

```yaml
# In ansible/roles/openwrt-dns/tasks/main.yml:
    - { name: "myapp.home", ip: "192.168.8.248" }
```

```bash
cd ansible && ansible-playbook playbooks/setup_network.yml
```

---

## Step 9 — First Manual Build and Push

Before the pipeline runs, the image must exist in Harbor at least once:

```bash
cd /path/to/myapp
docker build -t registry.home/badorius/myapp:latest .
pass-cli item view --vault-name homelab --item-title harbor-badorius --field password | docker login registry.home -u badorius --password-stdin
docker push registry.home/badorius/myapp:latest
```

---

## Step 10 — Activate Repo in Woodpecker and Test

1. Login to `https://ci.home`
2. Go to your repository → **Activate**
3. Trigger a test pipeline: `git commit --allow-empty -m "test pipeline" && git push`
4. Watch the pipeline at `https://ci.home`

---

## Pipeline Debugging

```bash
# Check Woodpecker agent logs
kubectl logs -n woodpecker woodpecker-agent-0 --tail=50

# Check Woodpecker server logs
kubectl logs -n woodpecker woodpecker-server-0 --tail=50

# Check ArgoCD app status after deploy step
kubectl get application myapp -n argocd -o jsonpath='{.status.sync.status}'

# Check pod is pulling new image
kubectl describe pod -n myapp -l app=myapp | grep -A10 Events
```

---

## Checklist

- [ ] App repo created in Gitea
- [ ] `infra/k8s/myapp/` manifests created (deployment, service, ingress with TLS)
- [ ] `.woodpecker.yml` created with test/publish/deploy steps
- [ ] Harbor project and badorius user configured
- [ ] `harbor-pull-secret` created in app namespace
- [ ] App secrets injected via `kubectl create secret`
- [ ] ArgoCD can access the Gitea repo (`argocd repo add`)
- [ ] ArgoCD Application added to `applications.yaml`
- [ ] Woodpecker secrets set (`docker_password`, `argocd_server`, `argocd_token`)
- [ ] DNS entry added and applied
- [ ] First manual image build + push done
- [ ] Repo activated in Woodpecker UI
- [ ] Pipeline runs successfully end-to-end
