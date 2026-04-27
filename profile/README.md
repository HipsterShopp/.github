#  HipsterShop — Cloud-Native GitOps Platform

> A production-grade, cloud-native e-commerce platform powered by 12 microservices, dual Kubernetes clusters, ArgoCD GitOps, GitHub OIDC, Argo Rollouts Blue-Green deployments, and a fully shared CI pipeline.

## Kubernetes Repo and CI workflows : [HipsterShop](https://github.com/Yampss/HipsterShop)
## Documentation of final Project : [Documentation](https://github.com/Yampss/HipsterShop/.github)
---

##  What Makes This Special

| Feature | Detail |
|---|---|
|  **Dual Clusters** | Separate Dev & Prod Kubernetes clusters, each watched by ArgoCD |
|  **Shared CI Workflows** | One workflow library (`hipstershop-shared-workflows`) powers all 12 services |
|  **Blue-Green Deployments** | Frontend uses Argo Rollouts with manual promotion gate |
|  **GitHub OIDC Native CD** | ShippingService deploys to prod with a short-lived JWT — zero stored credentials |
|  **Sealed Secrets** | All cluster secrets encrypted in Git via Bitnami Sealed Secrets |
|  **Dual Registries** | DockerHub for all services; GHCR exclusively for adservice in prod |
|  **Loose Coupling** | Every service is an independent repo; CI is purely parameterised |
|  **MongoDB Replica Set** | 3-node RS with per-service databases and SCRAM-SHA-256 auth |
|  **AI Assistant** | Google Gemini-powered shopping assistant (assistantservice) |
|  **Manual Approval Gate** | Human approval required before any production deployment |

---

##  Repository Layout

The project is split across **14 repositories** in the `HipsterShopp` GitHub organisation:

```
HipsterShopp/
├── hipstershop-shared-workflows      ← Shared CI library (feature/loose-coupling)
├── hipstershop-helm-charts           ← All Helm charts + ArgoCD ApplicationSets
│   ├── hipstershop-argoapps/         ← ArgoCD Application & ApplicationSet manifests
│   ├── hipstershop-infra/            ← MongoDB, kgateway, RBAC, backend-common
│   ├── hipstershop-micro-apps/       ← Per-service Helm charts (values.yaml / .dev / .prod)
│   └── sealed-secrets/               ← Encrypted SealedSecret manifests
│
├── hipstershop-adservice             ← Java 21 / Gradle — GHCR prod registry
├── hipstershop-authservice           ← Go — JWT + MongoDB auth
├── hipstershop-cartservice           ← Cart (Redis-backed)
├── hipstershop-checkoutservice       ← Checkout orchestrator
├── hipstershop-currencyservice       ← Currency converter
├── hipstershop-emailservice          ← Email notifications
├── hipstershop-frontend              ← Node.js — Argo Rollouts Blue-Green
├── hipstershop-paymentservice        ← Node.js — Razorpay + MongoDB persistence
├── hipstershop-productcatalogservice ← Go — MongoDB + local JSON fallback
├── hipstershop-recommendationservice ← Product recommendations
├── hipstershop-shippingservice       ← GitHub OIDC native CD (no ArgoCD in prod)
└── hipstershop-assistantservice      ← Gemini AI shopping assistant
```

---

## 🔄 CI/CD Pipeline — End to End

```
Developer pushes code  (main or develop branch)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│               GitHub Actions CI  (per-service caller)        │
│                                                             │
│  [PR only]  sonar-and-snyk  ──→  quality gate + Snyk scan  │
│                  │                                          │
│                  ▼                                          │
│  generate-tag-and-build-image                               │
│    • develop branch → develop-<short-sha>                   │
│    • main branch    → <service>-v<semver>                   │
│    • Builds Docker image → saves as artifact                │
│    • [PR only] Trivy CRITICAL scan                          │
│                  │                                          │
│                  ▼                                          │
│  push  → downloads artifact → pushes to DockerHub           │
│          [main only] also pushes :latest + creates git tag  │
│                  │                                          │
│      ┌───────────┴────────────┐                             │
│      ▼ (develop)              ▼ (main)                      │
│  update-helm-values-dev    prod-approval-gate               │
│  (writes values.dev.yaml)  (human must approve ✋)          │
│                                 │                           │
│                                 ▼                           │
│                         update-helm-values-prod             │
│                         (writes values.prod.yaml)           │
│                                                             │
│  [adservice only, main]  also push-to-ghcr                  │
│  [shippingservice, main] deploy-to-prod via OIDC kubectl    │
│                                                             │
│  notify (always) → SendGrid email on failure                │
└─────────────────────────────────────────────────────────────┘
         │                         │
         ▼ (develop branch)        ▼ (main branch)
  ArgoCD Dev Cluster          ArgoCD Prod Cluster
  (watches develop)           (watches main)
  values.dev.yaml             values.prod.yaml
  prune: true                 prune: false (safety)
  ServerSideApply: on         ServerSideApply: off
```

---

## 🌐 Dual Cluster Setup

| | Dev Cluster | Prod Cluster |
|---|---|---|
| **ApplicationSet** | `micro-apps-dev` | `micro-apps-prod` |
| **Branch tracked** | `develop` | `main` |
| **Values files** | `values.yaml` + `values.dev.yaml` | `values.yaml` + `values.prod.yaml` |
| **Prune** |  Yes |  No (safety net) |
| **ServerSideApply** |  Yes |  No (Rollouts conflict fix) |
| **Argo Rollouts** |  |  (frontend blue-green) |
| **ShippingService** | Managed by ArgoCD | Native OIDC `kubectl` CD |
| **Services count** | 12 | 11 (shipping excluded from AppSet) |

---



---

## 🔐 GitHub OIDC — ShippingService Native CD

ShippingService **skips ArgoCD entirely in production**. Instead it uses a zero-credential GitHub OIDC flow:

```
GitHub Actions runner → requests short-lived JWT (expires ~5 min)
  sub: "repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main"
         │
         ▼
kube-apiserver validates JWT via --oidc-issuer-url (GitHub OIDC)
         │
         ▼
RBAC RoleBinding matches exact sub → grants deploy/patch on Deployment only
         │
         ▼
kubectl set image deployment/shippingservice ... -n hipster-backend
```

- **No kubeconfig stored.** No long-lived tokens. The `develop` branch has a different `sub` → automatically denied.
- `id-token: write` permission on the job is the only requirement on the GitHub side.
- `K8S_API_SERVER` and `K8S_CA_CERT` are the only two secrets needed (non-sensitive cluster endpoint info).

---

## 🔵🟢 Frontend — Argo Rollouts Blue-Green

```
CI pushes new image tag → values.prod.yaml updated → ArgoCD syncs Rollout
    │
    ▼
Argo Rollouts creates GREEN ReplicaSet
    │
    ▼
Green pods pass health checks → attached to frontend-preview (no live traffic)
    │
    ▼
  ⏸️  PAUSED — operator reviews preview URL
    │
    ▼
Manual promotion → frontend-active service flips to GREEN pods
    │
    ▼
Blue RS kept alive 60s (rollback window) → then scaled to 0
```

`ignoreDifferences` in the prod ApplicationSet suppresses ArgoCD drift caused by Rollouts mutating Services and Rollout replicas in-cluster.

---

## 🏛️ Infrastructure

### Namespaces

| Namespace | Purpose |
|---|---|
| `hipster-frontend` | Frontend Rollout, HPA, active/preview Services |
| `hipster-backend` | All 11 backend services + ConfigMaps + Secrets |
| `hipster-database` | MongoDB 3-node StatefulSet |
| `argocd` | ArgoCD server + ApplicationSets |
| `kube-system` | Sealed Secrets controller |
| `kgateway-system` | KGateway (Kubernetes Gateway API) ingress controller |

### ArgoCD Sync Wave Order

| Wave | Resources |
|---|---|
| 0 | `mongodb` StatefulSet + `backend-common` (ConfigMaps, Secrets, NetworkPolicy) |
| 2 | `gateway` (kgateway + HTTPRoutes) |
| 3 | All microservices (via ApplicationSet) |
| 4 | `loadgenerator` (disabled by default) |

### MongoDB — 3-Node Replica Set

- 3 pods in `hipster-database` namespace via a **headless Service** (`ClusterIP: None`)
- Stable DNS: `mongodb-{0,1,2}.mongo-headless.hipster-database.svc.cluster.local:27017`
- SCRAM-SHA-256 authentication; each service has its **own database + credentials**
- `prune: false` on the ArgoCD app — database is never auto-deleted
- authservice has an **init-container** that waits for a writable primary before starting

### ConfigMaps (backend-common)

5 ConfigMaps in `hipster-backend`:
- `hipster-config` — tracing, Gemini model, feature flags
- `service-addresses` — all inter-service addresses
- `service-ports` — all port numbers
- `mongodb-config` — replica set name, DB names, collection names
- `app-features` — feature on/off flags (profiler, random errors, DB seed)

---

## 🔒 Secret Management — Sealed Secrets

All secrets are encrypted with **Bitnami Sealed Secrets** using the cluster's public key:

```
plaintext secret → kubeseal --cert cluster.pem → SealedSecret (safe to Git commit)
  └── decrypted only by the cluster that holds the matching private key
```

Secrets sealed: `app-secrets` (JWT + Gemini keys), `mongodb-root`, `mongodb-users` (per-service credentials) — across both `hipster-backend` and `hipster-database` namespaces.

> Currently the SealedSecret ArgoCD Application is commented out (plain Helm-rendered secrets used for demo). A `generate-sealed-secrets.sh` script automates re-sealing for production hardening.

---

## 🛠️ Shared Workflow Library

All CI logic lives in `hipstershop-shared-workflows` on branch `feature/loose-coupling`. Each service repo contains only a thin **caller file** that passes parameters:

| Shared Workflow | Role |
|---|---|
| `sonar-and-snyk.yaml` | SonarQube quality gate + Snyk dependency scan (PR only) |
| `generate-tag-and-build-image-trivy.yaml` | Semantic/branch tag, Docker build, Trivy CRITICAL scan |
| `push.yaml` | Push versioned tag + `:latest` (main) to DockerHub; create git tag |
| `helm-updater-job.yaml` | Checkout helm-charts repo, update `tag:` in values file, commit+push |
| `notify.yaml` | SendGrid failure email with full job status matrix |

> Changing CI behaviour for **all services at once** = editing one shared workflow file.

---

