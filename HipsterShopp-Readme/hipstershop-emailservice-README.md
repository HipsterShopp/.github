# hipstershop-emailservice

> Python 3.11 microservice that sends order confirmation emails via Gmail SMTP, triggered by the checkout flow.

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
| **Framework** | Custom HTTP server (`email_server.py`) |
| **Port** | 8080 |
| **Database** | None (logs to MongoDB `notification_db` — see email_server.py) |
| **Docker Image** | `yampss/hipstershop-emailservice:{tag}` (dev) |
| **Namespace** | `hipster-backend` |

### What This Service Does

EmailService sends order confirmation emails to customers after a successful checkout. It is called by `checkoutservice` via the internal service address `emailservice:8080`. The service uses Gmail's SMTP server (port 587, STARTTLS) to deliver emails. It renders HTML email templates from the `templates/` directory and logs notification events.

A custom `logger.py` provides structured logging. The service also optionally persists email delivery events to MongoDB `notification_db`.

### What This Service Does NOT Do

EmailService does not handle authentication. It does not initiate sending — it only responds to requests from `checkoutservice`. It does not queue emails (synchronous delivery per request).

---

## Architecture Position

```
  [checkoutservice]
       │  POST /send-confirmation
       ▼
  [emailservice :8080]
       │
       ├── Gmail SMTP :587 (STARTTLS) → [External Gmail servers]
       └── Optional: MongoDB notification_db

  [kGateway]
       │  POST /api/email/send-confirmation → URLRewrite: /send-confirmation
       ▼
  [emailservice :8080]
```

**Network Policy Note:** Email service has a special NetworkPolicy `emailservice-allow-smtp` that explicitly grants egress to `0.0.0.0/0:587` — the only service in the platform permitted to make internet egress requests (for Gmail SMTP).

**Receives traffic from:**
| Source | Path | Why |
|--------|------|-----|
| checkoutservice | `/send-confirmation` | Send order confirmation after checkout |
| kGateway | `/api/email/send-confirmation` → `/send-confirmation` | Direct gateway access |

---

## API Reference

| Method | Path | Description |
|--------|------|-------------|
| POST | `/send-confirmation` | Send order confirmation email |
| GET | `/_healthz` | Health check |

---

## Local Development

### Prerequisites

- Python 3.11
- Gmail account with App Password (or SMTP credentials)

```bash
git clone https://github.com/HipsterShopp/hipstershop-emailservice
cd hipstershop-emailservice/src
pip install -r requirements.txt
```

### Environment Setup

```env
EMAIL_SENDER=your-gmail@gmail.com
EMAIL_PASSWORD=your-gmail-app-password
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
NOTIFICATION_MONGO_URI=  # optional
ENABLE_PROFILER=1
```

### Run Locally

```bash
cd src/
python email_server.py
```

### Run with Docker

```bash
docker build -t hipstershop-emailservice:local src/
docker run -p 8080:8080 \
  -e EMAIL_SENDER=your@gmail.com \
  -e EMAIL_PASSWORD=your-app-password \
  hipstershop-emailservice:local
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

**Build Strategy:** Multi-stage Python. Builder installs `g++` and `linux-headers` to compile native Python packages. Runtime only needs `libstdc++` (C++ standard library for runtime linking). Python libraries are copied from builder to runtime.

**Important Decisions:**
- `PYTHONDONTWRITEBYTECODE=1`: No `.pyc` bytecode files generated.
- `PYTHONUNBUFFERED=1`: All Python stdout/stderr written immediately to container logs.
- `ENABLE_PROFILER=1`: Profiling enabled at runtime (can be overridden via env).
- `WORKDIR /email_server`: Service runs from this directory (templates/ must be inside it).

---

## CI/CD Pipeline

**File:** `.github/workflows/emailservice.yaml`  
**Trigger:** Push/PR to `main`/`develop` on `src/**` changes

```yaml
sonar-and-snyk:
  uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
  with:
    service_name: emailservice
    service_path: src/
    runtime: python
    python-version: "3.11"
```

Snyk scan command:
```bash
pip install -r requirements.txt
snyk test --file=requirements.txt --package-manager=pip --severity-threshold=high
```

---

## Kubernetes Deployment

**Namespace:** `hipster-backend`  
**Port:** 8080

### Special Network Policy

The emailservice has a dedicated NetworkPolicy that allows SMTP egress:

```yaml
# From backend-common/templates/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emailservice-allow-smtp
  namespace: hipster-backend
spec:
  podSelector:
    matchLabels:
      app: emailservice       # Only emailservice
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0      # Internet access allowed ONLY for emailservice
    ports:
    - protocol: TCP
      port: 587               # Gmail SMTP STARTTLS port
```

All other backend pods are blocked from internet egress by `default-deny-all`.

---

## ArgoCD GitOps

**ArgoCD Application:** `emailservice` / `emailservice-dev`  
**Source Path:** `hipstershop-micro-apps/emailservice`

---

## Configuration

### Environment Variables

| Variable | Required | Description | Set Via |
|----------|----------|-------------|---------|
| `EMAIL_SENDER` | Yes | Gmail sender address | Secret or env |
| `EMAIL_PASSWORD` | Yes | Gmail App Password | Secret `app-secrets` or env |
| `SMTP_HOST` | No | SMTP server (default: smtp.gmail.com) | Env or code default |
| `SMTP_PORT` | No | SMTP port (587=STARTTLS) | Env or code default |
| `ENABLE_PROFILER` | No | `1` = enable profiling | Dockerfile ENV |
| `ENABLE_TRACING` | No | `"1"` | ConfigMap `hipster-config` |

### Secrets Used

| Secret Name | Key | Purpose |
|------------|-----|---------|
| `app-secrets` | (email credentials) | Gmail authentication |
| `mongodb-users` | `NOTIFICATION_MONGO_USERNAME` | MongoDB notification_db |
| `mongodb-users` | `NOTIFICATION_MONGO_PASSWORD` | MongoDB notification_db |

---

## Troubleshooting

**Issue: Emails not being sent**
```bash
kubectl logs -l app=emailservice -n hipster-backend --tail=100
# Look for: SMTP authentication error, connection refused

# Test SMTP from within the cluster (only emailservice is allowed port 587)
# Other pods will be blocked by NetworkPolicy
kubectl exec -it <emailservice-pod> -n hipster-backend -- \
  python -c "import smtplib; s=smtplib.SMTP('smtp.gmail.com', 587); s.ehlo(); s.starttls(); print('SMTP OK')"
```

**Issue: NetworkPolicy blocks SMTP**
```bash
# The emailservice-allow-smtp policy must exist
kubectl get networkpolicy emailservice-allow-smtp -n hipster-backend
kubectl describe networkpolicy emailservice-allow-smtp -n hipster-backend
```

**Issue: Gmail authentication failure**
```bash
# Verify App Password is set correctly (Gmail requires App Passwords when 2FA enabled)
# Check secret in cluster
kubectl get secret app-secrets -n hipster-backend -o yaml
```

**Issue: Checkout succeeds but email not sent**

This is expected behavior. CheckoutService calls emailservice but does NOT block checkout on email failure. EmailService errors are logged but do not propagate to the caller.

### General Debug Commands

```bash
kubectl get pods -n hipster-backend -l app=emailservice
kubectl logs -l app=emailservice -n hipster-backend --tail=100
kubectl port-forward svc/emailservice 8080:8080 -n hipster-backend
curl http://localhost:8080/_healthz
```
