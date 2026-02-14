# Backup & Restore Guide

## Vue d'ensemble

Les données importantes à sauvegarder :

| Service | Chemin | Priorité | Description |
|---------|--------|----------|-------------|
| Plex | `/home/muchini/media-data/config/plex/` | Haute | Bibliothèque, préférences, watch history |
| qBittorrent | `/home/muchini/media-data/config/qbittorrent/` | Moyenne | Configuration, trackers |
| Cloudflared | N/A | Basse | Stateless, configuration dans Helm |

## Backup

### Script de backup complet

```bash
#!/bin/bash
# backup-media-stack.sh

BACKUP_DIR="/home/muchini/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/media-stack-backup-${DATE}.tar.gz"

# Créer le répertoire de backup
mkdir -p "${BACKUP_DIR}"

# Arrêter les pods pour un backup cohérent (optionnel)
kubectl scale deployment -n media-stack plex --replicas=0
kubectl scale deployment -n media-stack qbittorrent --replicas=0

# Attendre l'arrêt
sleep 10

# Créer l'archive
tar -czvf "${BACKUP_FILE}" \
  /home/muchini/media-data/config/plex \
  /home/muchini/media-data/config/qbittorrent

# Redémarrer les pods
kubectl scale deployment -n media-stack plex --replicas=1
kubectl scale deployment -n media-stack qbittorrent --replicas=1

echo "Backup créé: ${BACKUP_FILE}"

# Nettoyer les backups de plus de 30 jours
find "${BACKUP_DIR}" -name "media-stack-backup-*.tar.gz" -mtime +30 -delete
```

### Backup Plex uniquement

```bash
# Sauvegarder la base de données Plex (plus rapide)
tar -czvf plex-db-backup.tar.gz \
  /home/muchini/media-data/config/plex/Library/Application\ Support/Plex\ Media\ Server/Plug-in\ Support/Databases/
```

### Automatisation avec cron

```bash
# Ajouter au crontab
crontab -e

# Backup quotidien à 3h du matin
0 3 * * * /home/muchini/scripts/backup-media-stack.sh >> /var/log/media-stack-backup.log 2>&1
```

## Restore

### Restauration complète

```bash
#!/bin/bash
# restore-media-stack.sh

BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup-file.tar.gz>"
  exit 1
fi

# Arrêter les pods
kubectl scale deployment -n media-stack plex --replicas=0
kubectl scale deployment -n media-stack qbittorrent --replicas=0

# Attendre l'arrêt
sleep 10

# Extraire l'archive
tar -xzvf "${BACKUP_FILE}" -C /

# Fixer les permissions
chown -R 1000:1000 /home/muchini/media-data/config/

# Redémarrer les pods
kubectl scale deployment -n media-stack plex --replicas=1
kubectl scale deployment -n media-stack qbittorrent --replicas=1

echo "Restauration terminée"
```

### Restauration sur nouveau cluster

1. **Installer K3s et ArgoCD**
   ```bash
   curl -sfL https://get.k3s.io | sh -
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Restaurer les données**
   ```bash
   # Créer les répertoires
   mkdir -p /home/muchini/media-data/config/{plex,qbittorrent,cloudflared}

   # Extraire le backup
   tar -xzvf media-stack-backup-XXXXXX.tar.gz -C /
   ```

3. **Déployer la stack**
   ```bash
   kubectl apply -f apps/root-app.yaml
   ```

## Vérification post-restore

```bash
# Vérifier les pods
kubectl get pods -n media-stack

# Vérifier les logs
kubectl logs -n media-stack deployment/plex --tail=50
kubectl logs -n media-stack deployment/qbittorrent --tail=50

# Tester les services
curl http://localhost:32400/web/index.html
curl http://localhost:8080
```

## Backup offsite (recommandé)

### Avec rclone vers un stockage cloud

```bash
# Installer rclone
sudo apt install rclone

# Configurer (ex: Google Drive, S3, etc.)
rclone config

# Sync le backup
rclone copy /home/muchini/backups/ remote:media-stack-backups/
```

### Avec restic (backup incrémental chiffré)

```bash
# Initialiser le repository
restic -r /path/to/backup init

# Backup
restic -r /path/to/backup backup /home/muchini/media-data/config/

# Restore
restic -r /path/to/backup restore latest --target /
```
