# ADR-0005: Classes de priorité

## Statut

Accepté

## Contexte

Le Raspberry Pi 5 a des ressources limitées (4-8 Go RAM, 4 cores). En cas de pression sur les ressources, Kubernetes doit savoir quels pods évacuer en premier et lesquels préserver.

## Décision

Définir trois classes de priorité dans `apps/priority-classes.yaml`:

| Classe | Valeur | Services | Justification |
|--------|--------|----------|---------------|
| `media-critical` | 1,000,000 | Cloudflared | DNS critique pour tous les services |
| `media-high` | 900,000 | Plex, qBittorrent | Services utilisateur principaux |
| `media-normal` | 800,000 | (Réservé) | Services secondaires |

L'ordre d'éviction en cas de pression mémoire sera:
1. `media-normal` (évacué en premier)
2. `media-high`
3. `media-critical` (protégé au maximum)

## Conséquences

### Positives
- Le DNS reste disponible même sous forte charge
- Comportement prévisible lors des évictions
- Permet une dégradation gracieuse des services

### Négatives
- Complexité de configuration supplémentaire
- Risque de famine pour les pods basse priorité

## Références

- [Kubernetes Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
