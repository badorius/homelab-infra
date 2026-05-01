# Secret Rotation Runbook

This runbook documents how to rotate all homelab credentials. Run it when:
- A credential may have been exposed (leaked commit, compromised machine)
- Making the repo public (audit + rotate as precaution)
- Scheduled rotation

## Prerequisites

- `pass-cli` authenticated: `pass-cli test`
- `kubectl` configured with cluster access: `kubectl get nodes`
- Cluster reachable from workstation

## Overview of managed secrets

| Service | K8s Secret | Namespace | Vault Item |
|---------|-----------|-----------|------------|
| Gitea admin | `gitea-admin-secret` | `gitea` | `gitea-admin` |
| Grafana admin | `grafana-admin-credentials` | `monitoring` | `grafana-admin` |
| Woodpecker agent shared secret | `woodpecker-server-secret`, `woodpecker-agent-secret` | `woodpecker` | `woodpecker-agent-secret` |
| PostgreSQL sewbase | `sewbase-db-secrets`, `sewbase-app-secrets` | `sewbase` | `sewbase-db` |
| Harbor admin | UI only | `harbor` | `harbor-admin` |
| Harbor badorius | `harbor-pull-secret` | `sewbase` | `harbor-badorius` |
| Homelab Root CA cert | — | — | `homelab-root-ca` (note) |

---

## Critical: do everything in ONE shell session

Shell variables do not persist between separate terminal commands. Generate,
store to vault, and apply to K8s all in the same script. If you split it across
commands, variables will be empty and secrets will be created with blank values.

Also: `pass-cli item update --field password=VALUE` does **not** update the
native password field of Login items. Always use `pass-cli item delete` followed
by `pass-cli item create login --password VALUE`.

---

## Step 1 — Preserve values that must NOT change

```bash
# Woodpecker OAuth credentials (tied to Gitea OAuth app — do not rotate without recreating the app)
WP_OAUTH_CLIENT=$(kubectl get secret woodpecker-server-secret -n woodpecker \
  -o jsonpath='{.data.WOODPECKER_GITEA_CLIENT}' | base64 -d)
WP_OAUTH_SECRET=$(kubectl get secret woodpecker-server-secret -n woodpecker \
  -o jsonpath='{.data.WOODPECKER_GITEA_SECRET}' | base64 -d)

# Sewbase NextAuth secret (rotate separately if needed — requires app rebuild)
AUTH_SECRET=$(kubectl get secret sewbase-app-secrets -n sewbase \
  -o jsonpath='{.data.AUTH_SECRET}' | base64 -d)

echo "OAuth client len=${#WP_OAUTH_CLIENT} | OAuth secret len=${#WP_OAUTH_SECRET} | AUTH len=${#AUTH_SECRET}"
```

---

## Step 2 — Generate new passwords

```bash
# Use --symbols false to avoid shell quoting issues and app policy conflicts
# Harbor requires uppercase+lowercase+digit — symbols false satisfies all policies
PW_GITEA=$(pass-cli password generate random --length 24 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
PW_GRAFANA=$(pass-cli password generate random --length 24 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
PW_WP=$(pass-cli password generate random --length 32 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
PW_SEWDB=$(pass-cli password generate random --length 24 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
PW_HARBOR_ADMIN=$(pass-cli password generate random --length 24 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")
PW_HARBOR_BADORIUS=$(pass-cli password generate random --length 24 --symbols false --output json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['password'])")

echo "gitea=${#PW_GITEA} grafana=${#PW_GRAFANA} wp=${#PW_WP} sewdb=${#PW_SEWDB} harbor_admin=${#PW_HARBOR_ADMIN} harbor_badorius=${#PW_HARBOR_BADORIUS}"
# All lengths should be non-zero before continuing
```

---

## Step 3 — Update Kubernetes secrets

```bash
# Gitea — requires BOTH username and password keys
kubectl delete secret gitea-admin-secret -n gitea --ignore-not-found
kubectl create secret generic gitea-admin-secret -n gitea \
  --from-literal=username=gitea_admin \
  --from-literal=password="${PW_GITEA}"

# Grafana
kubectl delete secret grafana-admin-credentials -n monitoring --ignore-not-found
kubectl create secret generic grafana-admin-credentials -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="${PW_GRAFANA}"

# Woodpecker — preserve OAuth creds, only rotate agent secret
kubectl delete secret woodpecker-server-secret -n woodpecker --ignore-not-found
kubectl create secret generic woodpecker-server-secret -n woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET="${PW_WP}" \
  --from-literal=WOODPECKER_GITEA_CLIENT="${WP_OAUTH_CLIENT}" \
  --from-literal=WOODPECKER_GITEA_SECRET="${WP_OAUTH_SECRET}"

kubectl delete secret woodpecker-agent-secret -n woodpecker --ignore-not-found
kubectl create secret generic woodpecker-agent-secret -n woodpecker \
  --from-literal=WOODPECKER_AGENT_SECRET="${PW_WP}"

# Sewbase DB
kubectl delete secret sewbase-db-secrets -n sewbase --ignore-not-found
kubectl create secret generic sewbase-db-secrets -n sewbase \
  --from-literal=postgres-password="${PW_SEWDB}"

kubectl delete secret sewbase-app-secrets -n sewbase --ignore-not-found
kubectl create secret generic sewbase-app-secrets -n sewbase \
  --from-literal=DATABASE_URL="postgresql://sewuser:${PW_SEWDB}@postgres:5432/sewbase" \
  --from-literal=AUTH_SECRET="${AUTH_SECRET}"

# Harbor pull secret (used by sewbase to pull images from registry.home)
kubectl delete secret harbor-pull-secret -n sewbase --ignore-not-found
kubectl create secret docker-registry harbor-pull-secret -n sewbase \
  --docker-server=registry.home \
  --docker-username=badorius \
  --docker-password="${PW_HARBOR_BADORIUS}" \
  --docker-email=salvadormuntane@gmail.com
```

---

## Step 4 — Update Proton Pass vault

Delete existing items and recreate (update --field does not work for native password fields):

```bash
HOMELAB_SHARE="MZIHb6UgjzrLPYBHW_Hg79lPNEWE1EPOUDPy-LD0m2Sx_-n3uKe5qL399TLk62zBa2lTry4Y504Ru67E3OzKgQ=="

# Get current item IDs
pass-cli item list homelab

# Delete each item by ID (replace IDs with current ones from list above)
pass-cli item delete --share-id "${HOMELAB_SHARE}" --item-id "<ID>"

# Recreate with correct passwords
pass-cli item create login --vault-name homelab --title gitea-admin \
  --username gitea_admin --password "${PW_GITEA}" --url https://gitea.home

pass-cli item create login --vault-name homelab --title grafana-admin \
  --username admin --password "${PW_GRAFANA}" --url https://grafana.home

pass-cli item create login --vault-name homelab --title woodpecker-agent-secret \
  --username woodpecker-agent --password "${PW_WP}" --url https://ci.home

pass-cli item create login --vault-name homelab --title sewbase-db \
  --username sewuser --password "${PW_SEWDB}" --url postgres://sewbase

pass-cli item create login --vault-name homelab --title harbor-admin \
  --username admin --password "${PW_HARBOR_ADMIN}" --url https://registry.home

pass-cli item create login --vault-name homelab --title harbor-badorius \
  --username badorius --password "${PW_HARBOR_BADORIUS}" --url https://registry.home
```

Verify all vault items have non-empty passwords:

```bash
for item in gitea-admin grafana-admin harbor-admin harbor-badorius woodpecker-agent-secret sewbase-db; do
  LEN=$(pass-cli item view --vault-name homelab --item-title "${item}" --output json 2>/dev/null \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d['item']['content']['content']['Login']['password']))" 2>/dev/null)
  echo "${item}: password length = ${LEN:-ERROR}"
done
```

---

## Step 5 — Rotate PostgreSQL password in the database

The K8s secret is updated but the database itself still uses the old password until you run ALTER USER.

```bash
# Connect using the pod's own env var (old password still in pod memory)
# Then change to the new password from the updated secret
NEW_PG_PASS=$(kubectl get secret sewbase-db-secrets -n sewbase \
  -o jsonpath='{.data.postgres-password}' | base64 -d)

kubectl exec -n sewbase statefulset/postgres -- \
  bash -c "PGPASSWORD=\"\$POSTGRES_PASSWORD\" psql -U \"\$POSTGRES_USER\" -d \"\$POSTGRES_DB\" \
  -c \"ALTER USER \$POSTGRES_USER WITH PASSWORD '${NEW_PG_PASS}';\""
# Expected output: ALTER ROLE
```

---

## Step 6 — Restart pods to pick up new secrets

```bash
kubectl rollout restart deployment gitea -n gitea
kubectl rollout restart deployment prometheus-grafana -n monitoring
kubectl rollout restart statefulset woodpecker-server -n woodpecker
kubectl rollout restart statefulset woodpecker-agent -n woodpecker
kubectl rollout restart statefulset postgres -n sewbase
kubectl rollout restart deployment sewbase -n sewbase
```

> **Gitea note**: Gitea uses a ReadWriteOnce PVC with a leveldb lock. Rolling
> restart can deadlock if the old pod holds the lock. If the new pod stays in
> CrashLoopBackOff with "unable to lock level db", scale to 0 then back to 1:
> ```bash
> kubectl scale deployment gitea -n gitea --replicas=0
> sleep 10
> kubectl scale deployment gitea -n gitea --replicas=1
> ```

---

## Step 7 — Rotate Harbor passwords (manual UI step)

Harbor stores passwords in its own internal database. K8s secrets do not control
Harbor's login credentials after initial setup.

1. Log in to `https://registry.home` with the current admin password
2. **Admin**: User Management → admin → Edit → change password to `${PW_HARBOR_ADMIN}`
3. **badorius**: User Management → badorius → Edit → change password to `${PW_HARBOR_BADORIUS}`

Verify via API after changing:

```bash
PW_HARBOR_ADMIN=$(pass-cli item view --vault-name homelab --item-title harbor-admin --output json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['item']['content']['content']['Login']['password'])")
PW_HARBOR_BADORIUS=$(pass-cli item view --vault-name homelab --item-title harbor-badorius --output json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['item']['content']['content']['Login']['password'])")

curl -sk -o /dev/null -w "Harbor admin: HTTP %{http_code}\n" \
  https://registry.home/api/v2.0/users -u "admin:${PW_HARBOR_ADMIN}"
curl -sk -o /dev/null -w "Harbor badorius: HTTP %{http_code}\n" \
  https://registry.home/api/v2.0/users/current -u "badorius:${PW_HARBOR_BADORIUS}"
# Both should return HTTP 200
```

---

## Step 8 — Verify all services

```bash
# Check all pods are Running
for ns in gitea monitoring woodpecker sewbase harbor; do
  kubectl get pods -n $ns --no-headers | awk -v ns="$ns" '{print ns": "$1" "$3}'
done

# Test Gitea login (password sync happens automatically via configure-gitea init container)
curl -sk -o /dev/null -w "Gitea: HTTP %{http_code}\n" \
  https://gitea.home/api/v1/user \
  -u "gitea_admin:$(pass-cli item view --vault-name homelab --item-title gitea-admin \
    --output json | python3 -c "import sys,json; print(json.load(sys.stdin)['item']['content']['content']['Login']['password'])")"

# Test PostgreSQL
NEW_PG=$(kubectl get secret sewbase-db-secrets -n sewbase \
  -o jsonpath='{.data.postgres-password}' | base64 -d)
kubectl exec -n sewbase statefulset/postgres -- \
  bash -c "PGPASSWORD=\"${NEW_PG}\" psql -U sewuser -d sewbase -c 'SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='"'"'public'"'"';'"
# Should return 11 (number of sewbase tables)
```

---

## Rotating the CA certificate

The homelab Root CA cert is stored as a Proton Pass note (not in the repo).
The cert expires **2026-07-12**. When cert-manager regenerates the CA:

```bash
# 1. Export new CA cert from cluster
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/homelab-new-ca.crt

# 2. Update vault note
HOMELAB_SHARE="MZIHb6UgjzrLPYBHW_Hg79lPNEWE1EPOUDPy-LD0m2Sx_-n3uKe5qL399TLk62zBa2lTry4Y504Ru67E3OzKgQ=="
CA_ITEM_ID=$(pass-cli item list homelab 2>&1 | grep homelab-root-ca | sed 's/.*\[\(.*\)\].*/\1/')
pass-cli item delete --share-id "${HOMELAB_SHARE}" --item-id "${CA_ITEM_ID}"
python3 -c "
import json
cert = open('/tmp/homelab-new-ca.crt').read()
print(json.dumps({'title': 'homelab-root-ca', 'note': cert}))
" | pass-cli item create note --vault-name homelab --from-template -

# 3. Re-run Ansible common role to distribute new cert to all hosts
cd ansible && ansible-playbook playbooks/setup_hosts.yml --tags ca_trust
```

---

## Git history audit

Before making the repo public, scan for leaked secrets:

```bash
# Scan current files
grep -rIE "(password|token|secret)\s*[=:]\s*['\"]?[A-Za-z0-9@#\$]{8,}" \
  --include="*.yaml" --include="*.yml" --include="*.md" . \
  | grep -v "YOUR_\|REDACTED\|pass-cli\|secretName\|from_secret\|kubectl"

# Scan all commit messages
git log --all --format="%H %B" | grep -iE "password|token|secret" \
  | grep -vE "REDACTED|YOUR_|pass-cli|vault|kubectl|secretName"

# Scan all file content in git history
git log --all -p | grep -E "^\+" \
  | grep -iE "(password|token|secret)\s*[=:]\s*['\"]?[A-Za-z0-9@#\$]{8,}" \
  | grep -vE "YOUR_|REDACTED|pass-cli|kubectl|secretName|placeholder"
```

If a secret is found in git history, rewrite with `git filter-branch`:

```bash
# Replace the leaked value in all commits
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch --msg-filter \
  'sed "s/LEAKED_VALUE/REDACTED/g"' -- --all

# Or remove a file entirely from all commits
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch --index-filter \
  'git rm --cached --ignore-unmatch path/to/secret-file' -- --all

# Clean up backup refs
git for-each-ref --format="delete %(refname)" refs/original/ | git update-ref --stdin
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push to remote
git push --force origin main
```
