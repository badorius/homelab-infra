# ArgoCD

GitOps continuous delivery tool. Watches `github.com/badorius/homelab-infra` and keeps the cluster in sync with the git state.

## Access

| Property | Value |
|----------|-------|
| URL | https://argocd.home |
| Username | admin |
| Password | `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' \| base64 -d` |
| Namespace | argocd |
| Managed by | Manual bootstrap (`kubectl apply -k kubernetes/infra/argocd/`) |

## Apps Managed

| App | Source | Target namespace |
|-----|--------|-----------------|
| gitea | kubernetes/services/gitea/ (this repo) | gitea |
| woodpecker | kubernetes/services/woodpecker/ (this repo) | woodpecker |
| harbor | kubernetes/services/harbor/ (this repo) | harbor |
| sewbase | infra/k8s/sewbase/ (gitea.home private repo) | sewbase |

## Key Configuration

`kubernetes/infra/argocd/kustomization.yaml` patches two ConfigMaps:

- `argocd-cm`: `kustomize.buildOptions: "--enable-helm"` — allows `helmCharts:` in kustomization.yaml files
- `argocd-cmd-params-cm`: `server.insecure: "true"` — ArgoCD server runs HTTP; Traefik handles TLS

## Common Operations

```bash
# Bootstrap (first time or after full wipe)
kubectl apply -k kubernetes/infra/argocd/
kubectl apply -f kubernetes/infra/argocd/applications.yaml

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo

# Login via CLI
argocd login argocd.home --insecure --username admin --password '<pass>' --grpc-web

# Force sync all apps
for app in gitea woodpecker harbor sewbase; do
  argocd app sync $app --force --grpc-web
done

# Generate API token (for Woodpecker deploy step)
argocd account generate-token --account admin --grpc-web
```

## App Sync Policy

All apps have `automated.prune: true` and `automated.selfHeal: true`:
- **prune**: resources removed from git are deleted from the cluster
- **selfHeal**: manual changes to the cluster are reverted

## Register a Private Gitea Repo

ArgoCD needs credentials to clone private repos from Gitea:

```bash
argocd repo add https://gitea.home/gitea_admin/REPO_NAME.git \
  --username gitea_admin \
  --password 'YOUR_GITEA_PASSWORD' \
  --insecure-skip-server-verification \
  --grpc-web
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| App stuck "Progressing" | Check pod logs: `kubectl logs -n <ns> -l app=<label>` |
| Sync fails with auth error | Check repo credentials: `argocd repo list --grpc-web` |
| ArgoCD can't reach GitHub | Check cluster internet access from vm01 |
| App "OutOfSync" but won't sync | `argocd app sync <name> --force --grpc-web` |
