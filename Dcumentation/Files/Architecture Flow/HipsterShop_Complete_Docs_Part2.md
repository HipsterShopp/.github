---
# HipsterShop Platform Documentation — Part 2

---

## 7. Helm-Based Deployment Strategy

### 7.1 Why Helm

Before Helm, each service needed manually written Kubernetes YAMLs. With 12 services each needing a Deployment + Service (minimum), that is 24+ files — all with repeated boilerplate. Values like namespace names and port numbers were copy-pasted, creating configuration drift. Helm eliminates this by centralizing all configuration and using template injection.

**Why Helm in this project specifically:**
- 12 services × 3 environments (base/dev/prod) = managed with just 3 values files per service
- ArgoCD uses Helm's template engine — `helm template` output is applied via kubectl
- Versioned releases with rollback capability
- One `sed` command in CI can update the image tag in values.yaml → ArgoCD picks it up

### 7.2 Repository Structure

```
hipstershop-helm-charts/
│
├── hipstershop-argoapps/              # ArgoCD Application/ApplicationSet manifests
│   ├── project.yaml                   # ArgoCD AppProject: hipstershop
│   └── apps/
│   │   ├── micro-apps-appset.yaml     # Main: 11 services, values.yaml
│   │   ├── micro-apps-appset-dev.yaml # Dev: 12 services, values.yaml + values.dev.yaml
│   │   └── micro-apps-appset-prod.yaml# Prod: 11 services, values.yaml + values.prod.yaml
│   └── infra/
│       ├── mongodb.yaml               # Wave 0
│       ├── backend-common.yaml        # Wave 0
│       ├── sealed-secrets.yaml        # Wave 1
│       ├── gateway.yaml               # Wave 2
│       └── loadgenerator.yaml         # Wave 3
│
├── hipstershop-infra/                 # Infrastructure Helm charts
│   ├── backend-common/                # Namespace, ConfigMaps, Secrets, NetworkPolicy
│   ├── gateway/                       # kGateway GatewayClass + Gateway + HTTPRoutes
│   ├── mongodb/                       # MongoDB StatefulSet, Services, Init Job
│   └── rbac/                          # GitHub OIDC RBAC for shippingservice
│
├── hipstershop-micro-apps/            # One Helm chart per microservice
│   ├── adservice/
│   │   ├── Chart.yaml
│   │   ├── values.yaml            # Base (main branch defaults)
│   │   ├── values.dev.yaml        # Dev overrides (lighter resources, develop image)
│   │   ├── values.prod.yaml       # Prod overrides (GHCR image, full resources)
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── [authservice, cartservice, checkoutservice, currencyservice,
│        emailservice, frontend, paymentservice, productcatalogservice,
│        recommendationservice, shippingservice, assistantservice]
│        └── (same structure)
│
└── sealed-secrets/
    ├── backend-common-secrets.yaml    # Encrypted app-secrets (safe to git commit)
    ├── mongodb-secrets.yaml           # Encrypted MongoDB credentials
    ├── generate-sealed-secrets.sh     # Generator script
    └── steps.txt                      # Operator runbook
```

### 7.3 Three-Layer Values Override System

| File | Used In | Purpose |
|------|---------|---------|
| `values.yaml` | Base/Fallback | Default config, base image tags |
| `values.dev.yaml` | Dev ApplicationSet | Lighter resources (50%), develop image tag |
| `values.prod.yaml` | Prod ApplicationSet | Full resources, GHCR image tags |

ArgoCD renders via:
```bash
helm template {service} hipstershop-micro-apps/{service} \
  -f hipstershop-micro-apps/{service}/values.yaml \
  -f hipstershop-micro-apps/{service}/values.dev.yaml    # or values.prod.yaml
```

### 7.4 Values Comparison (adservice)

```yaml
# values.yaml (base)
global:
  images:
    adservice: "yampss/hipstershop-adservice:adservice-v0.1.2"
  replicas:
    adservice: 1
  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi
  ports:
    adservice: 9555

# values.dev.yaml (dev overrides — lighter resources)
global:
  images:
    adservice: "yampss/hipstershop-adservice:develop-latest"  # Updated by CI
  resources:
    adservice:
      requests:
        cpu: 100m    # 50% of prod
        memory: 90Mi # 50% of prod
      limits:
        cpu: 200m
        memory: 256Mi

# values.prod.yaml (prod overrides — GHCR + full resources)
global:
  images:
    adservice: "ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.2.3"
  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi
```

### 7.5 backend-common ConfigMaps

5 ConfigMaps in `hipster-backend` (created by Wave 0):

| ConfigMap | Keys | Purpose |
|-----------|------|---------|
| `hipster-config` | ENABLE_TRACING, COLLECTOR_SERVICE_ADDR, FRONTEND_MESSAGE, ENABLE_ASSISTANT | App-wide feature flags |
| `service-addresses` | PRODUCT_CATALOG_SERVICE_ADDR, SHIPPING_SERVICE_ADDR, PAYMENT_SERVICE_ADDR, EMAIL_SERVICE_ADDR, CURRENCY_SERVICE_ADDR, CART_SERVICE_ADDR, GATEWAY_ADDR | Inter-service DNS addresses |
| `service-ports` | ADSERVICE_PORT=9555, AUTHSERVICE_PORT=8081, CARTSERVICE_PORT=7070, CHECKOUTSERVICE_PORT=5050, CURRENCYSERVICE_PORT=7000, EMAILSERVICE_PORT=8080, PAYMENTSERVICE_PORT=50051, PRODUCTCATALOG_PORT=3550, SHIPPINGSERVICE_PORT=50051 | All service ports |
| `mongodb-config` | MONGO_REPLICA_SET=rs0, AUTH_DB_NAME, CART_DB_NAME, CATALOG_DB_NAME, PAYMENT_DB_NAME, PAYMENT_CHARGES_COLLECTION | MongoDB database/collection names |
| `app-features` | CURRENCY_SERVICE_RANDOM_ERROR=1, PAYMENT_SERVICE_DISABLE_PROFILER=1, PRODUCT_CATALOG_SEED_DATABASE=true, GEMINI_MODEL=gemini-2.5-flash | Feature flags |

### 7.6 Helm Commands Reference

```bash
# NOTE: helm list returns empty — ArgoCD uses helm template + kubectl apply, not helm install
helm list -n hipster-backend   # Returns empty — EXPECTED

# Render templates locally (debugging)
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Render with dev overrides
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  -f hipstershop-micro-apps/adservice/values.dev.yaml

# Lint chart
helm lint hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Dry run
helm install adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  --dry-run
```

---

## 8. CI/CD Pipeline

### 8.1 Design Philosophy

**Goal:** Automate the full loop — code push → quality gate → build → scan → push → deploy.

Two environments:
- `develop` — continuous deployment, auto-synced by ArgoCD (no human gate)
- `main` — **manual approval gate** before production deployment

**Why shared workflows (not per-repo pipelines):**
Before `hipstershop-shared-workflows`, every service repo duplicated 200+ lines of CI YAML. Updating the Trivy action version required 12 separate PRs. With shared workflows, updating a tool version = **one PR** to `hipstershop-shared-workflows`, all 12 services pick it up automatically.

### 8.2 Shared Workflow Catalog

| Workflow | Purpose | Triggered On |
|----------|---------|-------------|
| `sonar-and-snyk.yaml` | SonarQube Quality Gate + Snyk HIGH+ dep scan | PR to main/develop |
| `generate-tag-and-build-image-trivy.yaml` | Semver/branch tag, docker build, Trivy CRITICAL scan | Push + PR |
| `push.yaml` | Push to DockerHub (:versioned + :latest), create git tag | Push to main/develop |
| `helm-updater-job.yaml` | Checkout helm-charts repo, `sed` tag in values file, commit+push | After push |
| `notify.yaml` | SendGrid failure email with job status dashboard | Always (on failure) |

All callers reference `@feature/loose-coupling` branch:
```yaml
uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
```

### 8.3 Full Pipeline Flow

```
push to develop or main (src/** changed)
            │
   ┌────────▼────────────────────────────────────────────────────────┐
   │ [PR only] sonar-and-snyk                                         │
   │           SonarQube Quality Gate                                 │
   │           Snyk: snyk test --severity-threshold=high              │
   │           Upload Snyk HTML report (14-day retention)             │
   └────────┬────────────────────────────────────────────────────────┘
            │ (skip on push; always run on PR success)
   ┌────────▼────────────────────────────────────────────────────────┐
   │ generate-tag-and-build-image-trivy                               │
   │   main:    mathieudutour/github-tag-action → adservice-v0.1.3    │
   │   develop: develop-{sha8} (e.g., develop-3f2a1c9b)              │
   │   Check if tag already exists on DockerHub                       │
   │   docker build -t {user}/hipstershop-{service}:{tag} src/        │
   │   docker save → /tmp/image.tar (artifact)                        │
   │   [PR] Trivy: severity=CRITICAL, ignore-unfixed=true            │
   └────────┬────────────────────────────────────────────────────────┘
            │ (push event only)
   ┌────────▼────────────────────────────────────────────────────────┐
   │ push                                                             │
   │   docker load -i image.tar                                       │
   │   docker login DockerHub                                         │
   │   docker push :versioned-tag                                     │
   │   docker push :latest (main only)                                │
   │   git tag {tag} && git push (main only)                          │
   └────────┬───────────────────────┬────────────────────────────────┘
            │ develop               │ main
            ▼                       ▼
   update-helm-values-dev    push-to-ghcr
   checkout helm-charts:develop    ghcr.io/hipstershopp/{service}:{tag}
   sed tag in values.dev.yaml      │
   git commit + push               prod-approval-gate
            │                      GitHub Environment: production ✋
            │                      │
            │                      update-helm-values-prod
            │                      checkout helm-charts:main
            │                      sed tag in values.prod.yaml
            │                      git commit + push
            │                      │
            │                      [shippingservice only]
            │                      deploy-to-prod (self-hosted runner)
            │                      GitHub OIDC → kubectl set image
            │
   ArgoCD detects diff (~3 min)   ArgoCD detects diff (~3 min)
   micro-apps-appset-dev           micro-apps-appset-prod
   auto-sync → rolling update      auto-sync → rolling update / blue-green

   [Always] notify → SendGrid email on any failure
```

### 8.4 Tag Strategy

```bash
# develop branch → branch-slug + short SHA
BRANCH_SLUG=$(echo "$GITHUB_REF_NAME" | sed 's|/|-|g')
SHORT_SHA=$(echo "$GITHUB_SHA" | cut -c1-8)
TAG="${BRANCH_SLUG}-${SHORT_SHA}"
# Example: develop-3f2a1c9b

# main branch → semantic version with service prefix
# Uses mathieudutour/github-tag-action@v6.2
# tag_prefix: "{service_name}-v"
# Example: adservice-v0.1.3
```

### 8.5 Docker Build Strategy

All services use **multi-stage builds** to minimize runtime image size:

| Service | Builder Image | Runtime Image | Why Runtime Choice |
|---------|-------------|-------------|-------------------|
| adservice | eclipse-temurin:24-jdk | eclipse-temurin:25-jre-alpine | JRE only (no compiler) at runtime |
| authservice | golang:1.26-alpine | gcr.io/distroless/static-debian12 | No OS = zero CVE surface |
| cartservice | dotnet/sdk:8.0 | dotnet/aspnet:8.0 | ASP.NET runtime only |
| checkoutservice | golang:1.26-alpine | gcr.io/distroless/static | Distroless Go |
| currencyservice | node:20-alpine | alpine:3.23 + nodejs | Minimal alpine + node |
| emailservice | python:3.11-alpine | python:3.11-alpine | Python Alpine |
| frontend | golang:1.26-alpine | gcr.io/distroless/static | Distroless + templates/static dirs |
| paymentservice | node:20-alpine | alpine:3.23 + nodejs | Same as currencyservice |
| productcatalogservice | golang:1.26-alpine | gcr.io/distroless/static | Distroless Go |
| recommendationservice | python:3.11-alpine | python:3.11-alpine | Python Alpine |
| shippingservice | golang:1.26-alpine | gcr.io/distroless/static | Distroless Go |
| assistantservice | — | python:3.10-slim | Single stage (no native compilation) |

**Key build flags:**
- `CGO_ENABLED=0` — All Go services: produces static binary, required for distroless
- `-ldflags="-s -w"` — Strips debug symbols, reduces binary size
- `--only=production` — Node.js: exclude devDependencies from runtime image
- `sed -i 's/\r$//' gradlew` — adservice: fix Windows CRLF line endings on Linux CI runners

### 8.6 Required Secrets per Service Repo

| Secret | Purpose | Set At |
|--------|---------|--------|
| `DOCKERHUB_USERNAME` | Docker Hub login | Repo secrets |
| `DOCKERHUB_TOKEN` | Docker Hub push | Repo secrets |
| `GHCR_TOKEN` | GHCR push (prod images) | Repo secrets |
| `HELM_CHARTS_TOKEN` | Push to helm-charts repo | Repo secrets |
| `SNYK_TOKEN` | Snyk scan authentication | Repo secrets |
| `SONAR_TOKEN` | SonarQube authentication | Repo secrets |
| `SONAR_HOST_URL` | SonarQube server URL | Repo secrets |
| `SENDGRID_API_KEY` | Failure email notifications | Repo secrets |
| `K8S_API_SERVER` | **shippingservice only** — cluster API URL | Repo secrets |
| `K8S_CA_CERT` | **shippingservice only** — cluster CA (base64) | Repo secrets |

### 8.7 Thin Caller Pattern (adservice example)

```yaml
# hipstershop-adservice/.github/workflows/adservice.yaml
name: adservice CI

on:
  push:
    branches: [main, develop]
    paths: ['src/**', '.github/workflows/adservice.yaml']
  pull_request:
    branches: [main, develop]
    paths: ['src/**', '.github/workflows/adservice.yaml']

permissions:
  contents: write
  packages: write

jobs:
  sonar-and-snyk:
    if: github.event_name == 'pull_request'
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
    with:
      service_name: adservice
      service_path: src/
      runtime: java
      java-version: "21"
      java-build-tool: "gradle"
    secrets:
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  generate-tag-and-build-image:
    needs: [sonar-and-snyk]
    if: |
      always() &&
      (
        (github.event_name == 'pull_request' && needs.sonar-and-snyk.result == 'success') ||
        github.event_name == 'push'
      )
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/generate-tag-and-build-image-trivy.yaml@feature/loose-coupling
    with:
      service_name: adservice
      service_path: src/
      run_trivy: ${{ github.event_name == 'pull_request' }}
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}

  # ... push, push-to-ghcr, prod-approval-gate, update-helm-values-prod, notify
```

### 8.8 helm-updater-job — The GitOps Trigger

This is the critical link between CI and ArgoCD:

```bash
# Checkout hipstershop-helm-charts repo (not the service repo)
git checkout hipstershop-helm-charts:develop

# Update ONLY the tag line in the values file
sed -i "s|^\(\s*tag:\s*\).*|\1${IMAGE_TAG}|" \
  hipstershop-micro-apps/adservice/values.dev.yaml

# Commit (idempotent — skips if no change)
git diff --cached --quiet && echo "No changes" && exit 0
git commit -m "chore(adservice): update image tag to ${IMAGE_TAG}"
git push origin develop
```

ArgoCD polls this repo every ~3 minutes. When it detects this commit, it triggers sync.

### 8.9 Pipeline Timing

| Stage | Typical Duration |
|-------|----------------|
| CI checks on PR (sonar + build + trivy) | 6–12 minutes |
| Docker build + push (push event) | 3–8 minutes |
| ArgoCD detection (git poll) | up to 3 minutes |
| Pod start + readiness (Java) | 60–90 seconds |
| Pod start + readiness (Go/Python/Node) | 5–15 seconds |
| **Total end-to-end (push → running pod)** | **~10–15 minutes** |

---

## 9. GitOps Deployment — ArgoCD

### 9.1 Why GitOps

**Problem with push-based CD (GitHub Actions running `kubectl apply`):**
- If someone manually changes a Kubernetes resource (`kubectl edit deployment/adservice`), the change is invisible
- No audit trail of what's running vs what's in git
- No automatic recovery from cluster drift

**ArgoCD GitOps solution:**
- Git is the **only** source of truth
- ArgoCD continuously reconciles cluster state against git (`selfHeal: true`)
- Drift detection: any manual change is automatically reverted
- One-click rollback to any previous git state

### 9.2 ArgoCD AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: hipstershop
  namespace: argocd
spec:
  description: "HipsterShop microservices platform"
  sourceRepos:
  - "https://github.com/HipsterShopp/hipstershop-helm-charts"
  destinations:
  - server: https://kubernetes.default.svc
    namespace: hipster-backend
  - server: https://kubernetes.default.svc
    namespace: hipster-frontend
  - server: https://kubernetes.default.svc
    namespace: hipster-database
  - server: https://kubernetes.default.svc
    namespace: kube-system
  clusterResourceWhitelist:
  - group: "*"
    kind: "*"
  orphanedResources:
    warn: true
```

### 9.3 ApplicationSet — Scaling Solution

Instead of 12 individual Application YAMLs, one ApplicationSet generates all of them:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: micro-apps-dev
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - name: adservice
        namespace: hipster-backend
      - name: assistantservice
        namespace: hipster-backend
      - name: authservice
        namespace: hipster-backend
      - name: cartservice
        namespace: hipster-backend
      - name: checkoutservice
        namespace: hipster-backend
      - name: currencyservice
        namespace: hipster-backend
      - name: emailservice
        namespace: hipster-backend
      - name: frontend
        namespace: hipster-frontend    # Only service NOT in hipster-backend
      - name: paymentservice
        namespace: hipster-backend
      - name: productcatalogservice
        namespace: hipster-backend
      - name: recommendationservice
        namespace: hipster-backend
      - name: shippingservice
        namespace: hipster-backend

  template:
    metadata:
      name: "{{name}}-dev"
      namespace: argocd
      annotations:
        argocd.argoproj.io/sync-wave: "3"
    spec:
      project: hipstershop
      source:
        repoURL: https://github.com/HipsterShopp/hipstershop-helm-charts
        targetRevision: develop
        path: "hipstershop-micro-apps/{{name}}"
        helm:
          valueFiles:
          - values.yaml
          - values.dev.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
        - CreateNamespace=false
        - ServerSideApply=true
```

**Adding a new microservice:** Add one element to `generators.list.elements` + create `hipstershop-micro-apps/{new-service}/` Helm chart. ArgoCD picks it up on next sync.

### 9.4 Sync Policies by Application

| Application | Auto-sync | selfHeal | prune | Reason |
|------------|----------|----------|-------|--------|
| mongodb | Yes | Yes | **No** | Never auto-delete database resources |
| backend-common | Yes | Yes | **No** | ConfigMaps should not be auto-deleted |
| gateway | Yes | Yes | Yes | Stateless config — safe to prune |
| micro-apps (dev) | Yes | Yes | Yes | Dev: always converge fast |
| micro-apps (prod) | Yes | Yes | **No** | Prod: safer, leave orphaned resources |

### 9.5 Sync Wave Order

```
Wave 0:   mongodb + backend-common
          ↓ (Synced + Healthy)
Wave 1:   sealed-secrets
          ↓ (Real Secrets decrypted in cluster)
Wave 2:   gateway
          ↓ (HTTPRoutes active)
Wave 3:   All 12 microservices via ApplicationSet
          ↓ (Services running and serving traffic)
Wave 4:   loadgenerator (disabled by default)
```

### 9.6 ArgoCD Commands

```bash
# Login
argocd login localhost:8080 --username admin --password <password> --insecure

# List all apps
argocd app list

# Get app status (sync status, health, current image)
argocd app get adservice-dev

# Force sync
argocd app sync adservice-dev

# Force refresh (re-poll git without syncing)
argocd app get adservice-dev --refresh

# View rendered manifests
argocd app manifests adservice-dev

# View diff (git vs cluster)
argocd app diff adservice-dev

# View history
argocd app history adservice-dev

# Rollback to previous version
argocd app rollback adservice-dev <history-id>

# Promote frontend blue-green (prod)
kubectl argo rollouts promote frontend -n hipster-frontend
```

### 9.7 Important: helm list Returns Empty

```bash
$ helm list -n hipster-backend
NAME  NAMESPACE  REVISION  STATUS  CHART  APP VERSION
(empty)
```

**This is EXPECTED.** ArgoCD uses `helm template` + `kubectl apply` — it never runs `helm install`. No Helm release is registered in cluster history. Use `argocd app list` instead.

---

## 10. Deployment Strategies

### 10.1 Rolling Update (All Services Except Frontend Prod)

Default strategy for all 12 microservices:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1    # 1 pod can be down — minimum availability
    maxSurge: 1          # 1 extra pod during update — avoids capacity issues
```

**Flow:**
1. New pod starts alongside old pod
2. New pod passes readiness probe
3. Old pod receives `SIGTERM`, waits `terminationGracePeriodSeconds: 5`
4. Old pod terminated — zero downtime (new pod already serving)

### 10.2 Blue-Green Deployment (Frontend Production)

**Why Blue-Green for frontend:** The frontend serves HTML templates and static assets that change visually. A rolling update mixing old and new frontend pods during promotion would serve inconsistent UI. Blue-Green ensures a clean cutover with the ability to preview before promoting.

```yaml
# Argo Rollouts Rollout (replaces Deployment in prod)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: frontend
  namespace: hipster-frontend
spec:
  strategy:
    blueGreen:
      activeService: frontend-active      # Routes live user traffic
      previewService: frontend-preview    # Staging preview (no live traffic)
      autoPromotionEnabled: false         # Human must approve promotion
      scaleDownDelaySeconds: 60           # Keep Blue alive 60s after promotion
```

**Blue-Green Flow:**
```
New image tag in values.prod.yaml
          │
          ▼
ArgoCD syncs Rollout spec
          │
          ▼
Argo Rollouts creates GREEN ReplicaSet
          │
          ▼
Green pods pass readiness probe → attached to frontend-preview service
          │
          ▼
⏸ PAUSED — operator reviews preview URL
          │
          ▼
kubectl argo rollouts promote frontend -n hipster-frontend
          │
          ▼
frontend-active service selector flips → GREEN pods serve live traffic
          │
          ▼
Blue RS kept 60s (rollback window) → scaled to 0
```

**Quick rollback:**
```bash
# One command reverts to previous known-good version
kubectl argo rollouts undo frontend -n hipster-frontend
```

### 10.3 GitHub OIDC Deploy (ShippingService Production)

ShippingService skips ArgoCD entirely in production. A self-hosted runner authenticates with GitHub OIDC and runs `kubectl set image` directly:

```yaml
deploy-to-prod:
  runs-on: self-hosted
  permissions:
    id-token: write    # Required to request OIDC token
    contents: read
  steps:
  - name: Request GitHub OIDC token
    run: |
      TOKEN=$(curl -sSf \
        -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
        "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=kubernetes" \
        | jq -r '.value')
      echo "token=$TOKEN" >> $GITHUB_OUTPUT

  - name: Configure kubectl (OIDC — no stored kubeconfig)
    run: |
      echo "${{ secrets.K8S_CA_CERT }}" | base64 -d > ca.crt
      kubectl config set-cluster prod-cluster \
        --server=${{ secrets.K8S_API_SERVER }} \
        --certificate-authority=ca.crt --embed-certs=true
      kubectl config set-credentials github-actions-oidc \
        --token="${{ steps.oidc.outputs.token }}"
      kubectl config set-context prod \
        --cluster=prod-cluster --user=github-actions-oidc
      kubectl config use-context prod

  - name: Deploy new image
    run: |
      IMAGE="yampss/hipstershop-shippingservice:${TAG}"
      kubectl set image deployment/shippingservice server="${IMAGE}" -n hipster-backend
      kubectl rollout status deployment/shippingservice -n hipster-backend --timeout=5m
```

**Security properties:**
- No `kubeconfig` stored in GitHub Secrets — only cluster URL and CA cert (non-sensitive)
- JWT expires in ~5 minutes after issuance
- RBAC RoleBinding locks to exact repo + exact branch (`refs/heads/main`)
- `develop` branch automatically denied (different OIDC `sub` claim)

---

## 11. Security Architecture

### 11.1 Defense in Depth

```
Layer 1: Network Policies       — default-deny-all; explicit allow rules only
Layer 2: Sealed Secrets         — all secrets encrypted in Git; cluster-specific decryption
Layer 3: RBAC                   — least-privilege roles for GitHub OIDC
Layer 4: Container Security     — distroless (no shell); non-root (cartservice: USER 1000)
Layer 5: Image Scanning         — Trivy CRITICAL CVE scan on every PR build
Layer 6: Dependency Scanning    — Snyk HIGH+ vulnerability scan on every PR
Layer 7: Transport Security     — inter-service via ClusterIP (cluster-internal network)
Layer 8: Authentication         — JWT (authservice); OIDC (shippingservice prod deploy)
Layer 9: Git Security           — Branch protection; PR required; human approval for prod
```

### 11.2 Sealed Secrets

```
plaintext Secret
      │
      ▼ kubeseal --cert cluster.pem
SealedSecret (safe to git commit)
      │
      ▼ Sealed Secrets controller (kube-system) decrypts with cluster private key
Kubernetes Secret (in cluster only)
```

**Secrets sealed:**
- `app-secrets` — `JWT_SECRET`, `GEMINI_API_KEY`
- `mongodb-root` — root admin credentials
- `mongodb-users` — per-service MongoDB credentials (auth, cart, catalog, order, payment, notification, analytics)

**Re-sealing after cluster rebuild:**
```bash
cd hipstershop-helm-charts/sealed-secrets/

# Re-fetch cluster public cert
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system > pub-cert.pem

# Re-export all secrets and re-seal
export JWT_SECRET="..."
# ... all other exports
bash generate-sealed-secrets.sh

git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: reseal secrets for new cluster"
git push
```

### 11.3 Container Security

| Practice | Services | Detail |
|---------|---------|--------|
| Distroless runtime | All Go services + some | No shell, no package manager, minimal CVE surface |
| Non-root user | cartservice | `USER 1000` in Dockerfile |
| SHA-pinned base images | Go, Node, Java services | Reproducible builds; upstream tag changes don't break builds |
| No hardcoded secrets | All services | Secrets from Kubernetes Secrets mounted as env vars |
| Multi-stage builds | All except assistantservice | Runtime image contains only compiled artifact |

### 11.4 TLS / Transport

- All inter-service traffic is within the cluster (ClusterIP to ClusterIP) — Kubernetes cluster network
- External traffic enters via kGateway on port 80 (HTTP)
- **Production recommendation:** TLS termination at kGateway level (cert-manager + Let's Encrypt)
- MongoDB connections use SCRAM-SHA-256 authentication

### 11.5 JWT Authentication

```
User POST /api/auth/login
         │
         ▼
authservice validates credentials → issues JWT signed with JWT_SECRET
         │
         ▼
Frontend stores JWT in cookie
         │
         ▼
Subsequent requests carry JWT → frontend validates with JWT_SECRET
(authservice handles revocation via /logout)
```

---

## 12. Observability

### 12.1 Current State

| Component | Status | Tool | Coverage |
|-----------|--------|------|---------|
| Distributed Tracing | ✅ Implemented | Jaeger (OpenTelemetry) | adservice (Java OTel agent), Go services (OTel SDK) |
| Structured Logging | ✅ Implemented | Logrus (JSON stdout) | authservice; all services log to stdout |
| Metrics | ⚠️ Planned | Prometheus (not deployed) | Gap — no alerting or dashboards |
| Log Aggregation | ⚠️ Planned | Loki/ELK (not deployed) | Gap — logs accessible via `kubectl logs` only |

### 12.2 Distributed Tracing (Jaeger)

All services that support OpenTelemetry connect to:
```
COLLECTOR_SERVICE_ADDR: "jaeger:4317"
ENABLE_TRACING:         "1"
```

adservice — bakes OTel Java agent into Docker image:
```dockerfile
RUN wget -O /opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/opentelemetry-javaagent.jar"
```

### 12.3 Structured Logging

authservice uses logrus for JSON-structured logs:
```go
log.WithFields(log.Fields{
  "method":     r.Method,
  "path":       r.URL.Path,
  "user_email": email,
}).Info("login attempt")
```

Output:
```json
{"level":"info","method":"POST","path":"/login","time":"2026-04-22T10:00:00Z","user_email":"user@test.com","msg":"login attempt"}
```

### 12.4 Viewing Logs

```bash
# Real-time logs
kubectl logs -f <pod-name> -n hipster-backend

# Logs from crashed container
kubectl logs <pod-name> -n hipster-backend --previous

# Logs by label (pod name changes with each deploy)
kubectl logs -l app=adservice -n hipster-backend --tail=100

# Logs from all replicas simultaneously
kubectl logs -l app=mongodb -n hipster-database --prefix=true

# Resource usage
kubectl top pods -n hipster-backend
kubectl top nodes
```

### 12.5 Recommended Future Observability Stack

```
Prometheus + Grafana (kube-prometheus-stack)
  → CPU, Memory, Request Rate, Error Rate, Latency per service
  → Alertmanager for threshold-based alerts (PagerDuty/Slack)

Grafana Loki + Promtail DaemonSet
  → Centralized log aggregation
  → Correlation IDs across services
  → 30-day log retention

Jaeger (already configured)
  → Request traces across all 12 services
  → Latency breakdown per service hop
```

---

## 13. Performance & Scalability

### 13.1 Resource Allocation

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit | Reason |
|---------|------------|----------|---------------|-------------|--------|
| adservice | 200m | 300m | 180Mi | 512Mi | JVM heap + OTel agent overhead |
| authservice | 100m | 200m | 64Mi | 128Mi | Go binary — minimal baseline |
| cartservice | 100m | 200m | 64Mi | 256Mi | .NET runtime overhead |
| checkoutservice | 100m | 200m | 64Mi | 128Mi | Go binary — orchestration only |
| currencyservice | 50m | 100m | 32Mi | 64Mi | Node.js — data transformation only |
| emailservice | 50m | 100m | 64Mi | 128Mi | Python — infrequent calls |
| frontend | 100m | 300m | 64Mi | 256Mi | Go binary + template rendering |
| paymentservice | 50m | 100m | 64Mi | 128Mi | Node.js |
| productcatalogservice | 100m | 200m | 64Mi | 128Mi | Go binary + MongoDB queries |
| recommendationservice | 50m | 100m | 64Mi | 128Mi | Python — read-only |
| shippingservice | 50m | 100m | 32Mi | 64Mi | Go — pure calculation |
| assistantservice | 100m | 300m | 128Mi | 512Mi | Python + AI inference calls |
| MongoDB (per pod) | 250m | 500m | 512Mi | 1Gi | Database engine |

Dev values use ~50% of above requests (resource savings on dev cluster).

### 13.2 Scalability Approach

- **Frontend HPA** — scales 1→5 replicas based on CPU (70%) and Memory (80%)  
- **Stateless services** — all 11 backend services are stateless (state in MongoDB); any can scale horizontally  
- **MongoDB replica set** — 3 nodes provide read scaling; primary handles writes  
- **Connection pooling** — Go MongoDB driver uses connection pool; Node.js services pool connections  

---

## 14. Networking Architecture

### 14.1 Traffic Flows

```
External User
     │ HTTP :80
     ▼
kGateway LoadBalancer (cluster external IP)
     │ path matching (11 HTTPRoute objects)
     │ URLRewrite filter
     ▼
ClusterIP Service (hipster-backend or hipster-frontend namespace)
     │ kube-proxy DNAT
     ▼
Pod (container port)
     │ (response)
     ▼
Back through gateway → User
```

### 14.2 Internal DNS

Kubernetes CoreDNS resolves service names:
```
authservice.hipster-backend.svc.cluster.local:8081
cartservice.hipster-backend.svc.cluster.local:7070
mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017
```

Services reference each other using short names configured in ConfigMaps:
```yaml
# service-addresses ConfigMap
CART_SERVICE_ADDR: "cartservice:7070"
EMAIL_SERVICE_ADDR: "emailservice:8080"
GATEWAY_ADDR: "hipstershop-gateway.hipster-backend.svc.cluster.local:80"
```

### 14.3 Namespace Isolation

```
hipster-frontend ──────────► hipster-backend  (via NetworkPolicy allow)
hipster-backend  ──────────► hipster-database (via NetworkPolicy allow)
hipster-backend  ──✗──────── internet         (blocked by NetworkPolicy, except SMTP)
hipster-database ──✗──────── hipster-frontend (no policy allows this)
```

---

## 15. Configuration Management

### 15.1 ConfigMap Strategy

All non-sensitive runtime configuration is stored in ConfigMaps in `hipster-backend`:

```yaml
# hipster-config — app-wide behavior
ENABLE_TRACING: "1"
COLLECTOR_SERVICE_ADDR: "jaeger:4317"
FRONTEND_MESSAGE: "Checkout HipsterShop today!"
ENABLE_ASSISTANT: "true"

# service-addresses — inter-service DNS
PRODUCT_CATALOG_SERVICE_ADDR: "productcatalogservice:3550"
CART_SERVICE_ADDR: "cartservice:7070"
GATEWAY_ADDR: "hipstershop-gateway.hipster-backend.svc.cluster.local:80"

# mongodb-config — DB/collection names
MONGO_REPLICA_SET: "rs0"
AUTH_DB_NAME: "auth_db"
CART_DB_NAME: "cart_db"
PAYMENT_DB_NAME: "payment_db"
PAYMENT_CHARGES_COLLECTION: "charges"

# app-features — feature flags
CURRENCY_SERVICE_RANDOM_ERROR: "1"       # Chaos simulation
PRODUCT_CATALOG_SEED_DATABASE: "true"   # Auto-seed products
GEMINI_MODEL: "gemini-2.5-flash"
```

### 15.2 Secret Management

| Secret | Location | Contents | Encrypted |
|--------|---------|---------|---------|
| `app-secrets` | hipster-backend | JWT_SECRET, GEMINI_API_KEY | Via Sealed Secrets |
| `mongodb-root` | hipster-backend + hipster-database | MONGO_ROOT_USERNAME/PASSWORD | Via Sealed Secrets |
| `mongodb-users` | hipster-backend + hipster-database | Per-service user/password pairs | Via Sealed Secrets |

### 15.3 Environment Variables Injection

```yaml
# In deployment template — multiple injection sources
containers:
- name: server
  envFrom:
  - configMapRef:
      name: hipster-config         # Bulk non-sensitive config
  - configMapRef:
      name: service-addresses      # Inter-service DNS
  env:
  - name: PORT
    valueFrom:
      configMapKeyRef:
        name: service-ports
        key: AUTHSERVICE_PORT
  - name: JWT_SECRET
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: JWT_SECRET
  - name: MONGO_USERNAME
    valueFrom:
      secretKeyRef:
        name: mongodb-users
        key: AUTH_MONGO_USERNAME
```

---

## 16. 12-Factor App Compliance

| Factor | Implementation |
|--------|--------------|
| **I — Codebase** | Each service in its own Git repo; one codebase, multiple deploys (dev/prod) |
| **II — Dependencies** | Explicitly declared (go.mod, package.json, requirements.txt, build.gradle, .csproj) |
| **III — Config** | All config in env vars from ConfigMaps/Secrets — no hardcoded values |
| **IV — Backing Services** | MongoDB treated as attached resource via URI env var — swap without code change |
| **V — Build/Release/Run** | CI builds Docker image artifact → Helm values release → ArgoCD deploys |
| **VI — Processes** | All containers are stateless; state exclusively in MongoDB StatefulSet |
| **VII — Port Binding** | Each service exports HTTP via PORT env var; no shared server |
| **VIII — Concurrency** | Scale via Kubernetes replica count + HPA |
| **IX — Disposability** | `terminationGracePeriodSeconds: 5`; fast startup (distroless Go); health probes |
| **X — Dev/Prod Parity** | Same Docker images; same Helm charts; different values.dev vs values.prod |
| **XI — Logs** | All services log to stdout (`kubectl logs`); structured JSON (authservice) |
| **XII — Admin Processes** | MongoDB init jobs run as Kubernetes Jobs; seed jobs as one-off Pods |

---

*Document continues in Part 3 — Runbook, Troubleshooting, ADRs, Appendix...*
