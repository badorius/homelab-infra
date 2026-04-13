# GitOps Implementation Session State

*Date: 2026-04-13*

This file serves as a memory checkpoint for the AI agent and the user to resume the homelab infrastructure deployment in a future session. 

## Completed Objectives
1. **Zero-Secret Public Repo Setup**: All hardcoded passwords/tokens were successfully stripped from the `kustomization.yaml` files. Secrets are now securely referenced via `existingSecret` pointers inside the Kubernetes `gitea`, `woodpecker`, and `harbor` namespaces.
2. **Dynamic TLS & Cert-Manager**: 
   - Deployed `cert-manager` and established a local CA (`homelab-ca`).
   - Upgraded all legacy services (`homepage`, `grafana`, `qbittorrent`) and new services to leverage the CA and issue on-the-fly TLS certificates on their Traefik ingresses.
   - The Root CA (`homelab-root-ca.crt`) was exported and imported by the user to trust these certificates locally.
3. **ArgoCD Bootstrapping (App of Apps)**: ArgoCD was installed and successfully orchestrated to watch the `badorius/homelab-infra` repository via the `applications.yaml` meta-definition.
4. **Service Deployments**: Harbor and Woodpecker were successfully recognized and spun up by ArgoCD. 

## Current Active Blocker / Unresolved Issue
**Target**: `gitea` deployment (Specifically `gitea-postgresql` database pod).
**Issue**: The Gitea PostgreSQL database is stuck in an `ErrImagePull / ImagePullBackOff` loop.
**Context**:
- The Gitea Helm chart (v10.1.4) default values point to `docker.io/bitnami/postgresql:16.2.0-debian-12-r8`.
- Bitnami has pruned this strict tag from Docker Hub, producing a `not found` error.
- We switched `postgresql-ha` off and enabled the standard `postgresql` subchart. 
- We applied Kustomize `valuesInline` and direct JSON patches targeting the `StatefulSet/gitea-postgresql` to force the image tag to `latest`.
- **The Loop**: Despite the patches being pushed to Git and ArgoCD reporting `Synced`, the Kubernetes node is still trying to spawn pods using the old `16` or `16.2.0` tags. This is likely due to the StatefulSet retaining its immutable template cache, or ArgoCD's subchart rendering logic ignoring the deep string replacement.

## Next Steps for the Next Session
1. **Solve Gitea DB StatefulSet**: Clean up the cached state by completely blowing away the Gitea namespace or the StatefulSet (`kubectl delete statefulset gitea-postgresql -n gitea`). Allow ArgoCD to re-render and recreate the StatefulSet from scratch with the `latest` tag that is currently in Git.
2. **Gitea Health Check**: Verify that `gitea-postgresql-0` successfully pulls `latest` and `gitea` pods spin up properly over `https://gitea.home`.
3. **Manual Human Step (OAuth)**: 
   - Log into `https://gitea.home` (admin/REDACTED).
   - Create the OAuth2 application for Woodpecker.
   - Inject the `Client ID` and `Client Secret` into the cluster using:
```bash
kubectl patch secret woodpecker-server-secret -n woodpecker \
  --type='merge' \
  -p="{\"stringData\":{\"WOODPECKER_GITEA_CLIENT\":\"YOUR_CLIENT\",\"WOODPECKER_GITEA_SECRET\":\"YOUR_SECRET\"}}"

kubectl rollout restart deploy/woodpecker-server -n woodpecker
```
4. **Final System Verification**: Test a Woodpecker pipeline pushing to Harbor.
