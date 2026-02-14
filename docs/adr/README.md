# Architecture Decision Records

Ce répertoire contient les Architecture Decision Records (ADRs) documentant les décisions architecturales significatives du projet media-stack-k8s.

## Index

| ADR | Titre | Statut |
|-----|-------|--------|
| [0000](0000-template.md) | Template ADR | - |
| [0001](0001-argocd-app-of-apps.md) | Pattern App of Apps ArgoCD | Accepté |
| [0002](0002-dnscrypt-proxy-fixed-clusterip.md) | IP fixe pour dnscrypt-proxy | Accepté |
| [0003](0003-hostnetwork-plex.md) | hostNetwork pour Plex | Accepté |
| [0004](0004-hostpath-volumes.md) | Volumes hostPath | Accepté |
| [0005](0005-priority-classes.md) | Classes de priorité | Accepté |
| [0006](0006-argocd-image-updater.md) | ArgoCD Image Updater | Accepté |

## Créer un nouvel ADR

1. Copier `0000-template.md` vers un nouveau fichier avec le prochain numéro
2. Remplir toutes les sections
3. Mettre à jour l'index dans ce README
