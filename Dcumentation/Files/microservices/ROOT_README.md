# HipsterShop Platform — Complete DevOps Engineering Documentation

> **Organization:** HipsterShopp  
> **Repositories:** 14  
> **Status:** Production  
> **Last Updated:** April 2026

---

## Table of Contents

1. [Platform Overview](#section-1--platform-overview)
2. [Phase 1: Dockerization](#section-2--phase-1-dockerization-story)
3. [Phase 2: Kubernetes Setup & Orchestration](#section-3--phase-2-kubernetes-setup--orchestration)
4. [Phase 3: Infrastructure Components](#section-4--phase-3-infrastructure-components)
5. [Phase 4: CI/CD Implementation](#section-5--phase-4-cicd-implementation)
6. [Phase 5: Monorepo to Multi-Repo Transition](#section-6--phase-5-monorepo-to-multi-repo-transition)
7. [Phase 6: GitOps with ArgoCD](#section-7--phase-6-gitops-with-argocd)
8. [Phase 7: Helm Integration](#section-8--phase-7-helm-integration)
9. [Complete End-to-End Flow](#section-9--complete-end-to-end-flow)
10. [Troubleshooting Guide](#section-10--troubleshooting-guide)
11. [Command Cheat Sheet](#section-11--command-cheat-sheet)
12. [Best Practices](#section-12--best-practices)
13. [Future Improvements](#section-13--future-improvements)

---

## Section 1 — Platform Overview

HipsterShop is a cloud-native e-commerce demonstration platform composed of **12 independently deployable microservices** spanning Go, Java, Python, Node.js, and .NET — all running on Kubernetes, managed via ArgoCD GitOps, deployed through GitHub Actions CI/CD, and versioned with Helm charts. Users browse products, authenticate, manage carts, place orders, process payments, receive email confirmations, and interact with an AI shopping assistant powered by Google Gemini.

### Architecture Diagram

```
                        ┌─────────────────────────────────────────────────────────────┐
                        │                  EXTERNAL USERS                             │
                        └───────────────────────┬─────────────────────────────────────┘
                                                │ HTTP :80
                        ┌───────────────────────▼─────────────────────────────────────┐
                        │          kGateway (Kubernetes Gateway API)                    │
                        │     GatewayClass: kgateway  |  Gateway: hipstershop-gateway   │
                        │            Namespace: hipster-backend                         │
                        └───┬───────┬───────┬───────┬───────┬───────┬───────┬──────────┘
                            │       │       │       │       │       │       │
          ┌─────────────────┼───────┼───────┼───────┼───────┼───────┼───────┼──────────────────┐
          │  hipster-frontend  │       │       │       │       │       │       │  hipster-backend  │
          │  ┌─────────────┐  │       │       │       │       │       │       │  ┌─────────────┐  │
          │  │  frontend   │◄─┘       │       │       │       │       │       │  │  authservice │  │◄─/api/auth/*
          │  │  :8080      │          │       │       │       │       │       │  │  :8081  Go   │  │
          │  └─────────────┘          │       │       │       │       │       │  └─────────────┘  │
          └────────┬──────────────────┼───────┼───────┼───────┼───────┼───────────────────────────┘
                   │                  │       │       │       │       │
                   │          ┌───────▼──┐ ┌──▼────┐ ┌──▼───┐ ┌──▼──────────┐ ┌──▼─────────────────┐
                   │          │cartservice│ │ product│ │check-│ │paymentservice│ │  emailservice       │
                   │          │ :7070 .NET│ │catalog │ │out   │ │ :50051 Node.j│ │  :8080 Python       │
                   │          └─────┬────┘ │:3550 Go│ │:5050 │ └──────┬───────┘ └────────────────────┘
                   │                │      └────────┘ │  Go  │         │
                   │                │                 └──────┘         │
                   │                └────────────────────────────────────────────────────────────────┐
                   │                                                                                  │
                   │    ┌─────────────────────────────────────────────────────────────────────────┐  │
                   │    │                        hipster-database                                  │  │
                   │    │         MongoDB StatefulSet (3 replicas, ReplicaSet: rs0)               │  │
                   │    │         PVC: nfs storage class  |  5Gi per replica                     │  │
                   │    └─────────────────────────────────────────────────────────────────────────┘  │
                   └───────────────────────────────────────────────────────────────────────────────────┘

   Additional services (hipster-backend):
   - adservice        :9555  Java (Gradle)   — serves personalized advertisements
   - shippingservice  :50051 Go              — calculates shipping quotes
   - currencyservice  :7000  Node.js         — converts currencies
   - recommendationservice :8080 Python      — product recommendations
   - assistantservice :8080  Python/FastAPI  — AI shopping assistant (Gemini 2.5 Flash)

CI/CD Flow:
   Developer Push → GitHub Actions → Docker Build → DockerHub / GHCR
        → Helm values.yaml updated → ArgoCD detects git diff → Kubernetes Rollout
```

### Technology Stack

| Layer | Technology | Purpose | Version |
|-------|-----------|---------|---------|
| Container Runtime | Docker | Build & ship microservice images | Buildx via GitHub Actions |
| Container Registry (dev) | Docker Hub (yampss/) | Store dev images | - |
| Container Registry (prod) | GitHub Container Registry (ghcr.io/hipstershopp/) | Store prod images | - |
| Orchestration | Kubernetes | Run containers at scale | Cluster with 3-node MongoDB StatefulSet |
| API Gateway | kGateway (Kubernetes Gateway API) | Route HTTP traffic to services | kgateway controller |
| Package Manager | Helm | Template & version K8s manifests | v3.x (managed by ArgoCD) |
| GitOps | ArgoCD | Sync git state to cluster | ApplicationSet-driven |
| Secret Encryption | Sealed Secrets | Encrypt K8s secrets for git | bitnami-labs/sealed-secrets |
| CI/CD | GitHub Actions | Build, test, scan, deploy | Shared reusable workflows |
| Code Quality | SonarQube | Static code analysis | Self-hosted |
| Dependency Scanning | Snyk | Dependency vulnerability scan | CLI via Actions |
| Container Scanning | Trivy | CVE scanning of images | aquasecurity/trivy-action |
| Email Notifications | SendGrid | CI failure alerts | licenseware/send-email-notification |
| Database | MongoDB 7.0 | Persistent data for all services | Replica set rs0 |
| Persistent Storage | NFS StorageClass | MongoDB data volumes | StorageClass name: `nfs` |
| AI/ML | Google Gemini 2.5 Flash | Shopping assistant LLM | langchain-google-genai |
| Tracing | Jaeger | Distributed tracing | Collector at jaeger:4317 |
| Language (Ad Service) | Java 25 / Gradle | Ad serving | eclipse-temurin:25 JRE |
| Language (Auth) | Go 1.26.1 | JWT auth + MongoDB | gcr.io/distroless/static |
| Language (Cart) | .NET 8 / ASP.NET Core | Cart management | mcr.microsoft.com/dotnet/aspnet:8.0 |
| Language (Checkout) | Go 1.26.1 | Orchestrates checkout flow | gcr.io/distroless/static |
| Language (Currency) | Node.js 20 | Currency conversion | alpine:3.23.3 |
| Language (Email) | Python 3.11 | SMTP / email notifications | python:3.11-alpine |
| Language (Frontend) | Go 1.26.1 | HTML frontend server | gcr.io/distroless/static |
| Language (Payment) | Node.js 20 / Express | Credit card processing | alpine:3.23.3 |
| Language (ProductCatalog)| Go 1.26.1 | Product catalogue REST | gcr.io/distroless/static |
| Language (Recommendation)| Python 3.11 | Product recommendations | python:3.11-alpine |
| Language (Shipping) | Go 1.26.1 | Shipping quote calculator | gcr.io/distroless/static |
| Language (Assistant) | Python 3.10 / FastAPI | Gemini-powered AI assistant | uvicorn |

### Repository Map

| Repository | Type | Purpose | Connected To |
|-----------|------|---------|-------------|
| `hipstershop-adservice` | Microservice (Java) | Serves targeted ads | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-assistantservice` | Microservice (Python) | AI chat assistant using Gemini | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-authservice` | Microservice (Go) | JWT-based authentication | MongoDB, hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-cartservice` | Microservice (.NET) | Shopping cart management | MongoDB, hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-checkoutservice` | Microservice (Go) | Orchestrates checkout | payment, shipping, email, cart, hipstershop-helm-charts |
| `hipstershop-currencyservice` | Microservice (Node.js) | Currency conversion | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-emailservice` | Microservice (Python) | Order confirmation emails | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-frontend` | Microservice (Go) | Web UI serving all pages | All backend services, hipstershop-helm-charts |
| `hipstershop-paymentservice` | Microservice (Node.js) | Credit card charge processing | MongoDB, hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-productcatalogservice` | Microservice (Go) | Product catalog REST API | MongoDB, hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-recommendationservice` | Microservice (Python) | Product recommendations | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-shippingservice` | Microservice (Go) | Shipping quotes (OIDC deploy) | hipstershop-helm-charts, hipstershop-shared-workflows |
| `hipstershop-shared-workflows` | Reusable CI/CD | Centralized GitHub Actions workflows | All 12 microservice repos |
| `hipstershop-helm-charts` | GitOps / Config | All Helm charts + ArgoCD Applications | ArgoCD in cluster |

### Quick-Start: Deploy This Platform From Scratch

**Prerequisites:**
- Kubernetes cluster (kubeadm or cloud) with `kubectl` context configured
- Helm v3 installed
- ArgoCD installed in `argocd` namespace
- NFS StorageClass named `nfs` provisioned
- kGateway controller installed (`kgateway` GatewayClass)
- Sealed Secrets controller in `kube-system`
- `kubeseal` CLI installed

```bash
# Step 1: Create namespaces
kubectl create namespace hipster-backend
kubectl create namespace hipster-frontend
kubectl create namespace hipster-database

# Step 2: Generate and apply sealed secrets (run from sealed-secrets/ folder)
cd hipstershop-helm-charts/sealed-secrets
export JWT_SECRET="your-jwt-secret"
export GEMINI_API_KEY="your-gemini-key"
export MONGO_ROOT_PASSWORD="your-mongo-root-pw"
# ... (set all DB passwords)
bash generate-sealed-secrets.sh
git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: add sealed secrets" && git push

# Step 3: Apply ArgoCD resources in wave order
kubectl apply -f hipstershop-argoapps/project.yaml
kubectl apply -f hipstershop-argoapps/infra/mongodb.yaml
kubectl apply -f hipstershop-argoapps/infra/backend-common.yaml
# Wait for mongodb and backend-common to be Synced+Healthy
kubectl apply -f hipstershop-argoapps/infra/sealed-secrets.yaml
# Wait for secrets to be created
kubectl apply -f hipstershop-argoapps/infra/gateway.yaml
kubectl apply -f hipstershop-argoapps/apps/micro-apps-appset.yaml

# Step 4: Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
# Password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## Section 2 — Phase 1: Dockerization (Story)

### 2.1 Why Docker Was Introduced

The team began with 12 microservices spanning 5 different languages and runtimes — Java with Gradle, Go, Node.js, Python, and .NET 8. Running them locally worked on individual machines, but deploying them consistently to a shared environment was impossible without Docker. A Go 1.22 binary compiled on one machine would require the exact same Go runtime on the target server; a Java service needed a specific JDK; a .NET service needed the ASP.NET runtime. Docker solved this by packaging each service with its exact runtime inside an immutable image.

The secondary reason was security: `gcr.io/distroless` base images were chosen for Go services specifically to eliminate the OS-level attack surface — no shell, no package manager, no system utilities.

### 2.2 Dockerfile Analysis

---

#### Service: adservice
**File:** `hipstershop-adservice/src/Dockerfile`  
**Base Image (builder):** `eclipse-temurin:24.0.2_12-jdk-noble` — Specific SHA-pinned JDK for Gradle compilation  
**Base Image (runtime):** `eclipse-temurin:25.0.2_10-jre-alpine` — Minimal JRE only (no JDK) for runtime

```dockerfile
FROM eclipse-temurin:24.0.2_12-jdk-noble@sha256:dacac8e9a0df0d2bd24e702b4431132875c249930b70555ebd7ca285b5bee684 AS builder
WORKDIR /app
COPY ["build.gradle", "gradlew", "./"]
COPY gradle gradle
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew
RUN ./gradlew downloadRepos
COPY . .
RUN chmod +x gradlew
RUN ./gradlew installDist
FROM eclipse-temurin:25.0.2_10-jre-alpine@sha256:f10d6259d0798c1e12179b6bf3b63cea0d6843f7b09c9f9c9c422c50e44379ec
WORKDIR /app
COPY --from=builder /app .
RUN wget -O /opentelemetry-javaagent.jar https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/opentelemetry-javaagent.jar"
EXPOSE 9555
ENTRYPOINT ["/app/build/install/hipstershop/bin/AdService"]
```

**Line-by-line explanation:**
- Lines 1–9: Builder stage — uses full JDK to compile the Gradle project. `sed -i 's/\r$//'` fixes Windows CRLF line endings in `gradlew` before making it executable.
- Line 6: `downloadRepos` downloads Gradle dependencies before copying source, enabling Docker layer caching — dependencies are re-downloaded only if `build.gradle` changes.
- Lines 10–12: Runtime stage — switches to JRE-only Alpine image, copies compiled distribution.
- Line 13: Downloads OpenTelemetry Java agent for distributed tracing.
- Line 14: Configures JVM to load the OTel agent on startup (connects to Jaeger at `COLLECTOR_SERVICE_ADDR`).
- Line 15: Exposes port 9555 (Java gRPC/HTTP ad server port).

---

#### Service: authservice
**File:** `hipstershop-authservice/src/Dockerfile`  
**Base Image (builder):** `golang:1.26.1-alpine` — Alpine Go for compilation  
**Base Image (runtime):** `gcr.io/distroless/static-debian12` — No OS at all; binary only

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

- `CGO_ENABLED=0`: Produces a fully static binary with no C library dependencies — required for distroless to work.
- `gcr.io/distroless/static-debian12`: Contains only CA certificates and timezone data. No shell, no `/bin/sh`, significantly reduces CVE surface.

---

#### Service: cartservice
**File:** `hipstershop-cartservice/src/Dockerfile`  
**Base Image (builder):** `mcr.microsoft.com/dotnet/sdk:8.0`  
**Base Image (runtime):** `mcr.microsoft.com/dotnet/aspnet:8.0`

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

- `USER 1000`: Runs as non-root. The only service in the platform that sets this in the Dockerfile.
- `DOTNET_EnableDiagnostics=0`: Disables .NET diagnostic ports (cleaner container, fewer open sockets).
- Port 7070 is non-standard for ASP.NET (default 5000/5001) — used to avoid port conflicts with other services.

---

#### Service: checkoutservice
**File:** `hipstershop-checkoutservice/src/Dockerfile`

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

- `-ldflags="-s -w"`: Strips debug symbols and DWARF info, reducing binary size significantly.
- SHA-pinned base image ensures reproducible builds regardless of upstream tag changes.

---

#### Service: currencyservice
**File:** `hipstershop-currencyservice/src/Dockerfile`

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

- Builder stage installs build tools (`python3`, `make`, `g++`) needed for native npm modules.
- Runtime stage uses a minimal `alpine` + manually installed `nodejs` (not the full `node` image), keeping the image smaller and avoiding npm/yarn in production.
- Same Dockerfile structure is used for `paymentservice` (port 50051 instead of 7000).

---

#### Service: emailservice
**File:** `hipstershop-emailservice/src/Dockerfile`

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
ENV ENABLE_PROFILER=1
RUN apk update \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*
WORKDIR /email_server
COPY --from=builder /usr/local/lib/python3.11/ /usr/local/lib/python3.11/
COPY --from=builder /usr/local/bin/ /usr/local/bin/
COPY . .
EXPOSE 8080
ENTRYPOINT [ "python", "email_server.py" ]
```

- Multi-stage Python build: builder installs `g++` and `linux-headers` to compile native Python packages. Runtime stage only needs `libstdc++`.
- `PYTHONDONTWRITEBYTECODE=1`: Prevents `.pyc` bytecode files (cleaner container).
- `PYTHONUNBUFFERED=1`: Ensures Python stdout/stderr goes directly to the container logs.
- Same exact pattern used for `recommendationservice`.

---

#### Service: frontend
**File:** `hipstershop-frontend/src/Dockerfile`

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

- Copies `templates/` and `static/` into the distroless image because the Go frontend serves HTML templates and static assets directly.

---

### 2.3 Image Tagging Strategy

Two tagging strategies are used depending on branch:

**On `develop` branch:** `{branch-slug}-{short-sha}` (e.g., `develop-3f2a1c9b`)  
**On `main` branch:** Semantic version using conventional commits (e.g., `adservice-v0.1.2`)

Extracted from `generate-tag-and-build-image-trivy.yaml`:
```bash
# develop branch
BRANCH_SLUG=$(echo "${{ github.ref_name }}" | sed 's|/|-|g')
SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-8)
COMPUTED_TAG="${BRANCH_SLUG}-${SHORT_SHA}"

# main branch — uses mathieudutour/github-tag-action for semver
tag_prefix: "${{ inputs.service_name }}-v"
```

### 2.4 DockerHub / Registry Integration

- **Dev images:** Pushed to `yampss/hipstershop-{service-name}:{tag}` (Docker Hub)
- **Prod images:** Cross-pushed to `ghcr.io/hipstershopp/hipstershop-{service-name}:{tag}` (GHCR)

Secrets used:
- `DOCKERHUB_USERNAME` — Docker Hub account username
- `DOCKERHUB_TOKEN` — Docker Hub access token
- `GHCR_TOKEN` — GitHub PAT with `write:packages` scope

```bash
# Login (from push.yaml)
echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin

# Push versioned tag
docker push "${{ secrets.DOCKERHUB_USERNAME }}/hipstershop-${SERVICE_NAME}:${TAG}"

# On main: also push :latest
docker push "${{ secrets.DOCKERHUB_USERNAME }}/hipstershop-${SERVICE_NAME}:latest"
```

### 2.5 Common Docker Issues & Fixes

**Issue:** Gradle `gradlew: not found` or `Permission denied`  
**When:** First run of adservice build on Linux CI runner  
**Root cause:** `gradlew` committed with Windows line endings; Linux cannot execute CRLF scripts  
**Fix:** `RUN sed -i 's/\r$//' gradlew && chmod +x gradlew`  
**Prevention:** `.gitattributes` added to force LF for shell scripts

**Issue:** `exec /authservice: no such file or directory` in distroless image  
**Root cause:** CGO was enabled, producing a dynamically linked binary; distroless has no `libc`  
**Fix:** `CGO_ENABLED=0` in Go build command  
**Prevention:** All Go services now use `CGO_ENABLED=0` in their Dockerfiles

---

## Section 3 — Phase 2: Kubernetes Setup & Orchestration

### 3.1 Why Kubernetes

Once Docker images were built and working, running them via `docker run` or `docker-compose` exposed three concrete problems: (1) no automated restart on crash, (2) no rolling updates — every deploy required taking the service down, (3) no horizontal scaling. Kubernetes was introduced to provide declarative deployment management, rolling update strategies, health-based scheduling, and service discovery across all 12 microservices.

### 3.2 Namespace Design

The platform uses 3 core namespaces, created by Helm charts and managed with `helm.sh/resource-policy: keep` annotation so ArgoCD never deletes them:

| Namespace | Purpose | Services Running In It |
|-----------|---------|----------------------|
| `hipster-backend` | All microservices | adservice, authservice, cartservice, checkoutservice, currencyservice, emailservice, paymentservice, productcatalogservice, recommendationservice, shippingservice, assistantservice + kGateway Gateway object |
| `hipster-frontend` | Frontend only | frontend (Rollout in prod, Deployment in dev) |
| `hipster-database` | Persistence | MongoDB StatefulSet (3 replicas) |
| `argocd` | GitOps control plane | ArgoCD server, repo-server, application-controller |
| `kube-system` | Platform infra | Sealed Secrets controller, CNI |

Namespaces are isolated by network policy (`default-deny-all`) with explicit allow rules between namespaces.

### 3.3 Deployment Manifests

All deployment manifests are Helm templates. The adservice deployment is representative:

```yaml
# hipstershop-helm-charts/hipstershop-micro-apps/adservice/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.global.namespaces.backend }}    # hipster-backend
  name: adservice
  labels:
    app: adservice
spec:
  replicas: {{ .Values.global.replicas.adservice }}     # 1 (dev), 1 (prod)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: {{ .Values.global.rollingUpdate.maxUnavailable }}  # 1
      maxSurge: {{ .Values.global.rollingUpdate.maxSurge }}              # 1
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        tier: backend
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: {{ .Values.global.images.adservice }}
        ports:
        - containerPort: {{ .Values.global.ports.adservice }}            # 9555
        envFrom:
        - configMapRef:
            name: hipster-config                    # ENABLE_TRACING, COLLECTOR_SERVICE_ADDR, etc.
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: service-ports
              key: ADSERVICE_PORT                   # "9555"
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 512Mi
        readinessProbe:
          initialDelaySeconds: 60                   # Java takes ~45-60s to start
          periodSeconds: 15
          httpGet:
            path: "/_healthz"
            port: 9555
        livenessProbe:
          initialDelaySeconds: 60
          periodSeconds: 15
          httpGet:
            path: "/_healthz"
            port: 9555
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      volumes:
      - name: tmp-volume
        emptyDir: {}
```

**Key decisions:**
- `replicas: 1` — Single replica for most services. MongoDB handles data redundancy with 3-replica StatefulSet.
- `initialDelaySeconds: 60` for adservice — Java JVM startup with OpenTelemetry agent takes ~50-60 seconds.
- `terminationGracePeriodSeconds: 5` — Fast drain for rolling updates.
- `tmp-volume` emptyDir — Provides writable `/tmp` for services that need temporary scratch space.

### 3.4 MongoDB StatefulSet

MongoDB runs as a 3-replica StatefulSet with NFS-backed persistent storage:

```yaml
# hipstershop-helm-charts/hipstershop-infra/mongodb/templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: hipster-database
  name: mongodb
spec:
  serviceName: mongo-headless
  replicas: 3           # ReplicaSet rs0 with 3 members
  template:
    spec:
      containers:
      - name: mongodb
        image: mongo:7.0
        command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-root
              key: MONGO_ROOT_USERNAME
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        readinessProbe:
          exec:
            command: ["mongosh", "--eval", "db.adminCommand('ping')"]
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          exec:
            command: ["mongosh", "--eval", "db.adminCommand('ping')"]
          initialDelaySeconds: 30
          periodSeconds: 20
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"    # NFS StorageClass provides persistent volumes
      resources:
        requests:
          storage: 5Gi
```

### 3.5 Service Manifests

All services use `ClusterIP` type — internal cluster DNS only. Traffic enters exclusively through the kGateway:

```yaml
# adservice service (representative example)
apiVersion: v1
kind: Service
metadata:
  namespace: hipster-backend
  name: adservice
spec:
  type: ClusterIP           # Internal only; gateway routes to it
  selector:
    app: adservice
  ports:
  - name: http
    port: 9555
    targetPort: 9555
```

### 3.6 Inter-Service Communication

Services communicate via DNS service names within the cluster. ConfigMaps in `hipster-backend` provide service addresses:

```yaml
# From backend-common configmaps.yaml (actual content):
PRODUCT_CATALOG_SERVICE_ADDR: "productcatalogservice:3550"
SHIPPING_SERVICE_ADDR:        "shippingservice:50051"
PAYMENT_SERVICE_ADDR:         "paymentservice:50051"
EMAIL_SERVICE_ADDR:           "emailservice:8080"
CURRENCY_SERVICE_ADDR:        "currencyservice:7000"
CART_SERVICE_ADDR:            "cartservice:7070"
GATEWAY_ADDR:                 "hipstershop-gateway.hipster-backend.svc.cluster.local:80"
```

MongoDB connection string (from paymentservice deployment template):
```
mongodb://$(MONGO_USERNAME):$(MONGO_PASSWORD)@mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-1.mongo-headless.hipster-database.svc.cluster.local:27017,mongodb-2.mongo-headless.hipster-database.svc.cluster.local:27017/payment_db?replicaSet=rs0&authSource=payment_db&authMechanism=SCRAM-SHA-256
```

### 3.7 Key kubectl Commands

```bash
# View all pods across platform namespaces
kubectl get pods -n hipster-backend
kubectl get pods -n hipster-frontend
kubectl get pods -n hipster-database
kubectl get pods -n argocd

# View deployments
kubectl get deployments -n hipster-backend

# Check a failing pod
kubectl describe pod <pod-name> -n hipster-backend

# Stream logs
kubectl logs -f <pod-name> -n hipster-backend

# Logs from crashed container
kubectl logs <pod-name> -n hipster-backend --previous

# Exec into a running pod (most images are distroless — exec may not work)
kubectl exec -it <pod-name> -n hipster-backend -- /bin/sh

# Port-forward to test a service locally
kubectl port-forward svc/adservice 9555:9555 -n hipster-backend

# Check MongoDB pods
kubectl get pods -n hipster-database -l app=mongodb

# Check rollout status after a deploy
kubectl rollout status deployment/adservice -n hipster-backend
```

---

## Section 4 — Phase 3: Infrastructure Components

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### COMPONENT: kGateway / Kubernetes Gateway API
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#### 4.1 Why an API Gateway

Without a gateway, every service would need a LoadBalancer service (one public IP per service) to receive external traffic — expensive and operationally complex. The kGateway (Kubernetes Gateway API implementation by kgateway) provides a single entry point that routes based on HTTP paths to the correct ClusterIP service inside the cluster. It also eliminates the need for Kubernetes Ingress resources.

#### 4.2 Gateway Configuration

```yaml
# hipstershop-helm-charts/hipstershop-infra/gateway/templates/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kgateway
spec:
  controllerName: kgateway.dev/kgateway
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  namespace: hipster-backend
  name: hipstershop-gateway
spec:
  gatewayClassName: kgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All       # Accepts routes from all namespaces
```

#### 4.3 HTTPRoute Configuration (from routes.yaml)

The gateway defines 11 HTTPRoute objects (one per microservice). All routes use `URLRewrite` filters to translate external API paths to internal service paths:

| External Path | Rewrites To | Backend Service | Port |
|--------------|------------|----------------|------|
| `/api/ads` | `/ads` | adservice | 9555 |
| `/api/auth/login` | `/login` | authservice | 8081 |
| `/api/auth/signup` | `/signup` | authservice | 8081 |
| `/api/auth/logout` | `/logout` | authservice | 8081 |
| `/api/auth/me` | `/me` | authservice | 8081 |
| `/api/cart/*` | `/cart/*` | cartservice | 7070 |
| `/api/checkout` | `/checkout` | checkoutservice | 5050 |
| `/api/currency/currencies` | `/currencies` | currencyservice | 7000 |
| `/api/currency/convert` | `/convert` | currencyservice | 7000 |
| `/api/email/send-confirmation` | `/send-confirmation` | emailservice | 8080 |
| `/api/payment/charge` | `/charge` | paymentservice | 50051 |
| `/api/products/*` | `/products/*` | productcatalogservice | 3550 |
| `/api/recommendations` | `/recommendations` | recommendationservice | 8080 |
| `/api/shipping/*` | `/` | shippingservice | 50051 |
| `/api/assistant/chat` | (no rewrite) | assistantservice | 8080 |
| `/, /product, /cart, /login, /signup, /logout, /static/*, /_healthz, /setCurrency` | (direct) | frontend | 80 |

#### 4.4 Traffic Flow

```
1. HTTP request arrives at Gateway LoadBalancer IP on port 80
2. kgateway matches request path against HTTPRoute rules
3. URLRewrite filter transforms /api/cart/add → /cart/add
4. Gateway forwards to ClusterIP service:port inside hipster-backend
5. Kubernetes kube-proxy routes to pod IP based on Service selector
6. Pod processes request and responds
7. Response travels back through Gateway to client
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### COMPONENT: NFS Persistent Storage
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#### 4.5 Why Persistent Storage Was Needed

MongoDB requires persistent data volumes. Even in a development environment, losing the database on pod restart is unacceptable. NFS was chosen over local `hostPath` because StatefulSet pods with `hostPath` can only be scheduled on the node that owns the data — NFS allows pods to be rescheduled to any node while retaining data.

#### 4.6 Kubernetes Integration

MongoDB StatefulSet uses `volumeClaimTemplates` to automatically create a PVC for each replica:

```yaml
volumeClaimTemplates:
- metadata:
    name: mongo-data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "nfs"     # Must be pre-provisioned in the cluster
    resources:
      requests:
        storage: 5Gi
```

This creates PVCs:
- `mongo-data-mongodb-0` → 5Gi
- `mongo-data-mongodb-1` → 5Gi  
- `mongo-data-mongodb-2` → 5Gi

Each MongoDB replica mounts its own PVC at `/data/db`.

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### COMPONENT: Network Policies
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#### 4.7 Why Network Policies

Without network policies, any pod in the cluster can talk to any other pod. This means a compromised frontend pod could directly query the MongoDB database. Network policies enforce microsegmentation.

#### 4.8 Actual Policies

```yaml
# Policy 1: Default deny all in hipster-backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: hipster-backend
spec:
  podSelector: {}         # Applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress

---

# Policy 2: Allow controlled traffic in hipster-backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow
  namespace: hipster-backend
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - ipBlock:
        cidr: 172.31.0.0/16       # VPC CIDR — allows traffic from within VPC
  - from:
    - namespaceSelector:
        matchLabels:
          name: hipster-frontend  # Allow frontend → backend
  - from:
    - podSelector: {}             # Allow backend pods to talk to each other
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: hipster-database  # Backend → MongoDB
  - to:
    - podSelector: {}             # Intra-backend pod communication
  - to:
    - namespaceSelector:
        matchLabels:
          name: hipster-frontend  # Backend → frontend (for gateway routing)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kgateway-system  # Backend → kGateway
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: TCP
      port: 443                   # kube-apiserver (for health checks)
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53                    # DNS
    - protocol: TCP
      port: 53                    # DNS TCP fallback

---

# Policy 3: Special rule for emailservice SMTP egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emailservice-allow-smtp
  namespace: hipster-backend
spec:
  podSelector:
    matchLabels:
      app: emailservice         # Only applies to emailservice pods
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0        # Gmail SMTP (any IP)
    ports:
    - protocol: TCP
      port: 587                 # SMTP with STARTTLS
```

**What is now blocked:** Any pod in `hipster-backend` that tries to connect to internet endpoints on non-DNS ports (except emailservice on port 587). Database pods can only receive connections from backend namespace.

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### COMPONENT: RBAC
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#### 4.9 RBAC Design

A GitHub OIDC Role exists specifically for the shippingservice's production deployment. When the CI runner needs to `kubectl set image` on the production cluster, it authenticates using GitHub OIDC (not a stored kubeconfig) and is granted only the minimum permissions:

```yaml
# hipstershop-helm-charts/hipstershop-infra/rbac/github-actions-oidc-shippingservice.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: github-actions-shippingservice-verifier
  namespace: hipster-backend
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch", "update"]   # Needed for kubectl set image
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list"]                       # Needed for rollout status
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]                       # Needed for pod status checks
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-shippingservice-verifier
  namespace: hipster-backend
subjects:
  - kind: User
    # OIDC subject claim from GitHub Actions
    name: "github:repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: github-actions-shippingservice-verifier
  apiGroup: rbac.authorization.k8s.io
```

**Why each permission was granted:**
- `deployments patch/update` — `kubectl set image` patches the deployment image field
- `replicasets get/list` — `kubectl rollout status` checks replica set health
- `pods get/list` — Final pod state verification after rollout

---

## Section 5 — Phase 4: CI/CD Implementation

### 5.1 CI/CD Design Philosophy

The goal was to automate the full loop: code push → quality gate → build → scan → push → deploy. Two environments exist: `develop` (continuous deployment, auto-synced by ArgoCD) and `main` (manual approval gate before production deployment). The pipeline is entirely defined in reusable workflows in `hipstershop-shared-workflows` so each microservice repo only has a thin caller workflow.

### 5.2 Pipeline Architecture

```
Developer Push to develop/main
           ↓
GitHub Actions (.github/workflows/{service}.yaml)
           ↓
    ┌──────────────────────────────────────────┐
    │  Job: sonar-and-snyk (PR only)           │
    │  - SonarQube scan + Quality Gate         │
    │  - Snyk dependency scan (HIGH+)          │
    └──────────────────┬───────────────────────┘
                       ↓ (success or push event)
    ┌──────────────────────────────────────────┐
    │  Job: generate-tag-and-build-image       │
    │  - Calculate semver tag (main) or        │
    │    branch-sha tag (develop)              │
    │  - Check if image already exists on Hub  │
    │  - docker build -t {image}:{tag} src/    │
    │  - Save to artifact (image.tar)          │
    │  - Trivy scan (CRITICAL CVEs, PR only)   │
    └──────────────────┬───────────────────────┘
                       ↓ (push event only)
    ┌──────────────────────────────────────────┐
    │  Job: push                               │
    │  - Load image from artifact              │
    │  - docker login → DockerHub              │
    │  - docker push {image}:{tag}             │
    │  - docker push {image}:latest (main)     │
    │  - git tag (main only)                   │
    └──────┬───────────────────────────────────┘
           │                    │
    develop branch         main branch
           ↓                    ↓
  update-helm-values-dev   push-to-ghcr
  (helm-updater-job.yaml)  (cross-push to GHCR)
                                ↓
                      prod-approval-gate
                      (manual approval in
                       GitHub environment)
                                ↓
                      update-helm-values-prod
                      (updates values.prod.yaml
                       with ghcr.io image tag)
           ↓                    ↓
    ArgoCD detects        ArgoCD detects
    develop branch        main branch change
    change → auto-sync    → auto-sync
           ↓                    ↓
  Kubernetes Rollout     Kubernetes Rollout
```

### 5.3 CI Flow — Pull Request Pipeline

On every PR to `main` or `develop`, the following jobs run (from `adservice.yaml`):

```yaml
name: adservice CI

on:
  push:
    branches: [main, develop]
    paths: ['src/**', '.github/workflows/adservice.yaml']
  pull_request:
    branches: [main, develop]
    paths: ['src/**', '.github/workflows/adservice.yaml']

permissions:
  contents: write
  packages: write

jobs:
  sonar-and-snyk:
    if: github.event_name == 'pull_request'
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
    with:
      service_name: adservice
      service_path: src/
      runtime: java
      java-version: "21"
      java-build-tool: "gradle"
    secrets:
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  generate-tag-and-build-image:
    needs: [sonar-and-snyk]
    if: |
      always() &&
      (
        (github.event_name == 'pull_request' && needs.sonar-and-snyk.result == 'success') ||
        github.event_name == 'push'
      )
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/generate-tag-and-build-image-trivy.yaml@feature/loose-coupling
    with:
      service_name: adservice
      service_path: src/
      run_trivy: ${{ github.event_name == 'pull_request' }}
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

**What a passing PR looks like:**
- `sonarqube` ✅ — Quality Gate passed
- `snyk` ✅ — No HIGH+ vulnerabilities
- `build-image` ✅ — Docker image built successfully
- `trivy` ✅ — No CRITICAL CVEs (or `continue-on-error: true` if trivy finds issues it marks fail but doesn't block)

### 5.4 CD Flow — Develop Deployment

On push to `develop`:
1. `sonar-and-snyk` is **skipped** (only runs on PR)
2. `generate-tag-and-build-image` runs — tag format: `develop-{shortsha}`
3. `push` runs — pushes to Docker Hub
4. `update-helm-values-dev` runs — calls `helm-updater-job.yaml`:

```yaml
# helm-updater-job.yaml (full workflow)
jobs:
  update-helm-values:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout hipstershop-helm-charts
        uses: actions/checkout@v4
        with:
          repository: HipsterShopp/hipstershop-helm-charts
          token: ${{ secrets.HELM_CHARTS_TOKEN }}
          ref: develop

      - name: Update image tag
        run: |
          FILE="${{ inputs.helm_values_path }}"
          sed -i "s|^\(\s*tag:\s*\).*|\1${{ inputs.image_tag }}|" "$FILE"

      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add "${{ inputs.helm_values_path }}"
          git diff --cached --quiet && echo "No changes" && exit 0
          git commit -m "chore(${{ inputs.service_name }}): update image tag to ${{ inputs.image_tag }}"
          git push origin ${{ inputs.target_branch }}
```

5. ArgoCD detects the commit to `develop` branch in `hipstershop-helm-charts` repo
6. ArgoCD auto-syncs `{service-name}-dev` application
7. Kubernetes rolling update applies new image

### 5.5 CD Flow — Production Deployment

Production deployment has a critical difference from dev — a **manual approval gate** and GHCR registry:

1. Push to `main` triggers same build
2. After `push` job: `push-to-ghcr` cross-pushes the image to `ghcr.io/hipstershopp/hipstershop-{service}:{tag}`
3. `prod-approval-gate` job pauses for **manual approval** in GitHub `production` environment
4. After approval: `update-helm-values-prod` updates `values.prod.yaml` with GHCR image tag:
   ```bash
   sed -i "s|ghcr.io/[^/]*/hipstershop-adservice:[^\"]*|ghcr.io/${OWNER}/hipstershop-adservice:${TAG}|g" \
     hipstershop-micro-apps/adservice/values.prod.yaml
   ```
5. ArgoCD `micro-apps-prod` ApplicationSet syncs `main` branch → cluster

**Note — shippingservice production deploy is different:** It uses a self-hosted runner with GitHub OIDC to run `kubectl set image` directly (not through ArgoCD Helm values update). This is the only service with a direct kubectl deploy path.

### 5.6 Reusable Workflows

The `hipstershop-shared-workflows` repo contains 5 reusable workflows:

| Workflow | Purpose | Used by |
|---------|---------|---------|
| `sonar-and-snyk.yaml` | SonarQube + Snyk scans | All services (PR only) |
| `generate-tag-and-build-image-trivy.yaml` | Tag generation, Docker build, Trivy scan | All services |
| `push.yaml` | Push image to Docker Hub, create git tag | All services |
| `helm-updater-job.yaml` | Update Helm values.yaml with new tag | All services (develop) |
| `notify.yaml` | Email failure notification via SendGrid | All services |

All callers reference `@feature/loose-coupling` branch.

### 5.7 Secrets Management

| Secret Name | Used In | Purpose | Stored In |
|------------|---------|---------|----------|
| `DOCKERHUB_USERNAME` | All service repos | Docker Hub login | GitHub repo secrets |
| `DOCKERHUB_TOKEN` | All service repos | Docker Hub push access | GitHub repo secrets |
| `GHCR_TOKEN` | All service repos | GHCR push (prod) | GitHub repo secrets |
| `HELM_CHARTS_TOKEN` | All service repos | Push to helm-charts repo | GitHub repo secrets |
| `SNYK_TOKEN` | All service repos | Snyk scan authentication | GitHub repo secrets |
| `SONAR_TOKEN` | All service repos | SonarQube authentication | GitHub repo secrets |
| `SONAR_HOST_URL` | All service repos | SonarQube server URL | GitHub repo secrets |
| `SENDGRID_API_KEY` | All service repos | Email failure notifications | GitHub repo secrets |
| `K8S_API_SERVER` | shippingservice only | Production cluster API URL | GitHub repo secrets |
| `K8S_CA_CERT` | shippingservice only | Cluster CA certificate (base64) | GitHub repo secrets |

### 5.8 Docker Build Command in CI

Extracted from `generate-tag-and-build-image-trivy.yaml`:

```bash
# Build
docker build -t ${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:${TAG} ${SERVICE_PATH}

# Save as artifact (avoids re-build in push job)
docker save ${IMAGE_TAG} -o /tmp/image.tar

# Push (in push.yaml)
docker load -i /tmp/image.tar
docker tag ${LOADED_IMAGE} ${IMAGE_TAG}
docker push ${IMAGE_TAG}

# Push :latest (main branch only)
docker push ${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:latest
```

---

## Section 6 — Phase 5: Monorepo to Multi-Repo Transition

### 6.1 Why the Split Happened

The platform was split into 14 separate repositories because a single monorepo created a **false coupling problem**: changing one service's code would trigger CI for all 12 services. Every push required waiting for all service build pipelines before knowing if any one change was safe. With independent repos, each service's CI/CD runs only when that service's code changes (enforced by `paths:` filters in GitHub Actions triggers).

Additionally, separate repos enable per-service permissions — a team that owns the payment service doesn't need write access to the frontend repo.

### 6.2 How Shared Workflows Solved Duplication

Before the `hipstershop-shared-workflows` repo, every microservice repo would have duplicated the same 200+ line workflow file. When a Trivy action version needed updating, that would require 12 separate PRs.

**Before (duplicated in every service repo):**
```yaml
# .github/workflows/adservice.yaml (would have been 200+ lines in every repo)
jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: SonarQube scan
        uses: SonarSource/sonarqube-scan-action@v6
        # ... repeated in every repo
  snyk:
    # ... repeated in every repo
  build-image:
    # ... repeated in every repo
```

**After (each service calls shared workflows):**
```yaml
# .github/workflows/adservice.yaml (actual — only 186 lines)
jobs:
  sonar-and-snyk:
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
    with:
      service_name: adservice
      runtime: java

  generate-tag-and-build-image:
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/generate-tag-and-build-image-trivy.yaml@feature/loose-coupling
    with:
      service_name: adservice
      service_path: src/
```

Now updating the Trivy action version requires **one PR** to `hipstershop-shared-workflows`. All 12 services pick up the change automatically on their next run.

### 6.3 Final Multi-Repo Structure

| Repository | Contains | Branch Strategy | Deployed By |
|-----------|---------|----------------|------------|
| `hipstershop-adservice` | Java source, Dockerfile, workflow | main + develop | ArgoCD (via helm-charts) |
| `hipstershop-authservice` | Go source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-cartservice` | .NET source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-checkoutservice` | Go source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-currencyservice` | Node.js source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-emailservice` | Python source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-frontend` | Go source, templates, static, Dockerfile | main + develop | ArgoCD |
| `hipstershop-paymentservice` | Node.js source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-productcatalogservice` | Go source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-recommendationservice` | Python source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-shippingservice` | Go source, Dockerfile, workflow (OIDC) | main + develop | Direct kubectl (prod), ArgoCD (dev) |
| `hipstershop-assistantservice` | Python source, Dockerfile, workflow | main + develop | ArgoCD |
| `hipstershop-shared-workflows` | Reusable GitHub Actions workflows | feature/loose-coupling | N/A |
| `hipstershop-helm-charts` | Helm charts, ArgoCD Applications | main + develop | Applied via kubectl/ArgoCD |

---

## Section 7 — Phase 6: GitOps with ArgoCD

### 7.1 Why GitOps for This Project

Push-based CD (GitHub Actions running `kubectl apply` directly) had a problem: if someone manually changes a Kubernetes resource in the cluster (`kubectl edit deployment/adservice`), the change is invisible and can be overwritten or persist unexpectedly. ArgoCD solves this by continuously reconciling cluster state against git — the git repo is the only source of truth.

### 7.2 ArgoCD Installation

```bash
# Install ArgoCD in its namespace
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Install Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller

# Verify
kubectl get pods -n kube-system | grep sealed-secrets
kubectl get pods -n argocd
```

### 7.3 How ArgoCD Works in This Platform

ArgoCD watches the `hipstershop-helm-charts` repository. Specifically:
- **Dev environment:** `develop` branch, `hipstershop-micro-apps/{service}/` paths
- **Prod environment:** `main` branch, `hipstershop-micro-apps/{service}/` paths
- **Infra:** `main` branch, `hipstershop-infra/` paths

When a developer merges a PR to `develop`, the GitHub Actions workflow automatically updates `hipstershop-micro-apps/{service}/values.dev.yaml` with a new image tag. ArgoCD polls the repo (default every 3 minutes) and detects the diff. The sync process runs:
```
helm template {release-name} hipstershop-micro-apps/{service} -f values.yaml -f values.dev.yaml
```
The resulting manifests are applied to `hipster-backend` (or `hipster-frontend` for frontend) in the cluster.

### 7.4 Desired State vs Live State — Real Example

When a new image `adservice-v0.1.3` is pushed to Docker Hub and the production flow completes:
1. GitHub Actions updates `hipstershop-micro-apps/adservice/values.prod.yaml` line 16: from `ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.1.2` to `ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.1.3`
2. ArgoCD detects git diff in `main` branch within ~3 minutes
3. Sync triggered automatically (selfHeal: true)
4. ArgoCD renders: `helm template adservice-prod hipstershop-micro-apps/adservice -f values.yaml -f values.prod.yaml`
5. The rendered Deployment manifest has `image: ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.1.3`
6. ArgoCD applies this — Kubernetes starts rolling update
7. Old pod terminates after new pod passes readiness probe

### 7.5 ArgoCD Application Resources

#### ArgoCD Project

```yaml
# hipstershop-helm-charts/hipstershop-argoapps/project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: hipstershop
  namespace: argocd
spec:
  description: "HipsterShop microservices platform — all charts and infra"
  sourceRepos:
    - "https://github.com/HipsterShopp/hipstershop-helm-charts"
  destinations:
    - server: https://kubernetes.default.svc
      namespace: hipster-backend
    - server: https://kubernetes.default.svc
      namespace: hipster-frontend
    - server: https://kubernetes.default.svc
      namespace: hipster-database
    - server: https://kubernetes.default.svc
      namespace: kube-system
    - server: https://kubernetes.default.svc
      namespace: ""
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  orphanedResources:
    warn: true
```

#### Infrastructure ArgoCD Applications

```yaml
# gateway.yaml — Wave 2 (depends on backend-common namespace)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: hipstershop
  source:
    repoURL: https://github.com/HipsterShopp/hipstershop-helm-charts
    targetRevision: main
    path: hipstershop-infra/gateway
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: hipster-backend
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=false
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
  ignoreDifferences:
    - group: gateway.networking.k8s.io
      kind: GatewayClass
      jsonPointers: [/metadata/annotations, /metadata/labels, /spec/parametersRef]
```

### 7.6 ApplicationSet — The Scaling Solution

Instead of 12 individual Application YAMLs, one ApplicationSet generates them all:

```yaml
# hipstershop-argoapps/apps/micro-apps-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: micro-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: assistantservice
            namespace: hipster-backend
          - name: authservice
            namespace: hipster-backend
          - name: cartservice
            namespace: hipster-backend
          - name: checkoutservice
            namespace: hipster-backend
          - name: currencyservice
            namespace: hipster-backend
          - name: emailservice
            namespace: hipster-backend
          - name: frontend
            namespace: hipster-frontend
          - name: paymentservice
            namespace: hipster-backend
          - name: productcatalogservice
            namespace: hipster-backend
          - name: recommendationservice
            namespace: hipster-backend
          - name: shippingservice
            namespace: hipster-backend

  template:
    metadata:
      name: "{{name}}"
      namespace: argocd
      annotations:
        argocd.argoproj.io/sync-wave: "3"
        argocd-image-updater.argoproj.io/image-list: "app=yampss/hipstershop-{{name}}"
        argocd-image-updater.argoproj.io/app.update-strategy: semver
        argocd-image-updater.argoproj.io/write-back-method: git
        argocd-image-updater.argoproj.io/git-branch: main
        argocd-image-updater.argoproj.io/app.helm.image-name: "global.images.{{name}}.repository"
        argocd-image-updater.argoproj.io/app.helm.image-tag: "global.images.{{name}}.tag"
    spec:
      project: hipstershop
      source:
        repoURL: https://github.com/HipsterShopp/hipstershop-helm-charts
        targetRevision: main
        path: "hipstershop-micro-apps/{{name}}"
        helm:
          valueFiles: [values.yaml]
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=false
          - ServerSideApply=true
```

**Adding a new microservice:** Add one element to the `generators.list.elements` array and create the corresponding Helm chart directory in `hipstershop-micro-apps/{new-service}/`. ArgoCD picks it up automatically on next sync.

**Three ApplicationSets exist:**
- `micro-apps-appset.yaml` — main branch (base/production using values.yaml)
- `micro-apps-appset-dev.yaml` — develop branch (values.yaml + values.dev.yaml)
- `micro-apps-appset-prod.yaml` — main branch (values.yaml + values.prod.yaml, prune: false)

### 7.7 Helm in ArgoCD — The Critical Concept

> ⚠️ **IMPORTANT:** `helm list -n hipster-backend` returns **empty**. This is **expected behavior**.

ArgoCD uses its own internal Helm rendering engine. It never runs `helm install` — it runs `helm template` and then `kubectl apply`. The Helm release is not registered in the cluster's Helm release history.

```bash
# Wrong — always returns empty
helm list -n hipster-backend
helm list -n hipster-frontend

# Correct — use ArgoCD to see what's deployed
argocd app list
argocd app get adservice
argocd app manifests adservice
```

### 7.8 ArgoCD Sync Policies

| Application | Auto-sync | selfHeal | prune | Reason |
|------------|----------|----------|-------|--------|
| mongodb | Yes | Yes | **No** | Don't prune to avoid accidental data loss |
| backend-common | Yes | Yes | **No** | ConfigMaps should not be pruned automatically |
| gateway | Yes | Yes | Yes | Stateless config — safe to prune |
| All micro-apps (dev) | Yes | Yes | Yes | Dev environments: always converge fast |
| All micro-apps (prod) | Yes | Yes | **No** | Prod: safer to leave orphaned resources |

### 7.9 ArgoCD Access

```bash
# Port-forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open in browser
# https://localhost:8080
# Username: admin

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login via CLI
argocd login localhost:8080 --username admin --password <password> --insecure
```

---

## Section 8 — Phase 7: Helm Integration

### 8.1 Why Helm for This Project

Before Helm, every service needed manually written Kubernetes YAML files. With 12 services each needing a Deployment, Service, and potentially more resources, that's 24+ YAML files — all with repeated boilerplate. Worse, values like namespace names and port numbers were copy-pasted across files, creating drift. Helm templates eliminated this by centralizing all configuration in `values.yaml` and using `{{ .Values.xxx }}` injection.

### 8.2 Helm Repo Structure

```
hipstershop-helm-charts/
├── hipstershop-argoapps/          # ArgoCD Application/ApplicationSet definitions
│   ├── project.yaml               # ArgoCD AppProject: hipstershop
│   ├── apps/
│   │   ├── adservice.yaml         # Individual app (commented out — replaced by AppSet)
│   │   ├── micro-apps-appset.yaml      # Main branch AppSet (11 services)
│   │   ├── micro-apps-appset-dev.yaml  # Develop branch AppSet (12 services)
│   │   └── micro-apps-appset-prod.yaml # Prod AppSet with ignoreDifferences
│   └── infra/
│       ├── backend-common.yaml    # Wave 0: namespace, configmaps, secrets, netpol
│       ├── gateway.yaml           # Wave 2: kGateway + HTTPRoutes
│       ├── gateway-prod.yaml      # Wave 2: prod variant
│       ├── loadgenerator.yaml     # Wave 3: load generator job
│       ├── mongodb.yaml           # Wave 0: MongoDB StatefulSet
│       └── sealed-secrets.yaml   # Wave 1: Sealed Secrets controller ArgoCD App
│
├── hipstershop-infra/             # Infrastructure Helm charts
│   ├── backend-common/            # Namespaces, ConfigMaps, Secrets, NetworkPolicies
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── configmaps.yaml    # hipster-config, service-addresses, service-ports, mongodb-config, app-features
│   │       ├── namespace.yaml     # hipster-backend namespace
│   │       ├── network-policy.yaml
│   │       └── secrets.yaml       # app-secrets, mongodb-root, mongodb-users
│   ├── gateway/                   # kGateway GatewayClass, Gateway, HTTPRoutes
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values.prod.yaml
│   │   └── templates/
│   │       ├── gateway.yaml
│   │       └── routes.yaml        # 11 HTTPRoute objects
│   ├── loadgenerator/             # Load testing Deployment
│   ├── mongodb/                   # MongoDB StatefulSet, Services, Namespaces, Secrets
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── init-job.yaml
│   │       ├── namespace.yaml     # hipster-database namespace + mongodb-root/users secrets
│   │       ├── network-policy.yaml
│   │       ├── seed-migrate-jobs.yaml
│   │       ├── services.yaml
│   │       └── statefulset.yaml
│   └── rbac/                      # GitHub OIDC RBAC for shippingservice
│       └── github-actions-oidc-shippingservice.yaml
│
├── hipstershop-micro-apps/        # One Helm chart per microservice
│   ├── adservice/
│   │   ├── Chart.yaml
│   │   ├── values.yaml            # Base values (main branch / production registry)
│   │   ├── values.dev.yaml        # Dev overrides (lighter resources, develop-latest tag)
│   │   ├── values.prod.yaml       # Prod overrides (GHCR image, prod resource requests)
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── [authservice, cartservice, checkoutservice, currencyservice,
│        emailservice, frontend, paymentservice, productcatalogservice,
│        recommendationservice, shippingservice, assistantservice]
│        └── (same structure)
│
└── sealed-secrets/                # Sealed Secrets generator tooling
    ├── .gitignore                  # Ignores pub-cert.pem
    ├── backend-common-secrets.yaml # Encrypted app-secrets (JWT_SECRET, GEMINI_API_KEY, mongodb creds)
    ├── generate-sealed-secrets.sh  # Script to generate sealed secrets using kubeseal
    ├── mongodb-secrets.yaml        # Encrypted MongoDB credentials
    └── steps.txt                   # Step-by-step operator runbook
```

### 8.3 Per-Service Values Overrides

| Value Key | Default (values.yaml) | Dev (values.dev.yaml) | Prod (values.prod.yaml) |
|-----------|-----------------------|-----------------------|------------------------|
| `global.images.adservice` | `yampss/hipstershop-adservice:adservice-v0.1.2` | `yampss/hipstershop-adservice:develop-latest` | `ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.2.3` |
| `global.replicas.adservice` | 1 | 1 | 1 |
| `global.resources.adservice.requests.cpu` | 200m | 100m | 200m |
| `global.resources.adservice.requests.memory` | 180Mi | 90Mi | 180Mi |
| `global.resources.adservice.limits.cpu` | 300m | 200m | 300m |
| `global.resources.adservice.limits.memory` | 512Mi | 256Mi | 512Mi |

Dev environment uses half the resource requests — reduces cost on dev clusters.

---

## Section 9 — Complete End-to-End Flow

### 9.1 The Scenario

Developer fixes a bug in the `adservice` (Java) that was causing incorrect ads to be returned. Here's exactly what happens from commit to running pod:

### 9.2 Step-by-Step Trace

**STEP 1: Developer creates feature branch**
```bash
git checkout -b fix/ad-targeting-logic
# Developer modifies src/src/main/java/hipstershop/AdService.java
git add src/
git commit -m "fix(adservice): correct ad targeting logic for user context"
git push origin fix/ad-targeting-logic
```

**STEP 2: Pull Request opened to `develop`**
- GitHub triggers workflow: `adservice CI`
- Jobs that run:
  1. `sonar-and-snyk` — SonarQube static analysis + Snyk dependency scan
  2. `generate-tag-and-build-image` — Tag: `fix-ad-targeting-logic-{sha8}`, builds Docker image, runs Trivy CRITICAL CVE scan
- What must pass: SonarQube Quality Gate + Trivy (no CRITICAL CVEs)

**STEP 3: PR Merged to `develop`**
- Workflow triggered: `adservice CI` (push event)
- Jobs in order:
  1. `generate-tag-and-build-image` — Tag computed: `develop-{sha8}` (no sonar/snyk on push)
  2. `push` — `docker push yampss/hipstershop-adservice:develop-{sha8}`; also pushes `:latest` if main
  3. `update-helm-values-dev` — Checks out `hipstershop-helm-charts:develop`, runs `sed` to update tag in `hipstershop-micro-apps/adservice/values.dev.yaml`, commits `chore(adservice): update image tag to develop-{sha8}`

**STEP 4: Docker Image in Registry**
- Available at: `yampss/hipstershop-adservice:develop-{sha8}`
- Helm values file updated with new tag

**STEP 5: Helm Values Updated**
- File: `hipstershop-micro-apps/adservice/values.dev.yaml`
- Key: `.global.images.adservice`
- From: `yampss/hipstershop-adservice:develop-abc12345`
- To: `yampss/hipstershop-adservice:develop-def67890`

**STEP 6: ArgoCD Detects Change**
- ArgoCD polls `hipstershop-helm-charts:develop` every ~3 minutes
- Detects diff in `hipstershop-micro-apps/adservice/values.dev.yaml`
- ApplicationSet `micro-apps-dev` syncs `adservice-dev` application automatically
- ArgoCD renders: `helm template adservice hipstershop-micro-apps/adservice -f values.yaml -f values.dev.yaml`

**STEP 7: Kubernetes Updated**
```
Old pod: adservice-7d4b9f-xxxxx  (Running)
New pod: adservice-8e5c0g-yyyyy  (Starting → readiness probe at /_healthz)
After 60s (initialDelaySeconds): readiness probe passes
Old pod: terminated (terminationGracePeriodSeconds: 5)
New pod: Running
```

**STEP 8: Verification**
```bash
# Check rollout status
kubectl rollout status deployment/adservice -n hipster-backend

# Verify new image
kubectl get deployment adservice -n hipster-backend -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check pod logs
kubectl logs -l app=adservice -n hipster-backend --tail=50

# Check ArgoCD sync status
argocd app get adservice-dev
```

### 9.3 Timing Summary

| Stage | Typical Duration |
|-------|----------------|
| CI Checks (PR — sonar + build + trivy) | 6–12 minutes |
| Docker Build & Push (push event) | 3–8 minutes |
| ArgoCD Detection (git poll) | up to 3 minutes |
| Kubernetes Pod Start + Readiness | 60–90 seconds (Java), 5–15 seconds (Go/Python/Node) |
| **Total End-to-End (push to running)** | **~10–15 minutes** |

---

## Section 10 — Troubleshooting Guide

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #1: CrashLoopBackOff
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** `kubectl get pods -n hipster-backend` shows `CrashLoopBackOff`

**Debug Steps:**
```bash
# Step 1: Get crash logs
kubectl logs <pod-name> -n hipster-backend --previous

# Step 2: For authservice — check if MONGO_URI is missing
kubectl exec <pod-name> -n hipster-backend -- env | grep MONGO

# Step 3: Check events
kubectl get events -n hipster-backend --sort-by='.lastTimestamp' | tail -20

# Step 4: Describe pod for exit code
kubectl describe pod <pod-name> -n hipster-backend
```

**Common root causes for this platform:**
- `authservice`: `MONGO_URI` not set → service crashes immediately (`log.Fatalf("Required environment variable MONGO_URI is not set")`)
- `adservice`: OOM killed (Java heap too large for memory limit)
- `paymentservice`: MongoDB connection refused → payment store init fails (but service continues without persistence)

**Fix for missing secret (authservice):**
```bash
# Verify sealed secret was decrypted
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
# If missing, re-apply sealed secrets:
kubectl apply -f sealed-secrets/backend-common-secrets.yaml
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #2: ImagePullBackOff
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** Pod stuck in `ImagePullBackOff`

```
Failed to pull image "yampss/hipstershop-adservice:develop-abc12345": ...
```

**Debug Steps:**
```bash
kubectl describe pod <pod-name> -n hipster-backend | grep -A5 "Events:"
```

**Root causes:**
- Tag doesn't exist on Docker Hub yet (CI push job failed silently)
- Rate limit on Docker Hub (unauthenticated pull)
- GHCR image used in prod but no `imagePullSecret` configured

**Fix:**
```bash
# Check if image exists
docker pull yampss/hipstershop-adservice:develop-abc12345

# Check ArgoCD values are correct
argocd app manifests adservice-dev | grep image:
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #3: ArgoCD OutOfSync — Never Heals
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** ArgoCD shows `OutOfSync` but sync always fails or immediately goes OutOfSync again

**Debug Steps:**
```bash
argocd app get <app-name>
argocd app diff <app-name>
```

**Root cause:** A field is being modified by a controller (e.g., kGateway setting annotations on GatewayClass) that ArgoCD keeps trying to revert.

**Fix:** Add `ignoreDifferences` to the Application YAML (as done for gateway):
```yaml
ignoreDifferences:
  - group: gateway.networking.k8s.io
    kind: GatewayClass
    jsonPointers:
      - /metadata/annotations
      - /metadata/labels
      - /spec/parametersRef
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #4: `helm list` Returns Empty
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:**
```bash
$ helm list -n hipster-backend
NAME    NAMESPACE   REVISION    UPDATED STATUS  CHART   APP VERSION
(empty)
```

**This is EXPECTED BEHAVIOR.** ArgoCD uses `helm template` + `kubectl apply`, not `helm install`. No Helm release is registered.

```bash
# Use these instead:
argocd app list
argocd app get adservice
argocd app manifests adservice-dev
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #5: GitHub Actions Fails on Docker Push
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** `push` job fails with `unauthorized: authentication required`

**Debug Steps:**
- Check that `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets are set in the microservice repo
- Check that the Docker Hub token has `Read & Write` scope

**Fix:** Regenerate Docker Hub access token, update GitHub secret.

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #6: Service Unreachable (via Gateway)
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** Gateway returns 502/503 for a specific service path

**Debug Steps:**
```bash
# Step 1: Check if service pods are running
kubectl get pods -n hipster-backend -l app=<service>

# Step 2: Check endpoints (pods registered with service?)
kubectl get endpoints <service> -n hipster-backend

# Step 3: Check HTTPRoute is Accepted
kubectl get httproute -n hipster-backend
kubectl describe httproute <route-name> -n hipster-backend

# Step 4: Check Network Policy
kubectl describe networkpolicy -n hipster-backend

# Step 5: Test from inside cluster
kubectl exec -it <any-pod> -n hipster-backend -- curl http://<service>:<port>/_healthz
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #7: Helm Values Not Reflecting in Deployed Pods
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** Pod is running old image even though `values.dev.yaml` was updated

**Debug Steps:**
```bash
# Step 1: Verify ArgoCD synced
argocd app get <service>-dev

# Step 2: Check ArgoCD rendered the right image
argocd app manifests <service>-dev | grep image:

# Step 3: Check the actual deployment image
kubectl get deployment <service> -n hipster-backend \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Step 4: Force refresh and sync
argocd app get <service>-dev --refresh
argocd app sync <service>-dev
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #8: MongoDB Pod Pending
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** `mongodb-0` pod in `Pending` state

**Debug Steps:**
```bash
kubectl describe pod mongodb-0 -n hipster-database
# Look for: "no persistent volumes available for this claim"
kubectl get pvc -n hipster-database
kubectl get storageclass
```

**Root cause:** NFS StorageClass not present or NFS provisioner not running.

**Fix:** Ensure NFS provisioner is deployed and `storageClassName: nfs` exists:
```bash
kubectl get storageclass nfs
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #9: Pod Pending — Insufficient Resources
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** Pod in `Pending` with `Insufficient cpu` or `Insufficient memory`

```bash
kubectl describe pod <pod-name> -n hipster-backend | grep -A5 "Events:"
# "0/3 nodes are available: 3 Insufficient cpu"
```

**Fix (temporary):**
```bash
# Reduce resource requests for dev
# Edit hipstershop-micro-apps/{service}/values.dev.yaml
# Commit and push — ArgoCD will apply
```

### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
### ISSUE #10: ArgoCD Sync Failed — Helm Rendering Error
### ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Symptom:** ArgoCD shows `ComparisonError: failed to render Helm chart`

**Debug Steps:**
```bash
# Render locally to see error
helm template test hipstershop-micro-apps/<service> \
  -f hipstershop-micro-apps/<service>/values.yaml \
  -f hipstershop-micro-apps/<service>/values.dev.yaml

# Lint chart
helm lint hipstershop-micro-apps/<service> \
  -f hipstershop-micro-apps/<service>/values.yaml
```

---

## Section 11 — Command Cheat Sheet

### Kubernetes Commands

```bash
# ━━ VIEWING RESOURCES ━━
# List pods in all platform namespaces
kubectl get pods -n hipster-backend
kubectl get pods -n hipster-frontend
kubectl get pods -n hipster-database
kubectl get pods -n argocd

# Get detailed pod info (use when pod is in bad state)
kubectl describe pod <pod-name> -n hipster-backend

# Get logs from running pod
kubectl logs <pod-name> -n hipster-backend

# Get logs from crashed container
kubectl logs <pod-name> -n hipster-backend --previous

# Follow logs in real-time
kubectl logs -f <pod-name> -n hipster-backend

# Get logs by label (useful when pod name keeps changing)
kubectl logs -l app=adservice -n hipster-backend --tail=100

# ━━ DEPLOYMENTS ━━
# Check deployment status
kubectl rollout status deployment/adservice -n hipster-backend

# Restart a deployment (triggers new pod creation)
kubectl rollout restart deployment/adservice -n hipster-backend

# Rollback last deployment
kubectl rollout undo deployment/adservice -n hipster-backend

# See rollout history
kubectl rollout history deployment/adservice -n hipster-backend

# ━━ DEBUGGING ━━
# Port-forward to test service locally (most images are distroless, exec won't work)
kubectl port-forward svc/adservice 9555:9555 -n hipster-backend
kubectl port-forward svc/frontend 8080:80 -n hipster-frontend
kubectl port-forward svc/argocd-server 8080:443 -n argocd

# Check events (shows why pod failed to schedule)
kubectl get events -n hipster-backend --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -n hipster-backend
kubectl top nodes

# ━━ SERVICES & NETWORKING ━━
# List all services
kubectl get svc -n hipster-backend

# Check endpoints (verify pods are registered)
kubectl get endpoints adservice -n hipster-backend

# List all network policies
kubectl get networkpolicy -n hipster-backend

# ━━ SECRETS ━━
# Verify sealed secrets were decrypted
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-root -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
kubectl get secret mongodb-root -n hipster-database
kubectl get secret mongodb-users -n hipster-database

# ━━ MONGODB ━━
# Connect to MongoDB (if mongosh available)
kubectl exec -it mongodb-0 -n hipster-database -- mongosh \
  --username admin --password mongopassword --authenticationDatabase admin
```

### ArgoCD Commands

```bash
# Login
argocd login localhost:8080 --username admin --password <password> --insecure

# List all applications
argocd app list

# Get app details (includes sync status, health, image)
argocd app get adservice-dev
argocd app get adservice-prod

# Sync an application
argocd app sync adservice-dev

# Force refresh (re-poll git without syncing)
argocd app get adservice-dev --refresh

# View rendered manifests (what ArgoCD will apply)
argocd app manifests adservice-dev

# View app history
argocd app history adservice-dev

# Rollback to previous version
argocd app rollback adservice-dev <history-id>
```

### Docker Commands

```bash
# Build (format used in this project for dev)
docker build -t yampss/hipstershop-adservice:develop-abc12345 src/

# Push to Docker Hub
docker push yampss/hipstershop-adservice:develop-abc12345

# Test image locally
docker run -p 9555:9555 \
  -e PORT=9555 \
  yampss/hipstershop-adservice:develop-abc12345

# Check image layers
docker history yampss/hipstershop-adservice:develop-abc12345
```

### Helm Commands

```bash
# NOTE: helm list returns empty — ArgoCD manages releases
helm list -n hipster-backend   # Returns empty — EXPECTED

# Render templates locally (debugging)
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Render with dev overrides
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  -f hipstershop-micro-apps/adservice/values.dev.yaml

# Lint chart
helm lint hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Dry run
helm install adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  --dry-run
```

---

## Section 12 — Best Practices

### 12.1 GitOps Practices

- **Single Source of Truth:** All Kubernetes state is defined in `hipstershop-helm-charts`. No manual `kubectl apply` except for initial bootstrapping.
  - Evidence: All ArgoCD Applications point to `hipstershop-helm-charts` repo; `selfHeal: true` reverts manual changes.

- **Sync Wave Ordering:** Infrastructure bootstrapped before applications via `argocd.argoproj.io/sync-wave` annotations.
  - Evidence: Wave 0 (mongodb, backend-common) → Wave 1 (sealed-secrets) → Wave 2 (gateway) → Wave 3 (micro-apps)

### 12.2 Branching Strategy

| Branch | Purpose | Who merges to it | What it triggers |
|--------|---------|-----------------|--------------------|
| `develop` | Active development | Developers via PR | Dev image build → ArgoCD dev sync |
| `main` | Production-ready | Tech lead via PR from develop | Prod image build → Manual approval → Prod deploy |
| `feature/*` | Feature work | Developer (self) | PR CI checks |
| `fix/*` | Bug fixes | Developer (self) | PR CI checks |

### 12.3 Secrets Handling

- Secrets are encrypted using Bitnami Sealed Secrets before being committed to git.
- `pub-cert.pem` (the cluster's RSA public key) is in `.gitignore` — never committed.
- The raw secret values (`JWT_SECRET`, `GEMINI_API_KEY`, MongoDB passwords) are only ever in environment variables during the `generate-sealed-secrets.sh` execution.
- CI secrets (Docker Hub credentials, SonarQube tokens) are stored in GitHub repo-level secrets.

### 12.4 Docker Best Practices Used

- ✅ Multi-stage builds (all services)
- ✅ Specific base image versions (SHA-pinned for most services)
- ✅ Non-root user (cartservice: `USER 1000`)
- ✅ Layer caching optimization (go.mod/package.json copied separately before source)
- ✅ `PYTHONDONTWRITEBYTECODE=1` in Python services
- ❌ `.dockerignore` files not confirmed in repos
- ❌ Health checks in Dockerfile not present (health checks are in Kubernetes Deployment probes instead)

### 12.5 Kubernetes Best Practices Used

- ✅ Resource requests and limits on all containers
- ✅ Liveness and readiness probes on all containers
- ✅ Network Policies with default-deny-all
- ✅ Namespaces for environment isolation
- ✅ Rolling update strategy with maxUnavailable/maxSurge defined
- ✅ GitHub OIDC (keyless auth) for shippingservice production deploy
- ✅ Sealed Secrets for secret management
- ❌ Pod Disruption Budgets not present
- ❌ Pod anti-affinity rules not defined
- ❌ HorizontalPodAutoscaler absent for most services (frontend has HPA config in values.yaml)

---

## Section 13 — Future Improvements

### Monitoring (Prometheus + Grafana)
**Current State:** Distributed tracing configured via OpenTelemetry to Jaeger (`COLLECTOR_SERVICE_ADDR: "jaeger:4317"`), but Prometheus metrics scraping and Grafana dashboards are not present.  
**Gap:** No alerting on latency spikes, OOM kills, or pod restarts. No dashboards for request rates or error rates per service.  
**Recommended Solution:** Deploy kube-prometheus-stack via Helm; configure ServiceMonitors for each microservice.  
**Effort:** Medium | **Impact:** High

### Log Aggregation (Loki / ELK)
**Current State:** Logs are accessible via `kubectl logs` only. No centralized log aggregation.  
**Gap:** No ability to search across services, no log retention, no correlation between services.  
**Recommended Solution:** Deploy Grafana Loki + Promtail DaemonSet; view in Grafana.  
**Effort:** Medium | **Impact:** High

### Secret Management (External Secrets Operator)
**Current State:** Sealed Secrets provides encryption at rest in git but requires manual re-sealing when cluster RSA key changes.  
**Gap:** Secrets rotation requires regenerating all sealed secrets and pushing to git.  
**Recommended Solution:** External Secrets Operator pulling from AWS Secrets Manager or HashiCorp Vault.  
**Effort:** High | **Impact:** High

### Horizontal Pod Autoscaling
**Current State:** Frontend has HPA configuration in `values.yaml` (minReplicas: 1, maxReplicas: 5), but actual HPA resource is not confirmed deployed. Other services have no HPA.  
**Gap:** Services cannot scale on traffic spikes.  
**Recommended Solution:** Deploy HPA resources for high-traffic services (frontend, productcatalogservice, checkoutservice).  
**Effort:** Low | **Impact:** Medium

### Security Scanning in CI (Trivy already present — expand)
**Current State:** Trivy scans Docker images for CRITICAL CVEs on PRs only. Does not scan Helm charts or Kubernetes manifests.  
**Gap:** Misconfigured Kubernetes manifests (missing securityContext, privileged containers) are not detected.  
**Recommended Solution:** Add `trivy config` scan on Helm templates in CI.  
**Effort:** Low | **Impact:** Medium

### Disaster Recovery / Backup
**Current State:** MongoDB data on NFS, no backup job defined.  
**Gap:** Data loss risk if NFS volume is lost.  
**Recommended Solution:** Velero for cluster backup + MongoDB backup CronJob writing to S3.  
**Effort:** Medium | **Impact:** High
