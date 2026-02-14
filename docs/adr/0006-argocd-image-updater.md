# ADR-0006: ArgoCD Image Updater

## Statut

Accepté

## Contexte

Les images des containers (Plex, qBittorrent, Cloudflared) reçoivent régulièrement des mises à jour de sécurité et de fonctionnalités. Sans automatisation, les mises à jour nécessitent:
- Surveillance manuelle des nouvelles versions
- Modification manuelle des fichiers values.yaml
- Commit et push vers Git
- Attente de la synchronisation ArgoCD

Ce processus manuel est fastidieux et peut entraîner des retards dans l'application des correctifs de sécurité.

## Décision

Implémenter ArgoCD Image Updater avec la configuration suivante:

### Stratégie de mise à jour
- **Semver**: Toutes les versions (Major, Minor, Patch)
- Contrainte: `*` (aucune restriction de version)
- Filtre regex: `^[0-9]+\.[0-9]+\.[0-9]+$` (ignore les tags comme `latest`, `dev`)

### Write-back
- **Méthode**: Git (commit direct vers le repository)
- **Cible**: `values.yaml` de chaque chart Helm
- **Branche**: `main`
- **Authentification**: Secret `git-creds` avec token GitHub

### Images surveillées
| Application | Registry | Image |
|------------|----------|-------|
| Cloudflared | Docker Hub | `cloudflare/cloudflared` |
| Plex | LinuxServer | `lscr.io/linuxserver/plex` |
| qBittorrent | LinuxServer | `lscr.io/linuxserver/qbittorrent` |

## Conséquences

### Positives
- Mises à jour automatiques de toutes les versions (major, minor, patch)
- Correctifs de sécurité appliqués rapidement
- Nouvelles fonctionnalités disponibles automatiquement
- Traçabilité via commits Git automatiques
- Respect du workflow GitOps (état désiré dans Git)
- Rollback facile via Git revert

### Négatives
- Risque de breaking changes avec les mises à jour majeures
- Nécessite un token GitHub avec accès en écriture
- Dépendance à un composant supplémentaire (Image Updater)
- Les mises à jour peuvent arriver à des moments inopportuns

### Mitigations
- Le filtre regex évite les tags instables (latest, dev, etc.)
- Monitoring des logs Image Updater recommandé
- PodDisruptionBudgets garantissent la disponibilité
- Rollback via `git revert` en cas de problème

## Références

- [ArgoCD Image Updater Documentation](https://argocd-image-updater.readthedocs.io/)
- [Semver Update Strategy](https://argocd-image-updater.readthedocs.io/en/stable/configuration/images/#update-strategy-semver)
- [Git Write-Back Method](https://argocd-image-updater.readthedocs.io/en/stable/configuration/basics/#git-write-back-method)
