# hipstershop-adservice

> Java (Gradle) microservice that serves personalized advertisements to the HipsterShop frontend.

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
| **Language/Runtime** | Java 25 (JRE Alpine), built with Java 24 JDK |
| **Framework** | Gradle build system, custom gRPC/HTTP ad server |
| **Port** | 9555 |
| **Database** | None |
| **Docker Image** | `yampss/hipstershop-adservice:{tag}` (dev) / `ghcr.io/hipstershopp/hipstershop-adservice:{tag}` (prod) |
| **Namespace** | `hipster-backend` |
| **Build Tool** | Gradle (Kotlin DSL or Groovy) |

### What This Service Does

AdService serves contextually relevant advertisements based on request parameters. It exposes an HTTP endpoint (`/ads`) that accepts a request and returns a list of advertisement objects. The service instruments itself with the OpenTelemetry Java agent at startup for distributed tracing to Jaeger. It is queried by the frontend when rendering product listing pages.

### What This Service Does NOT Do

AdService does not store ad click data (no persistence layer). It does not authenticate requests. It does not manage ad inventory via a database — ad data is likely embedded in the application code.

---

## Architecture Position

```
                [kGateway]
                    │
        GET /api/ads → URLRewrite → /ads
                    │
            [adservice :9555]
                    │
              (returns ad list)
                    │
              [frontend]
```

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| kGateway HTTPRoute | `GET /api/ads` → `/ads` | Frontend requests ads for product pages |

**Sends traffic to:** None (leaf service, no downstream dependencies)

---

## API Reference

### Endpoints

| Method | Path | Description | Auth Required |
|--------|------|-------------|--------------|
| GET | `/ads` | Returns list of advertisements | No |
| GET | `/_healthz` | Health check | No |

### Request/Response: GET /ads

**Request:** Query params (context keys for targeting)

**Response:**
```json
[
  {
    "redirectUrl": "/product/OLJCESPC7Z0",
    "text": "Film camera for sale. Vintage cameras who can."
  }
]
```

---

## Local Development

### Prerequisites

- Java 21+ JDK installed
- Gradle 8.x (or use the included `gradlew` wrapper)

```bash
git clone https://github.com/HipsterShopp/hipstershop-adservice
cd hipstershop-adservice
```

### Run Locally

```bash
cd src/
# Fix gradlew line endings if on Linux/Mac
sed -i 's/\r$//' gradlew && chmod +x gradlew

# Build and run
./gradlew installDist
./build/install/hipstershop/bin/AdService
```

### Run with Docker

```bash
# Build
docker build -t hipstershop-adservice:local src/

# Run
docker run -p 9555:9555 \
  -e PORT=9555 \
  hipstershop-adservice:local
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

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

**Build Strategy:** Multi-stage. Builder stage uses full JDK for Gradle compilation; runtime uses JRE-only Alpine image (smaller, no JDK overhead).

**Base Image (runtime):** `eclipse-temurin:25.0.2_10-jre-alpine` — SHA-pinned for reproducible builds. JRE only (no compiler) reduces attack surface.

**Important Decisions:**
- `sed -i 's/\r$//'`: Fixes Windows CRLF line endings in `gradlew` that cause `exec format error` on Linux CI runners.
- `./gradlew downloadRepos` before `COPY . .`: Enables Docker layer caching — dependency layer is only invalidated when `build.gradle` changes.
- OpenTelemetry agent downloaded at build time and baked into image: No runtime download needed.

---

## CI/CD Pipeline

### Pipeline Overview

```
push/PR to main or develop (src/** changed)
              ↓
    GitHub Actions: adservice CI
              ↓
    [PR only] sonar-and-snyk
    (SonarQube Quality Gate + Snyk HIGH+)
              ↓
    generate-tag-and-build-image
    (tag: adservice-v0.x.x on main, develop-{sha8} on develop)
    (Docker build + Trivy CRITICAL scan on PR)
              ↓ [push event]
    push to DockerHub: yampss/hipstershop-adservice:{tag}
              ↓
    develop → update-helm-values-dev (values.dev.yaml)
    main    → push-to-ghcr + prod-approval-gate → update-helm-values-prod (values.prod.yaml)
```

### Workflow Details

**File:** `.github/workflows/adservice.yaml`  
**Trigger:** Push to `main`/`develop` (paths: `src/**`), PR to `main`/`develop`

**Jobs Breakdown:**

| Job | Runs On | Trigger | Purpose |
|-----|---------|---------|---------|
| `sonar-and-snyk` | ubuntu-latest | PR only | SonarQube + Snyk scan |
| `generate-tag-and-build-image` | ubuntu-latest | Always | Tag, build, Trivy scan |
| `push` | ubuntu-latest | Push event | Push to Docker Hub |
| `push-to-ghcr` | ubuntu-latest | Push to main | Cross-push to GHCR |
| `update-helm-values-dev` | ubuntu-latest | Push to develop | Update dev values |
| `prod-approval-gate` | ubuntu-latest | Push to main | Manual approval pause |
| `update-helm-values-prod` | ubuntu-latest | After approval | Update prod values |
| `notify` | ubuntu-latest | Always | Email on failure |

### Required Secrets

| Secret | Purpose | Set At |
|--------|---------|--------|
| `DOCKERHUB_USERNAME` | Docker Hub login | Repo secrets |
| `DOCKERHUB_TOKEN` | Docker Hub push | Repo secrets |
| `GHCR_TOKEN` | GHCR push (prod) | Repo secrets |
| `HELM_CHARTS_TOKEN` | Push to helm-charts repo | Repo secrets |
| `SNYK_TOKEN` | Snyk scan | Repo secrets |
| `SONAR_TOKEN` | SonarQube auth | Repo secrets |
| `SONAR_HOST_URL` | SonarQube server | Repo secrets |
| `SENDGRID_API_KEY` | Failure email | Repo secrets |

### How Shared Workflows Are Used

```yaml
# Caller (this repo):
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: adservice
    service_path: src/
    runtime: java
    java-version: "21"
    java-build-tool: "gradle"
```

The shared workflow:
1. Checkouts code with `fetch-depth: 0` (full history for SonarQube blame analysis)
2. Runs `SonarSource/sonarqube-scan-action@v6` against `src/` directory
3. Blocks on SonarQube Quality Gate result
4. Runs Snyk against `build.gradle` for dependency vulnerabilities
5. Uploads Snyk HTML report as artifact (retained 14 days)

---

## Kubernetes Deployment

### Resources Created

| Resource | Name | Namespace | Purpose |
|---------|------|-----------|---------|
| Deployment | adservice | hipster-backend | Runs AdService pods |
| Service | adservice | hipster-backend | ClusterIP endpoint :9555 |
| HTTPRoute | ad-route | hipster-backend | Routes /api/ads → adservice |

### Deployment Configuration

```yaml
# hipstershop-helm-charts/hipstershop-micro-apps/adservice/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: hipster-backend
  name: adservice
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: yampss/hipstershop-adservice:adservice-v0.1.2
        ports:
        - containerPort: 9555
        envFrom:
        - configMapRef:
            name: hipster-config
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: service-ports
              key: ADSERVICE_PORT
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 512Mi
        readinessProbe:
          initialDelaySeconds: 60
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

**Key Configuration Decisions:**
- **Replicas:** 1 — AdService is stateless; single replica sufficient for current load
- **Update Strategy:** RollingUpdate, maxUnavailable: 1, maxSurge: 1 — zero-downtime deploy
- **CPU Request:** 200m — Java JVM needs baseline CPU even at idle
- **Memory Limit:** 512Mi — Java heap + OTel agent overhead; set higher than most Go services
- **initialDelaySeconds: 60** — Java JVM + OpenTelemetry agent startup takes ~45-60 seconds
- **tmp-volume emptyDir** — OTel agent requires writable `/tmp` space

### Health Checks

```yaml
readinessProbe:
  initialDelaySeconds: 60
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
```

**Liveness:** If `/_healthz` fails after startup, pod is restarted.  
**Readiness:** Pod doesn't receive gateway traffic until `/_healthz` returns 200.  
**60s delay:** Java startup with OTel javaagent is slow; premature probing causes false failures.

### Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: hipster-backend
  name: adservice
spec:
  type: ClusterIP
  selector:
    app: adservice
  ports:
  - name: http
    port: 9555
    targetPort: 9555
```

**Type: ClusterIP** — Internal only. All external traffic enters via kGateway HTTPRoute.

---

## ArgoCD GitOps

**ArgoCD Application Name:** `adservice` (main), `adservice-dev` (develop)  
**Source Repo:** `https://github.com/HipsterShopp/hipstershop-helm-charts`  
**Source Path:** `hipstershop-micro-apps/adservice`  
**Target Namespace:** `hipster-backend`  
**Sync Policy:** Automated (selfHeal: true, prune: true)

### GitOps Flow

1. PR merged to `develop` in `hipstershop-adservice`
2. GitHub Actions builds Docker image: `yampss/hipstershop-adservice:develop-{sha8}`
3. Workflow updates `hipstershop-micro-apps/adservice/values.dev.yaml`
4. ArgoCD detects change in `hipstershop-helm-charts:develop` within ~3 minutes
5. ArgoCD renders: `helm template adservice hipstershop-micro-apps/adservice -f values.yaml -f values.dev.yaml`
6. Applied to cluster: Deployment image field updated
7. Kubernetes rolling update in `hipster-backend`

### Values File

```yaml
# hipstershop-micro-apps/adservice/values.yaml (base)
global:
  namespaces:
    backend: hipster-backend
  images:
    adservice: "yampss/hipstershop-adservice:adservice-v0.1.2"
  replicas:
    adservice: 1
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
  ports:
    adservice: 9555
  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi
```

```yaml
# values.dev.yaml — dev overrides
global:
  replicas:
    adservice: 1
  resources:
    adservice:
      requests:
        cpu: 100m
        memory: 90Mi
      limits:
        cpu: 200m
        memory: 256Mi
  images:
    adservice: "yampss/hipstershop-adservice:develop-latest"
```

```yaml
# values.prod.yaml — prod overrides (GHCR image)
global:
  replicas:
    adservice: 1
  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi
  images:
    adservice: "ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.2.3"
```

---

## Configuration

### Environment Variables

| Variable | Required | Source | Description |
|----------|----------|--------|-------------|
| `PORT` | Yes | ConfigMap `service-ports` key `ADSERVICE_PORT` | HTTP listen port (9555) |
| `ENABLE_TRACING` | No | ConfigMap `hipster-config` | Enable OTel tracing ("1") |
| `COLLECTOR_SERVICE_ADDR` | No | ConfigMap `hipster-config` | Jaeger collector address ("jaeger:4317") |
| `FRONTEND_MESSAGE` | No | ConfigMap `hipster-config` | Platform message |
| `ENABLE_ASSISTANT` | No | ConfigMap `hipster-config` | Feature flag |
| `JAVA_TOOL_OPTIONS` | Set in Dockerfile | Dockerfile ENV | `-javaagent:/opentelemetry-javaagent.jar` |

---

## Troubleshooting

### Service-Specific Issues

**Issue: Pod stuck in `Pending` after deploy**
```bash
kubectl describe pod -l app=adservice -n hipster-backend | grep Events -A10
# Check for: Insufficient cpu or Insufficient memory
# Fix: Reduce resource requests in values.dev.yaml
```

**Issue: Pod starts then enters `CrashLoopBackOff` immediately**
```bash
kubectl logs -l app=adservice -n hipster-backend --previous
# Likely: OOM kill (Java heap exceeds memory limit) or OTel agent download failed
```

**Issue: Readiness probe fails for 60+ seconds after pod start**
```
# This is expected! Java JVM + OTel agent startup takes ~50-60 seconds.
# The initialDelaySeconds: 60 handles this. Only investigate if pod never becomes Ready.
```

### General Debug Commands

```bash
# Check pods
kubectl get pods -n hipster-backend -l app=adservice

# Logs
kubectl logs -l app=adservice -n hipster-backend --tail=100

# Port forward for local testing
kubectl port-forward svc/adservice 9555:9555 -n hipster-backend
curl http://localhost:9555/ads
curl http://localhost:9555/_healthz

# Check env vars in pod
kubectl exec -it <pod-name> -n hipster-backend -- env | grep -E 'PORT|TRACING|COLLECTOR'

# Check if service has endpoints (pods registered)
kubectl get endpoints adservice -n hipster-backend
```
