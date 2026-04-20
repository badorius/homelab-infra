# GitOps CI/CD Architecture

This document describes the automated GitOps continuous integration and continuous deployment pipeline running on the homelab K3s cluster.

## Architecture Overview

The infrastructure relies on four core components to deliver zero-touch deployments:

1. **Gitea (`gitea.home`)**: The version control system. It holds the source code and triggers webhooks on every `git push`.
2. **Woodpecker CI (`ci.home`)**: The continuous integration engine. It intercepts webhooks from Gitea, runs unit tests, builds the Docker images, and pushes them to the registry.
3. **Harbor (`registry.home`)**: The private container registry. It stores all built Docker images securely.
4. **ArgoCD (`argocd.home`)**: The continuous deployment GitOps engine. It monitors the `homelab-infra` repository and application repositories. When a new image tag is detected or Kubernetes manifests change, ArgoCD automatically synchronizes the state to the K3s cluster.

## The GitOps Flow

1. **Developer Action**: Code is pushed to `main` branch on a Gitea repository (e.g., `sewbase_guitea`).
2. **CI Trigger**: Gitea sends a webhook to Woodpecker CI.
3. **Pipeline Execution**: Woodpecker reads the `.woodpecker.yml` file, runs `npm test`, builds the Docker container, and pushes `latest` (or tagged) to Harbor.
4. **CD Sync**: ArgoCD polls Harbor and/or Gitea for manifest changes. Upon detecting a new image/commit, ArgoCD performs a rollout restart of the affected Kubernetes deployments (e.g., `sewbase`).

## Day 2 Operations & Troubleshooting

### Woodpecker CI "Client ID not registered" Error

Since Woodpecker relies on Gitea for OAuth2 authentication, its deployment initially uses placeholder secrets. If you experience an authorization error when logging into `https://ci.home`, the OAuth2 application needs to be registered.

**Steps to resolve:**

1. Log into Gitea at `https://gitea.home`.
2. Go to **Settings -> Applications**.
3. Under **Manage OAuth2 Applications**, create a new application:
   - **Application Name**: `Woodpecker CI`
   - **Redirect URI**: `https://ci.home/authorize`
4. Gitea will provide a `Client ID` and a `Client Secret`.
5. Update the Kubernetes secret in the `woodpecker` namespace with these values:
   ```bash
   kubectl patch secret woodpecker-server-secret -n woodpecker \
     -p="{\"data\":{\"WOODPECKER_GITEA_CLIENT\": \"<base64-encoded-client-id>\", \"WOODPECKER_GITEA_SECRET\": \"<base64-encoded-client-secret>\"}}"
   ```
6. Restart the Woodpecker server to apply the new secrets:
   ```bash
   kubectl rollout restart statefulset woodpecker-server -n woodpecker
   ```

### No Secrets in the Repository

As a core rule of this GitOps workflow, **no passwords, API keys, or OAuth secrets are ever committed to the repository.**
All sensitive information is managed directly via Kubernetes Secrets (`kubectl create secret` or `kubectl patch`), ensuring the public GitHub/Gitea mirrors remain completely secure.
