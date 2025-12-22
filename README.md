# Media Stack K8s

Stack média déployée sur K3s avec ArgoCD (GitOps).

## Services

| Service | Description | Port | Namespace |
|---------|-------------|------|-----------|
| Cloudflared | DNS over HTTPS (anti-censure) | ClusterIP 5053 | media-stack |
| Plex | Media Server avec transcodage HW | 32400 (hostNetwork) | media-stack |
| qBittorrent | Client torrent (anti-seeding) | 8080 (hostPort) | media-stack |
| Home Assistant | Domotique open source | 8123 (hostNetwork) | home-assistant |

## Déploiement

```bash
# Appliquer le root app (App of Apps pattern)
kubectl apply -f apps/root-app.yaml

# Suivre le déploiement
kubectl get applications -n argocd -w
```

## Accès

- **ArgoCD**: https://192.168.1.51:30443
- **Plex**: http://192.168.1.51:32400/web
- **qBittorrent**: http://192.168.1.51:8080
- **Home Assistant**: http://192.168.1.51:8123

## Structure

```
├── apps/               # ArgoCD Application manifests
├── base/               # Ressources de base (namespace)
└── charts/             # Helm Charts
    ├── cloudflared/
    ├── homeassistant/
    ├── plex/
    └── qbittorrent/
```

