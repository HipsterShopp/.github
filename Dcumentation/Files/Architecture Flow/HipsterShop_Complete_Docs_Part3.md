---
# HipsterShop Platform Documentation — Part 3
---

## 17. Disaster Recovery & Backup

### 17.1 Current State

| Data | Location | Backup | RPO | RTO |
|------|---------|--------|-----|-----|
| MongoDB data | NFS PVCs (5Gi × 3) | ❌ No automated backup | Undefined | Undefined |
| Git state (configs) | GitHub repos | ✅ Git history | Near-zero | Minutes |
| Docker images | DockerHub + GHCR | ✅ Registry redundancy | Near-zero | Minutes |
| Helm charts | GitHub + ArgoCD | ✅ Git history | Near-zero | Minutes |
| Sealed Secrets | Git (encrypted) | ✅ Git + private key safe | Near-zero | Minutes |

### 17.2 MongoDB Backup Strategy (Recommended)

```yaml
# CronJob: Daily MongoDB backup to S3
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: hipster-database
spec:
  schedule: "0 2 * * *"         # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mongo:7.0
            command:
            - sh
            - -c
            - |
              mongodump \
                --host mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017 \
                --username $MONGO_ROOT_USERNAME \
                --password $MONGO_ROOT_PASSWORD \
                --authenticationDatabase admin \
                --out /backup/$(date +%Y%m%d)
              # Upload to S3 or NFS backup location
            envFrom:
            - secretRef:
                name: mongodb-root
          restartPolicy: OnFailure
```

### 17.3 Restore Procedure

```bash
# 1. Restore from mongodump
kubectl exec -it mongodb-0 -n hipster-database -- bash
mongorestore \
  --host mongodb-0.mongo-headless.hipster-database.svc.cluster.local:27017 \
  --username admin --password <root-password> \
  --authenticationDatabase admin \
  /backup/20260422/

# 2. Rollback ArgoCD to previous version
argocd app history adservice-prod
argocd app rollback adservice-prod <history-id>

# 3. Rollback frontend blue-green
kubectl argo rollouts undo frontend -n hipster-frontend

# 4. Rollback Kubernetes deployment directly
kubectl rollout undo deployment/adservice -n hipster-backend
kubectl rollout history deployment/adservice -n hipster-backend
```

### 17.4 Cluster Rebuild Checklist

```bash
# 1. Re-install ArgoCD
# 2. Re-install Sealed Secrets controller
# 3. Re-fetch cluster public cert for re-sealing
kubeseal --fetch-cert --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system > pub-cert.pem

# 4. Re-seal secrets with new cluster key
bash generate-sealed-secrets.sh
git add backend-common-secrets.yaml mongodb-secrets.yaml
git commit -m "chore: reseal for new cluster" && git push

# 5. Re-apply ArgoCD bootstrap sequence (waves 0→3)
# 6. Re-configure GitHub OIDC in kube-apiserver flags
# 7. Re-add GitHub secrets: K8S_API_SERVER, K8S_CA_CERT
```

---

## 18. Runbook — Operations Guide

### 18.1 Standard Deploy Flow

```bash
# Dev (automatic after PR merge to develop)
# 1. Developer merges PR to develop
# 2. GitHub Actions: build → push DockerHub → update values.dev.yaml
# 3. ArgoCD: auto-detects → auto-syncs → rolling update
# Verify:
argocd app get <service>-dev
kubectl rollout status deployment/<service> -n hipster-backend

# Prod (requires manual approval)
# 1. Developer merges PR to main
# 2. GitHub Actions: build → push GHCR → PAUSE for approval
# 3. Team lead approves GitHub Environment "production"
# 4. Actions: update values.prod.yaml
# 5. ArgoCD: auto-syncs → rolling update (or blue-green for frontend)
# Verify:
argocd app get <service>-prod
kubectl get deployment <service> -n hipster-backend \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 18.2 Frontend Blue-Green Promotion

```bash
# 1. CI pushes new image tag → values.prod.yaml updated → ArgoCD syncs Rollout
# 2. Argo Rollouts creates GREEN ReplicaSet
# 3. Green pods pass readiness → attached to frontend-preview service

# Check rollout status
kubectl argo rollouts get rollout frontend -n hipster-frontend --watch

# Preview new version (access frontend-preview service)
kubectl port-forward svc/frontend-preview 8080:80 -n hipster-frontend
curl http://localhost:8080/_healthz

# Promote GREEN to live traffic
kubectl argo rollouts promote frontend -n hipster-frontend

# Emergency abort (if green has issues)
kubectl argo rollouts abort frontend -n hipster-frontend

# Rollback (revert to blue)
kubectl argo rollouts undo frontend -n hipster-frontend
```

### 18.3 Emergency Rollback Procedures

```bash
# Option 1: ArgoCD rollback (preferred — reverts git state)
argocd app history adservice-dev
argocd app rollback adservice-dev <revision-id>

# Option 2: Kubernetes rollback (immediate — doesn't update git)
kubectl rollout undo deployment/adservice -n hipster-backend
kubectl rollout status deployment/adservice -n hipster-backend

# Option 3: Manual image tag update
# Edit hipstershop-micro-apps/adservice/values.dev.yaml
# Change image tag to last known good
git commit -m "revert: rollback adservice to adservice-v0.1.1"
git push  # ArgoCD picks up and syncs

# Option 4: shippingservice rollback (OIDC deploy)
# Must run new deployment manually or trigger CI again
kubectl set image deployment/shippingservice \
  server=yampss/hipstershop-shippingservice:<previous-tag> \
  -n hipster-backend
```

### 18.4 Scaling Operations

```bash
# Manual scale (temporary — ArgoCD will revert to values.yaml replica count)
kubectl scale deployment/adservice --replicas=3 -n hipster-backend

# Permanent scale — update values.yaml in helm-charts and push
# Edit: hipstershop-micro-apps/adservice/values.yaml
# global.replicas.adservice: 3
git commit -m "scale: increase adservice replicas to 3"
git push   # ArgoCD syncs

# Restart a deployment (e.g., to pick up ConfigMap changes)
kubectl rollout restart deployment/adservice -n hipster-backend

# Force ArgoCD re-sync
argocd app sync adservice-dev --force
```

### 18.5 Secret Rotation

```bash
# 1. Generate new secret value
NEW_JWT_SECRET=$(openssl rand -base64 32)

# 2. Re-seal
cd hipstershop-helm-charts/sealed-secrets/
export JWT_SECRET="$NEW_JWT_SECRET"
# ... export other secrets unchanged
bash generate-sealed-secrets.sh

# 3. Commit and push new sealed secrets
git add backend-common-secrets.yaml
git commit -m "chore: rotate JWT_SECRET"
git push

# 4. Also update GitHub Secrets if needed (for CI workflows)
# Go to each service repo → Settings → Secrets → Update

# 5. ArgoCD will sync new SealedSecret → new Kubernetes Secret
# 6. Bounce authservice to pick up new JWT_SECRET
kubectl rollout restart deployment/authservice -n hipster-backend
```

---

## 19. Troubleshooting Guide

### 19.1 Pod States Reference

| State | Meaning | First Action |
|-------|---------|-----------|
| `Pending` | Pod can't be scheduled | `kubectl describe pod <name> -n hipster-backend` — check Events |
| `ImagePullBackOff` | Can't pull image from registry | Check image tag exists on DockerHub/GHCR |
| `CrashLoopBackOff` | Container starts then crashes | `kubectl logs <pod> -n hipster-backend --previous` |
| `OOMKilled` | Out of memory | Increase memory limit in values.yaml |
| `Running 0/1` | Pod running but not Ready | Readiness probe failing — check logs + `/_healthz` endpoint |

### 19.2 Issue: CrashLoopBackOff

```bash
# Step 1: Get crash reason
kubectl logs <pod-name> -n hipster-backend --previous

# Step 2: Check exit code
kubectl describe pod <pod-name> -n hipster-backend | grep -A3 "Last State"

# Step 3: Check events
kubectl get events -n hipster-backend --sort-by='.lastTimestamp' | tail -20

# Common causes:
# authservice: "Required environment variable MONGO_URI is not set"
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend

# adservice: OOM killed — increase memory limit
kubectl top pod -l app=adservice -n hipster-backend

# paymentservice: MongoDB refused — service continues but check logs
kubectl logs -l app=paymentservice -n hipster-backend | head -20
```

### 19.3 Issue: ImagePullBackOff

```bash
kubectl describe pod <pod-name> -n hipster-backend | grep -A5 "Events:"
# "Failed to pull image ..."

# Check if image exists on DockerHub
docker pull yampss/hipstershop-adservice:develop-abc12345

# Check ArgoCD rendered the right image
argocd app manifests adservice-dev | grep image:

# Check the actual deployment
kubectl get deployment adservice -n hipster-backend \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### 19.4 Issue: ArgoCD OutOfSync (Never Heals)

```bash
# Check what is different
argocd app diff adservice-dev

# Common cause: controller modifies fields ArgoCD keeps reverting
# For gateway — add ignoreDifferences:
ignoreDifferences:
- group: gateway.networking.k8s.io
  kind: GatewayClass
  jsonPointers: [/metadata/annotations, /metadata/labels, /spec/parametersRef]

# For frontend blue-green — add ignoreDifferences on prod AppSet:
ignoreDifferences:
- group: ""
  kind: Service
  name: frontend-active
  namespace: hipster-frontend
  jsonPointers: [/spec/selector]
```

### 19.5 Issue: Service Unreachable (502/503 via Gateway)

```bash
# Step 1: Check if pod is Running and Ready
kubectl get pods -n hipster-backend -l app=<service>

# Step 2: Check if Service has Endpoints (pods registered)
kubectl get endpoints <service> -n hipster-backend
# Should show: ADDRESSES not empty

# Step 3: Check HTTPRoute status
kubectl get httproute -n hipster-backend
kubectl describe httproute <route-name> -n hipster-backend

# Step 4: Check NetworkPolicy (is traffic blocked?)
kubectl describe networkpolicy -n hipster-backend

# Step 5: Test from inside cluster
kubectl exec -it <any-running-pod> -n hipster-backend -- \
  curl http://<service>:<port>/_healthz
```

### 19.6 Issue: MongoDB Pod Pending

```bash
kubectl describe pod mongodb-0 -n hipster-database
# Look for: "no persistent volumes available for this claim"
# OR: "node had taints that the pod didn't tolerate"

# Check PVC status
kubectl get pvc -n hipster-database

# Check if NFS StorageClass exists
kubectl get storageclass nfs

# Check NFS provisioner
kubectl get pods -n default | grep nfs    # or wherever provisioner runs

# Fix if StorageClass missing: re-install NFS provisioner (see section 5.4)
```

### 19.7 Issue: Sealed Secret Not Decrypting

```bash
# Check controller is running
kubectl get pods -n kube-system | grep sealed-secrets

# Check for events on the SealedSecret
kubectl describe sealedsecret app-secrets -n hipster-backend

# Check controller logs
kubectl logs -l app.kubernetes.io/name=sealed-secrets -n kube-system --tail=50

# Common cause: sealed with different cluster key (after cluster rebuild)
# Fix: re-seal all secrets with new cluster public key (see section 17.4)
```

### 19.8 Issue: GitHub Actions Fails on Docker Push

```bash
# Error: unauthorized: authentication required
# 1. Check DOCKERHUB_TOKEN is set in repo secrets
# 2. Token must have Read & Write scope
# 3. Regenerate at hub.docker.com → Account Settings → Security → New Access Token
# 4. Update GitHub secret in repo Settings → Secrets and variables → Actions
```

### 19.9 Issue: helm-updater-job Fails

```bash
# Error: Permission denied
# HELM_CHARTS_TOKEN must have write access to hipstershop-helm-charts repo
# Scope: repo (full access) or at minimum: contents:write
# Regenerate at GitHub → Settings → Developer settings → Personal access tokens

# Error: Nothing to commit
# Image tag is the same (CI ran but image unchanged)
# This is harmless — sed found no change, git exit 0
```

### 19.10 Issue: authservice Crashes on Startup

```bash
kubectl logs -l app=authservice -n hipster-backend --previous
# "Required environment variable MONGO_URI is not set" → secrets missing
# "Required environment variable JWT_SECRET is not set" → secrets missing

# Verify sealed secrets were applied and decrypted
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend

# If secrets missing, apply sealed secrets file
kubectl apply -f hipstershop-helm-charts/sealed-secrets/backend-common-secrets.yaml

# Verify ArgoCD sealed-secrets app is Synced
argocd app get sealed-secrets
```

---

## 20. Challenges & Solutions

| # | Challenge | Root Cause | Solution |
|---|-----------|-----------|---------|
| 1 | `CrashLoopBackOff` on adservice | OOM killed — JVM heap exceeded 256Mi limit | Increased memory limit to 512Mi to accommodate JVM + OTel agent |
| 2 | `exec format error` in adservice CI | gradlew committed with Windows CRLF; Linux can't execute CRLF scripts | Added `sed -i 's/\r$//' gradlew && chmod +x gradlew` to Dockerfile |
| 3 | `no such file or directory` in distroless Go images | CGO was enabled → dynamically linked binary; distroless has no libc | Added `CGO_ENABLED=0` to all Go build commands |
| 4 | ArgoCD perpetually `OutOfSync` for gateway | kGateway controller mutates GatewayClass annotations; ArgoCD keeps reverting | Added `ignoreDifferences` for GatewayClass annotations/labels |
| 5 | `helm list` returns empty — "nothing deployed" | ArgoCD uses `helm template + kubectl apply`, not `helm install` | Documented expected behavior; use `argocd app list` instead |
| 6 | ShippingService prod deploy — "Unauthorized" | OIDC JWT sub claim didn't match RoleBinding subject exactly | Corrected subject to: `github:repo:HipsterShopp/hipstershop-shippingservice:ref:refs/heads/main` |
| 7 | MongoDB pod Pending — no PVC bound | NFS StorageClass not provisioned before StatefulSet deployment | Pre-provision NFS StorageClass as part of cluster bootstrap (wave 0 prerequisite) |
| 8 | ARM race: frontend ArgoCD + Argo Rollouts fighting over Service selector | Rollouts controller modifies Service selector during promotion; ArgoCD reverts | Added `ignoreDifferences` for frontend-active/frontend-preview Service selectors |
| 9 | MongoDB authservice init-container timing | authservice tries to connect before MongoDB primary is elected | Added init-container that polls MongoDB until a writable primary is available |
| 10 | PR CI running for non-src changes | Workflow triggered on any push, rebuilding even when only README changed | Added `paths: ['src/**']` filter to GitHub Actions triggers |
| 11 | 12 repos × 200-line CI files = maintenance nightmare | No shared CI library — every repo had its own full pipeline YAML | Created `hipstershop-shared-workflows` with 5 reusable workflows; callers are 30-100 lines |
| 12 | Sealed Secrets incompatible after cluster rebuild | Sealed Secrets are encrypted with cluster-specific RSA key pair | Documented re-sealing process; `generate-sealed-secrets.sh` automates it |
| 13 | currencyservice random errors confusing checkout debugging | `CURRENCY_SERVICE_RANDOM_ERROR: "1"` is intentional chaos | Added documentation — this is intentional; set to "0" for debugging |

---

## 21. Architecture Decision Records

### ADR-001: Kubernetes over Docker Swarm / VMs

**Decision:** Use Kubernetes for container orchestration.

**Reason:**  
- Industry standard for production microservices
- Self-healing via liveness probes and restart policies
- Rolling updates without downtime
- HPA for CPU/memory-based autoscaling
- Native support for StatefulSets (required for MongoDB)
- Rich ecosystem: ArgoCD, Sealed Secrets, Argo Rollouts all require Kubernetes

**Alternative Considered:** Docker Swarm — simpler but lacks StatefulSets, HPA, Gateway API, and the ecosystem tools used here.

**Trade-off:** Significantly higher operational complexity than Docker Swarm. New engineers require Kubernetes training.

---

### ADR-002: Helm for Kubernetes Manifest Templating

**Decision:** Use Helm v3 as the Kubernetes manifest templating system.

**Reason:**  
- Eliminates 24+ manually written YAML files (12 services × Deployment + Service)
- Centralizes all configuration in `values.yaml`
- Three-layer override system (base / dev / prod) maps cleanly to environments
- ArgoCD has native Helm support — renders `helm template` output without `helm install`
- Versioned chart releases support rollbacks

**Alternative Considered:** Kustomize — more declarative but less powerful for multi-environment value management.

**Trade-off:** Helm's templating language (Go templates) has a learning curve. `helm list` returning empty confuses engineers unfamiliar with ArgoCD + Helm integration.

---

### ADR-003: ArgoCD with ApplicationSet for GitOps

**Decision:** Use ArgoCD with ApplicationSet-driven deployment instead of Flux or push-based CD.

**Reason:**  
- Declarative GitOps — git is the only source of truth
- `selfHeal: true` automatically reverts any manual cluster changes
- ApplicationSet generates all 12 Application manifests from one template
- Sync waves ensure correct deployment order (database → config → gateway → services)
- Web UI provides visual sync status for all services

**Alternative Considered:** Flux v2 — similar capabilities but less mature web UI; ArgoCD team had more experience with it.

**Trade-off:** ArgoCD adds operational complexity (separate server to maintain). `helm list` shows no releases — confuses engineers used to direct Helm deployment.

---

### ADR-004: Shared GitHub Actions Workflows

**Decision:** Create `hipstershop-shared-workflows` repository with 5 shared workflows. All 12 service repos call these shared workflows via `uses:` reference.

**Reason:**  
- Without shared workflows: 12 repos × 200-line YAML = 2,400 lines of duplicated CI config
- Any tool version update (Trivy, SonarQube scanner) requires 12 PRs
- With shared workflows: one PR to shared repo propagates to all 12 services

**Trade-off:** Pipeline debugging requires looking at two repos (service repo for trigger, shared-workflows for logic). Adds coupling on `feature/loose-coupling` branch name.

---

### ADR-005: GitHub OIDC for ShippingService Production Deploy

**Decision:** ShippingService production deployment uses GitHub OIDC JWT authentication directly to kubectl — not through ArgoCD.

**Reason:**  
- Demonstrates zero-credential production deployment pattern
- No stored kubeconfig or long-lived tokens in GitHub Secrets
- JWT expires in ~5 minutes — minimal attack window
- RBAC RoleBinding locked to exact repo + exact branch

**Trade-off:** ShippingService ArgoCD Application may show `OutOfSync` after production deploy (running image differs from values.prod.yaml). Hybrid approach adds complexity.

---

### ADR-006: Sealed Secrets for Git-Safe Secret Management

**Decision:** Use Bitnami Sealed Secrets to encrypt Kubernetes Secrets before committing to git.

**Reason:**  
- Cannot store plain Kubernetes Secret manifests in git (base64 is not encryption)
- Sealed Secrets encrypts with cluster RSA public key — only the specific cluster can decrypt
- Safe to commit encrypted SealedSecret objects to public or private repos
- Simple workflow: `kubeseal --cert cluster.pem < secret.yaml > sealed-secret.yaml`

**Alternative Considered:** External Secrets Operator with AWS Secrets Manager — more powerful (rotation, audit trail) but significantly more complex setup.

**Trade-off:** Cluster-specific — sealed secrets must be re-generated if cluster RSA key rotates or cluster is rebuilt.

---

### ADR-007: Multi-Repo Architecture (14 repos vs 1 monorepo)

**Decision:** 14 separate repositories — one per microservice, plus shared-workflows, plus helm-charts.

**Reason:**  
- Monorepo with 12 services: any code change triggers CI for ALL services simultaneously
- Independent repos: CI only runs when that service's `src/**` changes
- Separate repos enable per-service access control (payment team doesn't need frontend access)
- ArgoCD monitors `hipstershop-helm-charts` only — not individual service repos

**Trade-off:** More repos to manage. Cross-service changes require coordinating multiple PRs. Developer onboarding requires cloning multiple repos.

---

### ADR-008: MongoDB Replica Set (3 nodes) over Single Instance

**Decision:** Deploy MongoDB as a 3-node replica set (`rs0`) using StatefulSet.

**Reason:**  
- Single MongoDB instance = single point of failure for all services
- Replica set provides automatic failover when primary fails
- Read scaling — replica nodes can serve read queries
- Services authenticate with SCRAM-SHA-256 using per-service dedicated databases and credentials

**Trade-off:** 3× the resource consumption. NFS storage required (not hostPath). MongoDB replica set initialization complexity (init job required).

---

## 22. Environment Strategy

### 22.1 Three-Environment Model

| | Dev | Prod |
|--|-----|------|
| Branch | `develop` | `main` |
| Image registry | DockerHub (`yampss/`) | DockerHub + GHCR (`ghcr.io/hipstershopp/`) |
| Image tag format | `develop-{sha8}` | `service-vX.Y.Z` |
| Resource requests | 50% of prod | Full |
| Argo Rollouts | No (Deployment) | Yes (frontend Blue-Green) |
| ArgoCD prune | Yes | No |
| Human approval gate | No | Yes |
| shippingservice deploy | ArgoCD (values.dev.yaml) | GitHub OIDC kubectl |

### 22.2 Promoting from Dev to Prod

```bash
# 1. Verify dev environment is healthy
argocd app list   # All services Synced + Healthy on dev cluster
kubectl get pods -n hipster-backend    # All Running

# 2. Create PR from develop → main
# GitHub Actions: sonar + snyk + build + trivy + push DockerHub

# 3. Team lead reviews and approves PR
# GitHub Actions: push-to-ghcr → PAUSE at prod-approval-gate

# 4. Team lead approves GitHub Environment "production"
# GitHub Actions: update-helm-values-prod → ArgoCD syncs prod cluster

# 5. Verify prod
argocd app list --server <prod-argocd-server>
kubectl get pods -n hipster-backend   # All Running on prod cluster
```

---

## 23. Release & Versioning Strategy

### 23.1 Semantic Versioning

All services follow semantic versioning with service-name prefix:
- **Format:** `{service-name}-vMAJOR.MINOR.PATCH`
- **Example:** `adservice-v0.1.3`, `frontend-v1.2.0`

Version is calculated automatically from conventional commit messages:
- `feat:` → MINOR bump (`v0.1.x` → `v0.2.0`)
- `fix:` → PATCH bump (`v0.1.2` → `v0.1.3`)
- `feat!:` or `BREAKING CHANGE:` → MAJOR bump

### 23.2 Release Lifecycle

```
feature/* branch (developer)
        │ PR to develop
        ▼
develop branch → CI build → DockerHub (develop-{sha8}) → Dev cluster
        │ PR to main (tech lead review)
        ▼
main branch → CI build → DockerHub + GHCR (service-vX.Y.Z) → Prod approval
        │ human approval
        ▼
Production cluster updated
        │ git tag created automatically
        ▼
argocd app history shows versioned release (rollback-able)
```

### 23.3 Git Tag Conventions

```bash
# Service-specific tags (created automatically by push.yaml on main)
adservice-v0.1.3
frontend-v1.2.0
authservice-v0.3.1

# View all tags for a service
git tag --list "adservice-*" | sort -V

# Rollback to previous tag
argocd app history adservice-prod
argocd app rollback adservice-prod <revision>
```

---

## 24. Incident Management & SLA

### 24.1 Severity Levels

| Severity | Description | Example | Response Time | Resolution Target |
|----------|------------|---------|--------------|-----------------|
| P1 — Critical | Full platform outage | Gateway down, all services unreachable | ≤ 15 minutes | ≤ 2 hours |
| P2 — High | Core feature unavailable | Checkout broken, MongoDB primary down | ≤ 30 minutes | ≤ 4 hours |
| P3 — Medium | Non-critical service degraded | emailservice down, recommendations unavailable | ≤ 2 hours | ≤ 24 hours |
| P4 — Low | Minor issues | Slow AI assistant, currency random errors | Next business day | ≤ 72 hours |

### 24.2 Incident Response Runbook

```bash
# P1: Full Platform Outage

# 1. Check gateway
kubectl get pods -n kgateway-system
kubectl get gateway hipstershop-gateway -n hipster-backend -o yaml

# 2. Check all pods
kubectl get pods -n hipster-backend
kubectl get pods -n hipster-frontend
kubectl get pods -n hipster-database

# 3. Check ArgoCD sync status
argocd app list

# 4. Check recent deployments (was a bad deploy the cause?)
kubectl rollout history deployment/frontend -n hipster-frontend
argocd app history frontend-prod

# 5. If bad deploy: rollback
argocd app rollback <app-name> <revision>
# OR:
kubectl rollout undo deployment/<service> -n hipster-backend

# 6. Check node health
kubectl get nodes
kubectl describe node <node-name> | grep -A10 Conditions

# P2: MongoDB Primary Down
kubectl get pods -n hipster-database
kubectl exec -it mongodb-0 -n hipster-database -- \
  mongosh --eval "rs.status()" --username admin --password <root-pw> --authenticationDatabase admin
# Should show PRIMARY + 2 SECONDARYs
# If no primary: force re-election
kubectl exec -it mongodb-0 -n hipster-database -- \
  mongosh --eval "rs.reconfig({...}, {force: true})"
```

### 24.3 Alerting (Current / Recommended)

**Current:** SendGrid email on CI pipeline failure only (via `notify.yaml` shared workflow).

**Recommended additional alerts:**
```yaml
# Prometheus AlertManager rules (to implement)
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
  for: 5m
  annotations:
    summary: "Pod {{ $labels.pod }} is crash looping"

- alert: MongoDBReplicaSetDegraded
  expr: mongodb_rs_members_health{state!="1"} < 2
  for: 2m
  annotations:
    summary: "MongoDB replica set has < 2 healthy members"

- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
  for: 5m
  annotations:
    summary: "Error rate > 5% on {{ $labels.service }}"
```

---

## 25. Non-Functional Requirements

| NFR | Target | Current Implementation | Status |
|-----|--------|----------------------|--------|
| Availability | ≥ 99.9% | Rolling updates + MongoDB replica set + ArgoCD self-healing | ✅ Design supports |
| Scalability | HPA 1→5× frontend | HPA configured (CPU 70%, Memory 80%) | ✅ Implemented |
| Security | RBAC + OIDC + Sealed Secrets | Network policies + Sealed Secrets + RBAC + distroless | ✅ Implemented |
| Performance | < 500ms p99 API response | Distroless Go services (fast); Java 60s startup | ⚠️ Java needs probe tuning |
| Reliability | Self-healing on probe failure | Liveness + readiness probes on all 12 services | ✅ Implemented |
| Auditability | Git history for all changes | All changes via git; ArgoCD audit log; CI logs | ✅ Implemented |
| Observability | Full request tracing | OpenTelemetry → Jaeger (partial); kubectl logs | ⚠️ Prometheus/Grafana pending |
| DR | RTO < 1 hour | ArgoCD re-sync restores services; MongoDB backup pending | ⚠️ Backup not automated |
| Data Isolation | Per-service databases | 7+ isolated MongoDB databases with per-service credentials | ✅ Implemented |

---

## 26. Cost & Resource Optimization

### 26.1 Current Optimizations

| Optimization | Example | Savings |
|-------------|---------|--------|
| Multi-stage Docker builds | All services — runtime image 60-80% smaller than build image | Faster pulls, lower registry storage |
| Distroless runtime images | All Go services — ~10MB runtime image | Faster pulls, minimal CVEs |
| Dev resource halving | CPU 200m→100m, Memory 180Mi→90Mi in values.dev.yaml | ~50% resource savings on dev cluster |
| Layer caching | go.mod/package.json copied before src — deps cached | Faster CI builds on unchanged deps |
| Single gateway (vs per-service LB) | kGateway replaces 12 LoadBalancer services | 11 fewer cloud load balancers |
| ApplicationSet (vs 12 Apps) | One ArgoCD ApplicationSet generates all 12 apps | Operational simplicity |

### 26.2 Recommended Optimizations

- **Enable HPA for backend services** — productcatalogservice and checkoutservice can benefit from autoscaling during peak traffic
- **Implement Pod Disruption Budgets** — prevent accidental full-service outage during node maintenance
- **Add Pod Anti-Affinity** — spread replicas across nodes for true HA
- **Consider multi-stage builds for assistantservice** — currently single-stage; add builder for pip dependency compilation
- **MongoDB connection pooling audit** — verify all services use connection pooling efficiently

---

## 27. Future Enhancements

| Enhancement | Current Gap | Recommended Solution | Effort | Impact |
|------------|------------|---------------------|--------|--------|
| Prometheus + Grafana | No metrics/alerting | kube-prometheus-stack Helm chart + ServiceMonitors | Medium | High |
| Grafana Loki | kubectl logs only | Loki + Promtail DaemonSet | Medium | High |
| External Secrets Operator | Sealed Secrets (manual re-seal) | ESO + AWS Secrets Manager / HashiCorp Vault | High | High |
| HPA for backend services | Frontend HPA only | Add HPA for productcatalogservice, checkoutservice | Low | Medium |
| Pod Anti-Affinity | Single-node risk | `podAntiAffinity` in Helm templates | Low | Medium |
| Pod Disruption Budgets | No maintenance protection | PDB for MongoDB and frontend | Low | Medium |
| TLS at Gateway | HTTP only | cert-manager + Let's Encrypt + TLS listener in kGateway | Medium | High |
| Trivy config scan | Image scan only | Add `trivy config` scan on Helm templates in CI | Low | Medium |
| MongoDB automated backup | Manual only | Velero + MongoDB CronJob → S3 | Medium | High |
| Istio service mesh | No mTLS | Istio sidecar injection + mTLS policies | High | High |
| Cost optimization | Fixed resources | KEDA + custom metrics autoscaling | High | Medium |
| Canary deployments | Blue-green only | Argo Rollouts canary strategy for backend services | Medium | Medium |

---

## 28. Appendix — Full Command Reference

### 28.1 Kubernetes Commands

```bash
# ━━ PODS ━━
kubectl get pods -n hipster-backend
kubectl get pods -n hipster-frontend
kubectl get pods -n hipster-database
kubectl get pods -n argocd

kubectl describe pod <pod-name> -n hipster-backend
kubectl logs <pod-name> -n hipster-backend
kubectl logs <pod-name> -n hipster-backend --previous     # crashed container
kubectl logs -f <pod-name> -n hipster-backend             # follow
kubectl logs -l app=adservice -n hipster-backend --tail=100 # by label

# ━━ DEPLOYMENTS ━━
kubectl get deployments -n hipster-backend
kubectl rollout status deployment/adservice -n hipster-backend
kubectl rollout restart deployment/adservice -n hipster-backend
kubectl rollout undo deployment/adservice -n hipster-backend
kubectl rollout history deployment/adservice -n hipster-backend

# ━━ SERVICES & NETWORKING ━━
kubectl get svc -n hipster-backend
kubectl get endpoints adservice -n hipster-backend
kubectl get networkpolicy -n hipster-backend
kubectl get httproute -n hipster-backend
kubectl get gateway -n hipster-backend

# ━━ PORT-FORWARDING (for local testing) ━━
kubectl port-forward svc/adservice 9555:9555 -n hipster-backend
kubectl port-forward svc/authservice 8081:8081 -n hipster-backend
kubectl port-forward svc/cartservice 7070:7070 -n hipster-backend
kubectl port-forward svc/checkoutservice 5050:5050 -n hipster-backend
kubectl port-forward svc/currencyservice 7000:7000 -n hipster-backend
kubectl port-forward svc/emailservice 8080:8080 -n hipster-backend
kubectl port-forward svc/frontend 8080:80 -n hipster-frontend
kubectl port-forward svc/paymentservice 50051:50051 -n hipster-backend
kubectl port-forward svc/productcatalogservice 3550:3550 -n hipster-backend
kubectl port-forward svc/shippingservice 50051:50051 -n hipster-backend
kubectl port-forward svc/assistantservice 8080:8080 -n hipster-backend
kubectl port-forward svc/argocd-server 8080:443 -n argocd

# ━━ RESOURCES & DEBUG ━━
kubectl top pods -n hipster-backend
kubectl top nodes
kubectl get events -n hipster-backend --sort-by='.lastTimestamp'
kubectl get events -n hipster-database --sort-by='.lastTimestamp'

# ━━ SECRETS & CONFIGMAPS ━━
kubectl get secret app-secrets -n hipster-backend
kubectl get secret mongodb-root -n hipster-backend
kubectl get secret mongodb-users -n hipster-backend
kubectl get secret mongodb-root -n hipster-database
kubectl get secret mongodb-users -n hipster-database

kubectl get configmap hipster-config -n hipster-backend -o yaml
kubectl get configmap service-addresses -n hipster-backend -o yaml
kubectl get configmap service-ports -n hipster-backend -o yaml
kubectl get configmap mongodb-config -n hipster-backend -o yaml

# ━━ MONGODB ━━
kubectl get pods -n hipster-database -l app=mongodb
kubectl get statefulset mongodb -n hipster-database
kubectl get pvc -n hipster-database

kubectl exec -it mongodb-0 -n hipster-database -- mongosh \
  --username admin --password <root-password> --authenticationDatabase admin

# Check replica set status
kubectl exec -it mongodb-0 -n hipster-database -- \
  mongosh --eval "rs.status()" --username admin --password <pw> --authenticationDatabase admin

# ━━ STORAGE ━━
kubectl get storageclass
kubectl get pv
kubectl get pvc -n hipster-database
kubectl describe pvc mongo-data-mongodb-0 -n hipster-database
```

### 28.2 ArgoCD Commands

```bash
# Login
argocd login localhost:8080 --username admin --password <password> --insecure

# List all applications
argocd app list

# Get detailed app status
argocd app get adservice-dev
argocd app get frontend-prod

# Sync an application
argocd app sync adservice-dev
argocd app sync adservice-dev --force

# Force refresh (re-poll git)
argocd app get adservice-dev --refresh

# View rendered manifests (what ArgoCD applies)
argocd app manifests adservice-dev

# View diff (git state vs cluster state)
argocd app diff adservice-dev

# View history
argocd app history adservice-dev

# Rollback to previous revision
argocd app rollback adservice-dev <revision-id>
```

### 28.3 Helm Commands

```bash
# IMPORTANT: helm list returns empty — ArgoCD manages releases
helm list -n hipster-backend   # Always empty — EXPECTED

# From hipstershop-helm-charts directory:

# Render templates locally
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Render with dev overrides
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  -f hipstershop-micro-apps/adservice/values.dev.yaml

# Render with prod overrides
helm template adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  -f hipstershop-micro-apps/adservice/values.prod.yaml

# Lint chart
helm lint hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml

# Dry run (shows what would be applied without ArgoCD)
helm install adservice hipstershop-micro-apps/adservice \
  -f hipstershop-micro-apps/adservice/values.yaml \
  --dry-run --debug
```

### 28.4 Argo Rollouts Commands (Frontend)

```bash
# Check rollout status
kubectl argo rollouts get rollout frontend -n hipster-frontend
kubectl argo rollouts get rollout frontend -n hipster-frontend --watch

# Promote GREEN to live traffic (Blue-Green)
kubectl argo rollouts promote frontend -n hipster-frontend

# Abort rollout (revert to BLUE during promotion)
kubectl argo rollouts abort frontend -n hipster-frontend

# Undo rollout (rollback)
kubectl argo rollouts undo frontend -n hipster-frontend

# View history
kubectl argo rollouts history rollout frontend -n hipster-frontend
```

### 28.5 Docker Commands

```bash
# Build
docker build -t yampss/hipstershop-adservice:local src/

# Test locally
docker run -p 9555:9555 -e PORT=9555 yampss/hipstershop-adservice:local

# Check image layers
docker history yampss/hipstershop-adservice:adservice-v0.1.2

# Scan with Trivy
trivy image yampss/hipstershop-adservice:adservice-v0.1.2 \
  --severity CRITICAL --exit-code 1 --ignore-unfixed

# Cross-push to GHCR
docker tag yampss/hipstershop-adservice:adservice-v0.1.2 \
  ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.1.2
docker push ghcr.io/hipstershopp/hipstershop-adservice:adservice-v0.1.2
```

### 28.6 Health Check Endpoints

```bash
# Test all service health endpoints (run after port-forwarding)
curl http://localhost:9555/_healthz    # adservice
curl http://localhost:8081/_healthz    # authservice
curl http://localhost:7070/_healthz    # cartservice
curl http://localhost:5050/_healthz    # checkoutservice
curl http://localhost:7000/_healthz    # currencyservice
curl http://localhost:8080/_healthz    # emailservice / assistantservice / recommendationservice
curl http://localhost:8080/_healthz    # frontend (port-forward svc/frontend 8080:80)
curl http://localhost:50051/_healthz   # paymentservice / shippingservice
curl http://localhost:3550/_healthz    # productcatalogservice
```

### 28.7 Service-Specific Test Calls

```bash
# authservice — signup and login
curl -X POST http://localhost:8081/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test1234"}'

curl -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test1234"}'

# adservice — get ads
curl http://localhost:9555/ads

# productcatalogservice — list products
curl http://localhost:3550/products

# paymentservice — test charge
curl -X POST http://localhost:50051/charge \
  -H "Content-Type: application/json" \
  -d '{
    "amount":{"currencyCode":"USD","units":10,"nanos":0},
    "creditCard":{
      "creditCardNumber":"4432-8015-6152-0454",
      "creditCardCvv":672,
      "creditCardExpirationYear":2029,
      "creditCardExpirationMonth":1
    }
  }'

# currencyservice — list currencies
curl http://localhost:7000/currencies

# assistantservice — chat
curl -X POST http://localhost:8080/api/assistant/chat \
  -H "Content-Type: application/json" \
  -d '{"productId":"OLJCESPC7Z0","message":"Tell me about this product"}'
```

---

## Appendix: Sample values.yaml Structure

```yaml
# hipstershop-micro-apps/{service}/values.yaml
global:
  namespaces:
    backend: hipster-backend
    frontend: hipster-frontend
    database: hipster-database

  images:
    adservice: "yampss/hipstershop-adservice:adservice-v0.1.2"
    authservice: "yampss/hipstershop-authservice:authservice-v0.2.1"
    # ... all 12 services

  replicas:
    adservice: 1
    authservice: 1
    # ...

  ports:
    adservice: 9555
    authservice: 8081
    cartservice: 7070
    checkoutservice: 5050
    currencyservice: 7000
    emailservice: 8080
    frontend: 8080
    paymentservice: 50051
    productcatalogservice: 3550
    recommendationservice: 8080
    shippingservice: 50051
    assistantservice: 8080

  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1

  resources:
    adservice:
      requests:
        cpu: 200m
        memory: 180Mi
      limits:
        cpu: 300m
        memory: 512Mi

  hpa:
    frontend:
      minReplicas: 1
      maxReplicas: 5
      cpuUtilization: 70
      memoryUtilization: 80

  blueGreen:
    autoPromotionEnabled: false
    scaleDownDelaySeconds: 60

  gateway:
    name: hipstershop-gateway
    port: 80

  networkPolicy:
    vpcCidr: "172.31.0.0/16"
    kgatewayNamespace: "kgateway-system"
```

---

*End of HipsterShop Platform — Complete DevOps Engineering Documentation*

*Document generated: April 2026 | Organization: HipsterShopp | 14 Repositories | 12 Microservices*
