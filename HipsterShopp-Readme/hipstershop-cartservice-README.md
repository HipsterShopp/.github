# hipstershop-cartservice

> .NET 8 / ASP.NET Core microservice managing the HipsterShop shopping cart, backed by MongoDB.

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
| **Language/Runtime** | .NET 8 / C# |
| **Framework** | ASP.NET Core |
| **Port** | 7070 |
| **Database** | MongoDB (`cart_db`) |
| **Docker Image** | `yampss/hipstershop-cartservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |
| **Project File** | `cartservice.csproj` |

### What This Service Does

CartService manages the shopping cart for each user session. It stores cart items (product ID + quantity) to MongoDB `cart_db`. The service is an ASP.NET Core REST API with controllers for adding items to cart, getting the current cart, and emptying the cart.

The cartservice source contains `cartstore/` (MongoDB storage layer), `controllers/` (REST endpoints), and `models/` (data models).

### What This Service Does NOT Do

CartService does not process payments or place orders. It does not calculate pricing (that is currencyservice + checkout). It does not manage product catalog.

---

## Architecture Position

```
  [kGateway]
      │  PathPrefix /api/cart → ReplacePrefixMatch /cart
      ▼
  [cartservice :7070]
      │
  [MongoDB cart_db :27017]
```

**Receives traffic from:**
| Source | Path | Method | Why |
|--------|------|--------|-----|
| kGateway | `/api/cart/*` → `/cart/*` | POST/GET/DELETE | Frontend cart operations |
| assistantservice | via gateway → `/api/cart/add` | POST | AI assistant adding items |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/cart/{userId}` | Get user's cart |
| POST | `/cart/add` | Add item to cart |
| DELETE | `/cart/{userId}` | Empty cart |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- .NET 8 SDK
- MongoDB running locally

```bash
git clone https://github.com/HipsterShopp/hipstershop-cartservice
cd hipstershop-cartservice/src
```

### Environment Setup

```env
MONGO_URI=mongodb://cart_user:cart_password@localhost:27017/cart_db?authSource=cart_db
ASPNETCORE_HTTP_PORTS=7070
```

### Run Locally

```bash
cd src/
dotnet restore cartservice.csproj
dotnet run
```

### Run with Docker

```bash
docker build -t hipstershop-cartservice:local src/
docker run -p 7070:7070 \
  -e MONGO_URI=mongodb://localhost:27017 \
  hipstershop-cartservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder
WORKDIR /app
COPY cartservice.csproj .
RUN dotnet restore cartservice.csproj
COPY . .
RUN dotnet publish cartservice.csproj \
    -c release \
    -o /cartservice
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=builder /cartservice .
EXPOSE 7070
ENV DOTNET_EnableDiagnostics=0 \
    ASPNETCORE_HTTP_PORTS=7070
USER 1000
ENTRYPOINT ["dotnet", "cartservice.dll"]
```

**Build Strategy:** Multi-stage. SDK stage compiles and publishes; runtime stage uses `aspnet:8.0` (ASP.NET Core runtime only, no SDK).

**Important Decisions:**
- `USER 1000`: The only service in the platform that runs as a non-root user at the Dockerfile level.
- `DOTNET_EnableDiagnostics=0`: Disables .NET diagnostic IPC sockets — cleaner for containers.
- `ASPNETCORE_HTTP_PORTS=7070`: Sets the HTTP listen port via env (overrides default 5000/5001).
- Port 7070: Non-standard, chosen to avoid conflicts with other services (frontend at 8080, authservice at 8081).
- `-c release`: Compiles in Release mode (optimized, no debug symbols).

---

## CI/CD Pipeline

**File:** `.github/workflows/cartservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

### Shared Workflow Calls

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: cartservice
    service_path: src/
    runtime: dotnet
    dotnet-version: "8.0"
```

Snyk command for .NET:
```bash
dotnet restore
snyk test --file=cartservice.csproj --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 7070  
**Resources:** Defined in `hipstershop-micro-apps/cartservice/values.yaml`

### Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cartservice
  namespace: hipster-backend
spec:
  type: ClusterIP
  ports:
  - port: 7070
    targetPort: 7070
```

---

## ArgoCD GitOps

**ArgoCD Application:** `cartservice` (main), `cartservice-dev` (develop)  
**Managed by ApplicationSet:** `micro-apps-appset.yaml`  
**Source Path:** `hipstershop-micro-apps/cartservice`

### GitOps Flow

1. PR merged to `develop`
2. Build: `yampss/hipstershop-cartservice:develop-{sha8}`
3. `values.dev.yaml` updated with new tag
4. ArgoCD syncs from `develop` branch
5. Rolling update in `hipster-backend`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `MONGO_URI` | Yes | MongoDB connection string | Helm template (from secrets + configmap) |
| `MONGO_DATABASE` | No | Database name (default: cart_db) | ConfigMap `mongodb-config` |
| `ASPNETCORE_HTTP_PORTS` | Yes | Listen port (7070) | Dockerfile ENV |
| `DOTNET_EnableDiagnostics` | No | Diagnostics (0=disabled) | Dockerfile ENV |

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `mongodb-users` | `CART_MONGO_USERNAME` | MongoDB login (cart_user) |
| `mongodb-users` | `CART_MONGO_PASSWORD` | MongoDB password |

---

## Troubleshooting

**Issue: Service unreachable at port 7070**
```bash
kubectl get pods -n hipster-backend -l app=cartservice
kubectl logs -l app=cartservice -n hipster-backend --tail=50
# Check if running on correct port
kubectl exec <pod-name> -n hipster-backend -- env | grep ASPNETCORE
```

**Issue: MongoDB connection error**
```bash
kubectl logs -l app=cartservice -n hipster-backend --previous
# Look for: MongoDB connection timeout or authentication error
kubectl get secret mongodb-users -n hipster-backend
```

**Issue: Cart items not persisting**
```bash
# Check MongoDB cart_db is reachable
kubectl get endpoints cartservice -n hipster-backend
kubectl port-forward svc/cartservice 7070:7070 -n hipster-backend
curl http://localhost:7070/_healthz
```

**⚠️ Note:** cartservice image runs as `USER 1000`. If debugging with exec, some tools may be restricted. The dotnet/aspnet:8.0 image does have a shell unlike distroless images.

```bash
kubectl exec -it <pod-name> -n hipster-backend -- /bin/bash
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=cartservice
kubectl logs -l app=cartservice -n hipster-backend --tail=100
kubectl port-forward svc/cartservice 7070:7070 -n hipster-backend
curl http://localhost:7070/_healthz
```
