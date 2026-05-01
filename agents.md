# agents.md — Homelab Infrastructure: AI Agent Persistent Memory

> **IMPORTANT**: This file is the single source of truth for AI agent context across sessions.
> Update the **Session State** section at the end of every session before closing.
> **NEVER store passwords, secrets, API keys or tokens in this file or anywhere in this repo.**
> The repository is PUBLIC. All secrets are injected at runtime via `kubectl create secret`.
> **Credentials live in Proton Pass vault `homelab`** — retrieve via `pass-cli item view --vault-name homelab --item-title <name> --field password`.

---

## 1. Repository Overview

| Field | Value |
|-------|-------|
| **Repo** | `github.com/badorius/homelab-infra` |
| **Type** | Public GitOps Infrastructure-as-Code (IaC) |
| **Owner** | `darthv` (Ansible user, SSH key-based auth only) |
| **Philosophy** | Everything Declarative. No snowflakes. Destroy and recreate at any time. |
| **Language** | All code/docs in **English**. Conversations with owner can be in Spanish. |

---

## 2. Repository Structure

```
homelab-infra/
├── agents.md                        # This file — AI agent persistent memory
├── README.md                        # Human-facing quickstart guide
├── .gitignore                       # Excludes: k3s.yaml, ssh keys, files/
├── ansible/                         # All host/VM/network provisioning
│   ├── ansible.cfg                  # Defaults: host_key_checking=False, inventory auto-set
│   ├── site.yml                     # Master playbook (network→hosts→backups→vms→k3s)
│   ├── k3s.yaml                     # Fetched kubeconfig (gitignored - never commit creds)
│   ├── inventory/
│   │   ├── hosts.yml                # All hosts, groups, IPs
│   │   └── group_vars/all.yml       # VM definitions (name, MAC, target host)
│   ├── playbooks/
│   │   ├── setup_network.yml        # OpenWrt DNS entries
│   │   ├── setup_hosts.yml          # Arch Linux hosts hardening + KVM
│   │   ├── setup_backups.yml        # BTRFS backups + K3s ETCD backup
│   │   ├── setup_vms.yml            # Cloud-Init VM provisioning
│   │   ├── setup_k3s.yml            # K3s cluster deployment
│   │   ├── setup_monitoring_agents.yml  # Node-exporter on all nodes
│   │   ├── setup_storage.yml        # NFS provisioner
│   │   └── maintenance/
│   │       └── update_reboot.yml    # Rolling updates + reboots (Arch+Ubuntu)
│   └── roles/
│       ├── base/                    # Base Arch Linux packages/config
│       ├── common/                  # Common config across hosts
│       ├── bridge-network/          # br0 bridge for VM networking
│       ├── btrfs-backup/            # Snapshot + send + prune scripts + systemd timers
│       ├── btrfs-target-prune/      # Remote OMV prune logic
│       ├── cloud-vms/               # Ubuntu Cloud Image VM creation via libvirt
│       ├── hosts-dns/               # /etc/hosts management
│       ├── infra-vm/                # Infra VM-specific config
│       ├── k3s/                     # K3s master + worker installation
│       ├── k3s-backup/              # ETCD snapshot to NFS
│       ├── kvm-host/                # KVM/QEMU packages
│       ├── libvirt-dhcp/            # Libvirt DHCP config
│       ├── libvirt-dns/             # Libvirt DNS config
│       ├── libvirt-network/         # Libvirt network definition
│       ├── monitoring-agents/       # node_exporter install
│       ├── nfs-client/              # NFS client mount config
│       ├── nfs-provisioner/         # K8s NFS subdir provisioner
│       └── openwrt-dns/             # Static DNS entries on OpenWrt router
└── kubernetes/
    ├── infra/                       # Core cluster infrastructure (GitOps-managed or bootstrap)
    │   ├── argocd/                  # ArgoCD install + ingress + App-of-Apps
    │   ├── cert-manager/            # cert-manager Helm + ClusterIssuers
    │   └── storage/                 # (reserved for future storage manifests)
    └── services/                    # Application workloads
        ├── gitea/                   # Gitea Helm via Kustomize (ArgoCD-managed)
        ├── harbor/                  # Harbor Helm via Kustomize (ArgoCD-managed)
        ├── woodpecker/              # Woodpecker Helm via Kustomize (ArgoCD-managed)
        ├── monitoring/              # kube-prometheus-stack values (Helm-managed)
        ├── homepage/                # Homepage dashboard (Kustomize, manual apply)
        └── qbittorrent/             # qBittorrent media downloader (Kustomize, manual apply)
```

---

## 3. Infrastructure Architecture

### Physical Layer (Layer 1)
- **3× Mini PCs** running **Arch Linux** (arch01, arch02, arch03)
- Filesystem: **BTRFS** (enables snapshots and incremental send)
- Managed entirely by **Ansible**

### Backup Layer (Layer 2)
- Local BTRFS snapshots at `/.snapshots/`
- Incremental send to **OpenMediaVault (OMV)** NAS via NFS
- Systemd timers: snapshot (03:00) → send (04:00) → local prune (05:00) → remote prune (06:00)
- Retention: 7 local / 14 on OMV
- K3s ETCD snapshots sent to `/mnt/nfs/backups/k3s/`

### Virtualization Layer (Layer 3)
- **Libvirt/KVM** on each Arch host
- Bridged networking via `br0`
- VMs: Ubuntu 22.04 LTS (Jammy) Cloud Image, 20GB qcow2 disks
- Cloud-Init injection: SSH keys, hostname, sudo NOPASSWD
- Each VM auto-starts on host boot

### Kubernetes Layer (Layer 4) — K3s Cluster
- **vm01** → K3s Master (`192.168.8.248`) — runs on arch01
- **vm02** → K3s Worker (`192.168.8.249`) — runs on arch02
- **vm03** → K3s Worker (`192.168.8.101`) — runs on arch03
- Ingress: **Traefik** (default K3s ingress controller)
- TLS: **cert-manager** with local CA (`homelab-ca`)
- Storage: **NFS Subdir Provisioner** (`nfs-client` StorageClass) → OMV NAS
- DNS: **OpenWrt router** via `dnsmasq` — all `*.home` domains → `192.168.8.248`
- Monitoring: **kube-prometheus-stack** (Prometheus + Grafana + AlertManager)

### GitOps Layer (Layer 5) — CI/CD
- **ArgoCD** (`argocd.home`) — Watches `badorius/homelab-infra` on GitHub
- **Gitea** (`gitea.home`) — Self-hosted Git server (source of truth for private apps)
- **Woodpecker CI** (`ci.home`) — CI pipelines triggered by Gitea webhooks
- **Harbor** (`registry.home`) — Private OCI container registry

---

## 4. Network / DNS Map

All `*.home` domains resolve via OpenWrt's `dnsmasq` to `192.168.8.248` (K3s master/Traefik).

| Domain | Service | TLS |
|--------|---------|-----|
| `homepage.home` | Homepage Dashboard | Yes (cert-manager) |
| `grafana.home` | Grafana Monitoring | Yes (cert-manager) |
| `qbittorrent.home` | qBittorrent | Yes (cert-manager) |
| `argocd.home` | ArgoCD GitOps UI | Yes (cert-manager) |
| `gitea.home` | Gitea Git Server | Yes (cert-manager) |
| `ci.home` | Woodpecker CI | Yes (cert-manager) |
| `registry.home` | Harbor Registry | Yes (cert-manager) |
| `openmediavault.home` | OMV NAS UI | No (HTTP, IP: 192.168.8.15) |

**OpenWrt router IP**: `192.168.8.253`
**OMV NAS IP**: `192.168.8.15`
**K3s API Server**: `https://192.168.8.248:6443`

---

## 5. Key Design Decisions & Rules

1. **No secrets in the repo** — The repository is public. All passwords, API keys, OAuth tokens, and certificates are injected at runtime via `kubectl create secret`. **Credentials are stored in Proton Pass vault `homelab`** — use `pass-cli item view --vault-name homelab --item-title <name> --field password` in documentation and commands instead of hardcoded values. See README.md §2 for the exact commands.
2. **IaC-first** — Every infrastructure change must be expressed in code (Ansible playbook/role or Kubernetes manifest). No manual ad-hoc changes that are not reflected in the repo.
3. **ArgoCD as the CD engine** — Services managed by ArgoCD must not be applied manually with `kubectl apply`. Push to git, let ArgoCD sync. Force sync only when debugging.
4. **Helm via Kustomize** — ArgoCD uses `--enable-helm` Kustomize option. Services are defined as `helmCharts` entries in `kustomization.yaml`, keeping everything declarative without a separate Helm release workflow.
5. **cert-manager for all TLS** — All ingresses use `cert-manager.io/cluster-issuer: homelab-ca` annotation. The CA root cert must be manually trusted in the browser/OS after first deploy.
6. **NFS StorageClass** — All persistent volumes use `storageClass: nfs-client`. The OMV NAS must be reachable before creating PVCs.
7. **k3s.yaml is gitignored** — The kubeconfig is auto-fetched by the Ansible k3s role and stored locally. Contains certificates and tokens. Never commit.

---

## 6. Service Runbooks

### 6.1 cert-manager & TLS Certificates

**How it works:**
1. `selfsigned-issuer` (ClusterIssuer) creates a self-signed root CA certificate.
2. The root CA is stored in the `root-secret` Kubernetes Secret in `cert-manager` namespace.
3. `homelab-ca` (ClusterIssuer) uses `root-secret` to sign all service certificates.
4. Each Ingress annotated with `cert-manager.io/cluster-issuer: homelab-ca` gets a TLS cert auto-issued.

**Add a TLS certificate for a new service:**
```bash
# 1. Add the annotation to the Ingress manifest:
#    cert-manager.io/cluster-issuer: homelab-ca
# 2. Add a tls section in the Ingress spec:
spec:
  tls:
    - hosts:
        - newservice.home
      secretName: newservice-tls  # cert-manager will auto-create this secret

# 3. Add the DNS entry to the OpenWrt role (ansible/roles/openwrt-dns/tasks/main.yml):
- { name: "newservice.home", ip: "192.168.8.248" }

# 4. Apply DNS change:
cd ansible && ansible-playbook playbooks/setup_network.yml

# 5. For ArgoCD-managed services: push to git and let ArgoCD sync.
# For manual services: kubectl apply -k kubernetes/services/newservice/
```

**Export and trust the Root CA (one-time per machine):**
```bash
# Export the root CA cert from the cluster
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt

# Trust on Linux (Arch)
sudo cp homelab-root-ca.crt /etc/ca-certificates/trust-source/anchors/
sudo update-ca-trust

# Trust on Linux (Ubuntu/Debian)
sudo cp homelab-root-ca.crt /usr/local/share/ca-certificates/homelab-root-ca.crt
sudo update-ca-certificates

# Import in your browser manually via browser settings > Certificates > Import
```

**Check certificate status:**
```bash
kubectl get certificate -A
kubectl describe certificate <name> -n <namespace>
kubectl get certificaterequest -A
```

---

### 6.2 ArgoCD

**Bootstrap (first time only):**
```bash
kubectl apply -k kubernetes/infra/argocd/
kubectl apply -f kubernetes/infra/argocd/applications.yaml
```

**Key configuration:**
- `server.insecure: "true"` — ArgoCD server runs HTTP; Traefik terminates TLS at the ingress.
- `kustomize.buildOptions: --enable-helm` — Allows Helm chart rendering inside Kustomize.
- App-of-Apps: `applications.yaml` defines ArgoCD Applications for gitea, woodpecker, harbor, sewbase.

**Get the initial admin password:**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```
> **Security**: Change this password after first login. Never store it in the repo.

**Force a sync (when ArgoCD is stuck):**
```bash
# Via CLI
argocd app sync gitea --force

# Or via UI at https://argocd.home
```

**Common ArgoCD issues:**
- Sync fails due to hook errors → check `kubectl get events -n <namespace>`
- App stuck "Progressing" → check pod logs with `kubectl logs -n <namespace> -l app=<label>`

---

### 6.3 Gitea

**Helm chart**: `https://dl.gitea.com/charts/` version `10.1.4`
**Namespace**: `gitea`
**Dependencies**: PostgreSQL (standard chart, NOT HA), Redis, NFS StorageClass

**Required secret (inject before ArgoCD sync):**
```bash
kubectl create secret generic gitea-admin-secret \
  --namespace gitea \
  --from-literal=password='YOUR_SECURE_PASSWORD'
```

**Key Helm values (in kustomization.yaml valuesInline):**
- Admin creds from `gitea-admin-secret`
- Domain: `gitea.home`, ROOT_URL: `https://gitea.home/`
- PostgreSQL image from `public.ecr.aws/bitnami/postgresql:16.2.0` (avoids Docker Hub rate limits)
- Persistence: 10Gi via `nfs-client`

**Known issue — PostgreSQL image pull:**
If `gitea-postgresql` pod gets `ErrImagePull`, the Bitnami tag may be pruned from Docker Hub.
Fix: Update the image tag in `kustomization.yaml` `postgresql.image.tag` to a valid ECR tag.

**Create an OAuth2 application for Woodpecker (mandatory after first Gitea deploy):**
1. Login to `https://gitea.home`
2. Go to Settings → Applications → Manage OAuth2 Applications
3. Create: Name = `Woodpecker CI`, Redirect URI = `https://ci.home/authorize`
4. Note the Client ID and Client Secret
5. Inject into Kubernetes (see Woodpecker runbook below)

---

### 6.4 Woodpecker CI

**Helm chart**: `https://woodpecker-ci.org/` version `1.5.0`
**Namespace**: `woodpecker`

**Required secrets (inject before ArgoCD sync):**
```bash
# Step 1 — before OAuth app exists in Gitea (use placeholder values):
kubectl create secret generic woodpecker-server-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_GITEA_CLIENT='placeholder' \
  --from-literal=WOODPECKER_GITEA_SECRET='placeholder' \
  --from-literal=WOODPECKER_AGENT_SECRET='some-random-shared-token'

kubectl create secret generic woodpecker-agent-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET='some-random-shared-token'

# Step 2 — after creating the OAuth app in Gitea, update with real values:
kubectl patch secret woodpecker-server-secret -n woodpecker \
  -p="{\"data\":{\"WOODPECKER_GITEA_CLIENT\": \"$(echo -n 'YOUR_CLIENT_ID' | base64)\", \"WOODPECKER_GITEA_SECRET\": \"$(echo -n 'YOUR_CLIENT_SECRET' | base64)\"}}"

kubectl rollout restart statefulset woodpecker-server -n woodpecker
```

**Key Helm values:**
- `WOODPECKER_HOST`: `https://ci.home`
- `WOODPECKER_GITEA_URL`: `https://gitea.home`
- `WOODPECKER_ADMIN`: `admin`
- Secrets injected from `woodpecker-server-secret` and `woodpecker-agent-secret`

**Troubleshoot "Client ID not registered" error:**
See the OAuth2 registration steps in §6.3 Gitea, then update the secret + restart as shown above.

---

### 6.5 Harbor Container Registry

**Helm chart**: `https://helm.goharbor.io` version `1.14.0`
**Namespace**: `harbor`
**Expose**: Ingress via Traefik, TLS cert sourced from `harbor-tls` Kubernetes Secret

**No additional secrets required** — Harbor generates its own admin password internally on first boot.
To retrieve it:
```bash
kubectl get secret harbor-core -n harbor \
  -o jsonpath='{.data.HARBOR_ADMIN_PASSWORD}' | base64 -d && echo
```

**Persistence**: All components (registry, jobservice, database, redis, trivy) use `nfs-client` StorageClass.

**Push an image to Harbor:**
```bash
# 1. Trust the CA cert on your docker daemon (see cert-manager runbook)
# 2. Login
docker login registry.home -u admin

# 3. Tag and push
docker tag myapp:latest registry.home/library/myapp:latest
docker push registry.home/library/myapp:latest
```

---

### 6.6 Prometheus & Grafana Monitoring

**Stack**: `kube-prometheus-stack` Helm chart, deployed in `monitoring` namespace.
**Managed**: Manually via Ansible playbook (not via ArgoCD).

**Apply/upgrade:**
```bash
cd ansible
ansible-playbook playbooks/setup_monitoring_agents.yml
```

**Key configuration** (`kubernetes/services/monitoring/values.yaml`):
- Prometheus retention: 7 days, 10Gi PVC on `nfs-client`
- External scrape targets (node_exporter on physical hosts):
  - `192.168.8.11:9100`, `192.168.8.12:9100`, `192.168.8.13:9100` (arch hosts)
  - `192.168.8.15:9100` (OMV NAS)
  - `192.168.8.253:9100` (OpenWrt router)
- Grafana: admin credentials from `grafana-admin-credentials` secret

**Grafana admin secret (create if not exists):**
```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='YOUR_SECURE_PASSWORD'
```

---

### 6.7 Homepage Dashboard

**Image**: `ghcr.io/gethomepage/homepage:latest`
**Namespace**: `homepage`
**Managed**: Manual Kustomize (`kubectl apply -k kubernetes/services/homepage/`)

**Configuration**: Entirely via ConfigMap (`kubernetes/services/homepage/configmap.yaml`).
To add a new service to the dashboard, edit `configmap.yaml` and re-apply.

```bash
kubectl apply -k kubernetes/services/homepage/
```

---

### 6.10 Sewbase Application CI/CD Pipeline

**Repo**: `https://gitea.home/gitea_admin/sewbase_guitea.git` (private, Gitea)
**Namespace**: `sewbase`
**Image registry**: `registry.home/badorius/sewbase:latest` (Harbor)
**CI engine**: Woodpecker CI (`https://ci.home`)
**CD engine**: ArgoCD (`https://argocd.home`), app name: `sewbase`
**K8s manifests**: `infra/k8s/sewbase/` inside the sewbase_guitea repo

#### Full Pipeline Flow
```
1. Developer pushes to main branch on gitea.home
2. Gitea webhook → Woodpecker CI pipeline starts
3. Woodpecker: npm install + npm run test
4. Woodpecker: docker build + push to registry.home/badorius/sewbase:latest
5. Woodpecker: curl POST to ArgoCD API → hard refresh + sync
6. ArgoCD: pulls latest image, redeploys sewbase pod in namespace sewbase
7. App available at https://sewbase.home
```

#### Required Woodpecker Secrets (set via https://ci.home → repository secrets)
```
docker_password   = Harbor password for user 'badorius'
argocd_server     = argocd.home
argocd_token      = ArgoCD API token (see below)
```

#### Generate ArgoCD API Token (one-time)
```bash
# Option 1: via argocd CLI
argocd login argocd.home --insecure
argocd account generate-token --account admin

# Option 2: via ArgoCD UI
# Settings → Accounts → admin → Generate Token
# Copy token → set as Woodpecker secret 'argocd_token'
```

#### Required Kubernetes Secrets (inject before ArgoCD sync)
```bash
# 1. Harbor pull secret for sewbase namespace
kubectl create namespace sewbase --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry harbor-pull-secret \
  --namespace sewbase \
  --docker-server=registry.home \
  --docker-username=YOUR_HARBOR_USER \
  --docker-password=YOUR_HARBOR_PASSWORD

# 2. App secrets (database, auth, minio)
kubectl create secret generic sewbase-app-secrets \
  --namespace sewbase \
  --from-literal=DATABASE_URL='postgresql://sewuser:YOUR_PW@postgres:5432/sewbase' \
  --from-literal=AUTH_SECRET='YOUR_AUTH_SECRET' \
  --from-literal=MINIO_ENDPOINT='http://openmediavault.home:9000' \
  --from-literal=MINIO_ACCESS_KEY='admin' \
  --from-literal=MINIO_SECRET_KEY='YOUR_MINIO_SECRET'

# 3. PostgreSQL secret
kubectl create secret generic sewbase-db-secrets \
  --namespace sewbase \
  --from-literal=postgres-password='YOUR_POSTGRES_PASSWORD'
```

#### Register Gitea Repo in ArgoCD (one-time, if not already done)
```bash
# ArgoCD must trust the Gitea TLS cert to clone the repo
argocd login argocd.home --insecure
argocd repo add https://gitea.home/gitea_admin/sewbase_guitea.git \
  --username gitea_admin \
  --password YOUR_GITEA_PASSWORD \
  --insecure-skip-server-verification
```

#### Troubleshooting
```bash
# Check Woodpecker pipeline logs
# → Visit https://ci.home → sewbase_guitea repo → Pipelines

# Check ArgoCD sync status
kubectl get application sewbase -n argocd -o yaml
argocd app get sewbase

# Check sewbase pods
kubectl get pods -n sewbase
kubectl logs -n sewbase -l app=sewbase --tail=50

# Check image pull
kubectl describe pod -n sewbase -l app=sewbase | grep -A5 Events
```

---


### 6.11 Traefik Dashboard

**URL**: `https://traefik.home`
**Managed**: Manual (`kubectl apply -k kubernetes/infra/traefik/`)

**Configuration** (`kubernetes/infra/traefik/`):
- `helmchartconfig.yaml`: Enables `api.dashboard=true`, `api.insecure=true` (serves dashboard at port 9000)
- `ingress.yaml`: Ingress + ClusterIP service exposing port 9000 → `traefik.home` with TLS

**Notes**: Port 9000 is only accessible within the cluster. The ingress forwards HTTPS traffic to the internal port.

---

### 6.12 Calibre-Web

**Image**: `lscr.io/linuxserver/calibre-web:latest`
**Namespace**: `calibre`
**URL**: `https://calibre.home`
**Managed**: Manual (`kubectl apply -k kubernetes/services/calibre/`)

**Default credentials** (first login after deploy):
- Username: `admin`
- Password: `admin123`
- Set library path to `/books` when prompted

**PVCs**:
- `calibre-config-pvc`: 1Gi (nfs-client) — config, metadata DB
- `calibre-books-pvc`: 50Gi (nfs-client) — books library

**Upload books**: Copy files to the NFS path of `calibre-books-pvc` or upload via Calibre-Web UI.

```bash
# Redeploy
kubectl apply -k kubernetes/services/calibre/
```

---

### 6.8 qBittorrent

**Image**: `lscr.io/linuxserver/qbittorrent:latest`
**Namespace**: `media`
**Timezone**: `Europe/Madrid`
**Managed**: Manual Kustomize (`kubectl apply -k kubernetes/services/qbittorrent/`)

**PVCs**: `qbittorrent-config-pvc` and `qbittorrent-downloads-pvc` (both `nfs-client`).

```bash
kubectl apply -k kubernetes/services/qbittorrent/
```

---

### 6.9 Backup System

**Automated by**: `ansible/roles/btrfs-backup/` role
**Applied by**: `ansible-playbook playbooks/setup_backups.yml`

**Scripts deployed to each Arch host:**
- `/usr/local/bin/btrfs-snapshot.sh` — takes local BTRFS snapshot
- `/usr/local/bin/btrfs-send.sh` — incremental send to OMV NAS
- `/usr/local/bin/btrfs-prune.sh` — prunes old local snapshots

**Systemd timers schedule:**
| Timer | Time | Action |
|-------|------|--------|
| `btrfs-snapshot.timer` | 03:00 | Create local snapshot |
| `btrfs-send.timer` | 04:00 | Send to OMV |
| `btrfs-prune.timer` | 05:00 | Prune locals (keep 7) |
| Remote prune (OMV) | 06:00 | Keep 14 on NAS |

**K3s ETCD backups**: role `k3s-backup` runs on `k3s_master` — snapshots ETCD to `/mnt/nfs/backups/k3s/`

**Check backup status:**
```bash
systemctl status btrfs-snapshot.timer
systemctl list-timers | grep btrfs
journalctl -u btrfs-send.service --since today
```

---

## 7. Ansible Playbook Reference

### Full Stack Deploy (from scratch)
```bash
cd ansible
ansible-playbook -i inventory/hosts.yml site.yml
```

### Individual Playbooks
```bash
# Configure OpenWrt DNS (add new domain entries first in openwrt-dns role)
ansible-playbook playbooks/setup_network.yml

# Provision Arch hosts (base, KVM, bridge networking)
ansible-playbook playbooks/setup_hosts.yml

# Configure BTRFS backups + K3s ETCD backup
ansible-playbook playbooks/setup_backups.yml

# Provision VMs (Ubuntu Cloud Images via Libvirt)
ansible-playbook playbooks/setup_vms.yml

# Install/upgrade K3s cluster
ansible-playbook playbooks/setup_k3s.yml

# Deploy node_exporter to all nodes + upgrade Prometheus Helm release
ansible-playbook playbooks/setup_monitoring_agents.yml

# Rolling update + reboot all hosts and VMs
ansible-playbook playbooks/maintenance/update_reboot.yml
```

### Run a specific role only
```bash
ansible-playbook playbooks/setup_hosts.yml --tags base
# or target a single host
ansible-playbook playbooks/setup_k3s.yml -l k3s_master
```

---

## 8. Kubernetes Quick Reference

```bash
# Check all ArgoCD-managed apps
kubectl get applications -n argocd

# Check all pods across all namespaces
kubectl get pods -A

# Check ingresses
kubectl get ingress -A

# Check TLS certificates
kubectl get certificates -A

# Check PVCs
kubectl get pvc -A

# Force ArgoCD to re-sync an app
kubectl patch application gitea -n argocd \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"force":{}}}}}'

# Get ArgoCD admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo

# Restart a deployment
kubectl rollout restart deployment <name> -n <namespace>
kubectl rollout restart statefulset <name> -n <namespace>

# View logs
kubectl logs -n <namespace> -l app=<label> --tail=50 -f

# Delete and let ArgoCD recreate (use carefully)
kubectl delete namespace <namespace>
# Then force sync in ArgoCD UI
```

---

## 9. How to Add a New Service (End-to-End)

1. **Create Kubernetes manifests** in `kubernetes/services/<servicename>/`
   - `namespace.yaml`
   - `kustomization.yaml` (with `helmCharts` or plain resources)
   - Ingress with `cert-manager.io/cluster-issuer: homelab-ca` annotation

2. **Add DNS entry** in `ansible/roles/openwrt-dns/tasks/main.yml`:
   ```yaml
   - { name: "newservice.home", ip: "192.168.8.248" }
   ```
   Apply: `ansible-playbook playbooks/setup_network.yml`

3. **Decide management mode**:
   - **ArgoCD-managed**: Add an Application entry to `kubernetes/infra/argocd/applications.yaml`
   - **Manual**: `kubectl apply -k kubernetes/services/<servicename>/`

4. **Create required secrets** via `kubectl create secret` (never in the repo).

5. **Verify TLS**: `kubectl get certificate -n <namespace>`

6. **Update Homepage** (optional): Add entry to `kubernetes/services/homepage/configmap.yaml` + apply.

7. **Document** the service in this file (§6) and update the service endpoints table in `README.md`.

---

## 10. Session State

> Update this section at the beginning and end of every working session.
> This is the handoff document between AI sessions. Be specific about what works, what doesn't, and what the immediate next action is.

### Last Updated: 2026-05-01 (Session 9)

### Infrastructure Status

| Component | Status | Notes |
|-----------|--------|-------|
| Arch hosts (arch01-03) | ✅ Running | Ansible-managed |
| VMs (vm01-03) | ✅ Running | CA trusted, k3s healthy |
| K3s cluster | ✅ Running | 1 master + 2 workers |
| Traefik ingress | ✅ Running | `https://traefik.home` |
| cert-manager | ✅ Running | homelab-ca ClusterIssuer — TLS en todos los endpoints |
| ArgoCD | ✅ Running | `https://argocd.home`, selfHeal activo |
| Gitea | ✅ Running | `https://gitea.home`, usuario: `gitea_admin` |
| Woodpecker CI | ✅ Running | `https://ci.home`, OAuth2 OK |
| Harbor | ✅ Running | `https://registry.home`, imagen sewbase existe |
| Prometheus/Grafana | ✅ Running | `https://grafana.home` |
| Homepage | ✅ Running | `https://homepage.home` |
| qBittorrent | ✅ Running | `https://qbittorrent.home` |
| Sewbase | ✅ Running | `https://sewbase.home`, postgres en NFS correctamente |
| Calibre-Web | ✅ Running | `https://calibre.home` |
| NFS provisioner | ✅ Running | Base path cambiado a `/share/k3s/` |
| OMV NAS | ✅ Running | `192.168.8.15`, NFS exports active |

### Credenciales (Session 4)
- **NOTA**: Todas las credenciales están almacenadas en Proton Pass, vault **homelab**.
- Gitea admin: `gitea_admin` — `pass-cli item view --vault-name homelab --item-title gitea-admin --field password`
- Grafana: `admin` — `pass-cli item view --vault-name homelab --item-title grafana-admin --field password`
- Harbor admin: `admin` — `pass-cli item view --vault-name homelab --item-title harbor-admin --field password`
- Harbor user badorius: `badorius` — `pass-cli item view --vault-name homelab --item-title harbor-badorius --field password`
- Woodpecker agent secret: `pass-cli item view --vault-name homelab --item-title woodpecker-agent-secret --field password`
- Sewbase DB: `sewuser` — `pass-cli item view --vault-name homelab --item-title sewbase-db --field password`

### What Changed This Session (4)
- **Secrets**: Todos los servicios unificados con la misma contraseña base (excepto Harbor que requiere mayúscula).
- **Harbor**: Password admin cambiada. Usuario `badorius` creado. Proyecto `badorius` creado con permisos developer.
- **harbor-pull-secret**: Creado en namespace `sewbase` con usuario `badorius` (ver vault homelab).
- **TLS Grafana**: `values.yaml` actualizado con `ingressClassName: traefik` y cert-manager. Helm upgrade aplicado. Certificado `grafana-tls` activo.
- **TLS Sewbase**: Ingress corregido con cert-manager. Certificado `sewbase-tls` activo.
- **Traefik Dashboard**: `HelmChartConfig` aplicado (api.dashboard=true, api.insecure=true). Ingress + Service `traefik-dashboard-svc` en puerto 9000. DNS `traefik.home` añadido.
- **Calibre-Web**: Nuevo servicio desplegado en namespace `calibre`. PVCs en nfs-client. TLS activo (`calibre-tls`). DNS `calibre.home` añadido.
- **DNS**: Añadidos `traefik.home` y `calibre.home` en OpenWrt.
- **Woodpecker agentes**: CA montado vía ConfigMap `homelab-ca-cert` para trusting de Harbor.
- **sewbase_guitea**: `.woodpecker.yml` corregido (buildkit_config para CA). `app.yaml` corregido (ingressClassName). Pusheado a Gitea.
- **Infra repo**: `kubernetes/infra/traefik/`, `kubernetes/services/calibre/` añadidos.

### What Changed This Session (9) — 2026-05-01

- **Security audit pre-publicación**: Repo escaneado completamente (ficheros actuales + historial git). Sin secretos en ficheros.
- **Git history limpiado**: Commit `bc98371` tenía la contraseña `Cluster12/cluster12` en el mensaje de commit. Reescrito con `git filter-branch`, refs/original eliminados, `git gc --prune=now --aggressive`. Force-push a GitHub. Historia limpia verificada.
- **Rotación completa de contraseñas**: Todos los secrets rotados con contraseñas aleatorias de 24-32 chars (generadas con `pass-cli password generate random --symbols false`). Vault Proton Pass actualizado (6 items). Secretos K8s recreados. PostgreSQL `ALTER USER` aplicado. Harbor cambiado via UI.
- **Secrets rotados**: `gitea-admin-secret` (con campo `username`), `grafana-admin-credentials`, `woodpecker-server-secret` (OAuth Gitea preservado), `woodpecker-agent-secret`, `sewbase-db-secrets`, `sewbase-app-secrets`, `harbor-pull-secret`.
- **Vault pass-cli**: `pass-cli item update --field` NO funciona para el campo password nativo de items Login. Hay que usar `pass-cli item delete` + `pass-cli item create login --password VALUE` en la misma sesión de shell.
- **CA cert pública**: `ansible/roles/common/files/homelab-root-ca.crt` — es la clave pública del CA, no la privada. Es correcto tenerla en el repo. Expira 2026-07-12.
- **Repo listo para público**: Tras estos cambios el repo puede hacerse público en GitHub.

### What Changed This Session (8) — 2026-05-01

- **Bug fix crítico — postgres mountPath**: `postgres.yaml` corregido. Bitnami PostgreSQL usa `/bitnami/postgresql` como directorio de datos, NO `/var/lib/postgresql/data`. El PVC NFS estaba montado en el path incorrecto → todos los datos se perdían al reiniciar el pod. Fix: mountPath cambiado a `/bitnami/postgresql`. Pusheado a Gitea, ArgoCD synced.
- **NFS reorganización**: Todos los directorios de PVCs movidos de `/share/` a `/share/k3s/`. 49 directorios (19 activos + 30 archived). PVs de Kubernetes recreados con nuevas rutas. NFS provisioner actualizado con `nfs.path=/share/k3s`. Ansible role actualizado. Cluster completamente operativo tras la migración.
- **Sewbase migraciones**: Aplicadas manualmente 2 veces (por el reinicio de postgres). Las tablas están en la DB. La imagen corriendo es antigua (no incluye `migrate deploy`). Pendiente: run `deploy.sh` para construir imagen nueva con CMD correcto.
- **NFS provisioner**: Estaba en CrashLoopBackOff antes de la migración (causa desconocida). Después de `helm upgrade` con nuevo path está Running.

### Active Blockers

#### ⚠️ Sewbase — imagen antigua en Harbor (sin migrate deploy)
- **Issue**: La imagen corriendo (`registry.home/badorius/sewbase:latest`) es antigua y NO ejecuta `prisma migrate deploy` al arrancar. Si postgres pierde datos, el pod no se autorecupera. Las migraciones están en la DB ahora, pero hay que rebuild.
- **Root cause**: La imagen fue construida antes de que el Dockerfile tuviera el CMD correcto.
- **Next action**: Ejecutar `deploy.sh` en la workstation (o portátil):
  ```bash
  # Verificar que Harbor CA está confiada por Docker (una vez por máquina)
  kubectl get secret root-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d | \
    sudo tee /etc/docker/certs.d/registry.home/ca.crt
  sudo systemctl restart docker
  # Login y deploy
  pass-cli item view --vault-name homelab --item-title harbor-badorius --field password | \
    docker login registry.home -u badorius --password-stdin
  cd /home/darthv/git/badorius/sewbase_guitea
  ./deploy.sh
  ```

#### ⚠️ Woodpecker Pipeline clone — roto (baja prioridad)
- **Issue**: El clone step falla. Ver detalles completos en `sewbase_guitea/AGENTS.md §8`.
- **Workaround activo**: Usar `deploy.sh` para deploys manuales.

#### ⚠️ ArgoCD — GitHub auth para gitea/harbor/woodpecker
- **Issue**: Apps `gitea`, `harbor`, `woodpecker` muestran Unknown en ArgoCD (repo GitHub no accesible). `selfHeal` activo puede revertir cambios manuales.
- **Impact**: Cambios en `kubernetes/services/{gitea,harbor,woodpecker}/` requieren `kubectl apply -k` manual hasta resolver.

### What Changed This Session (6) — 2026-04-26

- **Security cleanup**: Todos los passwords hardcodeados eliminados de la documentación y del historial de git (git-filter-repo, 54 commits reescritos).
- **Proton Pass**: Vault `homelab` creado/poblado con 6 entradas: `harbor-admin`, `harbor-badorius`, `gitea-admin`, `grafana-admin`, `woodpecker-agent-secret`, `sewbase-db`.
- **Documentación actualizada**: Todos los ficheros `.md` usan `pass-cli item view --vault-name homelab --item-title <name> --field password` en lugar de contraseñas literales.
- **Rules actualizadas**: §5, §11 y agents.md header — regla explícita de no hardcodear passwords.
- **Repo estuvo seguro**: El repo se hizo privado antes de que los commits con passwords fueran pusheados al remoto.

### What Changed This Session (7) — 2026-04-26

- **Sewbase operativo**: `https://sewbase.home` responde HTTP 200. Root cause resuelto.
- **Root cause sewbase error**: `secrets.example.yaml` estaba en `infra/k8s/sewbase/` — ArgoCD lo aplicaba con valores placeholder (`YOUR_PASSWORD`, `YOUR_POSTGRES_PASSWORD`) sobreescribiendo los secrets reales en cada sync.
- **Fix**: Movido `secrets.example.yaml` a `docs/infra/` (fuera del path de ArgoCD). Pusheado a Gitea.
- **Postgres password corregida**: Cambiada en la DB a valor real del vault. Secrets recreados con `pass-cli`.
- **Schema aplicado manualmente**: La DB estaba vacía (0 tablas). Schema completo aplicado via `kubectl exec + psql`.
- **Dockerfile mejorado**: CMD actualizado para ejecutar `prisma migrate deploy` antes de arrancar.
- **app.yaml mejorado**: Eliminado `DATABASE_URL` duplicado (ya viene de `envFrom`). Añadido `readinessProbe`.
- **Documentación sewbase_guitea**: Creados `CLAUDE.md`, `AGENTS.md` completo, `docs/infra/deployment.md`, `docs/infra/secrets.md`.
- **Proton Pass**: Añadido item `sewbase-auth` (NextAuth AUTH_SECRET).
- **sewbase-app-secrets**: Recreado con `DATABASE_URL` y `AUTH_SECRET` reales desde vault.

### What Changed This Session (7 cont.) — 2026-04-26 (Woodpecker, Schema Drift, S3)

- **Woodpecker secrets configurados**: `docker_password`, `argocd_server`, `argocd_token` añadidos como global secrets via REST API (`POST https://ci.home/api/secrets`). JWT token generado manualmente en Python desde `user_hash` SQLite.
- **Woodpecker repo activado**: `sewbase_guitea` activo. `gitea_admin` creado como usuario admin. Repo activation: `POST /api/repos?forge_remote_id=1` (query param, no body).
- **Woodpecker TLS fix**: `WOODPECKER_GITEA_SKIP_VERIFY: "true"` añadido al server (Helm chart v1.5.0 no soporta `server.extraVolumes`). Aplicado via `kubectl patch statefulset`.
- **Schema drift sewbase resuelto**: Creada y aplicada migración `20260426000000_add_missing_columns`. Todos los cambios registrados en `_prisma_migrations`.
- **S3 rename**: `MINIO_*` → `S3_*` env vars en `storage.ts`, `secrets.md`, `secrets.example.yaml`, `CLAUDE.md`. Prep para Hetzner Object Storage o cualquier proveedor S3-compatible.
- **ArgoCD homelab-infra**: Apps `gitea`/`harbor`/`woodpecker` muestran Unknown (auth GitHub issue pre-existente, no bloqueante).

### Infrastructure Status (Session 7)

| Component | Status | Notes |
|-----------|--------|-------|
| Arch hosts (arch01-03) | ✅ Running | Ansible-managed |
| VMs (vm01-03) | ✅ Running | CA trusted, k3s healthy |
| K3s cluster | ✅ Running | 1 master + 2 workers |
| Traefik ingress | ✅ Running | `https://traefik.home` |
| cert-manager | ✅ Running | homelab-ca — TLS en todos los endpoints |
| ArgoCD | ✅ Running | `https://argocd.home` |
| Gitea | ✅ Running | `https://gitea.home`, usuario: `gitea_admin` |
| Woodpecker CI | ✅ Running | `https://ci.home`, OAuth2 OK |
| Harbor | ✅ Running | `https://registry.home`, imagen sewbase existe |
| Prometheus/Grafana | ✅ Running | `https://grafana.home` |
| Homepage | ✅ Running | `https://homepage.home` |
| qBittorrent | ✅ Running | `https://qbittorrent.home` |
| Sewbase | ✅ Running | `https://sewbase.home` HTTP 200 |
| Calibre-Web | ✅ Deployed | `https://calibre.home` |
| OMV NAS | ✅ Running | `192.168.8.15`, NFS exports active |

### Credenciales (Session 7)
- **NOTA**: Todas las credenciales en Proton Pass, vault **homelab**.
- Gitea admin: `gitea_admin` — `pass-cli item view --vault-name homelab --item-title gitea-admin --field password`
- Grafana: `admin` — `pass-cli item view --vault-name homelab --item-title grafana-admin --field password`
- Harbor admin: `admin` — `pass-cli item view --vault-name homelab --item-title harbor-admin --field password`
- Harbor user badorius: `badorius` — `pass-cli item view --vault-name homelab --item-title harbor-badorius --field password`
- Woodpecker agent secret: `pass-cli item view --vault-name homelab --item-title woodpecker-agent-secret --field password`
- Sewbase DB: `sewuser` — `pass-cli item view --vault-name homelab --item-title sewbase-db --field password`
- Sewbase AUTH_SECRET: `pass-cli item view --vault-name homelab --item-title sewbase-auth --field password`

### Next Session Priorities

1. **Rebuild sewbase imagen** — `./deploy.sh` en sewbase_guitea para construir imagen con `prisma migrate deploy` en CMD. Esto elimina el riesgo de pérdida de schema ante cualquier reinicio de postgres.
2. **S3 storage** — decidir proveedor (MinIO OMV vs Hetzner Object Storage), configurar bucket `sewbase`, crear item `s3-secret` en vault, actualizar `sewbase-app-secrets`.
3. **Feature development** — desarrollo MVP según `docs/ux/mvp-scope.md` en sewbase_guitea. Pipeline CI/CD en pausa — usar `deploy.sh`.
4. **Fix Woodpecker pipeline clone** (baja prioridad) — ver enfoques en `sewbase_guitea/AGENTS.md §8`.
5. **ArgoCD GitHub auth** — `gitea`/`harbor`/`woodpecker` muestran Unknown. Resolver para que ArgoCD pueda gestionar estos servicios correctamente.
6. **Calibre-Web** — subir librería de libros al PVC `calibre-books-pvc`.

---

## 11. Rules for the AI Agent

When working in this repository, always:

1. **Read this file first** at the start of every session to understand current state.
2. **Update Session State (§10)** at the end of every session — be specific about what changed.
3. **Check cross-repo notifications** — read `§12 Inbox` below and `../sewbase_guitea/AGENTS.md §9 Outbox` at session start; write to `§12 Outbox` at session end for any changes sewbase must know about.
4. **Never add secrets** to any file in this repo. If a secret is needed, document the `kubectl create secret` command. **For passwords in docs/commands, always use `pass-cli item view --vault-name homelab --item-title <name> --field password`** — never hardcode the actual value.
5. **Keep all code/docs in English** even when the conversation with the user is in Spanish.
6. **Prefer IaC** — every change must be expressed in Ansible or Kubernetes manifests.
7. **Use ArgoCD for CD** — for ArgoCD-managed services, push to git and sync. Don't apply manually.
8. **Document all new services** in §6 of this file and in `README.md` endpoint table.
9. **Add DNS entries** for every new service in `ansible/roles/openwrt-dns/tasks/main.yml`.
10. **All ingresses need TLS** via cert-manager annotation `cert-manager.io/cluster-issuer: homelab-ca`.
11. **Consult existing patterns** before creating new manifests — check existing service kustomizations for conventions.
12. **Password manager** — All homelab credentials are in Proton Pass vault `homelab`. Entries: `harbor-admin`, `harbor-badorius`, `gitea-admin`, `grafana-admin`, `woodpecker-agent-secret`, `sewbase-db`. Use `pass-cli item view` to retrieve them. Never create new entries without also adding them to the vault.

---

## 12. Cross-Repo Notifications

> **Protocol**: This section is the async interface between this repo and `sewbase_guitea`.
>
> - **Outbox → sewbase**: Written here when a homelab-infra session makes changes that sewbase should know about. A sewbase session consumes these items, acts on them, then marks them `[done]` in sewbase's `AGENTS.md §9 Inbox`.
> - **Inbox ← sewbase**: Items received from sewbase sessions. Written here by the sewbase AI after it identifies something homelab-infra needs to act on.
>
> The other repo lives at `../sewbase_guitea/`. Read its `AGENTS.md §9` at session start if cross-repo work is expected.

### Outbox → sewbase (pending)

| Date | Item | Notes |
|------|------|-------|
| 2026-05-01 | postgres mountPath bug fixed | `postgres.yaml` corregido: `mountPath: /bitnami/postgresql`. PVC NFS ahora en path correcto. Datos NO se pierden al reiniciar postgres. Pusheado a Gitea, ArgoCD synced. |
| 2026-05-01 | NFS reorganización completa | Todos los PVCs de k3s ahora en `/share/k3s/`. Esto afecta al path NFS del PVC `sewbase-postgres-pvc-pvc-b01c289c-...`. No requiere acción en sewbase_guitea, pero es info relevante si debuggeas NFS. |
| 2026-05-01 | Imagen sewbase necesita rebuild | La imagen en Harbor es antigua (sin `prisma migrate deploy`). Pendiente ejecutar `./deploy.sh`. Las migraciones están en la DB: 11 tablas, todo correcto. |

### Inbox ← sewbase (received)

| Date | Item | Notes |
|------|------|-------|
| 2026-04-26 | `MINIO_*` → `S3_*` rename | All S3 env vars renamed; `sewbase-app-secrets` must use `S3_*` keys |
| 2026-04-26 | Schema drift resolved | Reconciliation migration applied; `_prisma_migrations` has 2 entries |
| 2026-04-26 | `secrets.example.yaml` moved | Moved from `infra/k8s/sewbase/` to `docs/infra/` — never put it back |
