# hipstershop-checkoutservice

> Go microservice that orchestrates the complete checkout flow — coordinating payment, shipping, email, and cart services to complete an order.

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
| **Framework** | Standard `net/http` + `money/` package for currency math |
| **Port** | 5050 |
| **Database** | None directly (uses other services' databases via API calls) |
| **Docker Image** | `yampss/hipstershop-checkoutservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

CheckoutService is the orchestrator of the purchase flow. When a user clicks "Place Order," the frontend calls `/api/checkout` which routes here. The service:
1. Gets the user's cart from cartservice
2. Converts currency via currencyservice
3. Charges the payment method via paymentservice
4. Gets shipping quote from shippingservice
5. Sends order confirmation email via emailservice
6. Empties the cart

The `money/` subdirectory contains custom currency arithmetic operations (addition, multiplication) for handling multi-currency price calculations without floating-point errors.

### What This Service Does NOT Do

CheckoutService does not store orders permanently (checkout-only, no order history). It does not render HTML or serve a UI. It does not handle authentication — the JWT is passed through from the frontend.

---

## Architecture Position

```
  [kGateway]
      │  POST /api/checkout → URLRewrite: /checkout
      ▼
  [checkoutservice :5050]
      │
      ├── GET  → cartservice:7070         (get cart items)
      ├── POST → paymentservice:50051     (charge card)
      ├── GET  → shippingservice:50051    (get shipping quote)
      ├── POST → emailservice:8080        (send confirmation)
      ├── GET  → currencyservice:7000     (currency conversion)
      └── DELETE → cartservice:7070       (empty cart after order)
```

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| kGateway | POST `/api/checkout` → `/checkout` | Frontend submits order |

**Sends traffic to:**
| Service | Operation |
|---------|-----------|
| `cartservice:7070` | Get cart contents, empty cart |
| `paymentservice:50051` | Charge credit card |
| `shippingservice:50051` | Get shipping quote |
| `emailservice:8080` | Send order confirmation |
| `currencyservice:7000` | Convert prices |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| POST | `/checkout` | Process complete checkout |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- Go 1.26+

```bash
git clone https://github.com/HipsterShopp/hipstershop-checkoutservice
cd hipstershop-checkoutservice/src
go mod download
```

### Environment Setup

```env
PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
SHIPPING_SERVICE_ADDR=shippingservice:50051
PAYMENT_SERVICE_ADDR=paymentservice:50051
EMAIL_SERVICE_ADDR=emailservice:8080
CURRENCY_SERVICE_ADDR=currencyservice:7000
CART_SERVICE_ADDR=cartservice:7070
PORT=5050
```

### Run Locally

```bash
cd src/
go run .
# Requires all dependent services accessible at configured addresses
```

### Run with Docker

```bash
docker build -t hipstershop-checkoutservice:local src/
docker run -p 5050:5050 \
  -e CART_SERVICE_ADDR=localhost:7070 \
  -e PAYMENT_SERVICE_ADDR=localhost:50051 \
  -e SHIPPING_SERVICE_ADDR=localhost:50051 \
  -e EMAIL_SERVICE_ADDR=localhost:8080 \
  -e CURRENCY_SERVICE_ADDR=localhost:7000 \
  hipstershop-checkoutservice:local
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
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /checkoutservice .
FROM gcr.io/distroless/static
WORKDIR /src
COPY --from=builder /checkoutservice /src/checkoutservice
EXPOSE 5050
ENTRYPOINT ["/src/checkoutservice"]
```

- Same pattern as other Go services: multi-stage with distroless runtime.
- `-ldflags="-s -w"`: Strips debug symbols — reduces binary size.
- SHA-pinned builder image: `golang:1.26.1-alpine@sha256:2389...` — reproducible builds.
- **No shell in runtime image** — `kubectl exec -- /bin/sh` will fail. Use port-forward + curl for debugging.

---

## CI/CD Pipeline

**File:** `.github/workflows/checkoutservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

Snyk scan command (shared workflow):
```bash
go mod download
snyk test --file=go.mod --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 5050

The CheckoutService deployment (from helm template) includes:
- `envFrom: configMapRef: name: service-addresses` — all downstream service addresses
- `envFrom: configMapRef: name: service-ports` — port numbers

No secrets are mounted directly — all inter-service calls are unauthenticated HTTP to internal cluster DNS.

---

## ArgoCD GitOps

**ArgoCD Application:** `checkoutservice` / `checkoutservice-dev`  
**Source Path:** `hipstershop-micro-apps/checkoutservice`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | ConfigMap `service-addresses` |
| `SHIPPING_SERVICE_ADDR` | Yes | `shippingservice:50051` | ConfigMap `service-addresses` |
| `PAYMENT_SERVICE_ADDR` | Yes | `paymentservice:50051` | ConfigMap `service-addresses` |
| `EMAIL_SERVICE_ADDR` | Yes | `emailservice:8080` | ConfigMap `service-addresses` |
| `CURRENCY_SERVICE_ADDR` | Yes | `currencyservice:7000` | ConfigMap `service-addresses` |
| `CART_SERVICE_ADDR` | Yes | `cartservice:7070` | ConfigMap `service-addresses` |
| `PORT` | No | `5050` | ConfigMap `service-ports` |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |
| `COLLECTOR_SERVICE_ADDR` | No | `jaeger:4317` | ConfigMap `hipster-config` |

---

## Troubleshooting

**Issue: Checkout returns error — any single downstream service unavailable**

CheckoutService calls all downstream services synchronously. If any service is down, checkout fails.

```bash
# Test each downstream service health:
kubectl port-forward svc/cartservice 7070:7070 -n hipster-backend &
kubectl port-forward svc/paymentservice 50051:50051 -n hipster-backend &
kubectl port-forward svc/shippingservice 50051:50051 -n hipster-backend &
kubectl port-forward svc/emailservice 8080:8080 -n hipster-backend &
kubectl port-forward svc/currencyservice 7000:7000 -n hipster-backend &
```

**Issue: Currency conversion errors**
```bash
# Check currencyservice is running
kubectl get pods -n hipster-backend -l app=currencyservice
# Check CURRENCY_SERVICE_RANDOM_ERROR flag
kubectl get configmap app-features -n hipster-backend -o yaml
# CURRENCY_SERVICE_RANDOM_ERROR: "1" simulates random errors (intentional)
```

**Issue: Email not sent after checkout**
```bash
# EmailService may fail silently (checkout still succeeds even if email fails)
kubectl logs -l app=emailservice -n hipster-backend --tail=50
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=checkoutservice
kubectl logs -l app=checkoutservice -n hipster-backend --tail=100
kubectl port-forward svc/checkoutservice 5050:5050 -n hipster-backend
curl http://localhost:5050/_healthz

# Check all service addresses are configured
kubectl get configmap service-addresses -n hipster-backend -o yaml
```
