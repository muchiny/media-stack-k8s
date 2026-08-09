# ⬇️ qBittorrent - Client Torrent

Helm chart pour déployer **qBittorrent** avec configuration anti-seeding et DNS sécurisé.

## 🎯 Objectif

```mermaid
graph TB
    subgraph "☸️ Cluster K3s"
        QB[⬇️ qBittorrent<br/>Port 8080]
        Init[⏳ Init Container<br/>Attend DNS]
    end

    subgraph "💾 Stockage"
        Config[(📁 config/qbittorrent/)]
        Downloads[(⬇️ torrents/)]
        Media[(🎬 /media/)]
    end

    subgraph "🌐 Internet"
        Trackers[📡 Trackers]
    end

    Init -->|"nslookup"| DNS
    Init -->|"OK"| QB
    QB -->|"DNS"| DNS
    QB --> Trackers
    QB --> Config & Downloads & Media
```

## 📄 Fichiers

| Fichier | Description |
|---------|-------------|
| 📄 `Chart.yaml` | Métadonnées du chart (v1.0.0, appVersion 5.1.4) |
| ⚙️ `values.yaml` | Configuration par défaut |
| 📂 `templates/` | Templates Kubernetes |

### 📂 Templates

| Template | Ressource | Description |
|----------|-----------|-------------|
| 🔧 `_helpers.tpl` | - | Fonctions helper (labels, selectors) |
| 📋 `deployment.yaml` | Deployment | Pod avec init container et startupProbe |
| 🌐 `service.yaml` | Service | NodePort WebUI + Torrent |
| 🛡️ `pdb.yaml` | PodDisruptionBudget | Garantit disponibilité minimale |

## ⚙️ Configuration

```yaml
# values.yaml
image:
  repository: lscr.io/linuxserver/qbittorrent
  tag: "5.1.4"

service:
  webui:
    type: NodePort
    port: 8080
    nodePort: 30080
  torrent:
    type: NodePort
    port: 6881
    nodePort: 30881

persistence:
  config:
    hostPath: /home/muchini/media-data/config/qbittorrent
  downloads:
    hostPath: /home/muchini/media-data/torrents
  media:
    hostPath: /media

dns:

environment:
  PUID: "1000"
  PGID: "1000"
  TZ: "Europe/Paris"
  WEBUI_PORT: "8080"

nodeSelector:
  kubernetes.io/arch: arm64
```

## 🏗️ Architecture

```mermaid
flowchart TB
    subgraph "📦 Deployment"
        subgraph "⏳ Init Container"
            Init[busybox<br/>attend le DNS du cluster]
        end

        subgraph "🐳 Main Container"
            QB[qbittorrent:5.1.4]
        end

        subgraph "Volumes"
            V1[📁 /config]
            V2[⬇️ /downloads]
            V3[🎬 /media]
        end
    end

    subgraph "🌐 Services"
        WebUI[🖥️ NodePort 30080<br/>WebUI]
        Torrent[📡 NodePort 30881<br/>Torrent]
    end

    Init -->|"success"| QB
    QB --> V1 & V2 & V3
    WebUI & Torrent --> QB
```

## ⏳ Init Container - Attente DNS

```mermaid
sequenceDiagram
    participant Init as ⏳ Init Container
    participant QB as ⬇️ qBittorrent

    loop Toutes les 5 secondes
        Init->>DNS: nslookup google.com
        DNS-->>Init: ❌ Pas prêt
    end
    Init->>DNS: nslookup google.com
    DNS-->>Init: ✅ Résolu
    Init->>QB: 🚀 Démarrer
```

## 🚫 Configuration Anti-Seeding

```mermaid
mindmap
    root((🚫 Anti-Seeding))
        📊 Limites
            Upload: 1 KB/s
            Ratio: 0.01
            Seeding time: 0
        🔄 Comportement
            Pause après DL
            Pas de DHT
            Pas de PeX
```

**À configurer dans l'interface WebUI:**
- `Options > BitTorrent > Seeding Limits`
  - Max ratio: `0.01`
  - Max seeding time: `0 minutes`
  - Action: `Pause torrent`

## 🏥 Probes & Haute disponibilité

| Probe | Configuration |
|-------|---------------|
| **livenessProbe** | TCP 8080, delay 60s, period 30s |
| **readinessProbe** | TCP 8080, delay 30s, period 10s |
| **startupProbe** | TCP 8080, period 10s, 30 tentatives max (5 min) |
| **PDB** | minAvailable: 1 |
| **preStop** | sleep 10s (graceful shutdown) |

## ⚠️ Points critiques

| ⚠️ | Description |
|----|-------------|
| 📤 | **Partage activé** (2026-08-09) - `hostPort: 6881` TCP+UDP. Sans lui, qBittorrent annonçait 6881 aux trackers alors que seul le NodePort 30881 était ouvert : les pairs ne pouvaient jamais l'atteindre. |
| 🌐 | **Redirection box requise** - 6881 TCP+UDP vers `192.168.1.51`, sinon le partage reste inopérant |
| ⚖️ | **Ratio** - `ShareLimitAction=Stop` est défini mais **aucune limite de ratio ne l'est**, donc il ne se déclenche jamais. À régler dans la WebUI. |
| 🔓 | **Pas de VPN** - choix assumé : l'IP publique du domicile est visible dans les swarms |
| ⏳ | **Init container** - Attend que le DNS du cluster réponde |
| 📡 | **NodePort** - Accessible sur `30080` (WebUI) et `30881` (torrent) |
| 🖥️ | **arm64** - NodeSelector force le déploiement sur Raspberry Pi |

## 🔧 Commandes

```bash
# ✅ Valider le chart
helm lint charts/qbittorrent
helm template charts/qbittorrent

# 🔄 Forcer la sync ArgoCD
argocd app sync qbittorrent

# 📊 Vérifier le pod
kubectl get pods -n media-stack -l app=qbittorrent

# 📋 Voir les logs
kubectl logs -n media-stack -l app=qbittorrent -f

# ⏳ Voir les logs de l'init container
kubectl logs -n media-stack -l app=qbittorrent -c wait-for-dns

# 🌐 Accéder à l'interface
# http://192.168.1.51:8080 (via hostPort)
# Login par défaut: admin / adminadmin
```
