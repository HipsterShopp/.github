# hipstershop-assistantservice

> Python/FastAPI microservice that powers the HipsterShop AI shopping assistant using Google Gemini 2.5 Flash via LangChain/LangGraph.

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
| **Language/Runtime** | Python 3.10 |
| **Framework** | FastAPI served by Uvicorn |
| **Port** | 8080 |
| **Database** | None (stateless) |
| **Docker Image** | `yampss/hipstershop-assistantservice:{tag}` (dev) / `ghcr.io/hipstershopp/hipstershop-assistantservice:{tag}` (prod) |
| **Namespace** | `hipster-backend` |

### What This Service Does

AssistantService is an AI-powered shopping assistant that accepts customer questions about specific products and returns conversational responses. It uses Google Gemini 2.5 Flash via LangChain + LangGraph's `create_react_agent` to create an agentic loop that can:
1. Fetch product details by calling the `productcatalogservice` via the gateway (`/api/products/{product_id}`)
2. Add products to the user's cart by calling `cartservice` via the gateway (`/api/cart/add`)
3. Return natural language responses to the user

The agent uses the user's session cookie (`shop_session-id`) to associate cart operations with the correct user session.

### What This Service Does NOT Do

AssistantService does not store conversation history (each request is stateless). It does not access MongoDB. It does not process payments or access order history.

---

## Architecture Position

```
  [User Browser]
        │  POST /api/assistant/chat
        ▼
  [kGateway]
        │ (no URL rewrite for /api/assistant/chat)
        ▼
  [assistantservice :8080]
        │  tool: get_product_details
        ├──────────────────────► GET http://hipstershop-gateway.hipster-backend.svc.cluster.local:80/api/products/{id}
        │                              └──► [productcatalogservice]
        │  tool: add_to_cart
        └──────────────────────► POST http://hipstershop-gateway.hipster-backend.svc.cluster.local:80/api/cart/add
                                       └──► [cartservice]
```

**Receives traffic from:**
| Source | Method | Why |
|--------|--------|-----|
| kGateway | `POST /api/assistant/chat` | Frontend sends product questions |

**Sends traffic to:**
| Destination | Method | Why |
|------------|--------|-----|
| Gateway (internal) → productcatalogservice | GET | `get_product_details` tool fetches product info |
| Gateway (internal) → cartservice | POST | `add_to_cart` tool adds items |

---

## API Reference

### Endpoints

| Method | Path | Description | Auth Required |
|--------|------|-------------|--------------|
| POST | `/api/assistant/chat` | Chat with AI assistant | No (uses session cookie) |
| GET | `/_healthz` | Health check | No |
| OPTIONS | `/api/assistant/chat` | CORS preflight | No |

### Request/Response: POST /api/assistant/chat

**Request:**
```json
{
  "productId": "OLJCESPC7Z0",
  "message": "What's special about this camera? Can you add 2 to my cart?"
}
```

**Response:**
```json
{
  "reply": "This Vintage Camera is perfect for film photography enthusiasts. I have added 2 to your cart! Is there anything else you need?"
}
```

---

## Local Development

### Prerequisites

- Python 3.10+
- `pip`

```bash
git clone https://github.com/HipsterShopp/hipstershop-assistantservice
cd hipstershop-assistantservice/src
pip install -r requirements.txt
```

### Environment Setup

```env
GATEWAY_ADDR=localhost:8080
GEMINI_API_KEY=your-gemini-api-key-here
```

### Run Locally

```bash
cd src/
GEMINI_API_KEY=your-key GATEWAY_ADDR=localhost:8080 \
  uvicorn main:app --host 0.0.0.0 --port 8080
```

### Run with Docker

```bash
docker build -t hipstershop-assistantservice:local src/
docker run -p 8080:8080 \
  -e GEMINI_API_KEY=your-key \
  -e GATEWAY_ADDR=localhost:8080 \
  hipstershop-assistantservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Build Strategy:** Single stage. Unlike emailservice/recommendationservice which use multi-stage builds, this uses `python:3.10-slim` (Debian slim base). Simple, straightforward — no native compilation needed.

**Base Image:** `python:3.10-slim` — Debian-based slim Python image. `slim` variant excludes unnecessary packages.

**Note:** Uses `CMD` (not `ENTRYPOINT`), which means the command can be overridden at `docker run`. Other Python services use `ENTRYPOINT`.

---

## CI/CD Pipeline

**File:** `.github/workflows/assistantservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

**Required Secrets:** `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GHCR_TOKEN`, `HELM_CHARTS_TOKEN`, `SNYK_TOKEN`, `SONAR_TOKEN`, `SONAR_HOST_URL`, `SENDGRID_API_KEY`

### Shared Workflow Calls

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: assistantservice
    service_path: src/
    runtime: python
    python-version: "3.10"
```

The shared Snyk workflow runs:
```bash
snyk test --file=requirements.txt --package-manager=pip --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Helm chart:** `hipstershop-helm-charts/hipstershop-micro-apps/assistantservice/`

### Configuration

```yaml
global:
  images:
    assistantservice: "yampss/hipstershop-assistantservice:latest"
  replicas:
    assistantservice: 1
  ports:
    assistantservice: 8080
```

### Health Check

The service exposes `GET /_healthz` which returns `"ok"` (200 OK). This is used for both readiness and liveness probes in the Helm deployment template.

---

## ArgoCD GitOps

**ArgoCD Application:** `assistantservice` (main), `assistantservice-dev` (develop)  
**Managed by:** `micro-apps-appset.yaml` / `micro-apps-appset-dev.yaml` ApplicationSets  
**Source Path:** `hipstershop-micro-apps/assistantservice`

### GitOps Flow

1. PR merged to `develop`
2. GitHub Actions builds `yampss/hipstershop-assistantservice:develop-{sha8}`
3. `values.dev.yaml` updated via `helm-updater-job.yaml` shared workflow
4. ArgoCD detects change in `hipstershop-helm-charts:develop`
5. Helm renders: `helm template assistantservice hipstershop-micro-apps/assistantservice -f values.yaml -f values.dev.yaml`
6. Deployment rolls out in `hipster-backend`

---

## Configuration

### Environment Variables

| Variable | Required | Default | Description | Set Via |
|----------|----------|---------|-------------|---------|
| `GATEWAY_ADDR` | Yes | `hipstershop-gateway.hipster.svc.cluster.local:80` | Internal gateway address for tool API calls | ConfigMap `service-addresses` |
| `GEMINI_API_KEY` | Yes | None | Google Gemini API key for LLM access | Secret `app-secrets` key `GEMINI_API_KEY` |

**If `GEMINI_API_KEY` is missing:** Service returns HTTP 503 with `{"detail": "Assistant API key missing"}`.

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `app-secrets` | `GEMINI_API_KEY` | Authenticate with Google Gemini API |

---

## Troubleshooting

**Issue: 503 "Assistant API key missing"**
```bash
# Check if secret is mounted
kubectl exec <pod-name> -n hipster-backend -- env | grep GEMINI
# If empty, check sealed secret is decrypted
kubectl get secret app-secrets -n hipster-backend -o jsonpath='{.data.GEMINI_API_KEY}' | base64 -d
```

**Issue: 502 "Agent request failed"**
```bash
# Check if gateway is reachable from assistantservice pod
kubectl exec <pod-name> -n hipster-backend -- \
  curl http://hipstershop-gateway.hipster-backend.svc.cluster.local:80/api/products/OLJCESPC7Z0

# Check pod logs for Python traceback
kubectl logs -l app=assistantservice -n hipster-backend --tail=50
```

**Issue: Cart tool not adding items**
```bash
# Verify session cookie is being passed
# The service reads 'shop_session-id' cookie from the HTTP request
# Check cartservice is reachable via gateway
kubectl port-forward svc/cartservice 7070:7070 -n hipster-backend
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=assistantservice
kubectl logs -l app=assistantservice -n hipster-backend --tail=100
kubectl port-forward svc/assistantservice 8080:8080 -n hipster-backend
curl -X POST http://localhost:8080/api/assistant/chat \
  -H "Content-Type: application/json" \
  -d '{"productId":"OLJCESPC7Z0","message":"Tell me about this"}'
```
