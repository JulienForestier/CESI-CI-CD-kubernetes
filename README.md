# CESI-CI-CD-kubernetes

Dépôt GitOps (Kustomize + ArgoCD) pour le déploiement de l'application Collector.shop (CESI-CI-CD) sur un cluster k3s.

## Architecture

```
base/                       # source de vérité réelle (Deployment, Service, NetworkPolicy)
overlays/
  dev/                      # namespace cesi-ci-cd-dev, 1 replica, ENV=dev
  rec/                      # namespace cesi-ci-cd-rec, 2 replicas, ENV=recette
  prod/                     # namespace cesi-ci-cd-prod, 3 replicas
cluster/
  cert-manager/             # ClusterIssuer Let's Encrypt (staging + prod)
argocd/                     # 3 Application ArgoCD (dev/rec/prod), toutes sur la branche main
```

Chaque overlay référence `../../base` et n'applique que des **patches** (replicas, variable `ENV`) via Kustomize — pas de duplication de manifests. `service.yaml` et la `NetworkPolicy` sont mutualisés dans `base/` (identiques partout). Les `Ingress` et les secrets scellés restent propres à chaque overlay (spécifiques par environnement).

Une seule branche (`main`) sert de source de vérité pour les 3 environnements — la promotion se fait via le chemin (`overlays/dev|rec|prod`), pas via des branches séparées (cf. [bonnes pratiques ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/), qui déconseillent le "branch-per-environment").

## Flux GitOps

1. **Repo applicatif** (`CESI-CI-CD`) : la CI build l'image Docker et la pousse sur GHCR.
2. `deploy-front.yml` (repo applicatif) checkout toujours la branche `main` de ce repo, et exécute `kustomize edit set image` sur l'overlay ciblé (dev/rec/prod selon la branche source de l'app), puis commit + push.
3. **ArgoCD** surveille `main` (auto-sync + self-heal + prune) et applique les changements sur le cluster.

## Bootstrap du cluster (à rejouer si le cluster est reconstruit)

Ces composants sont installés **de façon impérative** via Helm (pratique standard pour les opérateurs d'infra, en dehors du flux GitOps applicatif).

### 1. k3s
```bash
curl -sfL https://get.k3s.io | sh -
```
Récupérer le kubeconfig (`/etc/rancher/k3s/k3s.yaml`), remplacer `127.0.0.1` par l'IP publique du serveur pour un usage à distance.

### 2. ArgoCD
Installé au préalable dans le namespace `argocd`. Les 3 manifests `Application` de ce repo (`argocd/*.yaml`) doivent être appliqués manuellement après (pas d'app-of-apps) :
```bash
kubectl apply -f argocd/application-dev.yaml -f argocd/application-rec.yaml -f argocd/application.yaml
```

### 3. Bitnami Sealed Secrets
```bash
helm repo add sealed-secrets https://bitnami.github.io/sealed-secrets
helm install sealed-secrets-controller sealed-secrets/sealed-secrets --namespace kube-system
```
Chaque secret GHCR (`ghcr-pull-secret`, un par namespace dev/rec/prod, scope strict namespace-bound) est scellé avec `kubeseal` puis committé (voir `overlays/*/ghcr-sealed-secret.yaml`) :
```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io --docker-username=<user> --docker-password=<token read:packages> \
  --docker-email=<email> -n <namespace> --dry-run=client -o yaml | \
kubeseal --format yaml --controller-name=sealed-secrets-controller --controller-namespace=kube-system \
  > overlays/<env>/ghcr-sealed-secret.yaml
```

Le BFF (`CESI-CI-CD.ApiService`) et l'IdentityServer (`CESI-CI-CD.IdentityService`) ont besoin de deux secrets scellés supplémentaires, un par namespace dev/rec/prod — **pas encore créés** (nécessitent un accès au cluster réel pour `kubeseal`), donc `identity-deployment.yaml`/`api-deployment.yaml` échoueront à démarrer tant qu'ils ne le sont pas :

```bash
# 1. Secret du client OAuth confidentiel "collector-shop-bff" (partagé entre apiservice et
#    identityservice — même valeur des deux côtés, sinon l'échange de code d'autorisation échoue)
BFF_SECRET=$(openssl rand -base64 32)
kubectl create secret generic collectorshop-bff \
  --from-literal=Bff__ClientSecret="$BFF_SECRET" \
  -n <namespace> --dry-run=client -o yaml | \
kubeseal --format yaml --controller-name=sealed-secrets-controller --controller-namespace=kube-system \
  > overlays/<env>/bff-sealed-secret.yaml

# 2. Certificat de signature RSA de l'IdentityServer (voir LoadOrCreateSigningCertificate dans
#    CESI-CI-CD.IdentityService/Program.cs) — sans lui, un certificat éphémère est régénéré à
#    chaque redémarrage du pod, invalidant tous les tokens/sessions en cours.
CERT_PASSWORD=$(openssl rand -base64 24)
openssl req -x509 -newkey rsa:2048 -keyout /tmp/identity-signing.key -out /tmp/identity-signing.crt \
  -days 730 -nodes -subj "/CN=collector-shop-identity"
openssl pkcs12 -export -out /tmp/identity-signing.pfx \
  -inkey /tmp/identity-signing.key -in /tmp/identity-signing.crt -password "pass:$CERT_PASSWORD"
kubectl create secret generic collectorshop-identity-signing \
  --from-literal=IdentityServer__SigningCertificate="$(base64 -i /tmp/identity-signing.pfx | tr -d '\n')" \
  --from-literal=IdentityServer__SigningCertificatePassword="$CERT_PASSWORD" \
  -n <namespace> --dry-run=client -o yaml | \
kubeseal --format yaml --controller-name=sealed-secrets-controller --controller-namespace=kube-system \
  > overlays/<env>/identity-signing-sealed-secret.yaml
rm /tmp/identity-signing.key /tmp/identity-signing.crt /tmp/identity-signing.pfx
```

Une fois ces deux fichiers créés pour un overlay, les ajouter à `resources:` dans son `kustomization.yaml`, puis câbler `Bff__ClientSecret`/`IdentityService__Authority` dans `base/api-deployment.yaml` (délibérément pas encore fait — apiservice est actuellement en prod avec l'ancienne auth JWT, câbler ces secrets avant qu'ils existent casserait son déploiement).

### 4. cert-manager
```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
kubectl apply -k cluster/cert-manager
```
Les ingress référencent `cert-manager.io/cluster-issuer: letsencrypt-prod` — les certificats sont émis automatiquement (HTTP-01 via Traefik).

### 5. Observabilité (Prometheus + Grafana + Loki)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.resources.requests.cpu=200m --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.limits.cpu=500m --set prometheus.prometheusSpec.resources.limits.memory=512Mi \
  --set grafana.resources.requests.cpu=50m --set grafana.resources.requests.memory=64Mi \
  --set grafana.resources.limits.cpu=200m --set grafana.resources.limits.memory=256Mi

# Loki (chart Monolithic, stockage filesystem, caches désactivés - inutiles à ce volume)
helm install loki grafana-community/loki -n monitoring -f bootstrap/loki-values.yaml \
  --set chunksCache.enabled=false --set resultsCache.enabled=false

# Promtail (envoi des logs vers Loki)
helm install promtail grafana/promtail --namespace monitoring \
  --set "config.clients[0].url=http://loki-gateway/loki/api/v1/push" \
  --set resources.requests.cpu=25m --set resources.requests.memory=64Mi \
  --set resources.limits.cpu=100m --set resources.limits.memory=128Mi
```
Dans Grafana, ajouter le datasource Loki : type `Loki`, URL `http://loki-gateway`.

## Accès aux interfaces (port-forward local)

```bash
# ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

## Requêtes utiles dans Grafana (Explore → datasource Loki)

```logql
# Tous les logs d'un environnement
{namespace="cesi-ci-cd-dev"}

# Logs d'un pod précis
{namespace="cesi-ci-cd-dev", pod=~"myapp.*"}

# Uniquement les erreurs
{namespace="cesi-ci-cd-dev"} |= "error"

# Logs des composants d'infra (ArgoCD, cert-manager, sealed-secrets)
{namespace="argocd"}
{namespace="cert-manager"}
{namespace="kube-system", container="sealed-secrets-controller"}

# Débit de logs par pod sur les 5 dernières minutes
sum by (pod) (rate({namespace="cesi-ci-cd-dev"}[5m]))
```

## Sécurité

- Secrets GHCR chiffrés (Bitnami Sealed Secrets), scope strict par namespace.
- TLS Let's Encrypt sur les 3 environnements (dev/rec/prod).
- `NetworkPolicy` restreignant l'ingress vers les pods applicatifs au seul namespace `kube-system` (Traefik).
- `resources.requests/limits` + `readinessProbe`/`livenessProbe` sur tous les déploiements.
- Serveur : pare-feu actif (ufw), SSH par clé uniquement, fail2ban, mises à jour de sécurité automatiques.
