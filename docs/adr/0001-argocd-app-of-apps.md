# ADR-0001: Pattern App of Apps ArgoCD

## Statut

Accepté

## Contexte

Nous avons besoin d'une stratégie de déploiement GitOps pour plusieurs services média sur K3s. Gérer les applications individuellement entraîne une dérive de configuration et nécessite des interventions manuelles.

## Décision

Utiliser le pattern "App of Apps" ArgoCD où:
- `apps/root-app.yaml` est l'Application parente
- Elle synchronise toutes les Applications enfants dans le répertoire `apps/`
- Chaque Application enfant déploie un chart Helm depuis `charts/`
- Les sync waves garantissent l'ordre de déploiement (0: infrastructure, 1: DNS, 2: services)

## Conséquences

### Positives
- Point d'entrée unique pour le déploiement (`kubectl apply -f apps/root-app.yaml`)
- Toutes les applications gérées déclarativement via Git
- Synchronisation automatique avec `selfHeal: true`
- Facile d'ajouter de nouveaux services
- Ordre de déploiement garanti par les sync waves

### Négatives
- Nécessite ArgoCD pré-installé
- Les changements kubectl manuels sont automatiquement annulés
- Complexité supplémentaire pour les nouveaux contributeurs

## Références

- [ArgoCD App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
