# host/ — ce qui tourne sur le Pi hors de Kubernetes

Ce dossier n'est **pas** deploye par ArgoCD (aucune Application ne pointe ici) :
c'est la copie versionnee de ce qui est installe a la main sur le noeud
`server`. La source de verite reste le systeme ; ce dossier doit etre recopie
apres chaque modification.

| Fichier du depot | Installe dans | Role |
|---|---|---|
| `bin/homelab-watch` | `/usr/local/bin/` | sonde toutes les 10 min, alertes ntfy, battement de coeur quotidien |
| `bin/homelab-db-backup` | `/usr/local/bin/` | copie coherente du datastore k3s (`VACUUM INTO`) et des bases Plex (`.backup`) vers `/var/backups/homelab/db` (root 700, fichiers 600, 14 j) |
| `systemd/homelab-watch.{service,timer}` | `/etc/systemd/system/` | sonde, hors k3s volontairement |
| `systemd/homelab-db-backup.{service,timer}` | `/etc/systemd/system/` | sauvegarde 02:30, `OnFailure=` vers la notification |
| `systemd/homelab-notify@.service` | `/etc/systemd/system/` | notification ntfy d'un echec d'unite (`--unit-failed`) |
| `systemd/homelab-fw.service` | `/etc/systemd/system/` | charge `/etc/nftables.d/homelab-fw.nft` (non versionne : prefixe IPv6 du domicile) |
| `etc/homelab-watch.conf.example` | `/etc/homelab-watch.conf` | **modele** ; la vraie conf (sujet ntfy) ne doit jamais entrer ici |

Pourquoi en systemd et non en CronJob : une sonde ou une sauvegarde qui tourne
dans le cluster meurt avec lui (panne du 2026-07-19 : kubelet fige 19 jours).

Ce que la sonde signale (toutes les 10 min, alerte apres un delai de grace,
rappel toutes les 6 h, message de retablissement) : horloge, k3s, boucle CRI,
noeud, pods, ArgoCD, argocd-image-updater, disque, temperature, DNS du cluster,
NVMe, archives media-config, age de la derniere copie du datastore (36 h),
pare-feu `homelab_fw`, unites `homelab-*` en echec. Plus un message quotidien
apres 9 h : **pas de message un matin = la sonde ne tourne plus**.

Pourquoi les copies du datastore ne sont plus dans `~/backups/db` (avant le
2026-11-04) : `~/backups` appartient a l'UID 1000 et il est monte en ecriture
dans le pod media-backup, qui pouvait y remplacer `db/`.

## Installer apres modification

```bash
sudo install -o root -g root -m 755 host/bin/homelab-watch host/bin/homelab-db-backup /usr/local/bin/
sudo install -o root -g root -m 644 host/systemd/homelab-* /etc/systemd/system/
sudo systemctl daemon-reload
```

## Tester sans notifier ni toucher l'etat reel

```bash
T=$(mktemp -d)
sudo env CONF=/dev/null NTFY_TOPIC= STATE_DIR="$T" TMPDIR="$T" host/bin/homelab-watch --dry-run --heartbeat
sudo env CONF=/dev/null NTFY_TOPIC= STATE_DIR="$T" TMPDIR="$T" host/bin/homelab-watch --dry-run --unit-failed homelab-db-backup.service
sudo rm -rf "$T"
```

## Verifier qu'aucun secret n'y est entre (avant chaque commit)

```bash
T=$(sudo sed -nE 's/^NTFY_TOPIC=["'\'']?([^"'\'' #]+).*/\1/p' /etc/homelab-watch.conf)
[ -n "$T" ] && ! /usr/bin/grep -rqF -- "$T" host/ && echo "sujet ntfy absent : OK"
# motifs ecrits [x] pour ne pas se trouver eux-memes dans ce README
! /usr/bin/grep -rqE 'gh[p]_|github_[p]at_|PRIVATE [K]EY|2a01[:]' host/ && echo "aucun jeton/cle/prefixe : OK"
unset T
```
