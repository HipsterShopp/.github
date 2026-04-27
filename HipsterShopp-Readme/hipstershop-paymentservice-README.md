# hipstershop-paymentservice

> Node.js/Express microservice that processes credit card charges and persists transaction records to MongoDB.

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
| **Language/Runtime** | Node.js 20 / Express |
| **Framework** | Express.js REST API |
| **Port** | 50051 |
| **Database** | MongoDB (`payment_db`, collection `charges`) |
| **Docker Image** | `yampss/hipstershop-paymentservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

PaymentService validates and processes credit card transactions. The core logic in `charge.js` validates card number, expiry date, and CVV, then returns a transaction ID without actually charging a real card (simulated). After each charge attempt, it writes a record to MongoDB `payment_db.charges` with:
- `transactionId` (indexed)
- `createdAt` (indexed)
- `status` (success/failed)
- `cardMeta.last4Masked` (e.g., `****4242`)
- `amount` (currencyCode, units, nanos)

If MongoDB is unavailable at startup, the service continues running without persistence (graceful degradation).

### What This Service Does NOT Do

PaymentService does not integrate with a real payment processor (Stripe, PayPal, etc.) — card processing is simulated. It does not handle refunds. It does not authenticate requests (auth is frontend/gateway responsibility).

---

## Architecture Position

```
  [checkoutservice]
       │  POST /charge (via paymentservice:50051)
       ▼
  [paymentservice :50051]
       │
       ├── charge.js (validate card, generate transactionId)
       └── MongoDB payment_db.charges (persist event)

  [kGateway]
       │  POST /api/payment/charge → URLRewrite: /charge
       ▼
  [paymentservice :50051]
```

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| checkoutservice | `/charge` via `paymentservice:50051` | Process payment during checkout |
| kGateway | `/api/payment/charge` → `/charge` | Direct API access |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| POST | `/charge` | Process a credit card charge |
| GET | `/_healthz` | Health check |

### Request: POST /charge

```json
{
  "amount": {
    "currencyCode": "USD",
    "units": 99,
    "nanos": 990000000
  },
  "creditCard": {
    "creditCardNumber": "4432-8015-6152-0454",
    "creditCardCvv": 672,
    "creditCardExpirationYear": 2029,
    "creditCardExpirationMonth": 1
  }
}
```

### Response: 200 OK

```json
{
  "transactionId": "some-uuid-transaction-id"
}
```

### Response: 400 Bad Request

```json
{
  "error": "Credit card validation failed: invalid expiration date"
}
```

---

## Local Development

### Prerequisites

- Node.js 20
- MongoDB (optional — service runs without it)

```bash
git clone https://github.com/HipsterShopp/hipstershop-paymentservice
cd hipstershop-paymentservice/src
npm install --only=production
```

### Environment Setup

```env
PORT=50051
PAYMENT_MONGO_URI=mongodb://payment_user:payment_password@localhost:27017/payment_db?authSource=payment_db
MONGO_DATABASE=payment_db
MONGO_PAYMENTS_COLLECTION=charges
DISABLE_PROFILER=1
```

### Run Locally

```bash
cd src/
PAYMENT_MONGO_URI=... node index.js
```

### Run with Docker

```bash
docker build -t hipstershop-paymentservice:local src/
docker run -p 50051:50051 \
  -e PORT=50051 \
  hipstershop-paymentservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM node:20.20.1-alpine AS builder
RUN apk add --update --no-cache \
    python3 \
    make \
    g++
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --only=production

FROM alpine:3.23.3
RUN apk add --no-cache nodejs
WORKDIR /usr/src/app
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY . .
EXPOSE 50051
ENTRYPOINT [ "node", "index.js" ]
```

**Same pattern as currencyservice** — multi-stage build with bare `alpine` + `nodejs`, not the full `node` image.

Port 50051 matches the gRPC convention (though this service uses REST/Express, not gRPC).

---

## CI/CD Pipeline

**File:** `.github/workflows/paymentservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: paymentservice
    service_path: src/
    runtime: node
    node-version: "20"
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 50051

### MongoDB Connection Configuration

The MongoDB URI is constructed directly in the Helm template:

```yaml
# From hipstershop-micro-apps/paymentservice/templates/deployment.yaml
- name: MONGO_URI
  value: "mongodb://$(MONGO_USERNAME):$(MONGO_PASSWORD)@mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-1.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-2.mongo-headless.hipster-database.svc.cluster.local:27017/payment_db?replicaSet=rs0&authSource=payment_db&authMechanism=SCRAM-SHA-256"
```

Where `$(MONGO_USERNAME)` and `$(MONGO_PASSWORD)` reference other env vars from secrets (Kubernetes variable substitution).

### Full Deployment Environment

| Env Var | Source |
|---------|--------|
| `PORT` | ConfigMap `service-ports` → `PAYMENTSERVICE_PORT` |
| `DISABLE_PROFILER` | ConfigMap `app-features` → `PAYMENT_SERVICE_DISABLE_PROFILER` |
| `MONGO_USERNAME` | Secret `mongodb-users` → `PAYMENT_MONGO_USERNAME` |
| `MONGO_PASSWORD` | Secret `mongodb-users` → `PAYMENT_MONGO_PASSWORD` |
| `MONGO_DATABASE` | ConfigMap `mongodb-config` → `PAYMENT_DB_NAME` |
| `MONGO_PAYMENTS_COLLECTION` | ConfigMap `mongodb-config` → `PAYMENT_CHARGES_COLLECTION` |
| `MONGO_URI` | Helm template string (builds from above) |

---

## ArgoCD GitOps

**ArgoCD Application:** `paymentservice` / `paymentservice-dev`  
**Source Path:** `hipstershop-micro-apps/paymentservice`

---

## Configuration

### Environment Variables

| Variable | Required | Default | Description | Set Via |
|----------|----------|---------|-------------|---------|
| `PORT` | No | `50051` | HTTP listen port | ConfigMap `service-ports` |
| `DISABLE_PROFILER` | No | `"1"` | Disable GCP profiler | ConfigMap `app-features` |
| `MONGO_URI` | No | None | MongoDB URI (persists charges) | Helm template |
| `MONGO_DATABASE` | No | `payment_db` | Target database | ConfigMap |
| `MONGO_PAYMENTS_COLLECTION` | No | `charges` | Collection name | ConfigMap |

**If `MONGO_URI` is not set:** Service starts normally but logs: `Payment Mongo persistence disabled.`

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `mongodb-users` | `PAYMENT_MONGO_USERNAME` | MongoDB `payment_user` |
| `mongodb-users` | `PAYMENT_MONGO_PASSWORD` | MongoDB payment password |

---

## Troubleshooting

**Issue: Payment succeeds but no record in MongoDB**
```bash
# Check if MongoDB connection was established at startup
kubectl logs -l app=paymentservice -n hipster-backend | head -30
# Look for: "Payment Mongo persistence enabled on payment_db.charges"
# If missing: "Payment Mongo persistence disabled." or "Mongo initialization failed"

# Verify MongoDB pods are healthy
kubectl get pods -n hipster-database
```

**Issue: Payment charge always returns 400**
```bash
kubectl port-forward svc/paymentservice 50051:50051 -n hipster-backend
curl -X POST http://localhost:50051/charge \
  -H "Content-Type: application/json" \
  -d '{"amount":{"currencyCode":"USD","units":10,"nanos":0},"creditCard":{"creditCardNumber":"4432-8015-6152-0454","creditCardCvv":672,"creditCardExpirationYear":2029,"creditCardExpirationMonth":1}}'

# Check charge.js validation logic
kubectl logs -l app=paymentservice -n hipster-backend --tail=20
```

**Issue: Card number masked incorrectly in logs**
```bash
# maskCard() function: keeps last 4 digits, replaces rest with ****
# This is expected behavior: "4432-8015-6152-0454" → "****0454"
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=paymentservice
kubectl logs -l app=paymentservice -n hipster-backend --tail=100
kubectl port-forward svc/paymentservice 50051:50051 -n hipster-backend
curl http://localhost:50051/_healthz
```
