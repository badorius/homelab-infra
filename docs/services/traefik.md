# Traefik

Ingress controller and reverse proxy. Routes all `*.home` HTTPS traffic into the K3s cluster. Installed automatically by K3s; configured via `HelmChartConfig` CRD.

## Access

| Property | Value |
|----------|-------|
| Dashboard URL | https://traefik.home |
| Namespace | kube-system |
| Version | 3.6.10 (rancher/mirrored-library-traefik) |
| Managed by | K3s built-in + `kubernetes/infra/traefik/` (manual apply) |

> **Note:** Traefik is NOT managed by ArgoCD. It is K3s's built-in ingress controller. The HelmChartConfig and extra Ingress/Service resources are applied manually via `kubectl apply`.

## Architecture

```
OpenWrt dnsmasq
  └─ *.home → 192.168.8.248 (vm01)
          │
          └─ Traefik LoadBalancer Service (kube-system)
               ├─ :80  (web entrypoint) — redirects to HTTPS
               ├─ :443 (websecure entrypoint) — TLS termination
               └─ :9000 (traefik entrypoint) — dashboard (internal only)
                    │
                    └─ Ingress rules route by host header to backend Services
```

## Configuration

Traefik is configured via `kubernetes/infra/traefik/helmchartconfig.yaml` (applied manually — not via ArgoCD):

Key settings enabled:
- `api.dashboard: true` + `api.insecure: true` — serve dashboard on port 9000
- `ingressRoute.dashboard.enabled: false` — disable default IngressRoute; use our custom Ingress instead
- `prometheus.io/scrape: "true"` — Prometheus scrapes Traefik metrics on port 8082
- `providers.kubernetesIngress.publishedService.enabled: true` — correct LoadBalancer IP on Ingress status

### Dashboard Ingress

The dashboard is exposed via a custom ClusterIP Service + Ingress in `kubernetes/infra/traefik/ingress.yaml`:

```
traefik-dashboard-svc (ClusterIP, port 9000)
  └─ targets Traefik pod on port 9000
Ingress (traefik.home)
  └─ → traefik-dashboard-svc:9000
  └─ TLS: traefik-dashboard-tls (cert-manager homelab-ca)
```

## Applying Configuration Changes

Since Traefik is not ArgoCD-managed, apply changes manually:

```bash
# Apply HelmChartConfig changes (triggers K3s Helm controller to update Traefik)
kubectl apply -f kubernetes/infra/traefik/helmchartconfig.yaml

# Apply dashboard Service + Ingress
kubectl apply -f kubernetes/infra/traefik/ingress.yaml

# Watch Traefik restart after HelmChartConfig update
kubectl rollout status daemonset traefik -n kube-system
```

## Ingress Convention

All services use the same pattern:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca    # auto-issue TLS cert
spec:
  ingressClassName: traefik                        # route via Traefik
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

> **Deprecated:** `kubernetes.io/ingress.class: traefik` annotation still works but prefer `spec.ingressClassName: traefik`.

## Common Operations

```bash
# Check all ingresses
kubectl get ingress -A

# View Traefik routing table (via dashboard at traefik.home)
# Or port-forward directly:
kubectl port-forward -n kube-system svc/traefik 9000:9000

# Check Traefik logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik -f

# Check Traefik version
kubectl get daemonset traefik -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# Verify a certificate is Ready after creating an Ingress
kubectl get certificate -n <namespace>
kubectl describe certificate <secretName> -n <namespace>
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `traefik.home` returns 404 | Verify `traefik-dashboard-svc` exists: `kubectl get svc -n kube-system traefik-dashboard-svc` |
| Dashboard redirect loop | Ensure `api.insecure: true` in HelmChartConfig, then `kubectl apply -f helmchartconfig.yaml` |
| Ingress not routing traffic | Check `spec.ingressClassName: traefik` is set (not just annotation) |
| TLS cert not issued | Verify cert-manager annotation is present: `cert-manager.io/cluster-issuer: homelab-ca` |
| Metrics not appearing in Prometheus | Check pod annotation `prometheus.io/scrape: "true"` on Traefik pod |
| HelmChartConfig not applying | Check K3s Helm controller: `kubectl logs -n kube-system -l app=helm-controller` |
