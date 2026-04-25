# Fresh Deploy Runbook

Complete from-scratch deployment of the entire homelab. Follow this if you are building from zero or recovering from a catastrophic failure.

---

## Prerequisites

- Arch Linux installed on arch01/02/03 with BTRFS root
- SSH key-based access configured (`~/.ssh/id_rsa`)
- OpenWrt router at 192.168.8.253 with SSH root access
- OMV NAS at 192.168.8.15 with NFS configured
- Local machine has: `ansible`, `kubectl`, `helm`, `argocd` CLI tools

---

## Phase 1 — Provision Physical Infrastructure (Ansible)

```bash
cd ansible

# Provision all Arch hosts + KVM + bridge network + backups + VMs + K3s
ansible-playbook -i inventory/hosts.yml site.yml
```

This runs in order:
1. `setup_network.yml` — OpenWrt DNS entries
2. `setup_hosts.yml` — Arch packages, KVM, bridge networking
3. `setup_backups.yml` — BTRFS timers + K3s ETCD backup
4. `setup_vms.yml` — Ubuntu VMs via Cloud-Init
5. `setup_k3s.yml` — K3s master + workers

After completion, `k3s.yaml` is fetched to `ansible/k3s.yaml` (gitignored).

```bash
export KUBECONFIG=$(pwd)/k3s.yaml
kubectl get nodes   # should show vm01/02/03 Ready
```

---

## Phase 2 — Bootstrap Kubernetes Infrastructure

### 2a. Install NFS Provisioner

```bash
ansible-playbook playbooks/setup_storage.yml
```

### 2b. Install cert-manager

```bash
kubectl apply -k kubernetes/infra/cert-manager/
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager --timeout=120s
```

### 2c. Export and Trust the Root CA

```bash
# Wait ~30s for cert-manager to issue the CA
sleep 30
kubectl get certificate homelab-ca -n cert-manager

# Export the root CA
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt

# Copy it to the Ansible common role for distribution
cp homelab-root-ca.crt ansible/roles/common/files/homelab-root-ca.crt

# Distribute to all hosts and VMs
ansible-playbook playbooks/setup_hosts.yml --tags ca_trust
ansible-playbook playbooks/setup_k3s.yml --tags ca_trust

# Trust on local Arch workstation
sudo cp homelab-root-ca.crt /etc/ca-certificates/trust-source/anchors/
sudo update-ca-trust

# Trust in Docker (for pushing images to Harbor)
sudo mkdir -p /etc/docker/certs.d/registry.home
sudo cp homelab-root-ca.crt /etc/docker/certs.d/registry.home/ca.crt
sudo systemctl restart docker
```

### 2d. Install ArgoCD

```bash
kubectl apply -k kubernetes/infra/argocd/
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### 2e. Enable Traefik Dashboard

```bash
kubectl apply -k kubernetes/infra/traefik/
```

---

## Phase 3 — Inject Secrets

**Never put real values in any file. Run these commands manually.**

```bash
# Gitea admin credentials
kubectl create namespace gitea
kubectl create secret generic gitea-admin-secret \
  --namespace gitea \
  --from-literal=username=gitea_admin \
  --from-literal=password='YOUR_SECURE_PASSWORD'

# Woodpecker (placeholder values first — update after Gitea OAuth app is created)
kubectl create namespace woodpecker
kubectl create secret generic woodpecker-server-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_GITEA_CLIENT='placeholder' \
  --from-literal=WOODPECKER_GITEA_SECRET='placeholder' \
  --from-literal=WOODPECKER_AGENT_SECRET='YOUR_SHARED_TOKEN'

kubectl create secret generic woodpecker-agent-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET='YOUR_SHARED_TOKEN'

# Woodpecker CA ConfigMap (needed for Harbor push)
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/homelab-ca.crt
kubectl create configmap homelab-ca-cert -n woodpecker \
  --from-file=homelab-root-ca.crt=/tmp/homelab-ca.crt

# Grafana admin credentials
kubectl create namespace monitoring
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='YOUR_SECURE_PASSWORD'

# Sewbase namespace and secrets
kubectl create namespace sewbase
kubectl create secret generic sewbase-db-secrets \
  --namespace sewbase \
  --from-literal=postgres-password='YOUR_DB_PASSWORD'

kubectl create secret generic sewbase-app-secrets \
  --namespace sewbase \
  --from-literal=DATABASE_URL='postgresql://sewuser:YOUR_DB_PASSWORD@postgres:5432/sewbase' \
  --from-literal=AUTH_SECRET='YOUR_AUTH_SECRET' \
  --from-literal=MINIO_ENDPOINT='http://openmediavault.home:9000' \
  --from-literal=MINIO_ACCESS_KEY='admin' \
  --from-literal=MINIO_SECRET_KEY='YOUR_MINIO_SECRET'
```

---

## Phase 4 — Deploy ArgoCD App-of-Apps

This deploys Gitea, Woodpecker, Harbor, and Sewbase:

```bash
kubectl apply -f kubernetes/infra/argocd/applications.yaml
```

Watch ArgoCD sync:
```bash
kubectl get applications -n argocd -w
```

Gitea, Woodpecker, and Harbor will start deploying. This takes 3-5 minutes.

---

## Phase 5 — Configure Gitea OAuth for Woodpecker

1. Login to `https://gitea.home` as `gitea_admin`
2. Go to **Settings → Applications → Manage OAuth2 Applications**
3. Create application:
   - Name: `Woodpecker CI`
   - Redirect URI: `https://ci.home/authorize`
4. Copy the **Client ID** and **Client Secret**

Update Woodpecker secret with real values:

```bash
kubectl patch secret woodpecker-server-secret -n woodpecker \
  -p="{\"data\":{
    \"WOODPECKER_GITEA_CLIENT\": \"$(echo -n 'YOUR_CLIENT_ID' | base64)\",
    \"WOODPECKER_GITEA_SECRET\": \"$(echo -n 'YOUR_CLIENT_SECRET' | base64)\"
  }}"

kubectl rollout restart statefulset woodpecker-server -n woodpecker
```

---

## Phase 6 — Configure Harbor

1. Login to `https://registry.home` as `admin`
2. Change admin password (must have uppercase, e.g., `REDACTED`)
3. Create user `badorius` with password `REDACTED`
4. Create project `badorius` (private)
5. Add `badorius` as Developer to the `badorius` project

Create pull secret for sewbase:

```bash
kubectl create secret docker-registry harbor-pull-secret \
  --namespace sewbase \
  --docker-server=registry.home \
  --docker-username=badorius \
  --docker-password='YOUR_HARBOR_PASSWORD'
```

---

## Phase 7 — Woodpecker Pipeline Secrets

1. Login to `https://ci.home` (Gitea OAuth)
2. **Settings → Token** → generate personal access token
3. Set pipeline secrets:

```bash
export WOODPECKER_SERVER=https://ci.home
export WOODPECKER_TOKEN=<your-token>

# Generate ArgoCD token first
argocd login argocd.home --insecure --username admin --password '<argo-pass>' --grpc-web
ARGOCD_TOKEN=$(argocd account generate-token --account admin --grpc-web)

# Set Woodpecker global secrets
/tmp/woodpecker-cli secret add --global --name docker_password --value 'YOUR_HARBOR_PASSWORD'
/tmp/woodpecker-cli secret add --global --name argocd_server --value argocd.home
/tmp/woodpecker-cli secret add --global --name argocd_token --value "$ARGOCD_TOKEN"
```

---

## Phase 8 — Deploy Manual Services

```bash
# Monitoring (Prometheus + Grafana)
ansible-playbook playbooks/setup_monitoring_agents.yml

# Homepage dashboard
kubectl apply -k kubernetes/services/homepage/

# qBittorrent
kubectl apply -k kubernetes/services/qbittorrent/

# Calibre-Web
kubectl apply -k kubernetes/services/calibre/
```

---

## Phase 9 — Build and Deploy Sewbase

```bash
# Build the sewbase image manually (first time)
cd /path/to/sewbase_guitea
docker build -t registry.home/badorius/sewbase:latest .
echo "YOUR_HARBOR_PASSWORD" | docker login registry.home -u badorius --password-stdin
docker push registry.home/badorius/sewbase:latest

# Register repo in ArgoCD (for private Gitea access)
argocd repo add https://gitea.home/gitea_admin/sewbase_guitea.git \
  --username gitea_admin \
  --password 'YOUR_GITEA_PASSWORD' \
  --insecure-skip-server-verification \
  --grpc-web

# Trigger ArgoCD sync
argocd app sync sewbase --grpc-web
```

---

## Phase 10 — Verify Everything

```bash
# All pods running
kubectl get pods -A

# All certificates issued
kubectl get certificates -A

# All ingresses with addresses
kubectl get ingress -A

# ArgoCD apps healthy
kubectl get applications -n argocd
```

Expected: all certificates `READY=True`, all ArgoCD apps `Synced/Healthy`.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Pod in ImagePullBackOff | Check harbor-pull-secret. Verify image exists in Harbor |
| ArgoCD stuck syncing | `argocd app sync <name> --force --grpc-web` |
| Certificate not Ready | `kubectl describe certificate <name> -n <ns>` → check CertificateRequest events |
| Gitea CrashLoop after restart | LevelDB lock. Scale to 0 then back to 1 |
| Woodpecker "Client ID not registered" | Recreate OAuth app in Gitea and update woodpecker-server-secret |
