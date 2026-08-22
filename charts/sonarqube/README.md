# 🔎 SonarQube

SonarQube Community Build + PostgreSQL, namespace dédié `sonarqube`.

| Élément | Valeur |
|---|---|
| Image | `sonarqube:26.8.0.126808-community` (arm64 disponible) |
| Base de données | PostgreSQL 17 (StatefulSet, PVC `local-path`) |
| Accès | `http://sonarqube.local/` via Traefik, ou `port-forward` sur 9000 |
| Identifiants initiaux | `admin` / `admin` (changement forcé) |
| Namespace | `sonarqube` — **hors** `media-stack`, donc hors ResourceQuota média |

## 🔐 Secret PostgreSQL (obligatoire, hors Git)

Le chart ne template **aucun** Secret. À créer une fois sur le cluster :

```bash
kubectl create namespace sonarqube --dry-run=client -o yaml | kubectl apply -f -
kubectl -n sonarqube create secret generic sonarqube-db \
  --from-literal=username=sonarqube \
  --from-literal=password="$(openssl rand -base64 24 | tr -d '/+=' | head -c 24)"
```

Sans ce Secret, les pods restent en `CreateContainerConfigError`.

## 🐳 Basculer sur la Docker Hardened Image (DHI)

Une DHI SonarQube existe bien : `dhi.io/sonarqube`, en **linux/amd64 et linux/arm64**,
base Debian 13, tags `26.x`. Elle n'est **pas** pullable anonymement — le registre
`dhi.io` renvoie 401 sans authentification. Il faut un compte Docker avec accès DHI
(essai Enterprise 30 jours, ou abonnement Select/Enterprise).

```bash
# 1. Vérifier l'accès depuis le Pi
docker login dhi.io
docker manifest inspect dhi.io/sonarqube:26 | grep architecture

# 2. Créer le pull secret dans le cluster
kubectl -n sonarqube create secret docker-registry dhi-registry \
  --docker-server=dhi.io \
  --docker-username='<user-docker>' \
  --docker-password='<PAT-docker>'
```

Puis dans `values.yaml` :

```yaml
image:
  repository: dhi.io/sonarqube     # ou <org>/dhi-sonarqube si repo mirroré
  tag: "26"
imagePullSecrets:
  - name: dhi-registry
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 65532                 # les DHI tournent en nonroot 65532
  runAsGroup: 65532
  fsGroup: 65532
```

⚠️ Le changement d'UID casse les PVC existants (fichiers en `1000:1000`). Sur une
instance déjà peuplée, rechown avant bascule :

```bash
sudo chown -R 65532:65532 /var/lib/rancher/k3s/storage/*sonarqube-{data,extensions,logs}*
```

## 🧠 Dimensionnement

SonarQube démarre 3 JVM (web, compute engine, Elasticsearch). Les heaps sont bornés
explicitement dans `values.yaml` (`javaOpts`) : sans cela elles se dimensionnent sur
les 8 Gi du Pi et le pod se fait OOM-kill par sa propre limite de 3 Gi.

`vm.max_map_count` du nœud vaut déjà 1048576 (> 262144 requis par Elasticsearch).

Aucune `priorityClassName` n'est définie : en cas de pression mémoire, SonarQube est
évincé avant Plex et qBittorrent (`media-high`, 900000).

## 🧪 Validation locale

```bash
helm lint charts/sonarqube
helm template charts/sonarqube | kubeconform -strict -summary
kube-linter lint charts/sonarqube --config .kube-linter.yaml
```
