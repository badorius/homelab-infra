# Runbook: Add a New Service

This runbook covers the complete process for adding a new service to the homelab K3s cluster. Follow every step in order.

---

## Decision: ArgoCD-managed or Manual?

| Choose ArgoCD-managed | Choose Manual |
|----------------------|---------------|
| Service changes frequently | One-time deploy, rarely updated |
| Part of CI/CD flow | Infrastructure tool or dashboard |
| Helm chart with many config options | Simple deployment (no Helm) |

Examples of ArgoCD-managed: Gitea, Woodpecker, Harbor, Sewbase.
Examples of manual: Homepage, qBittorrent, Calibre-Web, Traefik dashboard.

---

## Step 1 — Create Kubernetes Manifests

Create a directory under `kubernetes/services/<servicename>/`.

### Minimum required files:

**`namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myservice
```

**`kustomization.yaml`** (simple, no Helm)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myservice

resources:
  - namespace.yaml
  - deployment.yaml
```

**`deployment.yaml`** (example)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myservice
  namespace: myservice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myservice
  template:
    metadata:
      labels:
        app: myservice
    spec:
      containers:
        - name: myservice
          image: vendor/myservice:latest
          ports:
            - containerPort: 8080
          env:
            - name: TZ
              value: Europe/Madrid
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
  namespace: myservice
spec:
  selector:
    app: myservice
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myservice
  namespace: myservice
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myservice.home
      secretName: myservice-tls
  rules:
    - host: myservice.home
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myservice
                port:
                  number: 80
```

### If the service needs persistent storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myservice-data-pvc
  namespace: myservice
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 10Gi
```

Then add to the Deployment:
```yaml
spec:
  template:
    spec:
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: myservice-data-pvc
      containers:
        - name: myservice
          volumeMounts:
            - name: data
              mountPath: /data
```

### If the service uses a Helm chart (ArgoCD-managed):

**`kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myservice

resources:
  - namespace.yaml

helmCharts:
  - name: chart-name
    repo: https://charts.example.com/
    version: 1.0.0
    releaseName: myservice
    namespace: myservice
    valuesInline:
      ingress:
        enabled: true
        className: traefik
        annotations:
          cert-manager.io/cluster-issuer: homelab-ca
        hosts:
          - host: myservice.home
            paths:
              - path: /
        tls:
          - hosts:
              - myservice.home
            secretName: myservice-tls
      persistence:
        storageClass: nfs-client
```

---

## Step 2 — Add DNS Entry

Edit `ansible/roles/openwrt-dns/tasks/main.yml` and add:

```yaml
    - { name: "myservice.home", ip: "192.168.8.248" }
```

Apply:
```bash
cd ansible && ansible-playbook playbooks/setup_network.yml
```

---

## Step 3 — Inject Secrets (if needed)

If the service requires credentials, create the namespace first and inject secrets:

```bash
kubectl create namespace myservice
kubectl create secret generic myservice-credentials \
  --namespace myservice \
  --from-literal=username='admin' \
  --from-literal=password='YOUR_PASSWORD'
```

**Never put secret values in any file in this repository.**

---

## Step 4 — Deploy

### Option A: Manual deploy (kubectl)

```bash
kubectl apply -k kubernetes/services/myservice/
```

### Option B: ArgoCD-managed

Add an Application entry to `kubernetes/infra/argocd/applications.yaml`:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myservice
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/badorius/homelab-infra.git'
    targetRevision: HEAD
    path: kubernetes/services/myservice
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: myservice
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Then commit and push — ArgoCD will deploy automatically:

```bash
git add kubernetes/services/myservice/ kubernetes/infra/argocd/applications.yaml \
        ansible/roles/openwrt-dns/tasks/main.yml
git commit -m "feat: add myservice"
git push
```

---

## Step 5 — Verify

```bash
# Pod is running
kubectl get pods -n myservice

# Certificate issued
kubectl get certificate myservice-tls -n myservice
# READY should be True within ~30 seconds

# Ingress has addresses
kubectl get ingress -n myservice

# Service responds
curl -k https://myservice.home
```

---

## Step 6 — Update Homepage Dashboard (optional)

Edit `kubernetes/services/homepage/configmap.yaml` and add an entry under the appropriate category:

```yaml
    - My Category:
        - MyService:
            icon: myservice
            href: https://myservice.home
            description: What the service does
```

Apply:
```bash
kubectl apply -k kubernetes/services/homepage/
```

---

## Step 7 — Document the Service

1. Add a section in `agents.md` under §6 (Service Runbooks)
2. Create `docs/services/myservice.md` with operational details
3. Update the endpoints table in `README.md`

---

## Checklist

- [ ] Manifests created in `kubernetes/services/myservice/`
- [ ] Ingress uses `ingressClassName: traefik` and `cert-manager.io/cluster-issuer: homelab-ca`
- [ ] PVCs use `storageClassName: nfs-client`
- [ ] DNS entry added to `openwrt-dns` role and applied
- [ ] Secrets injected via `kubectl create secret` (not in git)
- [ ] Deployed (manual or ArgoCD)
- [ ] Certificate `READY=True`
- [ ] Service accessible at `https://myservice.home`
- [ ] Homepage updated (optional)
- [ ] Service documented in `agents.md` and `README.md`
