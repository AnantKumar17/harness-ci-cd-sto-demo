# Harness Pipeline Execution Links

## Project: Harness_Assignment_Setup

### Pipeline 1: PR Security Scan

**Pipeline Identifier:** `prsecurityscan`

**Purpose:** Shift-left security scanning on pull requests using Trivy for filesystem scanning.

**Execution Link:**https://app.harness.io/ng/account/RxaWg4t9QsiZh2OaJUuJIg/module/ci/orgs/default/projects/Harness_Assignment_Setup/pipelines/prsecurityscan/deployments/Z89MPpPcQB6QfEldRBUCOg/pipeline?storeType=INLINE

**Trigger:** GitHub PR (Open + Synchronize actions)

**Stages:**
- Shift-Left Trivy Scan (CI stage)
  - Clone repository
  - Run Trivy filesystem scan on source code
  - Exit with code 1 on CRITICAL or HIGH vulnerabilities
  - Report status back to GitHub PR

---

### Pipeline 2: Main CI/CD Pipeline

**Pipeline Identifier:** `harnesscicdstodemo`

**Purpose:** Complete CI/CD pipeline with 4 stages: Build, Security Scan, Deploy, Validate.

**Execution Link:**https://app.harness.io/ng/account/RxaWg4t9QsiZh2OaJUuJIg/module/ci/orgs/default/projects/Harness_Assignment_Setup/pipelines/harnesscicdstodemo/deployments/w7bUw3cGSPe1NeC9HMI6OQ/pipeline?storeType=INLINE

**Trigger:** Manual + GitHub PR merge to main

**Stages:**
1. **CI - Build and Push**
   - Run unit tests (pytest)
   - Code quality checks (Black, Flake8)
   - Build Docker image with layer caching
   - Push to Docker Hub with dual tags: sequential ID + `latest`

2. **STO - Security Scan**
   - Trivy container image scanning
   - CRITICAL severity gate (blocks deployment if found)
   - HIGH vulnerabilities reported but non-blocking

3. **CD - Deploy to Kubernetes**
   - Rolling update deployment (zero-downtime)
   - Automatic rollback on failure
   - Retry logic: 2 retries with 30s backoff

4. **Validation - Health Check**
   - Verify deployment rollout status
   - Test `/health` endpoint
   - Confirm service connectivity

---

## Key Features

 Dual-tag strategy: Both sequential ID (`:1`, `:2`, etc.) and `latest` pushed to Docker Hub  
 Helm templating: values.yaml with `<+artifacts.primary.image>` expression for dynamic image resolution  
 Zero-downtime deployments: RollingUpdate strategy with revision history for quick rollback  
 Comprehensive health checks: kubectl status + HTTP endpoint validation  
 Security gates: CRITICAL severity blocks deployment; HIGH logged but non-blocking  
 Automatic retry with backoff: 2 retries at 30s intervals on deployment failure  
 PR-based shift-left security: Source code scanned before merge  

---

## Accessing Pipelines

1. Go to [app.harness.io](https://app.harness.io)
2. Navigate to **Project: Harness_Assignment_Setup**
3. Click on **Pipelines**
4. Select either `prsecurityscan` or `harnesscicdstodemo`
5. View execution history and details