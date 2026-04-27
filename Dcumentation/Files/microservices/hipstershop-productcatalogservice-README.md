# hipstershop-productcatalogservice

> Go microservice providing the product catalog REST API — listing all products, fetching individual products, and seeding the MongoDB database from a static `products.json` file.

---

## Table of Contents
- [Service Overview](#service-overview)
- [Architecture Position](#architecture-position)
- [API Reference](#api-reference)
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
| **Framework** | Standard `net/http` |
| **Port** | 3550 |
| **Database** | MongoDB (`catalog_db`, collection `products`) |
| **Docker Image** | `yampss/hipstershop-productcatalogservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

ProductCatalogService is the product database layer. It reads from MongoDB `catalog_db` to:
1. Return a list of all products (`/products`)
2. Return a single product by ID (`/products/{id}`)

On startup, if `PRODUCT_CATALOG_SEED_DATABASE=true`, it seeds MongoDB from the embedded `products.json` file (containing the initial product catalog). The `products.json` is baked into the Docker image at build time and is embedded in the Go binary.

### What This Service Does NOT Do

ProductCatalogService does not manage inventory (stock counts). It does not handle pricing calculations or currency conversion. It does not process orders or manage the cart.

---

## Architecture Position

```
  [kGateway]
      ├── GET /api/products     → URLRewrite: /products       → [productcatalogservice :3550]
      └── GET /api/products/{id} → URLRewrite: /products/{id} → [productcatalogservice :3550]

  [checkoutservice]  → productcatalogservice:3550
  [recommendationservice] → productcatalogservice:3550
  [frontend] → productcatalogservice:3550
  [assistantservice] → gateway → /api/products/{id}
                                          │
                                    [MongoDB catalog_db.products]
```

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| kGateway | `/api/products`, `/api/products/{id}` | Public product browsing |
| checkoutservice | `productcatalogservice:3550/products/{id}` | Verify product exists at checkout |
| frontend | `productcatalogservice:3550/products` | Product listing pages |
| assistantservice | via gateway | Fetch product details for AI responses |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/products` | Return all products |
| GET | `/products/{id}` | Return product by ID |
| GET | `/_healthz` | Health check |

### Response: GET /products/{id}

```json
{
  "id": "OLJCESPC7Z0",
  "name": "Vintage Camera Lens",
  "description": "You won't find a camera like this anywhere else in the universe.",
  "picture": "/static/img/products/camera-lens.jpg",
  "priceUsd": {
    "currencyCode": "USD",
    "units": 12,
    "nanos": 490000000
  },
  "categories": ["photography", "vintage"]
}
```

---

## Local Development

### Prerequisites

- Go 1.26+
- MongoDB accessible at `localhost:27017`

```bash
git clone https://github.com/HipsterShopp/hipstershop-productcatalogservice
cd hipstershop-productcatalogservice/src
go mod download
```

### Environment Setup

```env
MONGO_URI=mongodb://catalog_user:catalog_password@localhost:27017/catalog_db?authSource=catalog_db
MONGO_DATABASE=catalog_db
CATALOG_PRODUCTS_COLLECTION=products
PRODUCT_CATALOG_SEED_DATABASE=true
PORT=3550
```

### Run Locally

```bash
cd src/
go run .
curl http://localhost:3550/products
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
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /productcatalogservice .
FROM gcr.io/distroless/static
WORKDIR /src
COPY --from=builder /productcatalogservice /src/productcatalogservice
COPY products.json .
EXPOSE 3550
ENTRYPOINT ["/src/productcatalogservice"]
```

**Key difference:** `COPY products.json .` — the static product catalog JSON file is copied into the distroless image. This allows the service to seed MongoDB from this file if configured to do so, without needing an external data source.

---

## CI/CD Pipeline

**File:** `.github/workflows/productcatalogservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: productcatalogservice
    service_path: src/
    runtime: go
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 3550

### Seed Database Flag

The `PRODUCT_CATALOG_SEED_DATABASE` is set via ConfigMap:

```yaml
# From backend-common/templates/configmaps.yaml (app-features ConfigMap)
PRODUCT_CATALOG_SEED_DATABASE: "true"
```

When this is `"true"`, on first startup the service reads `products.json` and inserts all products into MongoDB `catalog_db.products`. On subsequent restarts with data already present, it skips seeding (upsert logic).

---

## ArgoCD GitOps

**ArgoCD Application:** `productcatalogservice` / `productcatalogservice-dev`  
**Source Path:** `hipstershop-micro-apps/productcatalogservice`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `MONGO_URI` | Yes | MongoDB connection string | Helm template (from secrets) |
| `MONGO_DATABASE` | No | `catalog_db` | ConfigMap `mongodb-config` → `CATALOG_DB_NAME` |
| `CATALOG_PRODUCTS_COLLECTION` | No | `products` | ConfigMap `mongodb-config` → `CATALOG_PRODUCTS_COLLECTION` |
| `PRODUCT_CATALOG_SEED_DATABASE` | No | `"true"` | ConfigMap `app-features` |
| `PORT` | No | `3550` | ConfigMap `service-ports` → `PRODUCTCATALOG_PORT` |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `mongodb-users` | `CATALOG_MONGO_USERNAME` | MongoDB catalog_user |
| `mongodb-users` | `CATALOG_MONGO_PASSWORD` | MongoDB catalog password |

---

## Troubleshooting

**Issue: `/products` returns empty list**
```bash
# Check if database was seeded
kubectl logs -l app=productcatalogservice -n hipster-backend | head -20
# Look for: "Seeding database" or "Database already seeded"

# Verify MongoDB catalog_db has data
# Connect to MongoDB:
kubectl exec -it mongodb-0 -n hipster-database -- mongosh \
  --username admin --password mongopassword --authenticationDatabase admin \
  --eval "use catalog_db; db.products.countDocuments()"
```

**Issue: Product not found by ID**
```bash
kubectl port-forward svc/productcatalogservice 3550:3550 -n hipster-backend
curl http://localhost:3550/products/OLJCESPC7Z0
# Check if ID exists in products.json
```

**Issue: MongoDB connection timeout**
```bash
kubectl logs -l app=productcatalogservice -n hipster-backend --previous
kubectl get pods -n hipster-database -l app=mongodb
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=productcatalogservice
kubectl logs -l app=productcatalogservice -n hipster-backend --tail=100
kubectl port-forward svc/productcatalogservice 3550:3550 -n hipster-backend
curl http://localhost:3550/products
curl http://localhost:3550/_healthz
```
