# Homelab Infrastructure

Declarative homelab infrastructure using:

- Arch Linux Hosts
- Ansible Automation
- BTRFS Snapshots & Incremental Backups
- Libvirt Virtualization
- Cloud-Init Ubuntu VMs
- Kubernetes (k3s) Active Cluster
- Prometheus & Grafana Monitoring
- NFS Persistent Storage
- **GitOps CI/CD Pipeline (ArgoCD, Gitea, Woodpecker, Harbor)**

---

## Objective

Create a fully reproducible, automated, and resilient homelab environment where baremetal hosts and virtual machines can be destroyed and recreated at any time without data loss. The infrastructure provisions a fully functional Kubernetes (K3s) cluster on top of the underlying virtualization layer.

---

## Architecture Overview
```text
Mini PCs (Arch Linux)
├─ BTRFS Snapshots
├─ Ansible Deployment Agent
├─ Libvirt / KVM
│  ├─ Bridged Networking (br0)
│  └─ Cloud-Init Ubuntu VMs (vm01, vm02, vm03)
│     └─ Kubernetes (K3s) Cluster
└─ Backup & Storage → OpenMediaVault (NFS/BTRFS)
```

---

## Key Features

- Host backups with BTRFS incremental send to remote Target
- Fully declarative Ansible Infrastructure replacing fragmented manual steps.
- Automated VM creation mapped locally with Cloud-Init templates
- SSH key-based access only (No passwords)
- Zero-touch K3s High-Availability Deployment
- Internal DNS over OpenWrt (`*.home` domains)
- Centralized maintenance playbooks for updates and reboots
- Unified monitoring dashboard via Grafana
- Persistent storage for k8s workloads via NFS
- **Fully Automated GitOps Deployment tracking this repository via ArgoCD**

---

## Quick Start (Base Infrastructure)
Ensure your inventory variables in `ansible/inventory/hosts.yml` match your network topology (DHCP reservations).

```bash
git clone https://github.com/badorius/homelab-infra.git 
cd homelab-infra/ansible

# Deploy bare-metal hypervisors, automated backups, provision VMs, and install the K3s cluster:
ansible-playbook -i inventory/hosts.yml site.yml
```

## GitOps Stack Deployment (CI/CD)

This homelab utilizes a fully automated GitOps approach using ArgoCD. Services like Gitea (Repository), Woodpecker (CI), and Harbor (Registry) rely on Helm templates evaluated by ArgoCD dynamically. Since sensitive credentials are not stored in this Git repository, you must inject them into the cluster before bootstrapping the GitOps flow.

### 1. Create the Required Namespaces
```bash
kubectl create namespace cert-manager
kubectl create namespace argocd
kubectl apply -f kubernetes/services/gitea/namespace.yaml
kubectl apply -f kubernetes/services/woodpecker/namespace.yaml
kubectl apply -f kubernetes/services/harbor/namespace.yaml
```

### 2. Inject Secrets
Create the manual secrets required by the apps. Be sure to replace the placeholder strings with your actual secure passwords and OAuth credentials.
```bash
# Gitea Admin Password
kubectl create secret generic gitea-admin-secret \
  --namespace gitea \
  --from-literal=password='your-secure-password'

# Woodpecker Secrets
kubectl create secret generic woodpecker-server-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_GITEA_CLIENT='your-gitea-oauth-client-id' \
  --from-literal=WOODPECKER_GITEA_SECRET='your-gitea-oauth-secret' \
  --from-literal=WOODPECKER_AGENT_SECRET='your-shared-agent-hash'

kubectl create secret generic woodpecker-agent-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET='your-shared-agent-hash'
```
*(Note: You will need to start everything, create the Woodpecker OAuth Application in Gitea's UI, and update the server secret with the real Client/Secret afterwards).*

### 3. Bootstrap Core and GitOps Engine
Deploy `cert-manager` mapping your local CA, and `ArgoCD`.
```bash
# Install Cert-Manager and Local CA
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/component=webhook -n cert-manager --timeout=90s
kubectl apply -f kubernetes/infra/cert-manager/clusterissuers.yaml

# Install ArgoCD targetting our specific setup
kubectl apply -k kubernetes/infra/argocd/
```

### 4. Deploy All the Services (App of Apps)
Apply the meta-applications configuration. ArgoCD will synchronize to this repository and natively deploy Gitea, Woodpecker, and Harbor.
```bash
kubectl apply -f kubernetes/infra/argocd/applications.yaml
```

## Legacy / Extra Services Deployment

For services that are not yet managed via ArgoCD (or if you prefer the standard `kubectl` approach for infrastructure services), you can deploy them manually using Kustomize:

### Monitoring (Prometheus & Grafana)
Deploys the full Kube-Prometheus stack to monitor your cluster.
```bash
kubectl apply -k kubernetes/services/monitoring/
```

### Homepage & Media
Deploys the infrastructure dashboard and torrent client.
```bash
kubectl apply -k kubernetes/services/homepage/
kubectl apply -k kubernetes/services/qbittorrent/
```

---

## Infrastructure Maintenance

Use the maintenance playbooks to update and reboot all non-network infrastructure:

```bash
cd ansible
ansible-playbook playbooks/maintenance/update_reboot.yml
```

---

## Service Endpoints

Once the infrastructure is deployed, the following services are available via Traefik (secured via TLS by `cert-manager` for the GitOps stack):

| Service | Endpoint | Description |
| ------- | -------- | ----------- |
| Homepage | [http://homepage.home](http://homepage.home) | Infrastructure Overview |
| Grafana | [http://grafana.home](http://grafana.home) | Cluster Monitoring |
| qBittorrent | [http://qbittorrent.home](http://qbittorrent.home) | Media Downloads |
| ArgoCD | [https://argocd.home](https://argocd.home) | GitOps CD Dashboard |
| Gitea | [https://gitea.home](https://gitea.home) | Private Code Repository |
| Woodpecker CI | [https://ci.home](https://ci.home) | Continuous Integration |
| Harbor Registry | [https://registry.home](https://registry.home) | Container Registry |

> [!NOTE]
> DNS resolution depends on the OpenWrt router configuration managed via Ansible.
