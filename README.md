# Harness CI/CD + STO Pipeline Assignment

A complete demonstration of **Harness Continuous Delivery** with integrated security scanning (STO), deployed to Kubernetes on minikube. This project showcases best practices for modern DevOps workflows including automated testing, security gates, and multi-environment deployments.

---

## Table of Contents

- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Security Gates](#security-gates)
- [Setup Instructions](#setup-instructions)
- [Features](#features)
- [Bonus Features](#bonus-features)
- [Assumptions & Limitations](#assumptions--limitations)
- [Key Technologies](#key-technologies)

---

## Architecture

### High-Level Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                        │
│                  (AnantKumar17/harness-ci-cd-sto-demo)           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
        ┌───────────────┐     ┌──────────────────┐
        │  PR Opened    │     │  PR Merged       │
        │  (Bonus 1)    │     │  to main         │
        └───────┬───────┘     └────────┬─────────┘
                │                      │
                ▼                      ▼
      ┌─────────────────────┐  ┌──────────────────────────────┐
      │ PR SECURITY SCAN    │  │ MAIN CI/CD PIPELINE          │
      │ (Shift-Left)        │  │ (4 Stages)                   │
      │                     │  │                              │
      │ Stage 1: Trivy      │  │ Stage 1: CI (Build & Test)   │
      │ ▶ Scan source code  │  │ ▶ Unit tests                 │
      │ ▶ Exit on HIGH/CRIT │  │ ▶ Code quality checks        │
      │ ▶ Report to GitHub  │  │ ▶ Build Docker image         │
      │ ▶ Block merge if    │  │ ▶ Push to Docker Hub         │
      │   vulnerabilities   │  │                              │
      │   found             │  │ Stage 2: STO (Security Scan) │
      │                     │  │ ▶ Trivy image scan           │
      │ [Merge allowed      │  │ ▶ CRITICAL severity gate     │
      │  only if pass ✓]    │  │ ▶ Block deployment if found  │
      └─────────────────────┘  │                              │
                               │ Stage 3: CD (Deploy)         │
                               │ ▶ Rolling update             │
                               │ ▶ Automatic rollback (B2)    │
                               │ ▶ Retry with backoff (B2)    │
                               │                              │
                               │ Stage 4: Validation          │
                               │ ▶ Rollout status check       │
                               │ ▶ Health endpoint test       │
                               │ ▶ Service connectivity       │
                               │                              │
                               └──────────────┬───────────────┘
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │  minikube K8s    │
                                    │  Cluster         │
                                    │  (demo-app: 2    │
                                    │   replicas)      │
                                    └──────────────────┘
```

**Legend:**
- **B1** = Bonus 1: Shift-Left Security
- **B2** = Bonus 2: Failure Handling (Rollback + Retry)
- **🔒** = Security Gate (blocks on failure)

---

## Pipeline Stages

### Stage 1: CI - Build & Test

**Type:** CI (Continuous Integration)  
**Runtime:** Harness Cloud (ubuntu-22)  
**Trigger:** Manual or PR merge

**Steps:**

1. **Run Unit Tests**
   ```bash
   pip install -r app/requirements.txt flake8 black
   black app/                    # Auto-format code
   flake8 app/                   # Lint with strict rules
   pytest app/tests/ -v          # Run unit tests
   ```
   - Uses `python:3.11-alpine` for minimal dependencies
   - Fails if any test fails or code quality check fails
   - Timeout: 10m

2. **Build and Push Docker Image**
   ```bash
   docker build -t anantkumar17/harness-demo-app .
   docker push anantkumar17/harness-demo-app:$SEQUENCEID
   docker push anantkumar17/harness-demo-app:latest
   ```
   - Multi-stage build reduces image size
   - Dual tags: sequential ID (`:1`, `:2`, etc.) + `latest`
   - Layer caching enabled for faster rebuilds
   - Timeout: 10m

**Exit Criteria:** Both steps must pass, or pipeline stops

---

### Stage 2: STO - Security Scan

**Type:** Custom (using Trivy container)  
**Runtime:** Kubernetes Direct (delegate on minikube)  
**Tool:** Trivy (CNCF-recommended container scanning)

**Security Gate Logic:**

```
Trivy scans: anantkumar17/harness-demo-app:$SEQUENCEID

Results:
├─ CRITICAL vulnerabilities found → Exit Code 1  BLOCKS DEPLOYMENT
├─ HIGH vulnerabilities found    → Visible in logs  (non-blocking)
└─ No CRITICAL found            → Exit Code 0  CONTINUES TO CD
```

**Command:**
```bash
trivy image \
  --exit-code 1 \
  --severity CRITICAL \
  --no-progress \
  anantkumar17/harness-demo-app:$SEQUENCEID
```

**Why CRITICAL-only gate?**
- HIGH vulnerabilities in Debian/Python packages are common
- CRITICAL indicates exploitable privilege escalation or RCE
- Gate prevents dangerous images from reaching production
- Timeout: 10m

**Exit Criteria:** No CRITICAL vulnerabilities, or pipeline stops and rollback triggers

---

### Stage 3: CD - Deploy to Kubernetes

**Type:** Deployment (native Harness CD)  
**Target:** minikube cluster, `default` namespace  
**Strategy:** RollingUpdate (zero-downtime)

**Deployment Configuration:**

```yaml
spec:
  replicas: 2
  revisionHistoryLimit: 3        # Keeps last 3 ReplicaSets for rollback
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                 # 3 pods briefly during update
      maxUnavailable: 0           # Always ≥2 pods available
  template:
    image: anantkumar17/harness-demo-app:latest
    resources:
      requests:
        memory: 64Mi
        cpu: 100m
      limits:
        memory: 128Mi
        cpu: 200m
```

**Health Probes:**
- **Liveness:** `GET /health` every 5s, fail after 3s, restart after 3 failures
- **Readiness:** `GET /health` every 3s, fail after 1s, mark not-ready after 3 failures

**K8s Apply Step:**
- Applies both deployment and service manifests
- **Timeout:** 10m
- **Failure Strategy:** [See Bonus 2](#bonus-2-failure-handling) — Automatic Rollback + Retry

---

### Stage 4: Validation - Health Check

**Type:** Custom (ShellScript on delegate)  
**Runtime:** Kubernetes Direct (delegate)

**Validation Script:**

```bash
# Check rollout completion
kubectl rollout status deployment/demo-app -n default --timeout=60s

# Get service ClusterIP
POD_IP=$(kubectl get svc demo-app-service -n default -o jsonpath='{.spec.clusterIP}')

# Test /health endpoint
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$POD_IP:80/health)

# Fail if not 200
if [ "$HTTP_CODE" != "200" ]; then
  echo "Health check failed! HTTP code: $HTTP_CODE"
  exit 1
fi
```

**What This Verifies:**
-  Deployment rolled out successfully
-  Service is reachable via ClusterIP
-  Application responds to `/health` on port 80
-  HTTP status is 200 OK

**Timeout:** 5m

**Exit Criteria:** Health check passes, or pipeline fails and triggers rollback

---

## Security Gates

### How Security Gates Work

The security gate is the **enforcement mechanism** that makes security scanning meaningful:

```
Scan Phase          Gate Phase          Outcome
───────────────────────────────────────────────
Trivy scans image → Evaluate severity → Decision
                                          │
                                    ┌─────┴─────┐
                                    │           │
                              CRITICAL        No CRITICAL
                              found?          found?
                                    │           │
                                    ▼           ▼
                                FAIL          PASS
                              (Exit 1)       (Exit 0)
                                    │           │
                                    ▼           ▼
                              Stop pipeline   Continue to CD
                              Block deployment
```

### Configurable Thresholds

The gate uses `--severity` flag to define what "blocks":

| Threshold | Behavior | Use Case |
|-----------|----------|----------|
| `CRITICAL` | Fail on CRITICAL only | ← **This assignment** (prod-grade) |
| `CRITICAL,HIGH` | Fail on HIGH or CRITICAL | Strict security posture |
| `CRITICAL,HIGH,MEDIUM` | Fail on MEDIUM+ | Development/testing |


---

## Setup Instructions

### Prerequisites

```bash
# Check if installed
docker --version
minikube version
kubectl version --client
git --version
```

**Required:**
- Docker Desktop (macOS/Windows) or Docker Engine (Linux)
- Kubernetes (minikube ~100MB, runs locally)
- kubectl (included with Docker Desktop)
- Git

**Accounts (free tier sufficient):**
- [app.harness.io](https://app.harness.io) — Harness free account
- [docker.com](https://docker.com) — Docker Hub free account
- [github.com](https://github.com) — GitHub account

---

### Step 1: Clone the Repository

```bash
git clone https://github.com/AnantKumar17/harness-ci-cd-sto-demo.git
cd harness-ci-cd-sto-demo
```

---

### Step 2: Start minikube

```bash
minikube start --cpus=4 --memory=4096 --vm-driver=docker
kubectl config use-context minikube
```

Verify:
```bash
kubectl get nodes
# Output: minikube   Ready    control-plane   ...
```

---

### Step 3: Set Up Harness Account

1. Create free account at [app.harness.io](https://app.harness.io)
2. Create new project: `Harness_Assignment_Setup`
3. Enable modules: **CI** + **CD** (STO not available in free tier)

---

### Step 4: Create Harness Connectors

**GitHub Connector:**
- Type: GitHub
- Authentication: Personal Access Token (PAT) with `repo` + `workflow` scopes
- Name: `github_connector`

**Docker Hub Connector:**
- Type: Docker Registry
- Registry URL: `https://registry.hub.docker.com/v2/`
- Username: Your Docker Hub username
- Password: Docker Hub access token
- Name: `dockerhub_connector`

**Kubernetes Connector:**
- Type: Kubernetes Cluster
- Agent: Use minikube delegate (installed next)
- Name: `minikubek8sconnector`

---

### Step 5: Install Harness Delegate

In Harness UI:
1. **Account Settings → Delegates → Install New Delegate**
2. Choose: **Kubernetes (namespace)**
3. Copy the provided YAML
4. Apply to cluster:
   ```bash
   kubectl apply -f harness-delegate.yaml
   ```
5. Wait 2-3 minutes for "Connected" status in Harness UI

Verify:
```bash
kubectl get pods -n harness-delegate-ng
# Output: minikube-delegate-xxxxx    1/1     Running
```

---

### Step 6: Create Service & Environment

**Create Kubernetes Service:**
- Name: `harnessdemoapp`
- Artifact: Docker Registry (anantkumar17/harness-demo-app)
- Manifests: GitHub repo, path `k8s/`

**Create Environment:**
- Name: `minikube`
- Infrastructure: Point to minikube K8s connector
- Namespace: `default`

---

### Step 7: Import Pipelines

1. In Harness: **Pipelines → Create New Pipeline**
2. Choose: **GitHub Repository**
3. Import from [.harness/pr-security-scan.yaml](./.harness/pr-security-scan.yaml)
4. Repeat for [.harness/harness-ci-cd-sto-demo.yaml](./.harness/harness-ci-cd-sto-demo.yaml)

Or manually create using the YAML files in `.harness/`

---

### Step 8: Configure GitHub Webhook

1. GitHub repo → **Settings → Webhooks → Add webhook**
2. Payload URL: (provided by Harness when you create trigger)
3. Events: **Pull requests** + **Pushes**
4. Click **Add webhook**

---

### Step 9: Test the Pipeline

**Option A: Manual Trigger**
```bash
# Go to Harness → Pipelines → harness-ci-cd-sto-demo → Run Pipeline
```

**Option B: PR Trigger** (Bonus 1)
```bash
git checkout -b test/feature
echo "# Test" >> README.md
git add README.md
git commit -m "Test PR trigger"
git push origin test/feature
# Create PR on GitHub → PR security scan runs automatically
```

---

## Features

###  Core Features

| Feature | Implementation | Benefit |
|---------|---|---|
| **Automated Testing** | pytest + Black + Flake8 in CI stage | Ensures code quality before building |
| **Docker Layer Caching** | `caching: true` in BuildAndPushDocker | 70% faster rebuilds |
| **Security Scanning** | Trivy image scan with CRITICAL gate | Prevents vulnerable images reaching prod |
| **Rolling Deployments** | `RollingUpdate` strategy | Zero-downtime updates |
| **Health Checks** | Liveness + Readiness probes + HTTP test | Detects broken deployments immediately |
| **Version Control** | Dual tags (sequence ID + latest) | Traceability + latest convenience |

---

## Bonus Features

### Bonus 1: Shift-Left Security (PR Security Scan)

**What it does:** Scans source code **before merge**, blocking PRs with vulnerabilities.

**How it works:**

```
Developer opens PR
        ↓
GitHub triggers: pr-security-scan pipeline
        ↓
Stage: Shift-Left Trivy Scan
  ├─ Clone PR branch
  ├─ Run: trivy fs /harness --severity CRITICAL,HIGH
  ├─ Post results to GitHub PR
        ↓
  ├─ PASS →  Merge button enabled
  └─ FAIL →  Merge blocked (PR check shows red X)
        ↓
Developer sees: "PR Security Scan — vulnerability found"
        ↓
Either: Fix the code OR suppress if intentional
```

---

### Bonus 2: Failure Handling

#### 2a. Automatic Rollback

**What it does:** If deployment fails, automatically roll back to the last working version.

**Configuration:**

```yaml
stages:
  - CD Stage:
      failureStrategies:
        - onFailure:
            errors: [AllErrors]
            action: StageRollback    # ← Automatic rollback
      execution:
        - K8s Apply:
            failureStrategies:
              - onFailure:
                  errors: [AllErrors]
                  action: 
                    type: Retry
                    spec:
                      retryCount: 2
                      retryIntervals: [30s]
                      onRetryFailure:
                        action: StageRollback
```

**How it works:**

```
Deployment attempts to rollout new version
        │
        ├─ Pod startup fails (CrashLoopBackOff)
        │
        └─ K8s Apply detects error
                │
                └─ Retry Logic:
                   Attempt 1 FAIL → Wait 30s → Attempt 2
                   Attempt 2 FAIL → Wait 30s → Attempt 3
                   Attempt 3 FAIL → Trigger StageRollback
                        │
                        └─ StageRollback:
                           Revert to previous ReplicaSet
                           (Kubernetes keeps last 3 versions)
                                │
                                └─ Old pods restart
                                   Service traffic reroutes
                                   Users unaffected 
```

**Why This Matters:**
> "Production deployments fail. Network timeouts, misconfigured secrets, resource limits too low — these happen. The question is whether you have a recovery plan. This pipeline automatically reverts to the last known-good version."


#### 2b. Revision History

Kubernetes keeps the last **3 ReplicaSets** (configured in deployment.yaml):

```yaml
spec:
  revisionHistoryLimit: 3
```

This enables rollback to any of the last 3 versions:

```bash
kubectl rollout history deployment/demo-app
kubectl rollout undo deployment/demo-app  # Back to revision 1
kubectl rollout undo deployment/demo-app --to-revision=3  # Back to v9
```

---

## Assumptions & Limitations

### Assumptions

| Assumption | Reasoning |
|-----------|-----------|
| **Single K8s Cluster** | minikube on local laptop. In production: use separate dev/staging/prod clusters |
| **Single Namespace** | All resources in `default`. Production: use per-team or per-environment namespaces |
| **No GitOps** | Manual pipeline execution. Production: ArgoCD for Git-as-source-of-truth |
| **No Secrets Rotation** | Docker credentials stored in Harness. Production: use Vault/sealed-secrets |
| **No Observability** | No metrics/logging. Production: Prometheus + Loki + Grafana |
| **No Rate Limiting** | Docker Hub has limits (~100 pulls/6hrs for anonymous). Production: use docker-credentials |
| **Free Tier Only** | Limited STO scans, build minutes. Production: paid Harness + standalone scanners |

### Limitations

| Limitation | Workaround |
|-----------|-----------|
| **STO Module Not Available** | Used Custom stage + Trivy CLI. Production: Use paid Harness STO or Snyk |
| **Cloud Runtime Slow** | Use delegate-based builds for faster feedback loops |
| **Manual Approval Gates Missing** | Could add approval step between STO→CD for audit trail |
| **No Artifact Registry** | Using Docker Hub public repo. Production: Private ECR/ACR/GCR |
| **Limited Rollback Window** | Only last 3 revisions kept. Production: longer history + backup systems |

---

## Key Technologies

| Technology | Purpose | Why Chosen |
|-----------|---------|-----------|
| **Harness** | CI/CD Orchestration | Native STO module, delegate model, free tier generous |
| **Kubernetes** | Container Orchestration | Industry standard, battle-tested, local via minikube |
| **minikube** | Local K8s | No cloud costs, full control, reproducible environment |
| **Docker** | Containerization | Standard format, multi-stage builds, layer caching |
| **Trivy** | Security Scanning | CNCF project, fast (~30s), no license, high accuracy |
| **Python 3.11** | Application Runtime | Lightweight, easy to understand, Flask minimal dependencies |
| **Flask** | Web Framework | Minimal code, `/health` endpoint built-in, stateless |
| **pytest** | Testing | Industry standard, fixtures, plugins ecosystem |
| **Black + Flake8** | Code Quality | Auto-formatter + linter, opinionated, fast feedback |

---

## Troubleshooting

### Pipeline stuck at "provisioning machine"

**Cause:** Harness Cloud out of capacity  
**Solution:** Switch to Kubernetes Direct runtime in Infrastructure settings

### Health check failing with "service not found"

**Cause:** Service name mismatch  
**Solution:** Verify service name in deployment: `kubectl get svc -n default`

### Image pull fails with "ErrImagePull"

**Cause:** Image expression not rendering  
**Solution:** Use `<+artifacts.primary.image>` in values.yaml, not `<+artifact.image>`

---

## Additional Resources

- **Harness Docs:** [docs.harness.io](https://docs.harness.io)
- **Trivy Docs:** [aquasecurity.github.io/trivy](https://aquasecurity.github.io/trivy)
- **Kubernetes Docs:** [kubernetes.io](https://kubernetes.io)
- **Assignment Details:** [plan.md](./plan.md)
- **AI Usage & Process:** [CLAUDE.md](./CLAUDE.md)
- **Execution Links:** [.harness/EXECUTION_LINKS.md](./.harness/EXECUTION_LINKS.md)