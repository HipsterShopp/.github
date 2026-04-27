# hipstershop-authservice

> Go microservice providing JWT-based user authentication (signup, login, logout, me) backed by MongoDB.

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
| **Framework** | `gorilla/mux` router + `logrus` (JSON logging) |
| **Port** | 8081 |
| **Database** | MongoDB (`auth_db`) |
| **Docker Image** | `yampss/hipstershop-authservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

AuthService handles user identity: it registers new users (signup), validates credentials and issues JWTs (login), invalidates sessions (logout), and returns the current user's profile (me). It connects to MongoDB replica set `rs0` and stores user documents in the `auth_db` database. JSON-formatted logs are written to stdout using logrus with RFC3339 timestamps.

### What This Service Does NOT Do

AuthService does not send emails. It does not handle authorization beyond JWT issuance. It does not process payments or manage cart state.

---

## Architecture Position

```
  [kGateway]
      │
      ├── POST /api/auth/login   → URLRewrite: /login   → [authservice :8081]
      ├── POST /api/auth/signup  → URLRewrite: /signup  → [authservice :8081]
      ├── POST /api/auth/logout  → URLRewrite: /logout  → [authservice :8081]
      └── GET  /api/auth/me      → URLRewrite: /me      → [authservice :8081]
                                                               │
                                                     [MongoDB :27017 auth_db]
```

**Receives traffic from:**
| Source | Method | Why |
|--------|--------|-----|
| kGateway | POST `/api/auth/login` → `/login` | User login |
| kGateway | POST `/api/auth/signup` → `/signup` | User registration |
| kGateway | POST `/api/auth/logout` → `/logout` | User logout |
| kGateway | GET `/api/auth/me` → `/me` | Fetch current user profile |

**Sends traffic to:**
| Destination | Purpose |
|------------|---------|
| MongoDB `auth_db` (replica set rs0) | Store/retrieve user documents |

---

## API Reference

### Endpoints

| Method | Path | Description | Auth Required |
|--------|------|-------------|--------------|
| POST | `/signup` | Register new user | No |
| POST | `/login` | Authenticate, returns JWT | No |
| POST | `/logout` | Invalidate session | Yes (JWT) |
| GET | `/me` | Get current user profile | Yes (JWT) |
| GET | `/_healthz` | Health check | No |

### Internal Code Structure

From `src/main.go`:
```go
const port = "8081"

func main() {
    mongoURI := mustEnv("MONGO_URI")    // crashes if not set
    jwtSecret := mustEnv("JWT_SECRET")  // crashes if not set
    dbName := envOrDefault("MONGO_DATABASE", "auth_db")

    // 15s connection timeout to MongoDB
    client, err := mongo.Connect(ctx, options.Client().ApplyURI(mongoURI))
    
    r := mux.NewRouter()
    r.HandleFunc("/_healthz", ...).Methods(http.MethodGet)
    r.HandleFunc("/signup", h.signup).Methods(http.MethodPost)
    r.HandleFunc("/login", h.login).Methods(http.MethodPost)
    r.HandleFunc("/logout", h.logout).Methods(http.MethodPost)
    r.HandleFunc("/me", h.me).Methods(http.MethodGet)
}
```

`mustEnv()` calls `log.Fatalf()` if the variable is not set — the service will crash on startup if `MONGO_URI` or `JWT_SECRET` are missing.

---

## Local Development

### Prerequisites

- Go 1.26+
- MongoDB instance (or use Docker)

```bash
git clone https://github.com/HipsterShopp/hipstershop-authservice
cd hipstershop-authservice/src
go mod download
```

### Environment Setup

```env
MONGO_URI=mongodb://admin:mongopassword@localhost:27017/auth_db?authSource=admin
JWT_SECRET=team4hipstershopsecret
MONGO_DATABASE=auth_db
PORT=8081
```

### Run Locally

```bash
cd src/
MONGO_URI=... JWT_SECRET=... go run .
```

### Run with Docker

```bash
docker build -t hipstershop-authservice:local src/
docker run -p 8081:8081 \
  -e MONGO_URI=mongodb://admin:mongopassword@localhost:27017/auth_db?authSource=admin \
  -e JWT_SECRET=team4hipstershopsecret \
  hipstershop-authservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM golang:1.26.1-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /authservice .

FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=builder /authservice /authservice
EXPOSE 8081
ENTRYPOINT ["/authservice"]
```

**Build Strategy:** Multi-stage. Builder stage uses Alpine Go; runtime uses `gcr.io/distroless/static-debian12` — no OS shell, no package manager, only CA certificates and timezone data.

**Important Decisions:**
- `CGO_ENABLED=0`: Required for distroless. Produces fully static binary with no C library dependencies.
- `GOOS=linux`: Explicit cross-compilation target.
- `gcr.io/distroless/static-debian12`: Smallest safe base image for Go. No CVEs from OS packages. No shell available (cannot `kubectl exec -- /bin/sh`).

---

## CI/CD Pipeline

**File:** `.github/workflows/authservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

### Shared Workflow Calls

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: authservice
    service_path: src/
    runtime: go
    go-version: "1.22"
```

The shared Snyk workflow runs:
```bash
go mod download
snyk test --file=go.mod --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Helm chart:** `hipstershop-helm-charts/hipstershop-micro-apps/authservice/`

**Key resources:**
- `Deployment/authservice` — runs the Go binary
- `Service/authservice` — ClusterIP on port 8081
- HTTPRoutes in `gateway/templates/routes.yaml` — 4 auth routes

---

## ArgoCD GitOps

**ArgoCD Application:** `authservice` (main), `authservice-dev` (develop)  
**Managed by ApplicationSet:** `micro-apps-appset.yaml` / `micro-apps-appset-dev.yaml`  
**Source Path:** `hipstershop-micro-apps/authservice`

---

## Configuration

### Environment Variables

| Variable | Required | Default | Description | Set Via |
|----------|----------|---------|-------------|---------|
| `MONGO_URI` | **Yes** | None | Full MongoDB replica set URI | Built in Helm template from secret + configmap |
| `JWT_SECRET` | **Yes** | None | JWT signing secret | Secret `app-secrets` key `JWT_SECRET` |
| `MONGO_DATABASE` | No | `auth_db` | MongoDB database name | ConfigMap `mongodb-config` key `AUTH_DB_NAME` |
| `PORT` | No | `8081` | HTTP listen port | ConfigMap `service-ports` key `AUTHSERVICE_PORT` |

**MongoDB URI format (from Helm template):**
```
mongodb://$(MONGO_USERNAME):$(MONGO_PASSWORD)@mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-1.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-2.mongo-headless.hipster-database.svc.cluster.local:27017/auth_db?replicaSet=rs0&authSource=auth_db&authMechanism=SCRAM-SHA-256
```

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `app-secrets` | `JWT_SECRET` | Sign and verify JWTs |
| `mongodb-users` | `AUTH_MONGO_USERNAME` | MongoDB auth_user |
| `mongodb-users` | `AUTH_MONGO_PASSWORD` | MongoDB auth_password |

---

## Troubleshooting

**Issue: Service crashes immediately at startup**
```bash
kubectl logs -l app=authservice -n hipster-backend --previous
# Look for: "Required environment variable MONGO_URI is not set"
#       or: "Required environment variable JWT_SECRET is not set"
# These cause log.Fatalf() => immediate process exit

# Fix: Verify secrets are decrypted
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
```

**Issue: MongoDB connection fails**
```bash
# Check MongoDB pods are Running
kubectl get pods -n hipster-database -l app=mongodb

# Check if all 3 replicas are Ready
kubectl get statefulset mongodb -n hipster-database

# Test connectivity from authservice pod (distroless has no shell)
# Instead, test from another non-distroless pod:
kubectl run test-pod --image=curlimages/curl --rm -it -n hipster-backend -- \
  sh -c "nc -zv mongodb-0.mongo-headless.hipster-database.svc.cluster.local 27017"
```

**Issue: JWT tokens rejected by other services**
```bash
# Verify JWT_SECRET matches across services
kubectl get secret app-secrets -n hipster-backend -o jsonpath='{.data.JWT_SECRET}' | base64 -d
# Should match the sealed secret value
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=authservice
kubectl logs -l app=authservice -n hipster-backend --tail=100
kubectl port-forward svc/authservice 8081:8081 -n hipster-backend
curl http://localhost:8081/_healthz
curl -X POST http://localhost:8081/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test1234"}'
```
