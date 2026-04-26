# Woodpecker CI

CI/CD pipeline runner. Triggered by Gitea webhooks on push events.

## Access

| Property | Value |
|----------|-------|
| URL | https://ci.home |
| Auth | Gitea OAuth (login with gitea_admin — see Proton Pass vault homelab: **gitea-admin**) |
| Namespace | woodpecker |
| Chart | woodpecker-ci.org/woodpecker 1.5.0 |
| Managed by | ArgoCD (`woodpecker` Application) |

## Architecture

```
woodpecker-server (StatefulSet, 1 replica)
  ├─ Listens for Gitea webhooks
  ├─ Manages pipeline state (SQLite DB in PVC)
  └─ Distributes work to agents via gRPC port 9000

woodpecker-agent (StatefulSet, 2 replicas)
  ├─ Pulls pipeline steps from server
  ├─ Runs each step as a container (docker-in-docker or via host socket)
  └─ Mounts: homelab-ca-cert ConfigMap (for Harbor TLS trust)
```

## Required Kubernetes Secrets

| Secret | Namespace | Keys |
|--------|-----------|------|
| woodpecker-server-secret | woodpecker | WOODPECKER_GITEA_CLIENT, WOODPECKER_GITEA_SECRET, WOODPECKER_AGENT_SECRET |
| woodpecker-agent-secret | woodpecker | WOODPECKER_AGENT_SECRET |
| homelab-ca-cert | woodpecker | homelab-root-ca.crt (ConfigMap, not secret) |

The `WOODPECKER_AGENT_SECRET` must be identical in both server and agent secrets.

## Pipeline Secret Variables

These are stored in Woodpecker's database (not Kubernetes). Available to all pipelines:

| Secret name | Value | Purpose |
|-------------|-------|---------|
| `docker_password` | see Proton Pass vault homelab: **harbor-badorius** | Push images to Harbor as `badorius` |
| `argocd_server` | argocd.home | ArgoCD API host for deploy step |
| `argocd_token` | (generated) | ArgoCD API Bearer token |

Set via:
```bash
export WOODPECKER_SERVER=https://ci.home
export WOODPECKER_TOKEN=<from-ui-settings>
woodpecker-cli secret add --global --name docker_password --value "$(pass-cli item view --vault-name homelab --item-title harbor-badorius --field password)"
woodpecker-cli secret add --global --name argocd_server --value argocd.home
woodpecker-cli secret add --global --name argocd_token --value '<argocd-token>'
```

## Writing a Pipeline

Pipelines are defined in `.woodpecker.yml` at the repo root. Reference:

```yaml
steps:
  # Step 1: Run tests
  test:
    image: node:20-alpine
    commands:
      - npm ci
      - npm run test -- --passWithNoTests

  # Step 2: Build and push Docker image to Harbor
  publish:
    image: woodpeckerci/plugin-docker-buildx
    privileged: true
    settings:
      registry: registry.home
      repo: registry.home/badorius/myapp
      tags: latest
      username: badorius
      password:
        from_secret: docker_password
      buildkit_config: |
        [registry."registry.home"]
          insecure = true      # skip TLS verify for self-signed cert
    when:
      branch: main
      event: push

  # Step 3: Trigger ArgoCD sync
  deploy:
    image: curlimages/curl:latest
    environment:
      ARGOCD_SERVER:
        from_secret: argocd_server
      ARGOCD_TOKEN:
        from_secret: argocd_token
    commands:
      - |
        curl -k -s -f -X POST \
          "https://${ARGOCD_SERVER}/api/v1/applications/myapp/sync" \
          -H "Authorization: Bearer ${ARGOCD_TOKEN}" \
          -H "Content-Type: application/json" \
          -d '{"hardRefresh":true}'
    when:
      branch: main
      event: push
```

## Admin User

The `WOODPECKER_ADMIN: "admin"` env var in the Helm values means the Gitea user named `admin` is an admin in Woodpecker. In our setup, the Gitea admin is `gitea_admin`, so the first time `gitea_admin` logs into Woodpecker via OAuth, they become the Woodpecker admin.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Client ID not registered" | Gitea OAuth app missing. Recreate it (see Gitea docs) and update `woodpecker-server-secret` |
| Agent not connecting to server | `WOODPECKER_AGENT_SECRET` mismatch. Ensure both secrets have identical value |
| Docker push fails with cert error | `buildkit_config` missing `insecure = true` for registry.home |
| Docker push 401 Unauthorized | Wrong `docker_password`. Check Harbor credentials for `badorius` user |
| Pipeline not triggered | Gitea webhook not configured. Activate repo in Woodpecker UI |
| Steps not running | Agent pods not Ready: `kubectl get pods -n woodpecker` |
