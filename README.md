# harness-ci-cd-sto-demo

> Documentation in progress — will be populated after the pipeline is fully verified.

## Architecture

```
GitHub push
    │
    ▼
┌─────────────────────────────────────────────────┐
│  Harness Pipeline                               │
│                                                 │
│  Stage 1: CI          Stage 2: STO             │
│  ┌─────────────┐      ┌─────────────────────┐  │
│  │ Build image │ ───► │ Trivy scan          │  │
│  │ Run tests   │      │ Security gate       │  │
│  │ Push to Hub │      │ (block on CRITICAL) │  │
│  └─────────────┘      └─────────────────────┘  │
│                                  │              │
│  Stage 3: CD          Stage 4: Validate        │
│  ┌─────────────┐      ┌─────────────────────┐  │
│  │ kubectl     │ ───► │ /health check       │  │
│  │ apply       │      │ rollout status      │  │
│  └─────────────┘      └─────────────────────┘  │
└─────────────────────────────────────────────────┘
    │
    ▼
minikube (local Kubernetes)
```

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/) + `kubectl`
- [Harness Free Account](https://app.harness.io)
- Docker Hub account

---

## Local Setup

```bash
# 1. clone the repo
git clone https://github.com/AnantKumar17/harness-ci-cd-sto-demo.git
cd harness-ci-cd-sto-demo

# 2. install dependencies
pip install -r app/requirements.txt

# 3. run the app
python app/app.py

# 4. run tests
pytest app/tests/

# 5. build the docker image locally
docker build -t harness-demo-app:local .
docker run -p 5000:5000 harness-demo-app:local
```

---

## Pipeline Stages

| Stage | Tool | What it does |
|---|---|---|
| CI | Harness CI | Builds Docker image, runs unit tests, pushes to Docker Hub |
| STO | Harness STO + Trivy | Scans image for CVEs, blocks on CRITICAL/HIGH |
| CD | Harness CD | Applies k8s manifests to minikube via delegate |
| Validate | Harness CD | HTTP health check + `kubectl rollout status` |

### How the Security Gate Works

The STO stage runs a Trivy scan against the Docker image pushed in CI. A **security gate** policy is configured to fail the pipeline if any vulnerability with severity `CRITICAL` or `HIGH` is found. This means:

- A clean image → pipeline continues to CD
- A vulnerable image → pipeline halts at STO, CD never runs, nothing is deployed

This is intentional — broken images never reach the cluster.

---

## Bonus Features

### Shift-Left Security
A separate PR validation pipeline triggers on pull requests. It builds the PR branch image, scans it with Trivy, and posts the result as a PR status check. Merging is blocked if CRITICAL vulnerabilities are found. This catches security issues *before* they ever reach `main`.

### Failure Handling
The CD stage has a rollback step: if `kubectl apply` fails, Harness re-applies the last known-good image tag. The deployment step also retries twice with a 30-second backoff before triggering the rollback. A failure notification is sent on terminal pipeline failure.

---

## Assumptions

- Single local Kubernetes cluster via minikube
- Harness Delegate running inside minikube
- Docker Hub used as the container registry
- Free tier limits apply (~2,000 CI build-minutes/month)
- `python:3.11-slim` base image (Trivy may report MEDIUM CVEs; gate is set to block CRITICAL/HIGH only)

---

## AI-Assisted Development

See [CLAUDE.md](./CLAUDE.md) for a full account of how AI tooling was used in this project, what was reviewed and verified at each step, and where human judgment was applied.
