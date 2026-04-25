# Monitoring (kube-prometheus-stack)

Full observability stack: Prometheus collects metrics from all cluster nodes and services; Grafana provides dashboards and alerting.

## Access

| Property | Value |
|----------|-------|
| Grafana URL | https://grafana.home |
| Grafana username | admin |
| Grafana password | from Secret `grafana-admin-credentials` (key: `admin-password`) |
| Prometheus | internal only (no ingress) |
| Namespace | monitoring |
| Chart | prometheus-community/kube-prometheus-stack |
| Managed by | ArgoCD (`monitoring` Application) |

```bash
# Get Grafana password
kubectl get secret grafana-admin-credentials -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo
```

## Architecture

```
kube-prometheus-stack
  ├─ prometheus-operator (Deployment)
  │    └─ Manages Prometheus and Alertmanager CRDs
  ├─ prometheus (StatefulSet)
  │    ├─ Scrapes all pods/nodes/services
  │    ├─ Retention: 7 days
  │    └─ Storage: 10Gi PVC (nfs-client)
  ├─ alertmanager (StatefulSet)
  │    └─ Storage: 1Gi PVC (nfs-client)
  ├─ grafana (Deployment)
  │    ├─ Ingress: grafana.home (cert-manager TLS)
  │    └─ Storage: 2Gi PVC (nfs-client)
  ├─ node-exporter (DaemonSet)
  │    └─ Runs on every K3s node (vm01, vm02, vm03)
  └─ kube-state-metrics (Deployment)
       └─ Kubernetes object metrics
```

## External Scrape Targets

Prometheus scrapes physical hosts outside the cluster via static config:

| Host | IP | Port | Role |
|------|----|------|------|
| (unknown) | 192.168.8.11 | 9100 | Physical infrastructure |
| (unknown) | 192.168.8.12 | 9100 | Physical infrastructure |
| (unknown) | 192.168.8.13 | 9100 | Physical infrastructure |
| OMV NAS | 192.168.8.15 | 9100 | NAS / NFS server |
| (unknown) | 192.168.8.253 | 9100 | Physical infrastructure |

These hosts must have `node_exporter` running and accessible on port 9100.

## Required Kubernetes Secrets

| Secret | Namespace | Keys |
|--------|-----------|------|
| grafana-admin-credentials | monitoring | admin-user, admin-password |

Create the secret if missing:

```bash
kubectl create secret generic grafana-admin-credentials \
  -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='REDACTED'
```

## Common Operations

```bash
# Check all monitoring pods
kubectl get pods -n monitoring

# Check Prometheus targets (which nodes are being scraped)
# Port-forward Prometheus UI:
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Then open http://localhost:9090/targets

# Check Grafana is up
kubectl get ingress -n monitoring
kubectl get certificate -n monitoring

# Force restart Grafana (e.g., after password change)
kubectl rollout restart deployment -n monitoring -l app.kubernetes.io/name=grafana

# View Grafana logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -f
```

## Adding a New Scrape Target

Edit `kubernetes/services/monitoring/values.yaml` and add to `additionalScrapeConfigs`:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'my-new-service'
        static_configs:
          - targets:
              - '192.168.8.X:9100'
            labels:
              group: 'my-group'
```

Commit and push — ArgoCD syncs the change automatically.

## Persistent Storage

| Component | Size | StorageClass | Access |
|-----------|------|-------------|--------|
| Prometheus | 10Gi | nfs-client | ReadWriteOnce |
| Grafana | 2Gi | nfs-client | ReadWriteOnce |
| Alertmanager | 1Gi | nfs-client | ReadWriteOnce |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Grafana shows wrong password | Check Secret: `kubectl get secret grafana-admin-credentials -n monitoring -o yaml` |
| Prometheus not scraping a target | Check `Status → Targets` in Prometheus UI; verify port 9100 is open on target host |
| Grafana PVC full | Increase PVC size, or prune old dashboards/sessions |
| node-exporter not on a node | Check if DaemonSet has tolerations matching that node's taints |
| Grafana TLS cert missing | `kubectl describe certificate grafana-tls -n monitoring` — check cert-manager events |
