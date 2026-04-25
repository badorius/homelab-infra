# Runbook: Secret Management

This homelab's git repository is **public**. No secrets, passwords, tokens, or private keys must ever appear in any file in this repo. This document explains how secrets are managed at runtime.

---

## Where Secrets Live

| Secret type | Location | How to access |
|------------|----------|---------------|
| K8s service credentials | Kubernetes Secrets (per namespace) | `kubectl get secret -n <ns>` |
| Woodpecker CI pipeline secrets | Woodpecker DB | `https://ci.home` → Settings → Secrets |
| ArgoCD admin password | `argocd-initial-admin-secret` K8s secret | `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' \| base64 -d` |
| Harbor admin password | Harbor internal DB | Login at `https://registry.home` |
| TLS certificates | Kubernetes Secrets (auto-managed by cert-manager) | `kubectl get secret <name>-tls -n <ns>` |
| Root CA private key | `root-secret` in cert-manager ns | `kubectl get secret root-secret -n cert-manager` |

---

## Creating a New Secret

**Rule**: Always use `kubectl create secret`. Never write values to files.

```bash
# Generic secret (key-value pairs)
kubectl create secret generic my-app-secret \
  --namespace my-namespace \
  --from-literal=USERNAME='admin' \
  --from-literal=PASSWORD='your-password'

# Docker registry pull secret
kubectl create secret docker-registry harbor-pull-secret \
  --namespace my-namespace \
  --docker-server=registry.home \
  --docker-username=badorius \
  --docker-password='REDACTED'

# TLS secret (manual, rarely needed — cert-manager handles TLS automatically)
kubectl create secret tls my-tls-secret \
  --namespace my-namespace \
  --cert=path/to/cert.crt \
  --key=path/to/key.key
```

---

## Updating an Existing Secret

```bash
# Update a single key in a secret
kubectl patch secret my-app-secret -n my-namespace \
  -p="{\"data\":{\"PASSWORD\":\"$(echo -n 'new-password' | base64)\"}}"

# Replace the entire secret
kubectl delete secret my-app-secret -n my-namespace
kubectl create secret generic my-app-secret \
  --namespace my-namespace \
  --from-literal=USERNAME='admin' \
  --from-literal=PASSWORD='new-password'
```

After updating a secret, restart the affected pod so it picks up the new value:

```bash
kubectl rollout restart deployment my-app -n my-namespace
# or for StatefulSets:
kubectl rollout restart statefulset my-statefulset -n my-namespace
```

---

## Service-Specific Secrets

### Gitea Admin

```bash
kubectl create secret generic gitea-admin-secret \
  --namespace gitea \
  --from-literal=username=gitea_admin \
  --from-literal=password='YOUR_PASSWORD'
```

Used by: Gitea Helm chart to set the admin user on first install.
Note: Changing this secret after install does **not** change Gitea's admin password in the DB.
To change Gitea admin password after deploy: login to `https://gitea.home` → User Settings → Password.

### Woodpecker Server

```bash
kubectl create secret generic woodpecker-server-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_GITEA_CLIENT='<client-id-from-gitea-oauth-app>' \
  --from-literal=WOODPECKER_GITEA_SECRET='<client-secret-from-gitea-oauth-app>' \
  --from-literal=WOODPECKER_AGENT_SECRET='<shared-token>'
```

How to get the OAuth values: `https://gitea.home` → Settings → Applications → Manage OAuth2 Applications → Woodpecker CI.

### Woodpecker Agent

```bash
kubectl create secret generic woodpecker-agent-secret \
  --namespace woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET='<same-shared-token>'
```

The `WOODPECKER_AGENT_SECRET` must match exactly between server and agent.

### Grafana Admin

```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='YOUR_PASSWORD'
```

### Sewbase App

```bash
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

## Woodpecker Pipeline Secrets

Pipeline secrets are stored in Woodpecker's database, not in Kubernetes. Set them via:

**Web UI**: `https://ci.home` → Repository → Settings → Secrets

**CLI** (download from GitHub releases):
```bash
export WOODPECKER_SERVER=https://ci.home
export WOODPECKER_TOKEN=<your-personal-access-token>

# List secrets
woodpecker-cli secret ls --global

# Add global secret (available to all pipelines)
woodpecker-cli secret add --global --name docker_password --value 'YOUR_VALUE'

# Add repo-level secret (only for a specific repo)
woodpecker-cli secret add --repo gitea_admin/myapp --name my_secret --value 'YOUR_VALUE'

# Remove a secret
woodpecker-cli secret rm --global --name docker_password
```

Personal access token: `https://ci.home` → user icon → Settings → Token → Generate.

---

## ArgoCD API Token

Used by Woodpecker's deploy step to trigger ArgoCD syncs.

```bash
# Login to ArgoCD
argocd login argocd.home --insecure --username admin --password '<admin-pass>' --grpc-web

# Generate token
argocd account generate-token --account admin --grpc-web

# Store in Woodpecker
woodpecker-cli secret add --global --name argocd_token --value '<generated-token>'
```

Tokens don't expire by default. Regenerate if compromised.

---

## Rotating Secrets

### Rotating a Kubernetes Secret

1. Delete and recreate the secret:
   ```bash
   kubectl delete secret myapp-secret -n myapp
   kubectl create secret generic myapp-secret --namespace myapp --from-literal=PASSWORD='new-value'
   ```
2. Restart affected pods:
   ```bash
   kubectl rollout restart deployment myapp -n myapp
   ```
3. Verify the new value is used:
   ```bash
   kubectl exec -n myapp deploy/myapp -- env | grep PASSWORD
   ```

### Rotating the Harbor Password

Harbor password changes take effect immediately via the API or UI:

```bash
# Via Harbor API (requires current password)
curl -k -u admin:CURRENT_PASSWORD \
  -X PUT https://registry.home/api/v2.0/users/1/password \
  -H "Content-Type: application/json" \
  -d '{"old_password":"CURRENT_PASSWORD","new_password":"NEW_PASSWORD"}'
```

**Note**: Harbor enforces: 8-128 chars, at least 1 uppercase, 1 lowercase, 1 number.

After changing Harbor password, update all harbor-pull-secrets in every namespace:

```bash
for NS in sewbase; do
  kubectl delete secret harbor-pull-secret -n $NS
  kubectl create secret docker-registry harbor-pull-secret \
    --namespace $NS \
    --docker-server=registry.home \
    --docker-username=badorius \
    --docker-password='NEW_PASSWORD'
done

# Also update Woodpecker's docker_password secret
woodpecker-cli secret update --global --name docker_password --value 'NEW_PASSWORD'
```

### Rotating the Root CA

The root CA has no expiry by default. If you need to rotate it (e.g., compromise):

1. Delete the `root-secret` and `homelab-ca` certificate in cert-manager
2. cert-manager will reissue a new root CA
3. Export the new CA and redistribute to all hosts (Ansible `common` role)
4. Re-import in browser/OS

**Warning**: All existing TLS certificates will be invalid after CA rotation. cert-manager will reissue them automatically, but there may be a brief service disruption.

---

## Viewing a Secret Value

```bash
# Show all keys
kubectl get secret my-secret -n my-namespace -o jsonpath='{.data}' | \
  python3 -c "import sys,json,base64; [print(k,'=',base64.b64decode(v).decode()) for k,v in json.load(sys.stdin).items()]"

# Show a single key
kubectl get secret my-secret -n my-namespace \
  -o jsonpath='{.data.PASSWORD}' | base64 -d && echo
```

---

## What NOT to Put in This Repo

Never commit these to git, even in documentation:

- Actual passwords, tokens, or API keys
- Private SSH keys
- TLS private keys
- kubeconfig files (k3s.yaml is gitignored)
- OAuth client secrets
- Database connection strings with credentials
- `.env` files

If you accidentally commit a secret: rotate it immediately, then clean git history.
