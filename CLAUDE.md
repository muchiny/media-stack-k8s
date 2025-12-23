# 🤖 CLAUDE.md

Ce fichier fournit des instructions à Claude Code (claude.ai/code) pour travailler avec ce repository.

## 📋 Vue d'ensemble du projet

Stack média K3s déployée via ArgoCD GitOps sur **Raspberry Pi 5 (arm64)**. Utilise le pattern **App of Apps** où `apps/root-app.yaml` est l'application parente qui synchronise toutes les applications enfants.

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "☁️ GitHub"
        Repo[(📦 media-stack-k8s<br/>Repository)]
    end

    subgraph "🖥️ Raspberry Pi 5 - K3s"
        ArgoCD[🔄 ArgoCD<br/>Port: 30443]

        subgraph "📦 Namespace: media-stack"
            CF[🛡️ Cloudflared<br/>ClusterIP: 10.43.48.123<br/>Port: 5053]
            Plex[🎥 Plex<br/>hostNetwork<br/>Port: 32400]
            QB[⬇️ qBittorrent<br/>hostPort: 8080]
        end

        subgraph "🏠 Namespace: home-assistant"
            HA[🏡 Home Assistant<br/>hostNetwork<br/>Port: 8123]
        end

        CoreDNS[🌐 CoreDNS]
        Storage[(💾 /home/muchini/media-data)]
    end

    Repo -->|"GitOps Sync"| ArgoCD
    ArgoCD -->|"Déploie"| CF
    ArgoCD -->|"Déploie"| Plex
    ArgoCD -->|"Déploie"| QB
    ArgoCD -->|"Déploie"| HA
    CoreDNS -->|"Forward DNS"| CF
    Plex --> Storage
    QB --> Storage
    HA --> Storage
```

## 🎯 Décisions de conception clés

```mermaid
mindmap
  root((🏗️ Architecture))
    🛡️ Cloudflared
      ClusterIP fixe 10.43.48.123
      Intégration CoreDNS
      DNS-over-HTTPS
    🎥 Plex
      hostNetwork: true
      Découverte DLNA/GDM
      privileged pour /dev/dri
    ⬇️ qBittorrent
      Init container
      Attend Cloudflared DNS
      Anti-seeding
    🏡 Home Assistant
      hostNetwork: true
      mDNS/SSDP discovery
      Namespace séparé
    💾 Storage
      hostPath volumes
      /home/muchini/media-data/
```

| Composant | Configuration | Raison |
|-----------|--------------|--------|
| 🛡️ Cloudflared | ClusterIP fixe `10.43.48.123` | Intégration CoreDNS |
| 🎥 Plex | `hostNetwork: true` | Découverte DLNA/GDM |
| ⬇️ qBittorrent | Init container | Attend Cloudflared DNS |
| 🏡 Home Assistant | `hostNetwork: true` | Découverte mDNS/SSDP |
| 🏡 Home Assistant | Namespace `home-assistant` | Isolation |
| 💾 Tous les pods | `hostPath` volumes | Stockage `/home/muchini/media-data/` |

## 🔧 Commandes

### ☸️ Déploiement

```bash
# 📥 Déployer tout (initial ou après changements)
kubectl apply -f apps/root-app.yaml

# 👀 Surveiller le statut de sync
kubectl get applications -n argocd -w

# 📊 Vérifier les pods
kubectl get pods -n media-stack
kubectl get pods -n home-assistant

# 🌐 UI ArgoCD
# https://192.168.1.51:30443

# 🔄 Forcer la sync d'une app spécifique
argocd app sync cloudflared
argocd app sync plex
argocd app sync qbittorrent
argocd app sync homeassistant
```

### 🧪 Test des Helm Charts

```bash
# ✅ Valider les templates
helm template charts/cloudflared
helm template charts/plex
helm template charts/qbittorrent
helm template charts/homeassistant

# 🔍 Linter les charts
helm lint charts/cloudflared
helm lint charts/plex
helm lint charts/qbittorrent
helm lint charts/homeassistant

# 🔒 Kube-linter (sécurité)
kube-linter lint charts/
```

## ⚠️ Contraintes critiques

```mermaid
flowchart LR
    subgraph "🚫 INTERDIT"
        A[❌ Seeding qBittorrent]
        B[❌ Exposer Cloudflared]
        C[❌ Ajouter *arr services]
    end

    subgraph "✅ REQUIS"
        D[✔️ Plex privileged: true]
        E[✔️ selfHeal: true]
        F[✔️ hostPath volumes]
    end
```

| ⚠️ Règle | Description |
|---------|-------------|
| 🚫 **NE PAS** | Activer le seeding dans qBittorrent |
| 🚫 **NE PAS** | Exposer Cloudflared externellement (ClusterIP only) |
| 🚫 **NE PAS** | Ajouter les services *arr (Radarr, Sonarr, etc.) - intentionnellement exclus |
| ✅ **REQUIS** | Plex `privileged: true` pour transcodage HW via `/dev/dri` |
| ⚠️ **ATTENTION** | Toutes les apps ont `selfHeal: true` - les changements kubectl manuels seront annulés |

## 📂 Chemins des volumes

```mermaid
graph LR
    subgraph "💾 /home/muchini/media-data/"
        Config["📁 config/"]
        Torrents["📁 torrents/"]

        subgraph "Config Services"
            CFG1["cloudflared/"]
            CFG2["plex/"]
            CFG3["qbittorrent/"]
            CFG4["homeassistant/"]
        end
    end

    Config --> CFG1
    Config --> CFG2
    Config --> CFG3
    Config --> CFG4
```

| Type | Chemin |
|------|--------|
| 📁 Config | `/home/muchini/media-data/config/{service}/` |
| 🎬 Media | `/media/` |
| ⬇️ Torrents | `/home/muchini/media-data/torrents/` |

## 🔄 Workflow GitOps

```mermaid
sequenceDiagram
    participant Dev as 👨‍💻 Développeur
    participant GH as ☁️ GitHub
    participant Argo as 🔄 ArgoCD
    participant K8s as ☸️ K3s

    Dev->>GH: 📤 git push
    GH-->>Argo: 🔔 Webhook/Poll
    Argo->>GH: 📥 Fetch changes
    Argo->>Argo: 🔍 Compare desired vs actual
    Argo->>K8s: ⚡ Apply manifests
    K8s-->>Argo: ✅ Sync complete
    Note over Argo,K8s: selfHeal: true<br/>Auto-revert manual changes
```
