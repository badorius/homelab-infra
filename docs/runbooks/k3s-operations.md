# Runbook: K3s Cluster Operations

Day-2 operational procedures for the K3s Kubernetes cluster.

---

## Cluster Access

```bash
# From local workstation (kubeconfig must be set)
export KUBECONFIG=/home/darthv/git/badorius/homelab-infra/ansible/k3s.yaml

# Or on the K3s master VM directly
ssh darthv@192.168.8.248
sudo kubectl get nodes
# or: KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl get nodes
```

---

## Health Checks

```bash
# Node status
kubectl get nodes -o wide

# All pods across all namespaces
kubectl get pods -A

# All ingresses
kubectl get ingress -A

# All certificates (should all be READY=True)
kubectl get certificates -A

# All PVCs (should all be Bound)
kubectl get pvc -A

# ArgoCD application status
kubectl get applications -n argocd

# Traefik routing table
# Visit https://traefik.home/dashboard/ in browser
```

---

## Common Operations

### Restart a Deployment

```bash
kubectl rollout restart deployment <name> -n <namespace>
kubectl rollout status deployment <name> -n <namespace>
```

### Restart a StatefulSet

```bash
kubectl rollout restart statefulset <name> -n <namespace>
kubectl rollout status statefulset <name> -n <namespace>
```

### Scale a Deployment

```bash
# Scale down (useful for clearing locks, freeing resources)
kubectl scale deployment <name> -n <namespace> --replicas=0
# Scale back up
kubectl scale deployment <name> -n <namespace> --replicas=1
```

### View Pod Logs

```bash
# Current logs
kubectl logs -n <namespace> <pod-name> --tail=50

# Follow logs
kubectl logs -n <namespace> <pod-name> -f

# Logs for all pods with a label
kubectl logs -n <namespace> -l app=<label> --tail=50
```

### Exec into a Pod

```bash
kubectl exec -it -n <namespace> <pod-name> -- /bin/sh
# or bash if available:
kubectl exec -it -n <namespace> <pod-name> -- /bin/bash
```

### Describe a Resource (for debugging)

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl describe certificate <name> -n <namespace>
kubectl describe ingress <name> -n <namespace>
```

---

## ArgoCD Operations

### Force Sync an App

```bash
argocd login argocd.home --insecure --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath='{.data.password}' | base64 -d) --grpc-web

argocd app sync gitea --force --grpc-web
argocd app sync woodpecker --force --grpc-web
argocd app sync harbor --force --grpc-web
argocd app sync sewbase --force --grpc-web
```

### Check App Sync Status

```bash
argocd app list --grpc-web
argocd app get sewbase --grpc-web
```

### Get ArgoCD Admin Password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### Refresh an App (without sync)

```bash
argocd app get sewbase --refresh --grpc-web
```

---

## cert-manager Operations

### Check Certificate Status

```bash
kubectl get certificates -A
kubectl describe certificate <name> -n <namespace>
kubectl get certificaterequest -A
kubectl get order -n <namespace>    # ACME challenges (not used in homelab, but for reference)
```

### Force Certificate Renewal

```bash
# Delete the certificate object — cert-manager will reissue it
kubectl delete certificate <name> -n <namespace>

# Wait for new cert
kubectl get certificate <name> -n <namespace> -w
```

### Export Root CA

```bash
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt
```

---

## Storage Operations

### Check PVC Status

```bash
kubectl get pvc -A
kubectl describe pvc <name> -n <namespace>
```

### Delete a PVC (WARNING: destroys data)

```bash
kubectl delete pvc <name> -n <namespace>
# The NFS provisioner will delete the directory on the NAS
```

### Find Where a PVC's Data Lives on NAS

```bash
PV_NAME=$(kubectl get pvc <pvc-name> -n <namespace> -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV_NAME -o jsonpath='{.spec.nfs.path}'
# Returns something like: /share/<namespace>-<pvcname>-pvc-<uid>
# That path exists on the OMV NAS at 192.168.8.15
```

---

## Monitoring Operations

### Access Grafana

`https://grafana.home` — admin / see Proton Pass vault homelab: **grafana-admin**

Dashboards available:
- Kubernetes cluster overview
- Node resources (CPU, memory, disk)
- External node metrics (arch hosts, OMV, router via node_exporter)

### Upgrade Prometheus/Grafana

After editing `kubernetes/services/monitoring/values.yaml`:

```bash
cd ansible
ansible-playbook playbooks/setup_monitoring_agents.yml
```

### Check Scrape Targets

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
# Open http://localhost:9090/targets
```

---

## Node Operations

### SSH to Cluster Nodes

```bash
ssh darthv@192.168.8.248   # vm01 (master)
ssh darthv@192.168.8.249   # vm02
ssh darthv@192.168.8.101   # vm03
```

### Rolling Update of All Nodes

```bash
cd ansible
ansible-playbook playbooks/maintenance/update_reboot.yml
```

This updates packages and reboots each node in rolling fashion (K3s stays available).

### Drain a Node Before Maintenance

```bash
# Drain (evict pods safely)
kubectl drain vm02 --ignore-daemonsets --delete-emptydir-data

# Perform maintenance on vm02...

# Uncordon (allow pods back)
kubectl uncordon vm02
```

---

## Traefik Operations

### Dashboard

`https://traefik.home` — shows all routers, services, middlewares.

### View Traefik Config

```bash
kubectl get helmchartconfig traefik -n kube-system -o yaml
```

### Update Traefik Config

Edit `kubernetes/infra/traefik/helmchartconfig.yaml`, then:

```bash
kubectl apply -f kubernetes/infra/traefik/helmchartconfig.yaml
# K3s will run a new helm-install job and roll out the new config
kubectl rollout status deployment traefik -n kube-system
```

---

## Namespace Operations

### List All Namespaces

```bash
kubectl get namespaces
```

Expected namespaces:

| Namespace | Service |
|-----------|---------|
| argocd | ArgoCD |
| calibre | Calibre-Web |
| cert-manager | cert-manager |
| gitea | Gitea |
| harbor | Harbor |
| homepage | Homepage dashboard |
| kube-system | K3s system components |
| media | qBittorrent |
| monitoring | Prometheus + Grafana |
| nfs-provisioner | NFS storage provisioner |
| sewbase | Sewbase app |
| woodpecker | Woodpecker CI |

### Delete and Recreate a Service (Hard Reset)

Only for ArgoCD-managed services:

```bash
kubectl delete namespace <namespace>
# Wait for termination
kubectl get namespace <namespace>  # wait until gone

# ArgoCD will recreate it on next sync
# Force sync:
argocd app sync <appname> --grpc-web
```

**Note**: Deleting a namespace destroys all PVCs in it. PV data on NFS is retained (check NAS).

---

## Troubleshooting Common Issues

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Pod in `ImagePullBackOff` | Wrong registry credentials | Check/recreate `harbor-pull-secret` in that namespace |
| Pod in `Pending` | No PVC bound / no nodes available | `kubectl describe pod` → check Events |
| Certificate stuck `READY=False` | cert-manager issue | `kubectl describe certificate` → check CertificateRequest |
| ArgoCD app `Degraded` | Pod not starting | Check pod logs in the target namespace |
| Gitea `CrashLoopBackOff` after restart | LevelDB lock (two pods competing) | Scale deployment to 0 then back to 1 |
| Woodpecker "Client ID not registered" | Gitea OAuth app missing or wrong | Recreate OAuth app in Gitea, update `woodpecker-server-secret` |
| NFS PVC stuck in `Pending` | NFS server unreachable | Check OMV is running and `192.168.8.15:/share` is accessible |
| Harbor image push 401 | Wrong credentials | Verify `docker login registry.home -u badorius` — check Proton Pass vault homelab: **harbor-badorius** |
