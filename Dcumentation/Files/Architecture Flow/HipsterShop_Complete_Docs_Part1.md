# 🛒 HipsterShop Platform — Complete DevOps Engineering Documentation

> **Organization:** HipsterShopp | **Repositories:** 14 | **Status:** Production | **Last Updated:** April 2026

---

## 📋 Table of Contents

| # | Section |
|---|---------|
| 1 | Executive Summary |
| 2 | Business Requirements & Problem Statement |
| 3 | Architecture Overview — HLD |
| 4 | Detailed Design — LLD (All 12 Services) |
| 5 | Infrastructure Setup |
| 6 | Kubernetes Implementation Deep Dive |
| 7 | Helm-Based Deployment Strategy |
| 8 | CI/CD Pipeline |
| 9 | GitOps Deployment — ArgoCD |
| 10 | Deployment Strategies |
| 11 | Security Architecture |
| 12 | Observability |
| 13 | Performance & Scalability |
| 14 | Networking Architecture |
| 15 | Configuration Management |
| 16 | 12-Factor App Compliance |
| 17 | Disaster Recovery & Backup |
| 18 | Runbook — Operations Guide |
| 19 | Troubleshooting Guide |
| 20 | Challenges & Solutions |
| 21 | Architecture Decision Records |
| 22 | Environment Strategy |
| 23 | Release & Versioning Strategy |
| 24 | Incident Management & SLA |
| 25 | Non-Functional Requirements |
| 26 | Cost & Resource Optimization |
| 27 | Future Enhancements |
| 28 | Appendix — Full Command Reference |

---

## 1. Executive Summary

**HipsterShop** is a production-grade, cloud-native e-commerce platform composed of **12 independently deployable microservices** spanning Go, Java, Python, Node.js, and .NET — all running on Kubernetes, managed via **ArgoCD GitOps**, deployed through **GitHub Actions CI/CD**, and versioned with **Helm charts**.

Users browse products, authenticate with JWT, manage carts, place orders, process payments, receive email confirmations, and interact with an **AI shopping assistant** powered by Google Gemini 2.5 Flash.

### Core Technology Stack

| Layer | Technology | Why Used |
|-------|-----------|----------|
| 🐳 Container Runtime | **Docker** (multi-stage builds) | Package each service with its exact runtime; eliminate environment drift across 5 languages |
| ☸️ Orchestration | **Kubernetes** (kubeadm cluster) | Self-healing, rolling updates, horizontal scaling, service discovery |
| 📦 Package Manager | **Helm v3** | Eliminate 24+ manually written YAMLs; centralize config; per-environment overrides |
| 🔁 CI Pipeline | **GitHub Actions** (shared reusable workflows) | One shared library powers all 12 services; change once, propagate everywhere |
| 🚀 CD / GitOps | **Argo CD** (ApplicationSet-driven) | Git as single source of truth; drift detection; declarative reconciliation |
| 🔵🟢 Progressive Delivery | **Argo Rollouts** (frontend blue-green) | Zero-downtime frontend deployments with manual promotion gate |
| 🔒 Secret Encryption | **Bitnami Sealed Secrets** | Encrypt Kubernetes Secrets in Git; safe to commit; cluster-specific decryption |
| 🌐 API Gateway | **kGateway** (Kubernetes Gateway API) | Single entry point; path-based routing; eliminates per-service LoadBalancers |
| 🗄️ Database | **MongoDB 7.0** (3-node Replica Set) | Persistent data; SCRAM-SHA-256 auth; per-service isolated databases |
| 💾 Persistent Storage | **NFS StorageClass** | Dynamic PVC provisioning for MongoDB; pod-reschedulable without data loss |
| 🤖 AI/ML | **Google Gemini 2.5 Flash** (LangGraph) | AI shopping assistant with agentic tool use (product fetch + cart add) |
| 🔍 Tracing | **Jaeger** (via OpenTelemetry) | Distributed tracing across all services |
| 🔐 OIDC Auth | **GitHub OIDC** (shippingservice) | Zero-stored-credential production deploy; short-lived JWT |

### Key Outcomes

- ✅ **Automated CI/CD** — code push to running pod in ~10–15 minutes  
- ✅ **Zero/near-zero downtime** — rolling updates + blue-green for frontend  
- ✅ **GitOps** — git is the only source of truth; ArgoCD selfHeal reverts manual changes  
- ✅ **Keyless production deployment** — ShippingService deployed via GitHub OIDC JWT (no stored kubeconfig)  
- ✅ **Security by default** — Network policies, Sealed Secrets, RBAC, distroless images  
- ✅ **12 microservices, 14 repos** — loosely coupled, independently deployable  

---

## 2. Business Requirements & Problem Statement

### Why Microservices for E-Commerce

**Monolith Problems:**
- Scale the entire app when only one service is stressed (e.g., product catalog during flash sales)
- Bug in payment service requires full application redeployment
- All teams blocked on a single deployment pipeline
- Technology lock-in — one language/framework for everything

**Microservices Solutions:**

| Business Requirement | Microservices Solution |
|---------------------|----------------------|
| Independent scaling | Scale only `productcatalogservice` during browse spikes — other services unaffected |
| Fault isolation | Payment failure doesn't bring down catalog browsing |
| Faster release cycles | Each service has its own CI/CD pipeline; teams deploy independently |
| Technology flexibility | Java for JVM ecosystem (adservice), Go for performance (auth, checkout, shipping), Python for AI/ML |
| Security isolation | Separate MongoDB databases per service; network policies prevent cross-service DB queries |
| Auditability | Per-service logs, separate git history per service repo |

### Platform Goals

1. **High Availability** — 3-node MongoDB replica set; rolling updates; ArgoCD drift healing  
2. **Security** — Sealed Secrets; Network Policies (default-deny); RBAC; OIDC; distroless images  
3. **Developer Velocity** — One shared CI library; `git push` triggers full automation; ArgoCD auto-syncs dev  
4. **Observability** — OpenTelemetry → Jaeger for distributed tracing; structured JSON logs to stdout  
5. **Elasticity** — HPA on frontend (1–5 replicas); resource limits on all 12 services  

---

## 3. Architecture Overview — HLD

### 3.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL USERS (HTTP)                        │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ Port 80
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│         kGateway  (Kubernetes Gateway API — kgateway controller)    │
│   GatewayClass: kgateway  |  hipstershop-gateway  |  Port :80       │
│     11 HTTPRoute objects with URLRewrite filters                     │
│  Namespace: hipster-backend (allowedRoutes: from: All)               │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬────────────────┘
   │ /    │/api  │      │      │      │      │      │ Each path ─►
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼  ClusterIP:port

┌─────────────── Namespace: hipster-frontend ────────────────────────┐
│  frontend  Go :8080   (Service port :80)                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ PROD: Argo Rollouts Blue-Green                               │   │
│  │   frontend-active  (live traffic)                            │   │
│  │   frontend-preview (staging, manual promote)                 │   │
│  │   scaleDownDelaySeconds: 60  |  autoPromotionEnabled: false  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  HPA: min=1, max=5  (CPU: 70%, Memory: 80%)                        │
└──────────────────────────┬─────────────────────────────────────────┘
                           │ calls all backend services
                           ▼
┌─────────────── Namespace: hipster-backend ──────────────────────────────────────┐
│                                                                                   │
│  adservice          Java 25 / Gradle  :9555  Ads (OTel agent, GHCR prod)         │
│  authservice        Go 1.26           :8081  JWT auth (gorilla/mux + logrus)     │
│  cartservice        .NET 8 / C#       :7070  Cart CRUD (ASP.NET Core, USER 1000) │
│  checkoutservice    Go 1.26           :5050  Checkout orchestrator (5 services)  │
│  currencyservice    Node.js 20        :7000  Currency conversion                 │
│  emailservice       Python 3.11       :8080  SMTP order email (port 587 egress)  │
│  paymentservice     Node.js 20        :50051 Simulated charge + MongoDB persist  │
│  productcatalog     Go 1.26           :3550  Product REST API + MongoDB/JSON     │
│  recommendationservice Python 3.11   :8080  Product recommendations             │
│  shippingservice    Go 1.26           :50051 Shipping quotes (OIDC prod deploy)  │
│  assistantservice   Python 3.10       :8080  Gemini 2.5 Flash AI (LangGraph)     │
│                                                                                   │
│  ConfigMaps: hipster-config, service-addresses, service-ports,                   │
│              mongodb-config, app-features                                         │
│  Secrets:    app-secrets (JWT+Gemini), mongodb-root, mongodb-users               │
│  NetworkPolicies: default-deny-all, backend-allow, emailservice-allow-smtp       │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │ MongoDB connections via headless DNS
                                       ▼
┌─────────────── Namespace: hipster-database ──────────────────────────────────────┐
│  MongoDB StatefulSet  |  3 replicas  |  mongo:7.0  |  ReplicaSet: rs0            │
│  DNS: mongodb-{0,1,2}.mongo-headless.hipster-database.svc.cluster.local:27017   │
│  Auth: SCRAM-SHA-256  |  Per-service users + isolated databases                  │
│  Storage: NFS PVCs  |  5Gi per replica = 15Gi total                              │
│  Databases: auth_db, cart_db, catalog_db, payment_db + more                     │
└──────────────────────────────────────────────────────────────────────────────────┘

┌─────────────── Namespace: argocd ────────────────────────────────────────────────┐
│  ArgoCD server + repo-server + application-controller                             │
│  AppProject: hipstershop  |  3 ApplicationSets (main, dev, prod)                 │
│  Sync Waves: 0=mongodb+backend-common, 1=sealed-secrets, 2=gateway, 3=micro-apps │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 CI/CD Flow Diagram

```
Developer git push / PR
          │
          ▼
GitHub Actions — per-service caller workflow
          │
   ┌──────┴──────────────────────────────────────────────────────────┐
   │ [PR only]  sonar-and-snyk                                        │
   │            SonarQube Quality Gate + Snyk HIGH+ dep scan          │
   └──────────────────────────────┬──────────────────────────────────┘
                                  │ (success or push event)
   ┌──────────────────────────────▼──────────────────────────────────┐
   │ generate-tag-and-build-image-trivy                               │
   │   main branch → service-vX.Y.Z (semver via conventional commits) │
   │   develop    → develop-{sha8}                                    │
   │   docker build → saves image.tar artifact                        │
   │   [PR] Trivy CRITICAL CVE scan                                   │
   └──────────────────────────────┬──────────────────────────────────┘
                                  │ (push event only)
   ┌──────────────────────────────▼──────────────────────────────────┐
   │ push                                                             │
   │   docker push yampss/hipstershop-{service}:{tag}                │
   │   docker push :latest  (main branch only)                        │
   │   git tag  (main branch only)                                    │
   └──────┬──────────────────────────────────────┬───────────────────┘
          │ develop branch                        │ main branch
          ▼                                       ▼
   update-helm-values-dev              push-to-ghcr
   (sed tag in values.dev.yaml          (cross-push GHCR)
    commit hipstershop-helm-charts:develop)    │
          │                           prod-approval-gate (human ✋)
          │                                     │
          │                           update-helm-values-prod
          │                           (sed tag in values.prod.yaml
          │                            commit hipstershop-helm-charts:main)
          │                           [shippingservice] deploy-to-prod
          │                           (OIDC kubectl set image)
          ▼                                     ▼
ArgoCD polls helm-charts (~3 min)    ArgoCD polls helm-charts (~3 min)
micro-apps-appset-dev                micro-apps-appset-prod
auto-sync (prune:true)               auto-sync (prune:false)
          │                                     │
          ▼                                     ▼
Kubernetes rolling update            Kubernetes rolling update
Dev Cluster                          Prod Cluster (Blue-Green for frontend)

Failure at any step → notify.yaml → SendGrid failure email
```

### 3.3 Dual-Cluster Strategy

| Attribute | Dev Cluster | Prod Cluster |
|-----------|------------|-------------|
| ApplicationSet | `micro-apps-appset-dev.yaml` | `micro-apps-appset-prod.yaml` |
| Branch tracked | `develop` | `main` |
| Values files | `values.yaml` + `values.dev.yaml` | `values.yaml` + `values.prod.yaml` |
| Prune | ✅ Yes — fast convergence | ❌ No — safety net (orphaned resources kept) |
| ServerSideApply | ✅ Yes | ❌ No — Rollouts conflict prevention |
| Argo Rollouts | — | ✅ Frontend blue-green |
| ShippingService | Managed by ArgoCD | Direct OIDC `kubectl set image` |
| Services count | 12 | 11 (shipping excluded from AppSet) |

### 3.4 Repository Map

| Repository | Language | Purpose | Connected To |
|-----------|---------|---------|-------------|
| `hipstershop-adservice` | Java 25 / Gradle | Personalized ads (GHCR prod) | helm-charts, shared-workflows |
| `hipstershop-authservice` | Go 1.26 | JWT authentication + MongoDB | MongoDB, helm-charts |
| `hipstershop-cartservice` | .NET 8 / C# | Shopping cart + MongoDB | MongoDB, helm-charts |
| `hipstershop-checkoutservice` | Go 1.26 | Checkout orchestration (5 deps) | payment, shipping, email, cart, currency |
| `hipstershop-currencyservice` | Node.js 20 | Currency conversion | helm-charts |
| `hipstershop-emailservice` | Python 3.11 | SMTP order confirmation | helm-charts (SMTP egress) |
| `hipstershop-frontend` | Go 1.26 | HTML web UI + Argo Rollouts | All backend services |
| `hipstershop-paymentservice` | Node.js 20 | Simulated payment + MongoDB | MongoDB, helm-charts |
| `hipstershop-productcatalogservice` | Go 1.26 | Product catalog REST API | MongoDB, helm-charts |
| `hipstershop-recommendationservice` | Python 3.11 | Product recommendations | productcatalog, helm-charts |
| `hipstershop-shippingservice` | Go 1.26 | Shipping quotes (OIDC) | helm-charts |
| `hipstershop-assistantservice` | Python 3.10 / FastAPI | Gemini AI shopping assistant | productcatalog, cartservice via gateway |
| `hipstershop-shared-workflows` | YAML (GitHub Actions) | Reusable CI/CD library | All 12 service repos |
| `hipstershop-helm-charts` | Helm / YAML | GitOps control plane | ArgoCD in cluster |

---

## 4. Detailed Design — LLD

### 4.1 Service Design Summary

| Service | Language | Port | DB | Key Dependency | Why This Tech |
|---------|---------|------|-----|---------------|--------------|
| adservice | Java 25 | 9555 | None | OTel Java agent | JVM ecosystem for ad serving; preserves original demo design |
| authservice | Go 1.26 | 8081 | MongoDB auth_db | gorilla/mux, logrus | Fast startup, static binary, distroless-compatible |
| cartservice | .NET 8 | 7070 | MongoDB cart_db | ASP.NET Core | Demonstrates .NET in polyglot platform; built-in DI |
| checkoutservice | Go 1.26 | 5050 | None | 5 services | Go's concurrency model suits orchestration; custom money math |
| currencyservice | Node.js 20 | 7000 | None | — | JavaScript suitable for data transformation APIs |
| emailservice | Python 3.11 | 8080 | None | SMTP port 587 | Python's smtplib simplicity; consistent with other Python services |
| frontend | Go 1.26 | 8080 | None | All 7 backends | Go HTML templates; fast static binary; distroless |
| paymentservice | Node.js 20 | 50051 | MongoDB payment_db | Express.js, charge.js | Node.js rapid API development; graceful DB degradation |
| productcatalogservice | Go 1.26 | 3550 | MongoDB catalog_db | JSON fallback | Go REST + MongoDB driver; seed from embedded JSON |
| recommendationservice | Python 3.11 | 8080 | None | productcatalog | Python ML-friendly for future model integration |
| shippingservice | Go 1.26 | 50051 | None | GitHub OIDC | Pure calculation; OIDC deploy demo |
| assistantservice | Python 3.10 | 8080 | None | Gemini API, LangGraph | Python's AI/ML ecosystem; FastAPI async |

### 4.2 Inter-Service Communication Map

```
kGateway → frontend (80)
             │
             ├──► GET  productcatalogservice:3550   /products, /products/{id}
             ├──► GET/POST/DELETE cartservice:7070  /cart/*
             ├──► GET  currencyservice:7000         /currencies, /convert
             ├──► POST checkoutservice:5050         /checkout
             ├──► GET  recommendationservice:8080   /recommendations
             ├──► POST authservice:8081             /login, /signup, /logout, /me
             └──► GET  adservice:9555               /ads

kGateway → assistantservice:8080  POST /api/assistant/chat
             │
             ├──► GET  gateway → productcatalogservice  (tool: get_product_details)
             └──► POST gateway → cartservice            (tool: add_to_cart)

kGateway → checkoutservice:5050  POST /api/checkout
             │
             ├──► GET  cartservice:7070          get cart items
             ├──► POST paymentservice:50051      charge card
             ├──► GET  shippingservice:50051     get quote
             ├──► POST emailservice:8080         send confirmation
             ├──► GET  currencyservice:7000      convert prices
             └──► DELETE cartservice:7070        empty cart after order
```

### 4.3 Service API Reference

#### adservice (Java 25 / Port 9555)
```
GET  /ads       Query: context keys → [{redirectUrl, text}]
GET  /_healthz  → 200 OK
```

#### authservice (Go / Port 8081)
```
POST /signup    Body: {email, password}  → 201 user created
POST /login     Body: {email, password}  → 200 {token: "JWT..."}
POST /logout    Header: Authorization    → 200 logged out
GET  /me        Header: Authorization    → 200 {user object}
GET  /_healthz  → 200 OK
```

#### cartservice (.NET / Port 7070)
```
POST   /cart/add         Body: {userId, item: {productId, quantity}} → 200
GET    /cart/{userId}    → 200 {items: [{productId, quantity}]}
DELETE /cart/{userId}    → 204 cart emptied
GET    /_healthz         → 200 OK
```

#### checkoutservice (Go / Port 5050)
```
POST /checkout   Body: {userId, email, address, creditCard} → 200 {orderId}
GET  /_healthz   → 200 OK
```

#### currencyservice (Node.js / Port 7000)
```
GET  /currencies       → ["USD","EUR","GBP","INR",...]
POST /convert          Body: {from, to, amount} → converted amount
GET  /_healthz         → 200 OK
```

#### emailservice (Python / Port 8080)
```
POST /send-confirmation  Body: order details → 200 email sent
GET  /_healthz           → 200 OK
```

#### frontend (Go / Ports 8080/80)
```
GET/POST  /               Home page (product listing)
GET       /product/{id}   Product detail
GET/POST  /cart           Cart view / add item
GET/POST  /checkout       Checkout / place order
GET/POST  /login /signup  Auth pages
POST      /logout         Log out
POST      /setCurrency    Change display currency
GET       /_healthz       → "ok"
```

#### paymentservice (Node.js / Port 50051)
```
POST /charge   Body: {amount:{currencyCode,units,nanos}, creditCard:{...}}
               → 200 {transactionId: "uuid"} | 400 {error: "validation msg"}
GET  /_healthz → 200 OK
```

#### productcatalogservice (Go / Port 3550)
```
GET /products         → [{id, name, description, picture, priceUsd, categories}]
GET /products/{id}    → single product object
GET /_healthz         → 200 OK
```

#### shippingservice (Go / Port 50051)
```
GET  /getQuote   Query: items → {costUsd: {currencyCode, units, nanos}}
POST /shipOrder  Body: order → {trackingId: "uuid"}
GET  /_healthz   → 200 OK
```

#### assistantservice (Python / Port 8080)
```
POST /api/assistant/chat  Body: {productId, message} → {reply: "AI response"}
GET  /_healthz             → "ok"
OPTIONS /api/assistant/chat → CORS preflight
```

---

## 5. Infrastructure Setup

### 5.1 Prerequisites Checklist

```bash
# 1. Kubernetes cluster ready
kubectl cluster-info

# 2. Helm v3 installed
helm version

# 3. ArgoCD in argocd namespace
kubectl get pods -n argocd | grep argocd-server

# 4. NFS StorageClass named "nfs"
kubectl get storageclass nfs

# 5. kGateway controller
kubectl get pods -n kgateway-system

# 6. Sealed Secrets controller
kubectl get pods -n kube-system | grep sealed-secrets

# 7. kubeseal CLI
kubeseal --version

# 8. Argo Rollouts controller (for prod frontend)
kubectl get pods -n argo-rollouts
```

### 5.2 Complete Bootstrap Sequence

#### Step 1 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for readiness
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=300s

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080  |  user: admin
```

#### Step 2 — Install Sealed Secrets

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# Verify
kubectl get pods -n kube-system | grep sealed-secrets
```

#### Step 3 — Install kGateway

```bash
# Install Kubernetes Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Install kgateway controller (follow official kgateway.dev docs)
helm repo add kgateway https://kgateway.dev/charts
helm install kgateway kgateway/kgateway --namespace kgateway-system --create-namespace
```

#### Step 4 — Setup NFS StorageClass

**Why NFS over hostPath:** MongoDB StatefulSet pods may be rescheduled to any node. `hostPath` binds data to one node — if the pod moves, data is gone. NFS provides shared storage accessible from any node.

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=<YOUR-NFS-SERVER-IP> \
  --set nfs.path=/srv/nfs/hipstershop \
  --set storageClass.name=nfs

kubectl get storageclass nfs   # Should show: nfs (default provisioner)
```

#### Step 5 — Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
kubectl argo rollouts version
```

#### Step 6 — Configure GitHub OIDC (for shippingservice)

Add these flags to kube-apiserver manifest (`/etc/kubernetes/manifests/kube-apiserver.yaml`):

```yaml
- --oidc-issuer-url=https://token.actions.githubusercontent.com
- --oidc-client-id=kubernetes
- --oidc-username-claim=sub
- --oidc-username-prefix=github:
```

#### Step 7 — Create Namespaces

```bash
kubectl create namespace hipster-backend  --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace hipster-frontend --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace hipster-database --dry-run=client -o yaml | kubectl apply -f -
```

#### Step 8 — Clone Helm Charts Repo and Bootstrap ArgoCD

```bash
git clone https://github.com/HipsterShopp/hipstershop-helm-charts
cd hipstershop-helm-charts

# Wave 0 — Apply ArgoCD project and infrastructure foundations
kubectl apply -f hipstershop-argoapps/project.yaml
kubectl apply -f hipstershop-argoapps/infra/mongodb.yaml
kubectl apply -f hipstershop-argoapps/infra/backend-common.yaml

# Wait for wave 0 to sync
argocd app get mongodb
argocd app get backend-common

# Wave 1 — Generate and apply Sealed Secrets
cd sealed-secrets/

export JWT_SECRET="your-jwt-secret-here"
export GEMINI_API_KEY="your-gemini-api-key"
export MONGO_ROOT_PASSWORD="your-mongo-root-password"
export AUTH_PASS="your-auth-service-password"
export CART_PASS="your-cart-service-password"
export CATALOG_PASS="your-catalog-service-password"
export ORDER_PASS="your-order-service-password"
export PAYMENT_PASS="your-payment-service-password"
export NOTIFICATION_PASS="your-notification-service-password"
export ANALYTICS_PASS="your-analytics-service-password"

bash generate-sealed-secrets.sh

git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: add sealed secrets"
git push

kubectl apply -f ../hipstershop-argoapps/infra/sealed-secrets.yaml

# Verify secrets decrypted
kubectl get secret app-secrets   -n hipster-backend
kubectl get secret mongodb-root  -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
kubectl get secret mongodb-root  -n hipster-database
kubectl get secret mongodb-users -n hipster-database

# Wave 2 — Gateway
cd ..
kubectl apply -f hipstershop-argoapps/infra/gateway.yaml
argocd app get gateway   # Wait for Synced + Healthy

# Wave 3 — All Microservices via ApplicationSet
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset.yaml
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset-dev.yaml   # dev
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset-prod.yaml  # prod

# Watch all apps come online
argocd app list
```

### 5.3 ArgoCD Sync Wave Order

| Wave | Applications | Must Complete Before |
|------|-------------|---------------------|
| 0 | `mongodb`, `backend-common` | Namespaces + ConfigMaps + Secrets ready |
| 1 | `sealed-secrets` | Real Kubernetes Secrets created from SealedSecrets |
| 2 | `gateway` | kGateway + HTTPRoutes accepting traffic |
| 3 | All 12 microservices (via ApplicationSets) | Services running and healthy |

---

## 6. Kubernetes Implementation Deep Dive

### 6.1 Why Kubernetes

| Problem with docker-compose | Kubernetes Solution |
|---------------------------|-------------------|
| No restart on crash | Liveness probes + restart policy |
| Downtime during deploy | Rolling updates (maxUnavailable: 1) |
| No horizontal scaling | Replica count + HPA |
| Manual service discovery | CoreDNS (e.g., `authservice:8081`) |
| No resource governance | CPU/Memory requests and limits |
| Single node | Multi-node scheduling |
| No secret management | Secrets API + Sealed Secrets |

### 6.2 Full Deployment Manifest (adservice with annotations)

```yaml
# hipstershop-helm-charts/hipstershop-micro-apps/adservice/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: hipster-backend
  name: adservice
  labels:
    app: adservice
spec:
  replicas: 1                             # Single replica — stateless service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1                   # 1 pod can be unavailable during update
      maxSurge: 1                         # 1 extra pod allowed during update
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        tier: backend
    spec:
      terminationGracePeriodSeconds: 5    # Fast drain — pod has 5s to finish requests
      containers:
      - name: server
        image: yampss/hipstershop-adservice:adservice-v0.1.2   # Injected by CI via Helm values
        ports:
        - containerPort: 9555
        envFrom:
        - configMapRef:
            name: hipster-config          # ENABLE_TRACING, COLLECTOR_SERVICE_ADDR, GEMINI_MODEL
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: service-ports
              key: ADSERVICE_PORT         # "9555"
        resources:
          requests:
            cpu: 200m                     # JVM needs baseline CPU at idle
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 512Mi                 # JVM heap + OTel agent overhead
        readinessProbe:
          initialDelaySeconds: 60         # JVM + OTel agent takes 50-60s to start
          periodSeconds: 15
          httpGet:
            path: "/_healthz"
            port: 9555
        livenessProbe:
          initialDelaySeconds: 60
          periodSeconds: 15
          httpGet:
            path: "/_healthz"
            port: 9555
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume               # OTel agent writes to /tmp
      volumes:
      - name: tmp-volume
        emptyDir: {}                     # Ephemeral — cleared on pod restart
```

### 6.3 MongoDB StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: hipster-database
  name: mongodb
spec:
  serviceName: mongo-headless             # Required for stable DNS per pod
  replicas: 3                             # 3-node replica set rs0
  selector:
    matchLabels:
      app: mongodb
  template:
    spec:
      containers:
      - name: mongodb
        image: mongo:7.0
        command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-root
              key: MONGO_ROOT_USERNAME
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-root
              key: MONGO_ROOT_PASSWORD
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        readinessProbe:
          exec:
            command: ["mongosh", "--eval", "db.adminCommand('ping')"]
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          exec:
            command: ["mongosh", "--eval", "db.adminCommand('ping')"]
          initialDelaySeconds: 30
          periodSeconds: 20
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"             # Dynamic NFS provisioning
      resources:
        requests:
          storage: 5Gi                    # Creates: mongo-data-mongodb-0/1/2
```

### 6.4 kGateway Configuration

**Why kGateway instead of Ingress:**
- Kubernetes Gateway API is the successor to Ingress (more expressive, role-oriented)
- `URLRewrite` filters allow external path mapping without service changes
- Single resource manages routing for all 12 services

```yaml
# GatewayClass (one per cluster)
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kgateway
spec:
  controllerName: kgateway.dev/kgateway
---
# Gateway (one per platform)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  namespace: hipster-backend
  name: hipstershop-gateway
spec:
  gatewayClassName: kgateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All     # Accepts HTTPRoutes from all namespaces (including hipster-frontend)
```

### 6.5 HTTPRoute Routing Table

| External Path | Rewrites To | Backend Service | Port |
|-------------|------------|---------------|------|
| `GET /api/ads` | `/ads` | adservice | 9555 |
| `POST /api/auth/login` | `/login` | authservice | 8081 |
| `POST /api/auth/signup` | `/signup` | authservice | 8081 |
| `POST /api/auth/logout` | `/logout` | authservice | 8081 |
| `GET /api/auth/me` | `/me` | authservice | 8081 |
| `PathPrefix /api/cart` | `/cart/*` | cartservice | 7070 |
| `POST /api/checkout` | `/checkout` | checkoutservice | 5050 |
| `GET /api/currency/currencies` | `/currencies` | currencyservice | 7000 |
| `POST /api/currency/convert` | `/convert` | currencyservice | 7000 |
| `POST /api/email/send-confirmation` | `/send-confirmation` | emailservice | 8080 |
| `POST /api/payment/charge` | `/charge` | paymentservice | 50051 |
| `GET /api/products/*` | `/products/*` | productcatalogservice | 3550 |
| `GET /api/recommendations` | `/recommendations` | recommendationservice | 8080 |
| `PathPrefix /api/shipping` | `/` | shippingservice | 50051 |
| `POST /api/assistant/chat` | (no rewrite) | assistantservice | 8080 |
| `/, /product, /cart, /login, /signup, /static/*, /_healthz` | (direct) | frontend | 80 |

### 6.6 Network Policies (Zero-Trust)

```yaml
# Policy 1: default-deny-all (hipster-backend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: hipster-backend
spec:
  podSelector: {}       # All pods
  policyTypes:
  - Ingress
  - Egress
---
# Policy 2: backend-allow (controlled exceptions)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow
  namespace: hipster-backend
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - ipBlock:
        cidr: 172.31.0.0/16           # VPC CIDR (cluster nodes)
  - from:
    - namespaceSelector:
        matchLabels:
          name: hipster-frontend      # Allow frontend → backend
  - from:
    - podSelector: {}                  # Allow intra-backend communication
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: hipster-database      # Backend → MongoDB
  - to:
    - podSelector: {}                  # Intra-backend pods
  - to:
    - namespaceSelector:
        matchLabels:
          name: hipster-frontend
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kgateway-system
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53                         # DNS resolution
    - protocol: TCP
      port: 53
---
# Policy 3: emailservice SMTP egress only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emailservice-allow-smtp
  namespace: hipster-backend
spec:
  podSelector:
    matchLabels:
      app: emailservice
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0               # Gmail SMTP (any IP)
    ports:
    - protocol: TCP
      port: 587                        # SMTP with STARTTLS
```

### 6.7 RBAC for GitHub OIDC (ShippingService)

```yaml
# hipstershop-infra/rbac/github-actions-oidc-shippingservice.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: github-actions-shippingservice-verifier
  namespace: hipster-backend
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]   # kubectl set image
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list"]                       # rollout status
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]                       # verify pod state
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-shippingservice-verifier
  namespace: hipster-backend
subjects:
- kind: User
  # OIDC subject: exact repo + exact branch — develop branch automatically denied
  name: "github:repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: github-actions-shippingservice-verifier
  apiGroup: rbac.authorization.k8s.io
```

### 6.8 HPA — Frontend Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
  namespace: hipster-frontend
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: frontend
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 20    # Conservative scale-down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 5     # Aggressive scale-up for traffic spikes
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
```

---

*See Part 2 for: Helm, CI/CD, GitOps, Security, Observability, Runbook, Troubleshooting, ADRs, Appendix*
