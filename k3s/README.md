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

## 🔢 Version de CoreDNS : montée manuelle

À cause du `.skip`, k3s réécrit `coredns.yaml` à chaque démarrage **sans
l'appliquer** : l'image CoreDNS ne suit plus k3s.

| Date | k3s | CoreDNS en service | Digest de l'index multi-arch |
|---|---|---|---|
| 2026-11-03 | v1.36.5+k3s1 | 1.14.7 (était 1.14.3) | `sha256:7efd3c635b03efd68c4e8398fc45f0d993d0e9ab016f72c1cefb0fd6d01aa286` |

Après chaque montée de k3s, comparer l'image voulue par k3s et celle en service :

```bash
sudo grep 'image:' /var/lib/rancher/k3s/server/manifests/coredns.yaml
kubectl -n kube-system get deploy coredns \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Puis comparer tout le reste du manifeste (Deployment, RBAC, Service), **ConfigMap
exclue**. Lecture seule : `kubectl diff` ne modifie rien.

```bash
sudo cat /var/lib/rancher/k3s/server/manifests/coredns.yaml \
  | awk 'BEGIN{RS="\n---\n"; ORS="\n---\n"} !/kind: ConfigMap\n/' \
  | kubectl diff -f -
```

Normalement, seules l'image et la stratégie (`maxUnavailable` 0 ici, 1 chez k3s)
diffèrent : on monte alors **uniquement l'image**. Toute autre différence (règle
RBAC, args, sondes) doit être reportée à la main avant le `set image`. Ne jamais
faire `kubectl apply` de ce fichier : il contient la ConfigMap d'origine
(`forward . /etc/resolv.conf`) et ferait repartir le DNS en clair, sans aucune
erreur visible.

```bash
sudo k3s crictl pull docker.io/<image du manifeste k3s>
OLD=$(kubectl -n kube-system get pod -l k8s-app=kube-dns -o name)
kubectl -n kube-system set image deployment/coredns coredns=<image du manifeste k3s>
kubectl -n kube-system rollout status deployment/coredns --timeout=180s
kubectl -n kube-system wait --for=delete "$OLD" --timeout=90s
```

- Stratégie passée à `maxSurge: 1` / `maxUnavailable: 0` le 2026-11-03 : le
  nouveau pod est prêt avant l'arrêt de l'ancien, donc pas de coupure DNS.
  Avec la valeur de k3s (`maxUnavailable: 1`, une seule réplique), l'ancien pod
  était tué tout de suite.
- `revisionHistoryLimit: 0` : `kubectl rollout undo` ne marche pas. Retour
  arrière = `set image` vers la version précédente.
- Avant de monter, lire les notes de version (`notes/coredns-1.x.y.md` du dépôt
  coredns) pour `forward`, `hosts`, `cache` et `kubernetes`.
- Après, vérifier le chiffrement (section plus bas), les logs, et la résolution
  depuis le pod qBittorrent : c'est le seul dont la sortie DNS est filtrée par
  une NetworkPolicy.
- Pour tester les noms courts depuis un pod, utiliser `wget` (résolveur de la
  libc) et non le `nslookup` de busybox, qui ignore la liste `search`.

## Retour arrière du DoT (2 commandes)

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
