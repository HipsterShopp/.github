# hipstershop-shared-workflows

> Centralized GitHub Actions reusable workflows library for the HipsterShop platform. All 12 microservice CI/CD pipelines are built on these 5 workflows.

---

## Table of Contents
- [Purpose](#purpose)
- [Workflow Catalog](#workflow-catalog)
- [Usage Guide](#usage-guide)
- [Workflow: sonar-and-snyk.yaml](#workflow-sonar-and-snykyaml)
- [Workflow: generate-tag-and-build-image-trivy.yaml](#workflow-generate-tag-and-build-image-trivyyaml)
- [Workflow: push.yaml](#workflow-pushyaml)
- [Workflow: helm-updater-job.yaml](#workflow-helm-updater-jobyaml)
- [Workflow: notify.yaml](#workflow-notifyyaml)
- [Adding a New Workflow](#adding-a-new-workflow)
- [Troubleshooting](#troubleshooting)

---

## Purpose

Without shared workflows, each of the 12 microservice repositories would contain their own full CI/CD YAML (~200+ lines each). When any tool version needs updating (Trivy, SonarQube scanner, Snyk action), that would require 12 separate PRs across 12 repos.

This repo centralizes all pipeline logic. Each microservice repo contains a thin caller workflow (~30-100 lines) that calls these shared workflows with service-specific parameters. Updating a tool version here propagates to all services automatically on the next trigger.

**Reference branch:** All callers use `@feature/loose-coupling` pin:
```yaml
uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/{name}.yaml@feature/loose-coupling
```

---

## Workflow Catalog

| Workflow File | Purpose | Called On |
|---------------|---------|-----------|
| `sonar-and-snyk.yaml` | Static analysis + dependency scanning | PR to main/develop |
| `generate-tag-and-build-image-trivy.yaml` | Tag calculation, Docker build, Trivy CVE scan | Push + PR |
| `push.yaml` | Push image to Docker Hub + create git tag | Push to main/develop |
| `helm-updater-job.yaml` | Update image tag in Helm values file | After push to develop |
| `notify.yaml` | Send failure email via SendGrid | Always (after all jobs) |

---

## Usage Guide

### How to Call a Shared Workflow

```yaml
# In .github/workflows/{service-name}.yaml of any microservice repo
jobs:
  sonar-and-snyk:
    if: github.event_name == 'pull_request'
    uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
    with:
      service_name: myservice          # Required: service name (lowercase)
      service_path: src/               # Required: path to Dockerfile and source
      runtime: go                      # Required: java|go|python|node|dotnet
      go-version: "1.22"              # Required if runtime: go
    secrets:
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

---

## Workflow: sonar-and-snyk.yaml

### Purpose
Runs on pull requests to enforce code quality (SonarQube) and dependency security (Snyk) gates before images are built.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service_name` | Yes | — | Service name (used for SonarQube project key) |
| `service_path` | Yes | — | Path to source code directory (relative to repo root) |
| `runtime` | Yes | — | `java`, `go`, `python`, `node`, `dotnet` |
| `java-version` | No | `"21"` | Java version (runtime: java) |
| `java-build-tool` | No | `"gradle"` | `gradle` or `maven` (runtime: java) |
| `go-version` | No | `"1.22"` | Go version (runtime: go) |
| `python-version` | No | `"3.11"` | Python version (runtime: python) |
| `node-version` | No | `"20"` | Node.js version (runtime: node) |
| `dotnet-version` | No | `"8.0"` | .NET version (runtime: dotnet) |

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `SNYK_TOKEN` | Snyk authentication |
| `SONAR_TOKEN` | SonarQube authentication |
| `SONAR_HOST_URL` | SonarQube server URL |

### Outputs

| Output | Description |
|--------|-------------|
| `sonarqube_result` | `success` or `failure` |
| `snyk_result` | `success` or `failure` |

### What It Does

1. **SonarQube Job:**
   - Checkout with `fetch-depth: 0` (full git history for blame analysis)
   - Setup appropriate language runtime
   - For Java/Gradle: run `./gradlew build` to generate class files before scan
   - Run `SonarSource/sonarqube-scan-action@v6` against `service_path`
   - Check SonarQube Quality Gate — workflow fails if gate doesn't pass

2. **Snyk Job:**
   - Run appropriate Snyk scan based on runtime:
     - Java/Gradle: `snyk test --file=build.gradle --severity-threshold=high`
     - Go: `snyk test --file=go.mod --severity-threshold=high`
     - Python: `snyk test --file=requirements.txt --package-manager=pip --severity-threshold=high`
     - Node: `snyk test --severity-threshold=high`
     - .NET: `snyk test --file={project}.csproj --severity-threshold=high`
   - Upload HTML report as artifact (14-day retention)

---

## Workflow: generate-tag-and-build-image-trivy.yaml

### Purpose
Calculates an image tag using semantic versioning (main branch) or commit-based naming (other branches), builds the Docker image, and optionally runs Trivy vulnerability scanning.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service_name` | Yes | — | Service name (also the DockerHub repo suffix) |
| `service_path` | Yes | — | Path containing the Dockerfile |
| `run_trivy` | No | `false` | Whether to run Trivy scan |

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `DOCKERHUB_USERNAME` | Docker Hub login for tag check |
| `DOCKERHUB_TOKEN` | Docker Hub authentication |

### Outputs

| Output | Description |
|--------|-------------|
| `tag` | Computed image tag (e.g., `adservice-v0.1.2` or `develop-abc12345`) |
| `tag_exists` | `true` if this exact tag already exists on Docker Hub |
| `build_result` | `success` or `failure` |
| `trivy_result` | `success`, `failure`, or `skipped` |

### Tag Calculation Logic

```bash
# On main branch: semantic version using conventional commits
# Uses: mathieudutour/github-tag-action@v6.2
# tag_prefix: "{service_name}-v"
# Example output: "adservice-v0.1.3"

# On other branches: branch slug + short SHA
BRANCH_SLUG=$(echo "${{ github.ref_name }}" | sed 's|/|-|g')
SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-8)
COMPUTED_TAG="${BRANCH_SLUG}-${SHORT_SHA}"
# Example: "develop-3f2a1c9b" or "fix-login-bug-789abcde"
```

### Docker Build

```bash
docker build -t ${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:${TAG} ${SERVICE_PATH}
docker save ${IMAGE_TAG} -o /tmp/image.tar
```

The image is saved as an artifact (`image.tar`) so the `push` job can load it without rebuilding.

### Trivy Scan

When `run_trivy: true`:
```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: "${DOCKERHUB_USERNAME}/hipstershop-${service_name}:${tag}"
    format: "table"
    exit-code: "1"              # Fail on critical vulnerabilities
    severity: "CRITICAL"        # Only fail on CRITICAL
    ignore-unfixed: true        # Skip CVEs without fixes
```

---

## Workflow: push.yaml

### Purpose
Loads the pre-built Docker image artifact, pushes it to Docker Hub, pushes `:latest` on main branch, and creates a git tag.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service_name` | Yes | — | Service name |
| `git_tag` | Yes | — | Tag from generate-tag workflow output |
| `tag_exists` | No | `false` | Skip push if tag already on Hub |

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `DOCKERHUB_USERNAME` | Docker Hub login |
| `DOCKERHUB_TOKEN` | Docker Hub push |

### Outputs

| Output | Description |
|--------|-------------|
| `push_result` | `success` or `failure` |

### What It Does

```bash
# 1. Load pre-built image from artifact
docker load -i /tmp/image.tar

# 2. Login to Docker Hub
echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin

# 3. Push versioned tag
docker push "${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:${TAG}"

# 4. Push :latest (main branch only)
if [ "${GITHUB_REF_NAME}" = "main" ]; then
  docker tag "${IMAGE_TAG}" "${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:latest"
  docker push "${DOCKERHUB_USERNAME}/hipstershop-${SERVICE_NAME}:latest"
fi

# 5. Create git tag (main branch, if tag doesn't already exist)
git tag "${TAG}"
git push origin "${TAG}"
```

---

## Workflow: helm-updater-job.yaml

### Purpose
Updates the image tag in a specified Helm values file within the `hipstershop-helm-charts` repository and commits the change. This is what triggers ArgoCD to detect a new image and sync the cluster.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service_name` | Yes | — | Service name (used in commit message) |
| `image_tag` | Yes | — | New image tag to write |
| `helm_values_path` | Yes | — | Path to values file within helm-charts repo (e.g., `hipstershop-micro-apps/adservice/values.dev.yaml`) |
| `target_branch` | Yes | — | Branch of helm-charts repo to push to (`develop` or `main`) |

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `HELM_CHARTS_TOKEN` | PAT with write access to hipstershop-helm-charts repo |

### The Update Mechanism

```bash
# Checkout the helm-charts repo (NOT the caller service repo)
- uses: actions/checkout@v4
  with:
    repository: HipsterShopp/hipstershop-helm-charts
    token: ${{ secrets.HELM_CHARTS_TOKEN }}
    ref: ${{ inputs.target_branch }}

# Update the tag line in the values file
sed -i "s|^\(\s*tag:\s*\).*|\1${{ inputs.image_tag }}|" "${{ inputs.helm_values_path }}"

# Commit and push
git config user.name "github-actions"
git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
git add "${{ inputs.helm_values_path }}"
git diff --cached --quiet && echo "No changes" && exit 0
git commit -m "chore(${{ inputs.service_name }}): update image tag to ${{ inputs.image_tag }}"
git push origin ${{ inputs.target_branch }}
```

**The `sed` pattern matches lines like:**
```yaml
# Before:
    tag: develop-abc12345
# After:
    tag: develop-def67890
```

---

## Workflow: notify.yaml

### Purpose
Sends an HTML-formatted email notification via SendGrid when any part of the CI/CD pipeline fails. Always runs, regardless of prior job outcomes.

### Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `service_name` | Yes | Service name (appears in email subject) |
| `git_tag` | No | Current image tag |
| `sonarqube_result` | No | `success`, `failure`, or `skipped` |
| `snyk_result` | No | `success`, `failure`, or `skipped` |
| `generate_tag_result` | No | `success`, `failure`, or `skipped` |
| `build_result` | No | `success`, `failure`, or `skipped` |
| `trivy_result` | No | `success`, `failure`, or `skipped` |
| `push_result` | No | `success`, `failure`, or `skipped` |

### Secrets Required

| Secret | Purpose |
|--------|---------|
| `SENDGRID_API_KEY` | SendGrid API key for email delivery |

### What It Does

Only triggers email notification when at least one job previously failed (not just all-skipped). Uses `licenseware/send-email-notification@v2` action to send an HTML dashboard email with results grid:

| Job | Result |
|-----|--------|
| SonarQube | ✅ success |
| Snyk | ✅ success |
| Build & Tag | ❌ failure |
| Trivy | ⏭ skipped |
| Push | ⏭ skipped |

Email contains a link to the failed GitHub Actions run.

---

## Adding a New Workflow

1. Create `.github/workflows/new-workflow.yaml` in this repo on the `feature/loose-coupling` branch
2. Define `on: workflow_call:` with `inputs:` and `secrets:` sections
3. Update the caller workflow in the relevant microservice repo to use it

### Template Structure

```yaml
name: New Shared Workflow

on:
  workflow_call:
    inputs:
      service_name:
        required: true
        type: string
    secrets:
      MY_SECRET:
        required: true
    outputs:
      result:
        value: ${{ jobs.my-job.outputs.result }}

jobs:
  my-job:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.my-step.outputs.result }}
    steps:
      - name: My Step
        id: my-step
        run: echo "result=success" >> $GITHUB_OUTPUT
```

---

## Troubleshooting

**Issue: Caller workflow cannot find the shared workflow**
```bash
# Verify the branch 'feature/loose-coupling' exists in hipstershop-shared-workflows
# Check the uses: reference uses the correct org/repo/path
uses: HipsterShopp/hipstershop-shared-workflows/.github/workflows/sonar-and-snyk.yaml@feature/loose-coupling
#     ^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^
# Org             Repo                       Workflow path                                  Branch
```

**Issue: SonarQube quality gate times out**
```bash
# Quality gate check will poll the SonarQube server
# Ensure SONAR_HOST_URL secret is accessible from GitHub Actions (firewall rules)
# Check SonarQube background task queue in admin panel
```

**Issue: Snyk scan fails with authentication error**
```bash
# SNYK_TOKEN secret is missing or expired
# Regenerate from https://app.snyk.io/account
# Update secret in repo settings
```

**Issue: helm-updater-job fails with "Permission denied"**
```bash
# HELM_CHARTS_TOKEN is missing or doesn't have write access to hipstershop-helm-charts
# Token scope must include: repo (full repo access) or at minimum: contents:write
```

**Issue: notify.yaml never sends email**
```bash
# Check: did any prior job fail? (notify only sends on failure)
# Check: SENDGRID_API_KEY is correct
# Check: sender email domain is verified in SendGrid
```

**Issue: Trivy exits with code 1 for non-CRITICAL CVEs**
```bash
# Trivy is configured: severity: "CRITICAL" only
# Only CRITICAL CVEs cause a hard failure
# HIGH and lower are reported but don't fail the build
# If a CRITICAL CVE is found in the base image, use a newer base image
```
