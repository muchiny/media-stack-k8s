# ADR-0002: IP fixe pour dnscrypt-proxy

## Statut

Accepté

## Contexte

CoreDNS doit transmettre les requêtes DNS à dnscrypt-proxy pour le DNS-over-HTTPS. Les Services Kubernetes obtiennent des ClusterIPs aléatoires par défaut, ce qui nécessiterait de mettre à jour le ConfigMap CoreDNS à chaque redéploiement de dnscrypt-proxy.

Note: dnscrypt-proxy remplace cloudflared depuis la version 2026.2.0 qui a supprimé la fonctionnalité `proxy-dns`.

## Décision

Assigner une ClusterIP fixe `10.43.48.123` au service dnscrypt-proxy. Cette IP est dans le CIDR de service K3s (10.43.0.0/16) et peu susceptible d'entrer en conflit.

Configuration dans `charts/dnscrypt-proxy/values.yaml`:
```yaml
service:
  type: ClusterIP
  port: 5053
  clusterIP: "10.43.48.123"
```

## Conséquences

### Positives
- Configuration CoreDNS statique et fiable
- Pas besoin de mettre à jour CoreDNS lors des redémarrages de dnscrypt-proxy
- Chemin de résolution DNS prévisible
- dnscrypt-proxy offre plus de fonctionnalités (cache, DNSSEC)

### Négatives
- Risque de conflit IP si un autre service réclame 10.43.48.123
- Cette IP réservée doit être documentée

## Références

- K3s default service CIDR: 10.43.0.0/16
- CoreDNS forward plugin configuration
- [dnscrypt-proxy documentation](https://github.com/DNSCrypt/dnscrypt-proxy)
