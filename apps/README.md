# 📁 Apps - Applications ArgoCD

Ce dossier contient les définitions des **Applications ArgoCD** qui orchestrent le déploiement de la stack média via GitOps.

## 🏗️ Architecture App of Apps

```mermaid
graph TB
    subgraph "🔄 ArgoCD"
        RootApp[📦 root-app.yaml<br/>media-stack]
    end

    subgraph "📂 Applications Enfants"
        NS[🏷️ namespace.yaml]
        CF[🛡️ cloudflared.yaml]
        PX[🎥 plex.yaml]
        QB[⬇️ qbittorrent.yaml]
        HA[🏡 homeassistant.yaml]
    end

    subgraph "📊 Charts Helm"
        CFChart[charts/cloudflared/]
        PXChart[charts/plex/]
        QBChart[charts/qbittorrent/]
        HAChart[charts/homeassistant/]
    end

    RootApp -->|"sync"| NS
    RootApp -->|"sync"| CF
    RootApp -->|"sync"| PX
    RootApp -->|"sync"| QB
    RootApp -->|"sync"| HA

    CF -->|"déploie"| CFChart
    PX -->|"déploie"| PXChart
    QB -->|"déploie"| QBChart
    HA -->|"déploie"| HAChart
```

## 📄 Fichiers

| Fichier | Description |
|---------|-------------|
| 🚀 `root-app.yaml` | Application parente - Point d'entrée ArgoCD |
| 🏷️ `namespace.yaml` | Crée le namespace media-stack |
| 🛡️ `cloudflared.yaml` | Déploie le proxy DNS-over-HTTPS |
| 🎥 `plex.yaml` | Déploie Plex Media Server |
| ⬇️ `qbittorrent.yaml` | Déploie qBittorrent |
| 🏡 `homeassistant.yaml` | Déploie Home Assistant |

## 🔄 Flux de synchronisation

```mermaid
sequenceDiagram
    participant GH as ☁️ GitHub
    participant Argo as 🔄 ArgoCD
    participant Root as 📦 root-app
    participant Apps as 📂 Apps enfants

    GH->>Argo: 🔔 Changement détecté
    Argo->>Root: 📥 Sync root-app
    Root->>Apps: 📥 Découvre les apps enfants
    Apps->>Argo: ✅ Sync chaque application
    Note over Argo,Apps: selfHeal: true<br/>prune: true
```

## ⚙️ Configuration commune

Toutes les applications partagent:

- **`selfHeal: true`** - Annule les changements manuels kubectl
- **`prune: true`** - Supprime les ressources obsolètes
- **`CreateNamespace: true`** - Crée les namespaces automatiquement

## 🎯 Utilisation

```bash
# 📥 Déployer toute la stack
kubectl apply -f apps/root-app.yaml

# 👀 Surveiller les applications
kubectl get applications -n argocd

# 🔄 Forcer la sync d'une app
argocd app sync cloudflared
```
