# ADR-0003: hostNetwork pour Plex

## Statut

Accepté

## Contexte

Plex Media Server utilise les protocoles DLNA et GDM (G'Day Mate) pour la découverte automatique par les clients sur le réseau local. Ces protocoles reposent sur le multicast et le broadcast réseau qui ne fonctionnent pas correctement à travers l'overlay réseau Kubernetes.

## Décision

Configurer Plex avec `hostNetwork: true` pour lui permettre d'accéder directement à l'interface réseau de l'hôte.

Configuration dans `charts/plex/values.yaml`:
```yaml
hostNetwork: true
```

Cela nécessite également `dnsPolicy: ClusterFirstWithHostNet` pour continuer à résoudre les DNS du cluster.

## Conséquences

### Positives
- Découverte DLNA/GDM fonctionnelle sur le réseau local
- Les clients Plex (TV, téléphones) peuvent détecter automatiquement le serveur
- Latence réseau réduite pour le streaming

### Négatives
- NetworkPolicy a un effet limité avec hostNetwork
- Le port 32400 est directement exposé sur l'hôte
- Potentiel conflit de ports avec d'autres services

## Références

- [Plex Network Requirements](https://support.plex.tv/articles/200430283-network/)
- [DLNA Protocol](https://en.wikipedia.org/wiki/Digital_Living_Network_Alliance)
