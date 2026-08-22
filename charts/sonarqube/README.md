# 🔎 SonarQube

SonarQube Community Build + PostgreSQL, namespace dédié `sonarqube`.

| Élément | Valeur |
|---|---|
| Image | `sonarqube:26.8.0.126808-community` (arm64 disponible) |
| Base de données | PostgreSQL 17 (StatefulSet, PVC `local-path`) |
| Accès | `http://192.168.1.51:30900/` (NodePort direct) ou `http://sonarqube.local:30090/` (Traefik) |
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

## 🌐 URL publique du serveur

`sonar.core.serverBaseURL` est un reglage **stocke en base**, pas une propriete
process : le mapping `SONAR_*` du conteneur ne le prend pas (verifie, l'env var
reste sans effet). Il se pose une fois via l'API et survit aux redemarrages :

```bash
curl -u admin:<mdp> -X POST http://192.168.1.51:30900/api/settings/set \
  -d 'key=sonar.core.serverBaseURL' \
  --data-urlencode 'value=http://192.168.1.51:30900'
```

Sans lui, les liens renvoyes par l'API - donc par le serveur MCP - pointent sur
`localhost:9000`, injoignable depuis le PC de dev.

## 🦀 Analyse d'un projet Rust

L'analyseur `sonar-rust-plugin` est **inclus** dans cette version (85 règles, 78 actives
dans le profil `Sonar way` par défaut). Rien à installer côté serveur.

L'analyse tourne sur le **PC de dev**, pas sur le Pi : `sonar-scanner` lit les sources,
télécharge l'analyseur depuis le serveur et pousse le rapport. Le Pi n'exécute que le
Compute Engine.

`sonar-project.properties` à la racine du projet Rust :

```properties
sonar.projectKey=bridge-mcp
sonar.projectName=bridge-mcp
sonar.sources=src,crates,benches
sonar.tests=tests
sonar.host.url=http://192.168.1.51:30900

# Workspace Cargo : declarer chaque manifeste
sonar.rust.cargo.manifestPaths=Cargo.toml,crates/bridge-mcp-macros/Cargo.toml

# Clippy : l'analyseur peut lancer cargo clippy lui-meme...
sonar.rust.clippy.enabled=true
# ...ou importer un rapport deja produit en CI :
# sonar.rust.clippyReport.reportPaths=target/clippy.json

# Couverture (cargo-llvm-cov --lcov)
sonar.rust.lcov.reportPaths=target/lcov.info
```

Lancement depuis le PC de dev :

```bash
export SONAR_TOKEN='<token-utilisateur>'
cargo llvm-cov --workspace --lcov --output-path target/lcov.info   # optionnel
sonar-scanner
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
