# 🤖 CLAUDE.md

Ce fichier fournit des instructions à Claude Code (claude.ai/code) pour travailler avec ce repository.

## 📋 Vue d'ensemble du projet

Stack média K3s déployée via ArgoCD GitOps sur **Raspberry Pi 5 (arm64)**. Utilise le pattern **App of Apps** où `apps/root-app.yaml` est l'application parente qui synchronise toutes les applications enfants.

## 📂 Structure du projet

```
media-stack-k8s/
├── 📂 apps/                    # Applications ArgoCD
│   ├── root-app.yaml           # App parente (point d'entrée)
│   ├── namespace.yaml          # Namespace media-stack
│   ├── plex.yaml               # App Plex
│   ├── qbittorrent.yaml        # App qBittorrent
│   ├── sonarqube.yaml          # App SonarQube (namespace dedie)
│   ├── priority-classes.yaml   # Classes de priorité
│   └── resource-quota.yaml     # Quotas de ressources
├── 📂 base/                    # Ressources K8s de base
│   └── namespace.yaml          # Namespace media-stack
├── 📂 charts/                  # Helm Charts
│   ├── plex/
│   ├── qbittorrent/
│   └── sonarqube/
├── 📂 .github/workflows/       # CI/CD GitHub Actions
│   └── validate.yaml           # Pipeline de validation
├── .yamllint.yaml              # Config yamllint
├── .kube-linter.yaml           # Config kube-linter
└── CLAUDE.md                   # Ce fichier
```

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "☁️ GitHub"
        Repo[(📦 media-stack-k8s<br/>Repository)]
    end

    subgraph "🖥️ Raspberry Pi 5 - K3s"
        ArgoCD[🔄 ArgoCD<br/>Port: 30443]

        subgraph "📦 Namespace: media-stack"
            Plex[🎥 Plex<br/>hostNetwork<br/>Port: 32400]
            QB[⬇️ qBittorrent<br/>hostPort: 8080]
        end

        CoreDNS[🌐 CoreDNS]
        Storage[(💾 /home/muchini/media-data)]
    end

    Repo -->|"GitOps Sync"| ArgoCD
    ArgoCD -->|"Déploie"| Plex
    ArgoCD -->|"Déploie"| QB
    Plex --> Storage
    QB --> Storage
```

## 🎯 Décisions de conception clés

```mermaid
mindmap
  root((🏗️ Architecture))
    🎥 Plex
      hostNetwork: true
      Découverte DLNA/GDM
      privileged pour /dev/dri
    ⬇️ qBittorrent
      Init container
      Attend le DNS du cluster
      Anti-seeding
    💾 Storage
      hostPath volumes
      /home/muchini/media-data/
```

| Composant | Configuration | Raison |
|-----------|--------------|--------|
| 🎥 Plex | `hostNetwork: true` | Découverte DLNA/GDM |
| ⬇️ qBittorrent | Init container | Attend le DNS du cluster |
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

# 🌐 UI ArgoCD
# https://192.168.1.51:30443

# 🔄 Forcer la sync d'une app spécifique
argocd app sync plex
argocd app sync qbittorrent
```

### 🧪 Test des Helm Charts (local)

```bash
# ✅ Valider les templates
helm template charts/plex
helm template charts/qbittorrent

# 🔍 Linter les charts
helm lint charts/plex
helm lint charts/qbittorrent

# 📝 YAML Lint
yamllint -c .yamllint.yaml .

# 🔒 Kube-linter (sécurité)
kube-linter lint charts/

# ✅ Kubeconform (validation schémas K8s)
helm template charts/plex | kubeconform -strict -summary
```

### 🔄 CI/CD GitHub Actions

Le pipeline `.github/workflows/validate.yaml` s'exécute sur chaque push/PR et inclut:

```mermaid
graph LR
    subgraph "🔍 Lint"
        YL[📝 YAML Lint]
        HL[📦 Helm Lint]
        KL[🔒 Kube-linter]
    end

    subgraph "✅ Validate"
        KC[📋 Kubeconform<br/>Schémas K8s]
    end

    subgraph "🛡️ Security"
        TR[🔍 Trivy IaC]
        KS[🛡️ Kubescape<br/>NSA/MITRE]
        CH[✅ Checkov]
    end
```

| Job | Outils | Description |
|-----|--------|-------------|
| 🔍 Lint | yamllint, helm lint, kube-linter | Validation syntaxe et bonnes pratiques |
| ✅ Validate | kubeconform | Validation schémas Kubernetes |
| 🛡️ Security | Trivy, Kubescape, Checkov | Scan sécurité IaC |

## 📊 Gouvernance des ressources

### 🎯 Priority Classes

Définies dans `apps/priority-classes.yaml` pour gérer l'éviction des pods:

| Classe | Valeur | Services |
|--------|--------|----------|
| 🟠 `media-high` | 900,000 | Plex, qBittorrent |
| 🟢 `media-normal` | 800,000 | (Réservé) |

### 📏 Resource Quotas

Définis dans `apps/resource-quota.yaml` pour le namespace `media-stack`:

| Ressource | Requests | Limits |
|-----------|----------|--------|
| CPU | 4 cores | 6 cores |
| Memory | 6 Gi | 8 Gi |

### 🛡️ PodDisruptionBudgets

Chaque chart inclut un PDB (`templates/pdb.yaml`) avec `minAvailable: 1` pour garantir la disponibilité pendant les maintenances.

## ⚠️ Contraintes critiques

```mermaid
flowchart LR
    subgraph "🚫 INTERDIT"
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
| 🚫 **NE PAS** | Ajouter les services *arr (Radarr, Sonarr, etc.) - intentionnellement exclus |
| ⚠️ **DÉCISION** | Le partage qBittorrent est **activé** depuis le 2026-08-09 (`hostPort: 6881`). Le choix précédent de le bloquer a été levé volontairement. Nécessite une redirection 6881 TCP+UDP sur la box pour être effectif. |
| ⚠️ **ATTENTION** | Aucun VPN sur qBittorrent — choix assumé. L'IP publique du domicile est donc visible dans les swarms. |
| ✅ **REQUIS** | Plex `privileged: true` pour transcodage HW via `/dev/dri` |
| 📦 **HORS MEDIA** | SonarQube tourne dans son propre namespace `sonarqube`, hors ResourceQuota et sans PriorityClass : il est evince avant Plex/qBittorrent en cas de pression memoire. Secret PostgreSQL `sonarqube-db` cree hors Git. |
| ⚠️ **ATTENTION** | Toutes les apps ont `selfHeal: true` - les changements kubectl manuels seront annulés |

## 📂 Chemins des volumes

```mermaid
graph LR
    subgraph "💾 /home/muchini/media-data/"
        Config["📁 config/"]
        Torrents["📁 torrents/"]

        subgraph "Config Services"
            CFG2["plex/"]
            CFG3["qbittorrent/"]
        end
    end

    Config --> CFG2
    Config --> CFG3
```

| Type | Chemin |
|------|--------|
| 📁 Config | `/home/muchini/media-data/config/{service}/` (plex, qbittorrent) |
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
