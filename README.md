# CESI-CI-CD-kubernetes

Dépôt manifests Kubernetes pour le déploiement de l'application CESI-CI-CD sur k3s avec Argo CD.

## Architecture

- **base/** : Ressources de base communes (Deployment, Service, Kustomization)
- **overlays/dev/** : Surcharge pour environnement DEV (1 replica, namespace DEV)
- **overlays/rec/** : Surcharge pour environnement REC (2 replicas, namespace REC)
- **overlays/prod/** : Surcharge pour environnement PROD (3 replicas, namespace PROD)
- **argocd/** : Manifests Argo CD Application (une par environnement)

## Flux CI/CD

1. **Repo applicatif** : Code de l'application (ex: `CESI-CI-CD-app`)
2. **GitHub Actions** (dans repo applicatif) :
    - Build l'image Docker et la pousse dans GHCR (ou autre registry)
    - Clône ce repo manifests
    - Met à jour `overlays/prod/deployment.yaml` (image tag)
    - Commit + push les changements dans ce repo
3. **Argo CD** (dans k3s) :
    - Surveille ce repo sur la branche `main`
    - Détecte les changements de commit
    - Applique automatiquement les manifests du cluster
4. **Cluster k3s** : Les pods sont recrées avec la nouvelle image

## Installation sur k3s

### 1. Installer k3s sur le VPS
```bash
curl -sfL https://get.k3s.io | sh -