# hipstershop-currencyservice

> Node.js microservice that provides currency listing and conversion, serving real exchange rates to the HipsterShop frontend.

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
| **Language/Runtime** | Node.js 20 |
| **Framework** | Custom HTTP server (`server.js`) |
| **Port** | 7000 |
| **Database** | None (rates embedded in `data/` directory) |
| **Docker Image** | `yampss/hipstershop-currencyservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

CurrencyService provides two operations: returning a list of supported currencies, and converting a monetary amount from one currency to another. Exchange rate data is served from static files in the `data/` subdirectory — no external API calls. This service has an intentional random error mode controlled by the `CURRENCY_SERVICE_RANDOM_ERROR` feature flag (set to `"1"` in the platform's `app-features` ConfigMap), which simulates real-world transient failures for resilience testing.

### What This Service Does NOT Do

CurrencyService does not fetch live exchange rates from an external service. It does not store transaction data. It does not authenticate requests.

---

## Architecture Position

```
  [kGateway]
      ├── GET /api/currency/currencies → URLRewrite: /currencies → [currencyservice :7000]
      └── GET /api/currency/convert   → URLRewrite: /convert    → [currencyservice :7000]
                                                                        │
                                                              [data/ static rate files]
```

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| kGateway | GET `/api/currency/currencies` | Frontend currency dropdown |
| kGateway | GET `/api/currency/convert` | Price conversion during checkout |
| checkoutservice | direct `currencyservice:7000` | Price conversion for order totals |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/currencies` | List all supported currencies |
| GET | `/convert` | Convert amount from one currency to another |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- Node.js 20

```bash
git clone https://github.com/HipsterShopp/hipstershop-currencyservice
cd hipstershop-currencyservice/src
npm install --only=production
```

### Run Locally

```bash
cd src/
node server.js
```

### Run with Docker

```bash
docker build -t hipstershop-currencyservice:local src/
docker run -p 7000:7000 hipstershop-currencyservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM node:20.20.1-alpine@sha256:b88333c42c23fbd91596ebd7fd10de239cedab9617de04142dde7315e3bc0afa AS builder
RUN apk add --update --no-cache \
    python3 \
    make \
    g++
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --only=production

FROM alpine:3.23.3@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659
RUN apk add --no-cache nodejs
WORKDIR /usr/src/app
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY . .
EXPOSE 7000
ENTRYPOINT [ "node", "server.js" ]
```

**Build Strategy:** Multi-stage. Builder installs build tools (python3, make, g++) needed for native Node modules. Runtime uses bare `alpine` + manually installed `nodejs` — not the full node image.

**Why bare alpine + nodejs?** The full `node:20-alpine` image includes npm, npx, yarn, and build tools not needed at runtime. Using `alpine` + `apk add nodejs` produces a leaner image.

**Same Dockerfile pattern used for:** `paymentservice` (port 50051)

---

## CI/CD Pipeline

**File:** `.github/workflows/currencyservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: currencyservice
    service_path: src/
    runtime: node
    node-version: "20"
```

Snyk scan command:
```bash
npm install
snyk test --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 7000  
**Resources:** Standard (100m CPU, 64Mi memory requests; 200m CPU, 128Mi limits)

### Key Config

`CURRENCY_SERVICE_RANDOM_ERROR=1` is set via the `app-features` ConfigMap, enabling intentional random errors for resilience testing.

---

## ArgoCD GitOps

**ArgoCD Application:** `currencyservice` / `currencyservice-dev`  
**Source Path:** `hipstershop-micro-apps/currencyservice`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `CURRENCY_SERVICE_RANDOM_ERROR` | No | `"1"` enables random error injection | ConfigMap `app-features` |
| `ENABLE_TRACING` | No | `"1"` enables OTel tracing | ConfigMap `hipster-config` |
| `COLLECTOR_SERVICE_ADDR` | No | Jaeger collector address | ConfigMap `hipster-config` |

---

## Troubleshooting

**Issue: Random currency conversion errors**
```bash
# This is intentional when CURRENCY_SERVICE_RANDOM_ERROR=1
# Check the feature flag
kubectl get configmap app-features -n hipster-backend -o yaml | grep CURRENCY
# To disable: update backend-common values.yaml and let ArgoCD sync
```

**Issue: Currency service returning 500**
```bash
kubectl logs -l app=currencyservice -n hipster-backend --tail=50
kubectl port-forward svc/currencyservice 7000:7000 -n hipster-backend
curl http://localhost:7000/_healthz
curl http://localhost:7000/currencies
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=currencyservice
kubectl logs -l app=currencyservice -n hipster-backend --tail=100
kubectl port-forward svc/currencyservice 7000:7000 -n hipster-backend
```
