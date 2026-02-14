# ADR-0002: IP fixe pour Cloudflared

## Statut

Accepté

## Contexte

CoreDNS doit transmettre les requêtes DNS à Cloudflared pour le DNS-over-HTTPS. Les Services Kubernetes obtiennent des ClusterIPs aléatoires par défaut, ce qui nécessiterait de mettre à jour le ConfigMap CoreDNS à chaque redéploiement de Cloudflared.

## Décision

Assigner une ClusterIP fixe `10.43.48.123` au service Cloudflared. Cette IP est dans le CIDR de service K3s (10.43.0.0/16) et peu susceptible d'entrer en conflit.

Configuration dans `charts/cloudflared/values.yaml`:
```yaml
service:
  type: ClusterIP
  port: 5053
  clusterIP: "10.43.48.123"
```

## Conséquences

### Positives
- Configuration CoreDNS statique et fiable
- Pas besoin de mettre à jour CoreDNS lors des redémarrages de Cloudflared
- Chemin de résolution DNS prévisible

### Négatives
- Risque de conflit IP si un autre service réclame 10.43.48.123
- Cette IP réservée doit être documentée

## Références

- K3s default service CIDR: 10.43.0.0/16
- CoreDNS forward plugin configuration
