# hipstershop-frontend

> Go microservice serving the HipsterShop HTML frontend — templates, static assets, and the product/cart/checkout UI orchestration layer.

---

## Table of Contents
- [Service Overview](#service-overview)
- [Architecture Position](#architecture-position)
- [Web Routes](#web-routes)
- [Local Development](#local-development)
- [Docker](#docker)
- [CI/CD Pipeline](#cicd-pipeline)
- [Kubernetes Deployment](#kubernetes-deployment)
- [ArgoCD GitOps](#argocd-gitops)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Service Overview

| Field | Value |
|-------|-------|
| **Language/Runtime** | Go 1.26.1 |
| **Framework** | Standard `net/http` + Go HTML templates |
| **Port** | 8080 (container), 80 (Service) |
| **Database** | None (serves HTML, calls backend services) |
| **Docker Image** | `yampss/hipstershop-frontend:{tag}` (dev) |
| **Namespace** | `hipster-frontend` |
| **Rollout Type** | Blue-Green (Argo Rollouts in prod), Deployment (dev) |

### What This Service Does

Frontend is the user-facing web server. It serves HTML pages rendered from Go templates (`templates/` directory) and static assets (CSS, JS, images from `static/` directory). It orchestrates calls to backend REST APIs — product catalog, cart, recommendations, auth — to render dynamic pages. It also handles the AI assistant widget (sending chat messages to assistantservice).

The frontend renders pages and coordinates backend calls, but does not execute business logic (pricing, cart operations) itself.

### What This Service Does NOT Do

Frontend does not call MongoDB directly. It does not process payments. It does not have its own persistence. It does not validate JWT tokens — that is authservice's job.

---

## Architecture Position

```
  [External User]
       │ HTTP
       ▼
  [kGateway :80]
       │  Routes: /, /product/*, /cart/*, /login, /signup, /logout, /setCurrency,
       │          /static/*, /_healthz → frontend service :80 (hipster-frontend namespace)
       ▼
  [frontend :8080]
       │
       ├── → productcatalogservice:3550      (product listing, detail pages)
       ├── → cartservice:7070               (cart state)
       ├── → recommendationservice:8080     (recommended products)
       ├── → currencyservice:7000           (price conversion)
       ├── → checkoutservice:5050           (place order)
       ├── → authservice:8081               (check user session)
       └── → adservice:9555                (advertisements)
```

**Receives traffic from:** External users via kGateway

**Sends traffic to:** All 7 backend services via their ClusterIP DNS names

---

## Web Routes

| Method | Path | Renders |
|--------|------|---------|
| GET | `/` | Home page (product listing) |
| GET | `/product/{id}` | Product detail page |
| GET | `/cart` | Cart page |
| POST | `/cart` | Add to cart (redirect) |
| GET | `/checkout` | Checkout form |
| POST | `/checkout` | Place order |
| GET | `/login` | Login page |
| POST | `/login` | Submit login → authservice |
| GET | `/signup` | Register page |
| POST | `/signup` | Submit registration → authservice |
| POST | `/logout` | Log out |
| POST | `/setCurrency` | Change display currency |
| GET | `/_healthz` | Health check (returns "ok") |

---

## Local Development

### Prerequisites

- Go 1.26+

```bash
git clone https://github.com/HipsterShopp/hipstershop-frontend
cd hipstershop-frontend/src
go mod download
```

### Environment Setup

Frontend reads addresses from environment variables:

```env
PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
CART_SERVICE_ADDR=cartservice:7070
CURRENCY_SERVICE_ADDR=currencyservice:7000
CHECKOUT_SERVICE_ADDR=checkoutservice:5050
RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080
AUTH_SERVICE_ADDR=authservice:8081
AD_SERVICE_ADDR=adservice:9555
ASSISTANT_GATEWAY_ADDR=hipstershop-gateway.hipster-backend.svc.cluster.local:80
JWT_SECRET=team4hipstershopsecret
ENABLE_TRACING=1
COLLECTOR_SERVICE_ADDR=jaeger:4317
FRONTEND_MESSAGE=Checkout HipsterShop today!
ENABLE_ASSISTANT=true
```

### Run Locally

```bash
cd src/
go run .
# Access at http://localhost:8080
```

### Run with Docker

```bash
docker build -t hipstershop-frontend:local src/
docker run -p 8080:8080 \
  -e PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
  hipstershop-frontend:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM golang:1.26.1-alpine@sha256:2389ebfa5b7f43eeafbd6be0c3700cc46690ef842ad962f6c5bd6be49ed82039 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /go/bin/frontend .
FROM gcr.io/distroless/static
WORKDIR /src
COPY --from=builder /go/bin/frontend /src/server
COPY ./templates ./templates
COPY ./static ./static
EXPOSE 8080
ENTRYPOINT ["/src/server"]
```

**Key difference from pure Go services:** Frontend copies `templates/` and `static/` directories into the distroless image. The Go server reads these at runtime — they must be present in the container filesystem.

**`CGO_ENABLED=0` + distroless:** No shell access via exec. Use port-forward for debugging.

---

## CI/CD Pipeline

**File:** `.github/workflows/frontend.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

Sonar uses `golang` as the `runtime` input. Snyk scans `go.mod` for dependencies.

---

## Kubernetes Deployment

**Namespace:** `hipster-frontend`  
**Port:** Container 8080, Service 80

### Blue-Green Deployment (Production)

The frontend uses Argo Rollouts for **Blue-Green deployment** in production:

```yaml
global:
  blueGreen:
    autoPromotionEnabled: false   # Manual promotion required
    scaleDownDelaySeconds: 60     # Keep old version 60s after promotion
```

**Blue-Green services:**
- `frontend-active` — receives live traffic
- `frontend-preview` — receives traffic to preview before promotion

**ignoreDifferences in prod ArgoCD app:**
```yaml
ignoreDifferences:
  - group: ""
    kind: Service
    name: frontend-active
    namespace: hipster-frontend
    jsonPointers: [/spec/selector]
  - group: ""
    kind: Service
    name: frontend-preview
    namespace: hipster-frontend
    jsonPointers: [/spec/selector]
```

These `ignoreDifferences` entries prevent ArgoCD from fighting with the Argo Rollout controller over the service selector (which the rollout controller manages during promotion).

### Horizontal Pod Autoscaler (HPA)

Frontend has HPA configuration in `values.yaml`:

```yaml
global:
  hpa:
    frontend:
      minReplicas: 1
      maxReplicas: 5
      cpuUtilization: 70
      memoryUtilization: 80
      scaleDown:
        stabilizationWindowSeconds: 20
        percentValue: 50
        periodSeconds: 15
      scaleUp:
        stabilizationWindowSeconds: 5
        percentValue: 100
        podsValue: 2
        periodSeconds: 15
```

---

## ArgoCD GitOps

**ArgoCD Application:** `frontend` / `frontend-dev`  
**Namespace:** `hipster-frontend` (only service in this namespace)  
**Source Path:** `hipstershop-micro-apps/frontend`

```yaml
# From micro-apps-appset.yaml
- name: frontend
  namespace: hipster-frontend   # Different from rest (hipster-backend)
```

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | ConfigMap `service-addresses` |
| `CART_SERVICE_ADDR` | Yes | `cartservice:7070` | ConfigMap `service-addresses` |
| `CURRENCY_SERVICE_ADDR` | Yes | `currencyservice:7000` | ConfigMap `service-addresses` |
| `JWT_SECRET` | Yes | JWT signing secret | Secret `app-secrets` or frontend values `.global.secrets.jwtSecret` |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |
| `COLLECTOR_SERVICE_ADDR` | No | `jaeger:4317` | ConfigMap `hipster-config` |
| `FRONTEND_MESSAGE` | No | Banner message on homepage | ConfigMap `hipster-config` |
| `ENABLE_ASSISTANT` | No | `"true"` enables AI chat widget | ConfigMap `hipster-config` |

### Gateway Configuration

```yaml
global:
  gateway:
    name: hipstershop-gateway
    port: 80
  networkPolicy:
    vpcCidr: "172.31.0.0/16"
    kgatewayNamespace: "kgateway-system"
```

---

## Troubleshooting

**Issue: Frontend pod in hipster-frontend is not reachable**
```bash
# Check frontend is Running
kubectl get pods -n hipster-frontend -l app=frontend

# Check kGateway routes for frontend
kubectl get httproute frontend-route -n hipster-backend -o yaml

# Check frontend service has endpoints
kubectl get endpoints -n hipster-frontend
```

**Issue: Product page shows "Service unavailable"**
```bash
# Check productcatalogservice
kubectl get pods -n hipster-backend -l app=productcatalogservice
kubectl port-forward svc/productcatalogservice 3550:3550 -n hipster-backend
curl http://localhost:3550/products
```

**Issue: Can't exec into frontend pod**
```bash
# It's distroless — no shell. Use port-forward instead:
kubectl port-forward svc/frontend 8080:80 -n hipster-frontend
curl http://localhost:8080/_healthz
```

**Issue: Blue-green promotion stuck**
```bash
# Check rollout status (prod only)
kubectl get rollout frontend -n hipster-frontend
kubectl describe rollout frontend -n hipster-frontend
# Promote manually:
kubectl argo rollouts promote frontend -n hipster-frontend
```

### General Debug Commands

```bash
kubectl get pods -n hipster-frontend
kubectl logs -l app=frontend -n hipster-frontend --tail=100
kubectl port-forward svc/frontend 8080:80 -n hipster-frontend
curl http://localhost:8080/_healthz
kubectl get rollout -n hipster-frontend 2>/dev/null || echo "No rollouts (dev mode)"
```
