# CLAUDE.md — Homelab Infrastructure: AI Agent Rules

---

## MANDATORY SESSION STARTUP — DO THIS FIRST, EVERY TIME

Before responding to the user, execute these steps in order:

1. **Read `agents.md`** — current session state, infrastructure status, pending actions.
2. **Read `../sewbase_guitea/AGENTS.md §9 Outbox → homelab-infra`** — check for pending items from a sewbase session that affect this repo. If any items are listed there (not marked `[done]`), acknowledge them to the user immediately and process them.
3. **Begin the session.**

## MANDATORY SESSION TEARDOWN — DO THIS BEFORE CLOSING

Before ending the session, execute these steps:

1. **Assess cross-repo impact**: did any change in this session affect sewbase? (Examples: Woodpecker config changes, ArgoCD app changes, Harbor config, new cluster-level resources sewbase depends on, cert-manager or ingress changes.)
2. **If yes**: write the item(s) to `agents.md §12 Outbox → sewbase` with date and description, then commit and push `agents.md`.
3. **Update `agents.md §10 Session State`** with what changed this session.

---

## Repository Purpose

This repo contains all Infrastructure-as-Code for the homelab k3s cluster.
Everything is declarative — no snowflakes, no ad-hoc changes, destroy and recreate at any time.

Companion application repo: `../sewbase_guitea` — Next.js sewing app deployed on this cluster.

---

## Non-Negotiable Rules

1. **No secrets in this repo** — The repo is **public on GitHub**. No passwords, tokens, or credentials anywhere. Use `kubectl create secret` for runtime injection. For documentation, reference Proton Pass vault `homelab` using `pass-cli item view --vault-name homelab --item-title <name> --field password`.

2. **IaC-first** — Every infrastructure change must be expressed in Ansible playbooks or Kubernetes manifests. No ad-hoc `kubectl apply` for managed services.

3. **Code and docs in English** — All code, comments, and documentation must be in English. Conversations with the owner may be in Spanish.

4. **ArgoCD manages CD** — For ArgoCD-managed services, push to git and sync. Don't apply manifests manually (exception: secrets, which are never in git).

5. **TLS everywhere** — All ingresses must have `cert-manager.io/cluster-issuer: homelab-ca` annotation.

6. **DNS for every service** — Add DNS entries in `ansible/roles/openwrt-dns/tasks/main.yml` for every new service.

---

## Infrastructure Overview

| Component | Location | Notes |
|-----------|----------|-------|
| k3s cluster | vm01 (master) + vm02/vm03 (workers) | 1 master, 2 workers |
| ArgoCD | `https://argocd.home` | App-of-Apps pattern |
| Gitea | `https://gitea.home` | Private git server, user: `gitea_admin` |
| Woodpecker CI | `https://ci.home` | OAuth via Gitea |
| Harbor | `https://registry.home` | Container registry |
| cert-manager | In-cluster | homelab-ca ClusterIssuer (self-signed) |
| Traefik | In-cluster | Ingress controller |
| NFS StorageClass | `nfs-client` | Backed by OMV NAS at `192.168.8.15` |

Full details: `agents.md §3` and `agents.md §6`.

---

## Proton Pass Vault: `homelab`

All credentials are stored in Proton Pass. Retrieve with:
```bash
pass-cli item view --vault-name homelab --item-title <name> --field password
```

| Item title | Purpose |
|-----------|---------|
| `harbor-admin` | Harbor admin password |
| `harbor-badorius` | Harbor user `badorius` password |
| `gitea-admin` | Gitea admin password (user: `gitea_admin`) |
| `grafana-admin` | Grafana admin password |
| `woodpecker-agent-secret` | Woodpecker agent shared secret |
| `sewbase-db` | PostgreSQL password for sewbase `sewuser` |
| `sewbase-auth` | NextAuth AUTH_SECRET for sewbase |
| `s3-secret` | S3-compatible storage secret key (create when configured) |
