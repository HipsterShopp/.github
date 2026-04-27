# hipstershop-shippingservice

> Go microservice that calculates shipping quotes. Unique: uses GitHub Actions OIDC to deploy directly to the production cluster via kubectl — not through ArgoCD GitOps for production.

---

## Table of Contents
- [Service Overview](#service-overview)
- [Architecture Position](#architecture-position)
- [API Reference](#api-reference)
- [Local Development](#local-development)
- [Docker](#docker)
- [CI/CD Pipeline (Production OIDC Deploy)](#cicd-pipeline)
- [Kubernetes Deployment](#kubernetes-deployment)
- [ArgoCD GitOps](#argocd-gitops)
- [RBAC & GitHub OIDC](#rbac--github-oidc)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Service Overview

| Field | Value |
|-------|-------|
| **Language/Runtime** | Go 1.26.1 |
| **Framework** | Standard `net/http` |
| **Port** | 50051 |
| **Database** | None |
| **Docker Image** | `yampss/hipstershop-shippingservice:{tag}` (dev + prod) |
| **Namespace** | `hipster-backend` |
| **Prod Deploy Method** | GitHub OIDC → kubectl set image (NOT ArgoCD) |

### What This Service Does

ShippingService calculates shipping cost and returns a shipping quote given a list of items (products + quantities). Called by `checkoutservice` during the checkout flow. Shipping rates are calculated from product dimensions/weights embedded in the service code.

### What Makes This Service Unique

ShippingService is the **only microservice** in the platform that deploys to production via a **direct kubectl deploy** using GitHub Actions OIDC authentication — bypassing the ArgoCD GitOps flow used by all other services. A self-hosted runner authenticates to the Kubernetes API server using a GitHub OIDC JWT token, then runs `kubectl set image deployment/shippingservice`.

---

## Architecture Position

```
  [checkoutservice]
       │  GET /getQuote or /shipOrder (via shippingservice:50051)
       ▼
  [shippingservice :50051]
  (pure calculation — no DB)

  [kGateway]
       │  PathPrefix /api/shipping → ReplacePrefixMatch: /
       ▼
  [shippingservice :50051]
```

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/getQuote` | Calculate shipping cost estimate |
| POST | `/shipOrder` | Confirm and process shipment |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- Go 1.26+

```bash
git clone https://github.com/HipsterShopp/hipstershop-shippingservice
cd hipstershop-shippingservice/src
go mod download
```

### Run Locally

```bash
cd src/
go run .
curl http://localhost:50051/_healthz
```

---

## Docker

### Dockerfile Analysis

**File:** `src/Dockerfile`

```dockerfile
FROM golang:1.26.1-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /shippingservice .
FROM gcr.io/distroless/static
WORKDIR /src
COPY --from=builder /shippingservice /src/shippingservice
EXPOSE 50051
ENTRYPOINT ["/src/shippingservice"]
```

Standard Go multi-stage distroless pattern. Port 50051 (gRPC convention, though service uses HTTP).

---

## CI/CD Pipeline

**File:** `.github/workflows/shippingservice.yaml`

**This is the most complex pipeline in the platform.** It has two deploy paths:
1. **Develop:** Standard ArgoCD GitOps flow (helm-updater-job updates `values.dev.yaml`)
2. **Production:** Direct kubectl deploy via GitHub OIDC on a self-hosted runner

### Pipeline Flow

```
push/PR to main or develop
              ↓
  [PR] sonar-and-snyk (Java 21 / Gradle scan)
              ↓
  generate-tag-and-build-image
  [Trivy scan on PR, tag on main]
              ↓ [push event]
  push → DockerHub
              ↓
  develop branch:                  main branch:
  update-helm-values-dev           prod-approval-gate
  (values.dev.yaml on develop)     (manual approval)
  ArgoCD picks up                       ↓
                                   deploy-to-prod
                                   (self-hosted runner)
                                   [GitHub OIDC auth]
                                   kubectl set image
```

### Production Deploy Job (Full Extract)

```yaml
# From .github/workflows/shippingservice.yaml
deploy-to-prod:
  name: Deploy shippingservice to prod cluster
  needs: [prod-approval-gate, generate-tag-and-build-image]
  if: |
    always() &&
    needs.prod-approval-gate.result == 'success'
  runs-on: self-hosted          # Must be a self-hosted runner in the cluster VPC
  permissions:
    id-token: write             # Required to request GitHub OIDC token
    contents: read

  steps:
    - name: Install kubectl
      uses: azure/setup-kubectl@v4

    - name: Request GitHub OIDC token
      id: oidc
      run: |
        TOKEN=$(curl -sSf \
          -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
          "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=kubernetes" \
          | jq -r '.value')
        echo "::add-mask::$TOKEN"
        echo "token=$TOKEN" >> $GITHUB_OUTPUT

    - name: Configure kubectl (OIDC kubeconfig)
      run: |
        echo "${{ secrets.K8S_CA_CERT }}" | base64 -d > ca.crt
        kubectl config set-cluster prod-cluster \
          --server=${{ secrets.K8S_API_SERVER }} \
          --certificate-authority=ca.crt \
          --embed-certs=true
        kubectl config set-credentials github-actions-oidc \
          --token="${{ steps.oidc.outputs.token }}"
        kubectl config set-context prod \
          --cluster=prod-cluster \
          --user=github-actions-oidc
        kubectl config use-context prod

    - name: Deploy new image to prod
      run: |
        TAG="${{ needs.generate-tag-and-build-image.outputs.tag }}"
        IMAGE="yampss/hipstershop-shippingservice:${TAG}"
        kubectl set image deployment/shippingservice \
          server="${IMAGE}" \
          -n hipster-backend

    - name: Wait for rollout to complete
      run: |
        kubectl rollout status deployment/shippingservice \
          -n hipster-backend \
          --timeout=5m

    - name: Show final pod state
      if: always()
      run: |
        kubectl get pods \
          -n hipster-backend \
          -l app=shippingservice \
          -o wide
```

**Why `server=` in kubectl set image?** The container name in the Deployment spec is `server` (not `shippingservice`). This matches all other services in the platform.

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 50051

---

## ArgoCD GitOps

**Dev only:** ArgoCD manages `shippingservice-dev` from `develop` branch.  
**Prod:** ArgoCD Application `shippingservice` (or `shippingservice-prod`) exists but the image tag is NOT updated by `helm-updater-job.yaml` for production — instead, the `deploy-to-prod` job runs `kubectl set image` directly.

**⚠️ IMPORTANT:** Because production shippingservice is deployed via direct kubectl, the ArgoCD application may show `OutOfSync` after a production deploy (the running image doesn't match what's in `values.prod.yaml`). This is a known trade-off of the hybrid approach.

---

## RBAC & GitHub OIDC

GitHub Actions authenticates to the production cluster using OIDC (no stored certificates or tokens):

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
    verbs: ["get", "list", "patch", "update"]  # kubectl set image
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list"]                      # rollout status
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]                      # kubectl get pods
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-shippingservice-verifier
  namespace: hipster-backend
subjects:
  - kind: User
    # OIDC subject: repo + branch constraints
    name: "github:repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main"
    apiGroup: rbac.authorization.k8s.io
```

**How it works:**
1. GitHub Actions runner requests a JWT from GitHub's OIDC endpoint for `audience=kubernetes`
2. kube-apiserver is configured with `--oidc-issuer-url=https://token.actions.githubusercontent.com` and `--oidc-client-id=kubernetes`
3. OIDC token subject claim matches the RoleBinding subject exactly (repo + main branch)
4. Role grants only the minimum permissions needed for the deploy job

### Required Cluster Configuration

The kube-apiserver must have:
```
--oidc-issuer-url=https://token.actions.githubusercontent.com
--oidc-client-id=kubernetes
--oidc-username-claim=sub
--oidc-username-prefix=github:
```

### Required GitHub Secrets

| Secret | Purpose |
|--------|---------|
| `K8S_API_SERVER` | Production cluster API server URL (e.g., `https://1.2.3.4:6443`) |
| `K8S_CA_CERT` | Cluster CA certificate (base64 encoded) |

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `PORT` | No | `50051` | ConfigMap `service-ports` → `SHIPPINGSERVICE_PORT` |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |

---

## Troubleshooting

**Issue: GitHub OIDC token request fails**
```bash
# Error: "Unable to get ACTIONS_ID_TOKEN_REQUEST_TOKEN"
# Fix: Ensure workflow has: permissions: id-token: write
# Check the self-hosted runner is running and accessible
```

**Issue: kubectl set image fails — Unauthorized**
```bash
# Verify RoleBinding exists
kubectl get rolebinding github-actions-shippingservice-verifier -n hipster-backend

# Check OIDC token subject matches RoleBinding
# Token subject should be: github:repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main

# Verify kube-apiserver OIDC config
kubectl get --raw /openid/v1/jwks
```

**Issue: Rollout timeout (5m)**
```bash
kubectl rollout status deployment/shippingservice -n hipster-backend
kubectl describe deployment shippingservice -n hipster-backend
kubectl get pods -n hipster-backend -l app=shippingservice
kubectl logs -l app=shippingservice -n hipster-backend --previous
```

**Issue: ArgoCD shows OutOfSync for shippingservice after prod deploy**
```bash
# This is expected — kubectl set image updates the running spec but not values.prod.yaml
# Options:
# 1. Accept OutOfSync (monitor-only, no auto-sync for prod)
# 2. Manually update values.prod.yaml to match deployed image tag
```

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=shippingservice
kubectl logs -l app=shippingservice -n hipster-backend --tail=100
kubectl port-forward svc/shippingservice 50051:50051 -n hipster-backend
curl http://localhost:50051/_healthz
kubectl rollout status deployment/shippingservice -n hipster-backend
```
