# 🎬 Media Stack K8s

Stack média déployée sur K3s avec ArgoCD (GitOps) sur Raspberry Pi 5.

## 📋 Vue d'ensemble

```mermaid
graph TB
    subgraph "🌐 Internet"
        DNS[DNS Queries]
        Users[👤 Utilisateurs]
    end

    subgraph "🖥️ Raspberry Pi 5"
        subgraph "☸️ K3s Cluster"
            ArgoCD[🔄 ArgoCD<br/>GitOps Controller]

            subgraph "📦 Namespace: media-stack"
                CF[🛡️ Cloudflared<br/>DNS-over-HTTPS<br/>:5053]
                Plex[🎥 Plex<br/>Media Server<br/>:32400]
                QB[⬇️ qBittorrent<br/>Torrent Client<br/>:8080]
            end
        end

        Storage[(💾 /home/muchini/media-data)]
    end

    DNS --> CF
    Users --> Plex
    Users --> QB
    ArgoCD --> CF
    ArgoCD --> Plex
    ArgoCD --> QB
    Plex --> Storage
    QB --> Storage
```

## 🚀 Services

| Service | Description | Port | Namespace | Statut |
|---------|-------------|------|-----------|--------|
| 🛡️ Cloudflared | DNS over HTTPS (anti-censure) | ClusterIP 5053 | media-stack | ✅ |
| 🎥 Plex | Media Server avec transcodage HW | 32400 (hostNetwork) | media-stack | ✅ |
| ⬇️ qBittorrent | Client torrent (anti-seeding) | 8080 (hostPort) | media-stack | ✅ |

## 🔧 Déploiement

```bash
# 📥 Appliquer le root app (App of Apps pattern)
kubectl apply -f apps/root-app.yaml

# 👀 Suivre le déploiement
kubectl get applications -n argocd -w
```

## 🌐 Accès

| Service | URL |
|---------|-----|
| 🔄 ArgoCD | https://192.168.1.51:30443 |
| 🎥 Plex | http://192.168.1.51:32400/web |
| ⬇️ qBittorrent | http://192.168.1.51:8080 |

## 📁 Structure du projet

```mermaid
graph LR
    subgraph "📂 Repository"
        ROOT[📄 root-app.yaml]

        subgraph "📁 apps/"
            A1[cloudflared.yaml]
            A2[plex.yaml]
            A3[qbittorrent.yaml]
        end

        subgraph "📁 charts/"
            C1[☁️ cloudflared/]
            C2[🎥 plex/]
            C3[⬇️ qbittorrent/]
        end

        subgraph "📁 base/"
            B1[namespace.yaml]
        end
    end

    ROOT --> A1
    ROOT --> A2
    ROOT --> A3
    A1 --> C1
    A2 --> C2
    A3 --> C3
```

```
📦 media-stack-k8s/
├── 📁 apps/               # ArgoCD Application manifests
│   ├── 📄 root-app.yaml   # App of Apps parent
│   ├── 📄 cloudflared.yaml
│   ├── 📄 plex.yaml
│   └── 📄 qbittorrent.yaml
├── 📁 base/               # Ressources de base
│   └── 📄 namespace.yaml
└── 📁 charts/             # Helm Charts
    ├── ☁️ cloudflared/
    ├── 🎥 plex/
    └── ⬇️ qbittorrent/
```

## ⚠️ Contraintes importantes

> 🚫 **NE PAS** activer le seeding dans qBittorrent
> 🚫 **NE PAS** exposer Cloudflared externellement
> 🚫 **NE PAS** ajouter les services *arr (Radarr, Sonarr, etc.)

## 📊 Flux de données

```mermaid
sequenceDiagram
    participant U as 👤 Utilisateur
    participant QB as ⬇️ qBittorrent
    participant CF as 🛡️ Cloudflared
    participant DNS as 🌐 Cloudflare DNS
    participant P as 🎥 Plex
    participant S as 💾 Storage

    U->>QB: Ajoute torrent
    QB->>CF: Résolution DNS
    CF->>DNS: DNS-over-HTTPS
    DNS-->>CF: IP résolue
    CF-->>QB: Réponse DNS
    QB->>S: Télécharge fichier
    S-->>P: Fichier disponible
    U->>P: Stream média
    P->>S: Lecture fichier
    P-->>U: 🎬 Diffusion
```
