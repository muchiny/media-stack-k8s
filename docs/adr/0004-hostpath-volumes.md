# ADR-0004: Volumes hostPath

## Statut

Accepté

## Contexte

Le cluster K3s tourne sur un seul nœud Raspberry Pi 5. Nous avons besoin de stockage persistant pour:
- Les configurations des services
- Les fichiers média
- Les téléchargements torrents

Les solutions de stockage distribuées (Longhorn, Rook-Ceph) sont surdimensionnées pour un cluster single-node et consommeraient des ressources précieuses.

## Décision

Utiliser des volumes `hostPath` pointant vers `/home/muchini/media-data/` avec la structure:
```
/home/muchini/media-data/
├── config/
│   ├── cloudflared/
│   ├── plex/
│   └── qbittorrent/
├── torrents/
└── (lien vers /media pour les fichiers média)
```

## Conséquences

### Positives
- Simple et direct, pas de surcharge
- Performance optimale (accès direct au disque)
- Facile à sauvegarder avec des outils standard (rsync, tar)
- Pas de dépendances supplémentaires

### Négatives
- Pas de réplication des données
- Lié à un seul nœud (pas de migration de pods)
- Nécessite une gestion manuelle des sauvegardes

## Références

- [Kubernetes hostPath Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- Guide de sauvegarde: `docs/BACKUP-RESTORE.md`
