# hipstershop-recommendationservice

> Python 3.11 microservice that returns product recommendations based on the currently viewed product.

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
| **Language/Runtime** | Python 3.11 |
| **Framework** | Custom HTTP server |
| **Port** | 8080 |
| **Database** | MongoDB (`analytics_db`) — optional for persisting recommendation events |
| **Docker Image** | `yampss/hipstershop-recommendationservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

RecommendationService returns a list of recommended product IDs when given a product context. The frontend calls it on product detail pages to show "you might also like" items. The service may call `productcatalogservice` to fetch available product IDs and filter/rank them. Recommendations may be logged to MongoDB `analytics_db.recommendations`.

### What This Service Does NOT Do

RecommendationService does not use a machine learning model for recommendations (rule-based or random selection). It does not manage user profiles. It does not process orders.

---

## Architecture Position

```
  [kGateway]
      │  GET /api/recommendations → URLRewrite: /recommendations
      ▼
  [recommendationservice :8080]
      │
      ├── → productcatalogservice:3550 (get product catalog for recommendation pool)
      └── Optional: → MongoDB analytics_db (log recommendation events)

  [frontend]
      │  → recommendationservice:8080/recommendations
      ▼
  [recommendationservice :8080]
```

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/recommendations` | Return list of recommended product IDs |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- Python 3.11

```bash
git clone https://github.com/HipsterShopp/hipstershop-recommendationservice
cd hipstershop-recommendationservice/src
pip install -r requirements.txt
```

### Environment Setup

```env
PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
PORT=8080
```

### Run Locally

```bash
cd src/
python recommendation_server.py
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM python:3.11-alpine AS base
FROM base AS builder
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
RUN apk update \
    && apk add --no-cache g++ linux-headers \
    && rm -rf /var/cache/apk/*
COPY requirements.txt .
RUN pip install -r requirements.txt
FROM base
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
RUN apk update \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*
WORKDIR /recommendation_server
COPY --from=builder /usr/local/lib/python3.11/ /usr/local/lib/python3.11/
COPY --from=builder /usr/local/bin/ /usr/local/bin/
COPY . .
EXPOSE 8080
ENTRYPOINT [ "python", "recommendation_server.py" ]
```

**Identical pattern to emailservice** — multi-stage Python Alpine build. Builder with native compilation tools, runtime with only `libstdc++`. Python libraries copied from builder stage.

---

## CI/CD Pipeline

**File:** `.github/workflows/recommendationservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

```yaml
sonar-and-snyk:
  with:
    service_name: recommendationservice
    service_path: src/
    runtime: python
    python-version: "3.11"
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 8080

Same resource profile as standard backend services (100m CPU, 64Mi memory requests).

---

## ArgoCD GitOps

**ArgoCD Application:** `recommendationservice` / `recommendationservice-dev`  
**Source Path:** `hipstershop-micro-apps/recommendationservice`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `PORT` | No | `8080` | ConfigMap `service-ports` |
| `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | ConfigMap `service-addresses` |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |
| `COLLECTOR_SERVICE_ADDR` | No | `jaeger:4317` | ConfigMap `hipster-config` |

### Secrets Used (Optional)

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `mongodb-users` | `ANALYTICS_MONGO_USERNAME` | MongoDB analytics_db (optional) |
| `mongodb-users` | `ANALYTICS_MONGO_PASSWORD` | MongoDB analytics_db (optional) |

---

## Troubleshooting

```bash
kubectl get pods -n hipster-backend -l app=recommendationservice
kubectl logs -l app=recommendationservice -n hipster-backend --tail=100
kubectl port-forward svc/recommendationservice 8080:8080 -n hipster-backend
curl http://localhost:8080/_healthz
curl "http://localhost:8080/recommendations?productId=OLJCESPC7Z0"
```
