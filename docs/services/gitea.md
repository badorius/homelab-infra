# Gitea

Self-hosted private Git server. Hosts private application repos (sewbase_guitea, etc.) and acts as the OAuth provider for Woodpecker CI.

## Access

| Property | Value |
|----------|-------|
| URL | https://gitea.home |
| Admin username | gitea_admin |
| Admin password | REDACTED (from gitea-admin-secret) |
| Namespace | gitea |
| Chart | gitea/gitea 10.1.4 |
| Managed by | ArgoCD (`gitea` Application) |

## Architecture

```
ArgoCD → kubernetes/services/gitea/kustomization.yaml
  └─ Helm chart with:
       ├─ Gitea app (Deployment)
       ├─ PostgreSQL 16 (StatefulSet) — image from public.ecr.aws (avoids Docker Hub rate limits)
       └─ Redis (internal cache)
```

Storage: 10Gi PVC on nfs-client for Gitea data, shared with PostgreSQL.

## Repos Hosted

| Repo | Visibility | Purpose |
|------|-----------|---------|
| gitea_admin/sewbase_guitea | Private | Sewbase Next.js application |

## OAuth2 for Woodpecker

Woodpecker uses Gitea OAuth to authenticate users. The OAuth app is registered under gitea_admin's account:

- **Client ID**: stored in `woodpecker-server-secret.WOODPECKER_GITEA_CLIENT`
- **Redirect URI**: `https://ci.home/authorize`

If the OAuth app is lost (after namespace wipe), recreate it:

1. Login to `https://gitea.home` as gitea_admin
2. Settings → Applications → Manage OAuth2 Applications
3. Create: Name=`Woodpecker`, Redirect URI=`https://ci.home/authorize`
4. Update Woodpecker secret:
   ```bash
   kubectl patch secret woodpecker-server-secret -n woodpecker \
     -p="{\"data\":{
       \"WOODPECKER_GITEA_CLIENT\": \"$(echo -n 'CLIENT_ID' | base64)\",
       \"WOODPECKER_GITEA_SECRET\": \"$(echo -n 'CLIENT_SECRET' | base64)\"
     }}"
   kubectl rollout restart statefulset woodpecker-server -n woodpecker
   ```

## API Usage

```bash
# Create API token (for scripts)
curl -k -u gitea_admin:REDACTED \
  -X POST https://gitea.home/api/v1/users/gitea_admin/tokens \
  -H "Content-Type: application/json" \
  -d '{"name":"my-token"}'

# List repos
curl -k -u gitea_admin:REDACTED https://gitea.home/api/v1/repos/search

# Create repo
curl -k -u gitea_admin:REDACTED \
  -X POST https://gitea.home/api/v1/user/repos \
  -H "Content-Type: application/json" \
  -d '{"name":"myapp","private":true}'
```

## Git Remotes (using HTTPS)

Since the homelab CA is self-signed, configure git to use the CA cert:

```bash
# Per-repo
git config http.sslCAInfo /path/to/homelab-root-ca.crt

# Global (for all gitea.home repos)
git config --global http.https://gitea.home.sslCAInfo /path/to/homelab-root-ca.crt
```

Remote URL format: `https://gitea_admin:REDACTED@gitea.home/gitea_admin/REPO.git`

## Troubleshooting

| Issue | Fix |
|-------|-----|
| CrashLoopBackOff after restart | LevelDB lock: scale to 0 then to 1 |
| PostgreSQL image pull fails | Tag pruned from ECR. Update `postgresql.image.tag` in kustomization.yaml |
| "SSL certificate" git error | Configure `http.sslCAInfo` with the homelab CA |
| OAuth "Client ID not registered" | Recreate OAuth app and update woodpecker-server-secret |
