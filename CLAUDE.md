# Harness CI/CD + STO Pipeline Assignment

## Project Overview
This is a demonstration of a complete Harness CI/CD pipeline with security scanning (STO), deployed to Kubernetes on minikube.

---

## Architecture
- **CI Stage**: Build, test, and push Docker image
- **STO Stage**: Security scanning with Trivy (CRITICAL threshold)
- **CD Stage**: Deploy to Kubernetes with rolling updates
- **Validation Stage**: Health checks and rollout verification

---

## Key Features
- Automated image building with Docker layer caching
- Code quality checks (Black, Flake8, Pytest)
- Container image scanning (Trivy)
- Kubernetes deployment with rollback capability
- Health check validation
- PR-based security scanning (shift-left)

---

## Technologies
- Harness (CI/CD orchestration)
- Kubernetes (minikube)
- Docker (containerization)
- Trivy (security scanning)
- GitHub (version control)