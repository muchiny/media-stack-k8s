# 🔐 CoreDNS en DNS-over-TLS

> ⚠️ **Ce dossier n'est PAS déployé par ArgoCD.** CoreDNS vit dans `kube-system`,
> hors du périmètre de la stack média. Les fichiers ici sont la **source de vérité
> versionnée** d'une configuration appliquée manuellement sur le cluster.

## Pourquoi

Le `Corefile` par défaut de k3s se termine par `forward . /etc/resolv.conf`, donc
toutes les requêtes DNS du cluster partaient **en clair** vers `1.1.1.1` et
`8.8.8.8`. CoreDNS sait chiffrer lui-même : le `forward` accepte des destinations
`tls://`. Aucun conteneur supplémentaire n'est nécessaire.

Un chart `dnscrypt-proxy` avait été déployé pour ça auparavant. Il n'a **jamais
reçu la moindre requête** — CoreDNS ne lui transmettait rien. Il a été supprimé
le 2026-08-09 (commit `c36a74a`).

## Pourquoi pas la ConfigMap `coredns-custom`

k3s fournit un crochet `coredns-custom` (déjà monté par le déploiement CoreDNS,
`optional: true`), importé dans le `Corefile` :

```
import /etc/coredns/custom/*.override
forward . /etc/resolv.conf
```

Ce mécanisme est **additif**. Y injecter un second `forward .` pour la même zone
produit un doublon que CoreDNS refuse. Il permet d'*ajouter* des zones, pas de
*remplacer* l'amont. C'est probablement ce mur qui avait mené au déploiement d'un
pod dédié.

## Comment c'est appliqué

k3s régénère `/var/lib/rancher/k3s/server/manifests/coredns.yaml` à chaque
démarrage et réconcilie la ConfigMap (annotation `objectset.rio.cattle.io`).
Toute modification directe serait donc écrasée. On demande à k3s de ne plus la
gérer :

```bash
sudo touch /var/lib/rancher/k3s/server/manifests/coredns.yaml.skip
kubectl -n kube-system create configmap coredns \
  --from-file=Corefile=k3s/Corefile \
  --from-literal=NodeHosts="$(kubectl get cm coredns -n kube-system -o jsonpath='{.data.NodeHosts}')" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Le plugin `reload` de CoreDNS applique le changement **sans redémarrer le pod**
(~60 s). En cas de `Corefile` invalide, il conserve l'ancienne configuration.

## ⚠️ Contrepartie à connaître

**Les mises à jour de k3s ne toucheront plus CoreDNS.** Nouvelle version, correctif
de sécurité, changement de format du `Corefile` : c'est désormais à toi de suivre.
CoreDNS est le composant dont dépend tout le reste — à vérifier après chaque
montée de version de k3s.

## Retour arrière (2 commandes)

```bash
sudo rm /var/lib/rancher/k3s/server/manifests/coredns.yaml.skip
sudo kubectl apply -f /etc/k3s-custom/coredns-cm-ORIGINAL.yaml
```

Une copie de secours du `Corefile` d'origine et de la version DoT est conservée
hors dépôt dans `/etc/k3s-custom/`.

## Vérifier que le chiffrement est actif

```bash
sudo tcpdump -i any -nn 'host 1.1.1.1 and tcp port 853' -c 5
```

Du trafic sur le **port 853** = DNS-over-TLS. Du trafic sur le **port 53** vers un
résolveur public = requêtes en clair, donc problème.

## Ce que ça protège, et ce que ça ne protège pas

| | |
|---|---|
| ✅ | Ton FAI ne voit plus **quels domaines** tu résous |
| ❌ | Il voit toujours les **adresses IP** que tu contactes |
| ❌ | Cloudflare, lui, voit toujours tout — c'est un déplacement de confiance, pas une suppression |

---

# 🔀 Traefik : celui de k3s

Depuis le 2026-11-03, le Traefik du cluster est **celui intégré à k3s**. Il suit
donc les montées de version de k3s (k3s v1.36.5 → Traefik 3.7.13). Avant cette
date, un Traefik 3.6.8 installé à la main par Helm (février 2026, `--disable
traefik` côté k3s) n'était mis à jour par personne.

Sa configuration est dans [`traefik-config.yaml`](traefik-config.yaml) (un
`HelmChartConfig`), copié à la main dans
`/var/lib/rancher/k3s/server/manifests/traefik-config.yaml` :

- **NodePorts 30090 (HTTP) et 30453 (HTTPS)** : `servicelb` reste désactivé, donc
  pas de `LoadBalancer`. Le pare-feu `homelab_fw` ne les ouvre qu'au LAN.
- **Seules les IngressRoutes de `media-stack`** sont servies, sans Ingress classique.
- **RBAC par namespace** : Traefik ne lit les Secrets que de `kube-system` et
  `media-stack`. Il ne peut plus lire `argocd/git-creds`.

Modifier la configuration : éditer ce fichier, puis
`sudo install -m 600 k3s/traefik-config.yaml /var/lib/rancher/k3s/server/manifests/`.
k3s relance l'installation Helm tout seul.

Vérifier :

```bash
kubectl get helmchart,helmchartconfig -n kube-system
kubectl auth can-i get secret/git-creds -n argocd \
  --as=system:serviceaccount:kube-system:traefik   # → no
curl -sk -o /dev/null -w '%{http_code}\n' --resolve plex.local:30453:192.168.1.51 \
  https://plex.local:30453/identity                # → 200
```

⚠️ **Ne pas remettre `--disable traefik`** dans `/etc/systemd/system/k3s.service` :
k3s désinstallerait son Traefik.
