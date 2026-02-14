# 📁 Base - Ressources Kubernetes de base

Ce dossier contient les **ressources Kubernetes fondamentales** partagées par toutes les applications de la stack.

## 🎯 Objectif

```mermaid
graph LR
    subgraph "📂 base/"
        NS[🏷️ namespace.yaml]
        NSHA[🏷️ namespace-home-assistant.yaml]
    end

    subgraph "☸️ Cluster K3s"
        MediaStack[📦 Namespace<br/>media-stack]
        HAStack[🏠 Namespace<br/>home-assistant]
    end

    NS -->|"crée"| MediaStack
    NSHA -->|"crée"| HAStack
```

## 📄 Fichiers

| Fichier | Type | Description |
|---------|------|-------------|
| 🏷️ `namespace.yaml` | Namespace | Définit le namespace `media-stack` |
| 🏷️ `namespace-home-assistant.yaml` | Namespace | Définit le namespace `home-assistant` |

## 🏷️ Namespace media-stack

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media-stack
  labels:
    app.kubernetes.io/managed-by: argocd
```

### 📊 Services déployés dans ce namespace

```mermaid
graph TB
    subgraph "📦 Namespace: media-stack"
        CF[🛡️ Cloudflared<br/>DNS-over-HTTPS]
        PX[🎥 Plex<br/>Media Server]
        QB[⬇️ qBittorrent<br/>Torrent Client]
    end

    CF -.->|"DNS"| PX
    CF -.->|"DNS"| QB
```

## 🏠 Namespace home-assistant

Le namespace `home-assistant` est défini dans `namespace-home-assistant.yaml` et géré via GitOps. Il est séparé car Home Assistant nécessite une **isolation** pour:
- Sa propre configuration réseau (`hostNetwork`)
- Ses intégrations mDNS/SSDP
- Une gestion indépendante des mises à jour

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: home-assistant
  labels:
    app.kubernetes.io/managed-by: argocd
```

## 🔧 Utilisation

```bash
# 📥 Appliquer le namespace manuellement (si nécessaire)
kubectl apply -f base/namespace.yaml

# ✅ Vérifier l'existence
kubectl get namespace media-stack

# 📊 Voir les ressources du namespace
kubectl get all -n media-stack
```
