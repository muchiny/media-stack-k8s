# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

K3s-based media stack deployed via ArgoCD GitOps on Raspberry Pi 5 (arm64). Uses the **App of Apps** pattern where `apps/root-app.yaml` is the parent application that syncs all child applications.

## Architecture

```
                    ┌──────────────────┐
                    │  ArgoCD (GitOps) │
                    │   watches repo   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────────┐
        │Cloudflared│  │   Plex   │  │ qBittorrent  │
        │ DNS-over- │  │ hostNet  │  │  NodePort    │
        │  HTTPS    │  │  :32400  │  │   :30080     │
        └─────┬─────┘  └──────────┘  └──────┬───────┘
              │                             │
              │      CoreDNS forwards       │
              └─────────────────────────────┘
                  qBittorrent uses secure DNS
```

**Key design decisions:**
- Cloudflared has fixed ClusterIP (`10.43.48.123`) for CoreDNS integration
- Plex uses `hostNetwork: true` for DLNA/GDM discovery
- qBittorrent has init container waiting for Cloudflared DNS
- All pods use `hostPath` volumes pointing to `/home/muchini/media-stack/`

## Commands

```bash
# Deploy everything (initial or after repo changes)
kubectl apply -f apps/root-app.yaml

# Watch sync status
kubectl get applications -n argocd -w

# Check pods
kubectl get pods -n media-stack

# View ArgoCD UI
# https://192.168.1.51:30443

# Force sync a specific app
argocd app sync cloudflared
argocd app sync plex
argocd app sync qbittorrent

# Rollback to Docker (if K3s fails)
sudo systemctl stop k3s
cd ~/media-stack/docker && docker compose start
```

## Helm Chart Testing

```bash
# Validate chart templates
helm template charts/cloudflared
helm template charts/plex
helm template charts/qbittorrent

# Lint charts
helm lint charts/cloudflared
helm lint charts/plex
helm lint charts/qbittorrent
```

## Critical Constraints

1. **DO NOT** enable seeding in qBittorrent configuration
2. **DO NOT** expose Cloudflared externally (ClusterIP only)
3. **DO NOT** add *arr services (Radarr, Sonarr, etc.) - intentionally removed
4. Plex requires `privileged: true` for hardware transcoding via `/dev/dri`
5. All apps auto-sync with `selfHeal: true` - manual kubectl changes will be reverted

## Host Paths

All services mount from the Docker-era directory structure:
- Config: `/home/muchini/media-stack/docker/{service}/`
- Media: `/home/muchini/media-stack/media/`
- Torrents: `/home/muchini/media-stack/torrents/`
