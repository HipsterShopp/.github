# hipstershop-helm-charts

> The GitOps control plane for the HipsterShop platform. Contains all Helm charts for every microservice and infrastructure component, plus all ArgoCD Application/ApplicationSet definitions. This is the single source of truth for cluster state.

---

## Table of Contents
- [Repository Purpose](#repository-purpose)
- [Repository Structure](#repository-structure)
- [ArgoCD Bootstrap Guide](#argocd-bootstrap-guide)
- [Helm Charts Reference](#helm-charts-reference)
- [Infrastructure Charts](#infrastructure-charts)
- [Microservice Charts](#microservice-charts)
- [Values Override Pattern](#values-override-pattern)
- [Sealed Secrets](#sealed-secrets)
- [ArgoCD Application Management](#argocd-application-management)
- [Troubleshooting](#troubleshooting)

---

## Repository Purpose

This repo is the **destination** of the CI/CD pipeline. When a microservice's GitHub Actions workflow builds a new Docker image and wants to deploy it, it updates a `values.yaml` file here and commits. ArgoCD watches this repo and automatically syncs the cluster to match.

**Key principle:** No human or CI/CD system should run `kubectl apply` directly against the production cluster (except for initial bootstrap and shippingservice's OIDC deploy). Everything flows through git commits to this repo → ArgoCD reconciliation.

---

## Repository Structure

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
│       ├── mongodb.yaml               # Wave 0: MongoDB StatefulSet
│       ├── backend-common.yaml        # Wave 0: ConfigMaps, Secrets, NetworkPolicies
│       ├── sealed-secrets.yaml        # Wave 1: Sealed Secrets controller
│       ├── gateway.yaml               # Wave 2: kGateway + HTTPRoutes
│       ├── gateway-prod.yaml          # Wave 2: Production gateway variant
│       └── loadgenerator.yaml         # Wave 3: Load test job
│
├── hipstershop-infra/                 # Infrastructure Helm charts
│   ├── backend-common/                # Namespace, ConfigMaps, Secrets, NetworkPolicy
│   ├── gateway/                       # kGateway resources
│   ├── loadgenerator/                 # Load testing
│   ├── mongodb/                       # MongoDB StatefulSet
│   └── rbac/                          # GitHub OIDC RBAC for shippingservice
│
├── hipstershop-micro-apps/            # Microservice Helm charts (one per service)
│   ├── adservice/
│   ├── assistantservice/
│   ├── authservice/
│   ├── cartservice/
│   ├── checkoutservice/
│   ├── currencyservice/
│   ├── emailservice/
│   ├── frontend/
│   ├── paymentservice/
│   ├── productcatalogservice/
│   ├── recommendationservice/
│   └── shippingservice/
│
└── sealed-secrets/                    # Secret generation tooling
    ├── .gitignore                     # Ignores pub-cert.pem
    ├── backend-common-secrets.yaml    # Encrypted secrets (safe to commit)
    ├── mongodb-secrets.yaml           # Encrypted MongoDB secrets
    ├── generate-sealed-secrets.sh     # Generator script (run in WSL/Git Bash)
    └── steps.txt                      # Step-by-step operator runbook
```

---

## ArgoCD Bootstrap Guide

Run these steps in order. Each wave must be `Synced + Healthy` before the next.

### Prerequisites

```bash
# ArgoCD installed
kubectl get pods -n argocd | grep argocd-server

# Sealed Secrets controller installed
kubectl get pods -n kube-system | grep sealed-secrets

# NFS StorageClass exists
kubectl get storageclass nfs

# kGateway controller installed
kubectl get pods -n kgateway-system 2>/dev/null || echo "install kgateway"

# Namespaces must exist before sealed secrets can be generated
kubectl create namespace hipster-backend  --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace hipster-database --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace hipster-frontend --dry-run=client -o yaml | kubectl apply -f -
```

### Wave 0 — Foundations

Apply the ArgoCD project first, then both infrastructure foundations simultaneously:

```bash
kubectl apply -f hipstershop-argoapps/project.yaml
kubectl apply -f hipstershop-argoapps/infra/mongodb.yaml
kubectl apply -f hipstershop-argoapps/infra/backend-common.yaml
```

**Wait until both show Synced + Healthy:**
```bash
argocd app get mongodb
argocd app get backend-common
```

What is deployed:
- `mongodb`: MongoDB StatefulSet (3 replicas), Services, Namespace (`hipster-database`), and Kubernetes Secrets (mongodb-root, mongodb-users) from `hipstershop-infra/mongodb/`
- `backend-common`: Namespace (`hipster-backend`), 5 ConfigMaps (hipster-config, service-addresses, service-ports, mongodb-config, app-features), 3 Secrets (app-secrets, mongodb-root, mongodb-users), NetworkPolicies

### Wave 1 — Sealed Secrets

**First generate sealed secrets (run from WSL/Git Bash):**

```bash
cd sealed-secrets/

# Set real values (these stay in your terminal session only)
export JWT_SECRET="team4hipstershopsecret"
export GEMINI_API_KEY="your-actual-gemini-api-key"
export MONGO_ROOT_PASSWORD="your-real-root-password"
export AUTH_PASS="your-auth-password"
export CART_PASS="your-cart-password"
export CATALOG_PASS="your-catalog-password"
export ORDER_PASS="your-order-password"
export PAYMENT_PASS="your-payment-password"
export NOTIFICATION_PASS="your-notification-password"
export ANALYTICS_PASS="your-analytics-password"

bash generate-sealed-secrets.sh

# Commit the encrypted files (pub-cert.pem is gitignored)
git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: regenerate sealed secrets"
git push
```

**Then apply:**
```bash
kubectl apply -f hipstershop-argoapps/infra/sealed-secrets.yaml
```

**Verify secrets were decrypted into the cluster:**
```bash
kubectl get secret app-secrets   -n hipster-backend
kubectl get secret mongodb-root  -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
kubectl get secret mongodb-root  -n hipster-database
kubectl get secret mongodb-users -n hipster-database
```

### Wave 2 — Gateway

```bash
kubectl apply -f hipstershop-argoapps/infra/gateway.yaml
```

Wait for `Synced + Healthy`. This deploys the `GatewayClass`, `Gateway`, and all 11 `HTTPRoute` objects from `hipstershop-infra/gateway/`.

### Wave 3 — Microservices

```bash
# Apply the ApplicationSet — creates one Application per service
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset.yaml

# For dev environment:
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset-dev.yaml

# For prod environment:
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset-prod.yaml
```

ArgoCD will create 11-12 Applications and begin syncing from the helm chart paths. First sync may take 3-5 minutes as images are pulled.

---

## Helm Charts Reference

### Chart: backend-common

**Path:** `hipstershop-infra/backend-common/`  
**Managed by ArgoCD Application:** `backend-common` (Wave 0)

Creates the following Kubernetes resources in `hipster-backend`:

**Namespace:**
```yaml
# namespace.yaml
kind: Namespace
metadata:
  name: hipster-backend
  annotations:
    helm.sh/resource-policy: keep  # Never deleted by Helm/ArgoCD
```

**ConfigMaps (5 total):**

| ConfigMap | Keys | Purpose |
|-----------|------|---------|
| `hipster-config` | ENABLE_TRACING, COLLECTOR_SERVICE_ADDR, FRONTEND_MESSAGE, ENABLE_ASSISTANT | App-wide settings |
| `service-addresses` | GATEWAY_ADDR, PRODUCT_CATALOG_SERVICE_ADDR, SHIPPING_SERVICE_ADDR, PAYMENT_SERVICE_ADDR, EMAIL_SERVICE_ADDR, CURRENCY_SERVICE_ADDR, CART_SERVICE_ADDR | Inter-service DNS addresses |
| `service-ports` | ADSERVICE_PORT (9555), AUTHSERVICE_PORT (8081), CARTSERVICE_PORT (7070), CHECKOUTSERVICE_PORT (5050), CURRENCYSERVICE_PORT (7000), EMAILSERVICE_PORT (8080), FRONTEND_PORT (8080), PAYMENTSERVICE_PORT (50051), PRODUCTCATALOG_PORT (3550), RECOMMENDATIONSERVICE_PORT (8080), SHIPPINGSERVICE_PORT (50051) | All service ports |
| `mongodb-config` | MONGO_REPLICA_SET (rs0), AUTH_DB_NAME, CART_DB_NAME, CATALOG_DB_NAME, ORDER_DB_NAME, PAYMENT_DB_NAME, NOTIFICATION_DB_NAME, ANALYTICS_DB_NAME, CATALOG_PRODUCTS_COLLECTION, ORDER_ORDERS_COLLECTION, etc. | MongoDB database/collection names |
| `app-features` | CURRENCY_SERVICE_RANDOM_ERROR, PAYMENT_SERVICE_DISABLE_PROFILER, PRODUCT_CATALOG_SEED_DATABASE, GEMINI_MODEL | Feature flags |

**Secrets (3 total — overwritten by Sealed Secrets):**

| Secret | Keys | Purpose |
|--------|------|---------|
| `app-secrets` | JWT_SECRET, GEMINI_API_KEY | Application secrets |
| `mongodb-root` | MONGO_ROOT_USERNAME, MONGO_ROOT_PASSWORD | MongoDB root credentials |
| `mongodb-users` | AUTH_MONGO_USERNAME/PASSWORD, CART_MONGO_USERNAME/PASSWORD, CATALOG_MONGO_USERNAME/PASSWORD, ORDER_MONGO_USERNAME/PASSWORD, PAYMENT_MONGO_USERNAME/PASSWORD, NOTIFICATION_MONGO_USERNAME/PASSWORD, ANALYTICS_MONGO_USERNAME/PASSWORD | Per-service MongoDB credentials |

**NetworkPolicies (3 total):**
- `default-deny-all` — Blocks all ingress + egress in hipster-backend
- `backend-allow` — Allows intra-backend + to/from frontend, database, kgateway, DNS
- `emailservice-allow-smtp` — Allows emailservice egress to port 587 (Gmail SMTP)

**Values overview:**
```yaml
# backend-common/values.yaml (key sections)
global:
  secrets:
    jwtSecret: "team4hipstershopsecret"        # Overridden by SealedSecret
    geminiApiKey: "REPLACE_WITH_GEMINI_API_KEY" # Overridden by SealedSecret
  appConfig:
    enableTracing: "1"
    collectorServiceAddr: "jaeger:4317"
    frontendMessage: "Checkout HipsterShop today!"
    enableAssistant: "true"
  features:
    currencyServiceRandomError: "1"
    paymentServiceDisableProfiler: "1"
    productCatalogSeedDatabase: "true"
    geminiModel: "gemini-2.5-flash"
  mongodb:
    rootUsername: admin
    rootPassword: mongopassword           # Overridden by SealedSecret
```

---

### Chart: gateway

**Path:** `hipstershop-infra/gateway/`  
**Managed by ArgoCD Application:** `gateway` (Wave 2)

Creates `GatewayClass`, `Gateway`, and 11 `HTTPRoute` objects:

```yaml
# gateway.yaml (template)
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kgateway
spec:
  controllerName: kgateway.dev/kgateway
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: hipstershop-gateway
  namespace: hipster-backend
spec:
  gatewayClassName: kgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

All 11 `HTTPRoute` objects in `routes.yaml` use `URLRewrite` to translate external paths:

```yaml
# Example: ad-route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ad-route
  namespace: hipster-backend
spec:
  parentRefs:
    - name: hipstershop-gateway
  rules:
    - matches:
        - path:
            type: Exact
            value: /api/ads
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplaceFullPath
              replaceFullPath: /ads
      backendRefs:
        - name: adservice
          port: 9555
```

**`ignoreDifferences` on gateway Application:**
kGateway controller modifies `GatewayClass` annotations/labels after sync. ArgoCD's `ignoreDifferences` prevents it from fighting the controller:
```yaml
ignoreDifferences:
  - group: gateway.networking.k8s.io
    kind: GatewayClass
    jsonPointers: [/metadata/annotations, /metadata/labels, /spec/parametersRef]
```

---

### Chart: mongodb

**Path:** `hipstershop-infra/mongodb/`  
**Managed by ArgoCD Application:** `mongodb` (Wave 0, prune: false)

Creates:
- `Namespace/hipster-database` (with `helm.sh/resource-policy: keep`)
- `Secret/mongodb-root` and `Secret/mongodb-users` in hipster-database
- `Service/mongo-headless` (headless for StatefulSet DNS)
- `Service/mongo-nodeport` (for external access)
- `StatefulSet/mongodb` (3 replicas, mongo:7.0, NFS storage)
- Init job to bootstrap MongoDB replica set and create users

**Critical: prune: false** — The mongodb ArgoCD Application has `prune: false`. If resources are deleted from git, ArgoCD will NOT delete them from the cluster. This prevents accidental data loss.

---

## Microservice Charts

Each of the 12 microservices has its own Helm chart at `hipstershop-micro-apps/{service-name}/`:

```
{service}/
├── Chart.yaml
├── values.yaml           # Base values (main branch prod-ish defaults)
├── values.dev.yaml       # Dev overrides (lighter resources, develop-latest image)
├── values.prod.yaml      # Prod overrides (GHCR image, prod resource levels)
└── templates/
    ├── deployment.yaml   # Deployment (or Rollout for frontend in prod)
    └── service.yaml      # ClusterIP Service
```

### How the ApplicationSet Generates Applications

```yaml
# micro-apps-appset.yaml generates one Application per element:
generators:
  - list:
      elements:
        - name: adservice
          namespace: hipster-backend
        - name: frontend
          namespace: hipster-frontend  # frontend goes to different namespace
        # ... 9 more services

template:
  metadata:
    name: "{{name}}"
  spec:
    source:
      path: "hipstershop-micro-apps/{{name}}"  # Template variable
      helm:
        valueFiles: [values.yaml]
    destination:
      namespace: "{{namespace}}"               # Template variable
```

---

## Values Override Pattern

### Three-Layer Override System

| File | Used In | Purpose |
|------|---------|---------|
| `values.yaml` | Base/Fallback | Default config, base image tags |
| `values.dev.yaml` | Dev ApplicationSet | Lighter resources, develop branch image tags |
| `values.prod.yaml` | Prod ApplicationSet | Full resources, GHCR image tags |

ArgoCD Helm rendering merges them in order:
```bash
helm template {service} hipstershop-micro-apps/{service} \
  -f hipstershop-micro-apps/{service}/values.yaml \
  -f hipstershop-micro-apps/{service}/values.dev.yaml   # dev
  # OR
  -f hipstershop-micro-apps/{service}/values.prod.yaml  # prod
```

Later files override earlier ones. Only specified keys are overridden — the rest inherit from base.

### Example: adservice values comparison

```yaml
# values.yaml (base)
global:
  images:
    adservice: "yampss/hipstershop-adservice:adservice-v0.1.2"
  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi

# values.dev.yaml (override)
global:
  images:
    adservice: "yampss/hipstershop-adservice:develop-latest"  # Updated by CI
  resources:
    adservice:
      requests:
        cpu: 100m    # Half of prod
        memory: 90Mi # Half of prod

# values.prod.yaml (override)
global:
  images:
    adservice: "ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.2.3"  # GHCR
  resources:
    adservice:
      requests:
        cpu: 200m    # Full prod resources
        memory: 180Mi
```

---

## Sealed Secrets

Sealed Secrets encrypts Kubernetes Secret objects with the cluster's RSA public key. Encrypted `SealedSecret` objects are safe to commit to git. The Sealed Secrets controller running in `kube-system` decrypts them into real Kubernetes Secrets.

### Files

| File | Contents | Commit to Git? |
|------|----------|---------------|
| `backend-common-secrets.yaml` | Encrypted app-secrets (JWT_SECRET, GEMINI_API_KEY, MongoDB passwords) for hipster-backend | ✅ Yes |
| `mongodb-secrets.yaml` | Encrypted MongoDB credentials for hipster-database | ✅ Yes |
| `generate-sealed-secrets.sh` | Bash script to generate the above files | ✅ Yes |
| `pub-cert.pem` | Cluster RSA public key (cluster-specific) | ❌ No (gitignored) |
| `steps.txt` | Operator runbook for initial setup and re-sealing | ✅ Yes |

### Re-sealing After Cluster Rebuild

Sealed secrets are **cluster-specific** — the encryption uses the cluster's RSA key pair. If the cluster is replaced or the key pair is rotated, re-sealing is required:

```bash
# In WSL/Git Bash:
cd sealed-secrets/
# Re-export all secret values
export JWT_SECRET="..."
# [... all other exports]
bash generate-sealed-secrets.sh
git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: reseal secrets for new cluster"
git push
```

---

## ArgoCD Application Management

### Viewing Application Status

```bash
# List all applications
argocd app list

# Get detailed status for one service
argocd app get adservice-dev

# See rendered manifests (what ArgoCD will apply)
argocd app manifests adservice-dev

# See diff between git state and cluster state
argocd app diff adservice-dev

# Force sync
argocd app sync adservice-dev

# Rollback to previous version
argocd app history adservice-dev
argocd app rollback adservice-dev <id>
```

### Sync Wave Order Summary

| Wave | Applications | What Must Complete |
|------|-------------|-------------------|
| 0 | mongodb, backend-common | Namespaces + ConfigMaps + Secrets ready |
| 1 | sealed-secrets | Real Kubernetes Secrets created from SealedSecrets |
| 2 | gateway | kGateway + HTTPRoutes accepting traffic |
| 3 | All microservices (via ApplicationSets) | Services running and healthy |

**Important:** `argocd.argoproj.io/sync-wave` annotation must be set correctly on Application/ApplicationSet metadata.

---

## Troubleshooting

### `helm list` Returns Empty

```bash
helm list -n hipster-backend    # Returns empty — EXPECTED
# ArgoCD uses helm template + kubectl apply, not helm install
# Use argocd commands instead:
argocd app get adservice-dev
```

### ArgoCD Stuck OutOfSync

```bash
# Check what is different
argocd app diff adservice-dev

# Force refresh (fetch latest git state)
argocd app get adservice-dev --refresh

# Check for controller-modified fields (need ignoreDifferences)
kubectl describe application adservice-dev -n argocd | grep -A10 "Comparison Error"
```

### Helm Rendering Failed

```bash
# Render locally to see error
cd hipstershop-helm-charts/
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  -f hipstershop-micro-apps/adservice/values.dev.yaml

# Lint
helm lint hipstershop-micro-apps/adservice -f hipstershop-micro-apps/adservice/values.yaml
```

### Sealed Secret Not Decrypting

```bash
# Check controller is running
kubectl get pods -n kube-system | grep sealed-secrets

# Check for events on the SealedSecret
kubectl describe sealedsecret app-secrets -n hipster-backend

# Check controller logs
kubectl logs -l app.kubernetes.io/name=sealed-secrets -n kube-system --tail=50
```

### Image Not Updating After CI Push

```bash
# Step 1: Verify the values file was updated
git log --oneline hipstershop-micro-apps/adservice/values.dev.yaml

# Step 2: Check ArgoCD sees the latest commit
argocd app get adservice-dev --refresh

# Step 3: Verify ArgoCD rendered the correct image
argocd app manifests adservice-dev | grep image:

# Step 4: Verify the pod is using the expected image
kubectl get deployment adservice -n hipster-backend \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### MongoDB Pod Pending

```bash
# Check if NFS StorageClass exists
kubectl get storageclass nfs
# Check PVC status
kubectl get pvc -n hipster-database
# Describe pending PVC
kubectl describe pvc mongo-data-mongodb-0 -n hipster-database
```
