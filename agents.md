# agents.md — Homelab Infrastructure: AI Agent Persistent Memory

> **IMPORTANT**: This file is the single source of truth for AI agent context across sessions.
> Update the **Session State** section at the end of every session before closing.
> Never store passwords, secrets, API keys or tokens in this file or anywhere in this repo.
> The repository is PUBLIC. All secrets are injected at runtime via `kubectl create secret`.

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

1. **No secrets in the repo** — The repository is public. All passwords, API keys, OAuth tokens, and certificates are injected at runtime via `kubectl create secret`. See README.md §2 for the exact commands.
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

### Last Updated: 2026-04-23

### Infrastructure Status

| Component | Status | Notes |
|-----------|--------|-------|
| Arch hosts (arch01-03) | ✅ Running | Ansible-managed, BTRFS backups active |
| VMs (vm01-03) | ✅ Running | vm01=master(248), vm02=worker(249), vm03=worker(101) |
| K3s cluster | ✅ Running | 1 master + 2 workers |
| Traefik ingress | ✅ Running | Default K3s ingress |
| cert-manager | ✅ Running | homelab-ca ClusterIssuer active |
| ArgoCD | ✅ Running | `https://argocd.home` |
| Gitea | ⚠️ Blocked | PostgreSQL image pull issue (see below) |
| Woodpecker CI | ✅ Deployed | Waiting for Gitea OAuth setup |
| Harbor | ✅ Deployed | `https://registry.home` |
| Prometheus/Grafana | ✅ Running | `https://grafana.home` |
| Homepage | ✅ Running | `http://homepage.home` |
| qBittorrent | ✅ Running | `http://qbittorrent.home` |
| OMV NAS | ✅ Running | `192.168.8.15`, NFS exports active |
| BTRFS Backups | ✅ Active | Systemd timers running on all Arch hosts |
| K3s ETCD backup | ✅ Active | Snapshots going to `/mnt/nfs/backups/k3s/` |

### Active Blockers

#### ⚠️ Gitea PostgreSQL — ErrImagePull
- **Issue**: `gitea-postgresql-0` pod in `ErrImagePull` / `ImagePullBackOff`
- **Root cause**: Bitnami pruned the Docker Hub tag `bitnami/postgresql:16.2.0-debian-12-r8`
- **Current state**: `kustomization.yaml` updated to use `public.ecr.aws/bitnami/postgresql:16.2.0`
- **Next action**: 
  ```bash
  # Delete the stuck StatefulSet so ArgoCD recreates it from scratch with new image
  kubectl delete statefulset gitea-postgresql -n gitea
  # Then in ArgoCD UI, force sync the gitea application
  ```

#### ⚠️ Woodpecker CI — OAuth2 not configured
- **Issue**: Woodpecker shows "Client ID not registered" error at `https://ci.home`
- **Dependency**: Requires Gitea to be healthy first
- **Next action** (after Gitea is fixed):
  1. Login to `https://gitea.home`
  2. Settings → Applications → Create OAuth2 app named `Woodpecker CI`
  3. Redirect URI: `https://ci.home/authorize`
  4. Update Kubernetes secret with Client ID + Secret (see §6.4 Woodpecker runbook)
  5. Restart Woodpecker server

### Sewbase Application
- **Repo**: `https://gitea.home/gitea_admin/sewbase_guitea.git`
- **ArgoCD app**: defined in `applications.yaml`, path `infra/k8s/sewbase`, namespace `sewbase`
- **CI**: Woodpecker pipeline (`.woodpecker.yml` in the sewbase repo)
- **Registry**: Was migrated from local k3s registry → GHCR (GitHub Container Registry)
- **Status**: Blocked by Gitea/Woodpecker not fully operational

### Next Session Priorities

1. **Fix Gitea PostgreSQL** — delete StatefulSet, force ArgoCD sync, verify `gitea.home` is up
2. **Configure Woodpecker OAuth** — Gitea → Settings → OAuth2 app → inject secret → restart woodpecker-server
3. **Test full CI/CD pipeline** — push to sewbase repo → Woodpecker builds → Harbor push → ArgoCD deploys
4. **Verify all services** — run full health check across all endpoints

---

## 11. Rules for the AI Agent

When working in this repository, always:

1. **Read this file first** at the start of every session to understand current state.
2. **Update Session State (§10)** at the end of every session — be specific about what changed.
3. **Never add secrets** to any file in this repo. If a secret is needed, document the `kubectl create secret` command.
4. **Keep all code/docs in English** even when the conversation with the user is in Spanish.
5. **Prefer IaC** — every change must be expressed in Ansible or Kubernetes manifests.
6. **Use ArgoCD for CD** — for ArgoCD-managed services, push to git and sync. Don't apply manually.
7. **Document all new services** in §6 of this file and in `README.md` endpoint table.
8. **Add DNS entries** for every new service in `ansible/roles/openwrt-dns/tasks/main.yml`.
9. **All ingresses need TLS** via cert-manager annotation `cert-manager.io/cluster-issuer: homelab-ca`.
10. **Consult existing patterns** before creating new manifests — check existing service kustomizations for conventions.
