# Troubleshooting Guide

## Problèmes courants et solutions

### DNS / Cloudflared

#### Le pod Cloudflared ne démarre pas

```bash
# Vérifier les logs
kubectl logs -n media-stack deployment/cloudflared

# Vérifier les événements
kubectl describe pod -n media-stack -l app.kubernetes.io/name=cloudflared
```

**Causes possibles :**
- Image non disponible pour arm64
- Ressources insuffisantes

#### qBittorrent bloqué sur "Waiting for Cloudflared"

```bash
# Vérifier si le service DNS est accessible
kubectl exec -n media-stack -it deployment/qbittorrent -c wait-for-dns -- nslookup cloudflared.media-stack.svc.cluster.local

# Vérifier le service Cloudflared
kubectl get svc -n media-stack cloudflared
```

### Plex

#### Plex n'est pas accessible

```bash
# Vérifier que le pod est Running
kubectl get pods -n media-stack -l app.kubernetes.io/name=plex

# Vérifier que hostNetwork est activé
kubectl get pod -n media-stack -l app.kubernetes.io/name=plex -o jsonpath='{.items[0].spec.hostNetwork}'
```

**Solutions :**
- Vérifier que le port 32400 n'est pas utilisé par un autre processus
- Vérifier les règles firewall sur le Raspberry Pi

#### Transcodage lent ou échoué

```bash
# Vérifier l'accès GPU
kubectl exec -n media-stack deployment/plex -- ls -la /dev/dri

# Vérifier les logs de transcodage
kubectl logs -n media-stack deployment/plex | grep -i transcode
```

**Solutions :**
- S'assurer que `gpu.enabled: true` dans values.yaml
- Vérifier que le conteneur est en mode `privileged`

### qBittorrent

#### WebUI inaccessible

```bash
# Vérifier le port
kubectl get svc -n media-stack qbittorrent-webui

# Tester la connectivité
curl -v http://<node-ip>:30080
```

#### Téléchargements bloqués

Vérifier les logs pour des erreurs DNS :
```bash
kubectl logs -n media-stack deployment/qbittorrent | grep -i dns
```

### ArgoCD

#### Application OutOfSync

```bash
# Voir les différences
argocd app diff <app-name>

# Forcer la sync
argocd app sync <app-name> --force

# Vérifier la santé
argocd app get <app-name>
```

#### selfHeal annule les changements manuels

C'est le comportement attendu. Pour faire des modifications :
1. Modifier les fichiers dans le repository Git
2. Push les changements
3. ArgoCD synchronisera automatiquement

### Stockage

#### Permission denied sur les volumes

```bash
# Vérifier les permissions sur l'hôte
ls -la /home/muchini/media-data/

# Fixer les permissions
sudo chown -R 1000:1000 /home/muchini/media-data/
```

#### Espace disque insuffisant

```bash
# Vérifier l'espace
df -h /home/muchini/media-data/

# Nettoyer les fichiers temporaires
rm -rf /home/muchini/media-data/config/plex/Library/Application\ Support/Plex\ Media\ Server/Cache/*
```

## Commandes de diagnostic

```bash
# État global
kubectl get pods -n media-stack
kubectl get events -n media-stack --sort-by='.lastTimestamp'

# Ressources utilisées
kubectl top pods -n media-stack

# Logs en temps réel
kubectl logs -f -n media-stack deployment/<service>

# Entrer dans un conteneur
kubectl exec -it -n media-stack deployment/<service> -- /bin/sh
```

## Réinitialisation complète

En dernier recours :
```bash
# Supprimer les applications ArgoCD
argocd app delete root-app --cascade

# Recréer
kubectl apply -f apps/root-app.yaml
```
