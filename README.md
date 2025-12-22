# Media Stack K8s

Stack média déployée sur K3s avec ArgoCD (GitOps).

## Services

| Service | Description | Port |
|---------|-------------|------|
| Cloudflared | DNS over HTTPS (anti-censure) | ClusterIP 5053 |
| Plex | Media Server avec transcodage HW | 32400 (hostNetwork) |
| qBittorrent | Client torrent (anti-seeding) | NodePort 30080 |

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
- **qBittorrent**: http://192.168.1.51:30080

## Structure

```
├── apps/               # ArgoCD Application manifests
├── base/               # Ressources de base (namespace)
└── charts/             # Helm Charts
    ├── cloudflared/
    ├── plex/
    └── qbittorrent/
```

