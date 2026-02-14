# Contributing

Merci de votre intérêt pour contribuer à ce projet !

## Prérequis

- Git
- Helm 3.x
- kubectl
- pre-commit (optionnel mais recommandé)

## Configuration de l'environnement

```bash
# Cloner le repository
git clone https://github.com/YOUR_USERNAME/media-stack-k8s.git
cd media-stack-k8s

# Installer les pre-commit hooks
pip install pre-commit
pre-commit install
```

## Workflow de contribution

1. **Fork** le repository
2. **Créer une branche** pour votre feature ou fix
   ```bash
   git checkout -b feature/ma-nouvelle-feature
   ```
3. **Faites vos modifications**
4. **Validez localement**
   ```bash
   # Linting YAML
   yamllint -c .yamllint.yaml .

   # Linting Helm
   helm lint charts/*

   # Validation schémas K8s
   for chart in charts/*/; do
     helm template "$chart" | kubeconform -strict -summary
   done
   ```
5. **Committez** avec un message conventionnel
   ```bash
   git commit -m "feat: add new feature"
   git commit -m "fix: resolve issue with X"
   git commit -m "docs: update README"
   ```
6. **Push** et créez une **Pull Request**

## Conventions de commit

Ce projet utilise [Conventional Commits](https://www.conventionalcommits.org/):

| Type | Description |
|------|-------------|
| `feat` | Nouvelle fonctionnalité |
| `fix` | Correction de bug |
| `docs` | Documentation uniquement |
| `chore` | Maintenance (deps, CI, etc.) |
| `refactor` | Refactoring sans changement de comportement |
| `test` | Ajout ou modification de tests |

## Structure des Helm Charts

Chaque chart doit inclure :

```
charts/mon-service/
├── Chart.yaml          # Métadonnées du chart
├── values.yaml         # Valeurs par défaut
├── .helmignore         # Fichiers à ignorer
├── README.md           # Documentation
└── templates/
    ├── _helpers.tpl    # Fonctions helper
    ├── deployment.yaml
    ├── service.yaml
    ├── pdb.yaml        # PodDisruptionBudget
    ├── networkpolicy.yaml
    └── NOTES.txt       # Message post-install
```

## Règles importantes

- **NE PAS** activer le seeding dans qBittorrent
- **NE PAS** exposer dnscrypt-proxy externellement
- **NE PAS** ajouter les services *arr (Radarr, Sonarr, etc.)
- **TOUJOURS** inclure les ressources (requests/limits)
- **TOUJOURS** inclure les probes (liveness/readiness/startup)

## Questions ?

Ouvrez une issue pour toute question ou suggestion.
