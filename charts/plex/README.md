# 🎥 Plex - Media Server

Helm chart pour déployer **Plex Media Server** avec transcodage matériel sur Raspberry Pi 5.

## 🎯 Objectif

```mermaid
graph TB
    subgraph "📱 Clients"
        TV[📺 Smart TV]
        Phone[📱 Mobile]
        Browser[💻 Navigateur]
    end

    subgraph "☸️ Cluster K3s"
        Plex[🎥 Plex<br/>hostNetwork:32400]
    end

    subgraph "💾 Stockage"
        Config[(📁 config/plex/)]
        Media[(🎬 /media/)]
        Transcode[(⚡ /transcode<br/>emptyDir RAM)]
    end

    TV & Phone & Browser -->|"DLNA/HTTP"| Plex
    Plex --> Config
    Plex --> Media
    Plex --> Transcode
```

## 📄 Fichiers

| Fichier | Description |
|---------|-------------|
| 📄 `Chart.yaml` | Métadonnées du chart (v1.0.0, appVersion 1.43.0) |
| ⚙️ `values.yaml` | Configuration par défaut |
| 📂 `templates/` | Templates Kubernetes |

### 📂 Templates

| Template | Ressource | Description |
|----------|-----------|-------------|
| 🔧 `_helpers.tpl` | - | Fonctions helper (labels, selectors) |
| 📋 `deployment.yaml` | Deployment | Pod Plex avec GPU et probes |
| 🌐 `service.yaml` | Service | ClusterIP port 32400 |
| 🛡️ `pdb.yaml` | PodDisruptionBudget | Garantit disponibilité minimale |

## ⚙️ Configuration

```yaml
# values.yaml
image:
  repository: lscr.io/linuxserver/plex
  tag: "1.43.0"

# ⚠️ Requis pour découverte DLNA/GDM
hostNetwork: true

# 🎮 Accès /dev/dri (rend le conteneur privileged) : inutile sur Pi 5
gpu:
  enabled: false

persistence:
  config:
    hostPath: /home/muchini/media-data/config/plex
  media:
    hostPath: /media
    readOnly: true
  transcode:
    type: emptyDir
    medium: Memory  # RAM pour performance

environment:
  PUID: "1000"
  PGID: "1000"
  TZ: "Europe/Paris"

resources:
  limits:
    memory: 4Gi
    cpu: 3500m

service:
  type: ClusterIP
  port: 32400

nodeSelector:
  kubernetes.io/arch: arm64
```

## 🏗️ Architecture

```mermaid
flowchart TB
    subgraph "📦 Deployment"
        Pod[🐳 Pod plex<br/>non privilégié]

        subgraph "Volumes"
            V1[📁 /config<br/>hostPath]
            V2[🎬 /media<br/>hostPath RO]
            V3[⚡ /transcode<br/>emptyDir RAM]
        end
    end

    subgraph "🏠 Réseau Host"
        Port[🌐 Port 32400]
    end

    Pod --> V1 & V2 & V3
    Pod -.->|"hostNetwork"| Port
```

## 🖥️ Transcodage

Le transcodage est **logiciel** (CPU). Le Pi 5 n'a pas d'encodeur vidéo
matériel : `/dev/dri` n'expose que le GPU 3D (v3d) et l'affichage (vc4), que
Plex n'utilise pas (il ne gère que Intel Quick Sync, NVIDIA et AMD).
`gpu.enabled: true` monterait `/dev/dri` mais rendrait le conteneur
`privileged` : ne l'activer que sur du matériel compatible.

## ⚠️ Points critiques

| ⚠️ | Description |
|----|-------------|
| 🔒 | **Non privilégié** - `gpu.enabled: false`, aucun accès à `/dev/dri` |
| 🌐 | **hostNetwork: true** - Requis pour découverte DLNA/GDM |
| ⚡ | **emptyDir Memory** - Transcodage en RAM pour performance |
| 📁 | **Media readOnly** - Protection contre modifications accidentelles |

## 🔧 Commandes

```bash
# ✅ Valider le chart
helm lint charts/plex
helm template charts/plex

# 🔄 Forcer la sync ArgoCD
argocd app sync plex

# 📊 Vérifier le pod
kubectl get pods -n media-stack -l app=plex

# 📋 Voir les logs
kubectl logs -n media-stack -l app=plex -f

# 🌐 Accéder à l'interface
# http://192.168.1.51:32400/web

```
