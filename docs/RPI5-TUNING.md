# Guide de Performance Raspberry Pi 5

Ce guide couvre les optimisations de performance spécifiques à l'exécution de la stack média sur Raspberry Pi 5 (arm64).

## Configuration Hardware

### Mémoire
- RPi5 disponible en variantes 4 Go ou 8 Go
- Overhead K3s: ~512 Mo
- Les quotas de ressources limitent media-stack à 6 Gi requests / 8 Gi limits

### GPU (VideoCore VII)
- Transcodage matériel Plex via `/dev/dri`
- Nécessite `privileged: true` dans le contexte de sécurité du container
- Partage mémoire GPU configuré dans `/boot/firmware/config.txt`

```bash
# Vérifier l'allocation mémoire GPU
vcgencmd get_mem gpu
```

### Stockage
- Utiliser une microSD rapide (classe A2) ou NVMe via USB 3.0
- Volumes hostPath sur `/home/muchini/media-data/`
- Considérer un disque séparé pour les médias vs la config

## Optimisations K3s

### Désactiver les composants inutilisés

```bash
# Installation K3s avec empreinte minimale
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --disable servicelb \
  --disable local-storage
```

### Limites mémoire

```yaml
# Configuration K3s (/etc/rancher/k3s/config.yaml)
kubelet-arg:
  - "system-reserved=cpu=200m,memory=200Mi"
  - "kube-reserved=cpu=200m,memory=300Mi"
```

## Optimisations des containers

### Politique de pull des images
Tous les charts utilisent `pullPolicy: IfNotPresent` pour éviter les téléchargements inutiles.

### Requests/Limits des ressources

| Service | CPU Request | Memory Request | CPU Limit | Memory Limit |
|---------|-------------|----------------|-----------|--------------|
| Cloudflared | 50m | 32Mi | 100m | 64Mi |
| Plex | 1000m | 1Gi | 3500m | 4Gi |
| qBittorrent | 100m | 256Mi | 1000m | 1Gi |

### tmpfs pour les données transitoires
- Répertoire de transcodage Plex utilise `emptyDir` avec `medium: Memory`
- Données de session qBittorrent utilisent tmpfs pour réduire l'usure de la carte SD

## Optimisations réseau

### Services hostNetwork
- **Plex**: Requis pour la découverte DLNA/GDM
- Utilise `dnsPolicy: ClusterFirstWithHostNet` pour résoudre les DNS du cluster

### CoreDNS + Cloudflared
- Requêtes DNS routées via Cloudflared pour DoH
- Réduit la latence DNS de l'ISP et améliore la confidentialité

## Monitoring

### Utilisation des ressources

```bash
# Vérifier les ressources du nœud
kubectl top nodes

# Vérifier les ressources des pods
kubectl top pods -n media-stack

# Vérifier les événements OOM
dmesg | grep -i oom

# Voir les métriques détaillées
kubectl describe node
```

### Monitoring de la température

```bash
# Vérifier la température CPU
vcgencmd measure_temp

# Monitoring continu
watch -n 1 vcgencmd measure_temp

# Vérifier le throttling
vcgencmd get_throttled
# 0x0 = OK
# 0x50000 = throttling actif
```

### Seuils de température
- **< 60°C**: Normal
- **60-80°C**: Charge élevée, acceptable
- **> 80°C**: Throttling possible, améliorer le refroidissement
- **> 85°C**: Throttling actif, intervention requise

## Troubleshooting

### Utilisation CPU élevée

1. **Vérifier le transcodage Plex**
   ```bash
   kubectl top pods -n media-stack | grep plex
   ```
   Solution: Passer en lecture directe (Direct Play) dans les paramètres client

2. **Limiter les connexions qBittorrent**
   - WebUI > Options > Connexion
   - Réduire les connexions max par torrent

3. **Vérifier les intervalles de polling**
   - Réduire la fréquence de sync ArgoCD si nécessaire

### Pression mémoire

1. **Identifier les pods consommateurs**
   ```bash
   kubectl top pods -n media-stack --sort-by=memory
   ```

2. **Vérifier les fuites mémoire**
   ```bash
   # Surveiller l'évolution sur quelques heures
   watch -n 60 kubectl top pods -n media-stack
   ```

3. **Réduire le buffer de transcodage Plex**
   - Paramètres Plex > Transcodeur > Qualité du transcodage

4. **Augmenter le swap** (non recommandé pour cartes SD)
   ```bash
   # Si nécessaire, utiliser zram plutôt que swap fichier
   sudo apt install zram-tools
   ```

### Usure carte SD

1. **Utiliser tmpfs pour les données temporaires** (déjà configuré)
   - Transcodage Plex: emptyDir Memory
   - Session qBittorrent: tmpfs

2. **Déplacer les configs vers disque externe**
   ```bash
   # Monter un disque USB/NVMe pour /home/muchini/media-data
   ```

3. **Activer noatime**
   ```bash
   # Dans /etc/fstab
   /dev/mmcblk0p2 / ext4 defaults,noatime 0 1
   ```

4. **Surveiller la santé de la carte SD**
   ```bash
   # Vérifier les erreurs
   dmesg | grep -i mmc
   ```

## Recommandations de refroidissement

Pour des performances optimales sur RPi5:

1. **Dissipateur passif**: Minimum requis
2. **Ventilateur actif**: Recommandé pour charge continue
3. **Boîtier ventilé**: Éviter les boîtiers fermés sans ventilation
4. **Position**: Éviter les espaces confinés

## Commandes utiles

```bash
# État général du système
htop

# Informations RPi5
cat /proc/cpuinfo | grep Model
vcgencmd version

# Espace disque
df -h /home/muchini/media-data

# I/O disque
iostat -x 1 5

# Connexions réseau
ss -tunlp | grep -E '(5053|32400|8080|6881)'
```
