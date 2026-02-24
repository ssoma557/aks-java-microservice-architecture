# Secure CI/CD and GitOps with GitHub Actions, Argo CD, and Helm

This document describes a secure, reusable CI/CD design for a Java microservice running on AKS using:

- GitHub Actions for CI and image build.
- GitHub Actions reusable workflows for CD.
- Argo CD and Helm for GitOps‑based deployment.

---

## 1. Goals and architecture overview

### Objectives

- Build, test, and scan Java services on every pull request and push.
- Build and push immutable container images to Azure Container Registry (ACR).
- Promote changes to AKS through GitOps: Argo CD reconciles AKS to a Git “environment” repo.
- Use reusable workflows so multiple microservices can share the same pipeline logic.
- Follow security best practices for GitHub Actions, Azure auth (OIDC), and GitOps.

### Repository layout

We use two main repos:

- **App repo**: Java microservice code, Dockerfile, Helm chart, CI/CD caller workflow.
- **Env (GitOps) repo**: Argo CD `Application` manifests and environment‑specific Helm values for dev and prod.

Example:

```text
# App repo (java-microservice-app)
.
├─ src/
├─ pom.xml
├─ Dockerfile
├─ charts/
│  └─ java-microservice/   # Helm chart
└─ .github/workflows/
   └─ app-main.yml         # Calls reusable CI and CD workflows

# Env repo (env-aks-gitops)
.
└─ java-microservice/
   ├─ dev/
   │  └─ values-dev.yaml
   ├─ prod/
   │  └─ values-prod.yaml
   ├─ app-dev.yaml         # Argo CD Application (dev)
   └─ app-prod.yaml        # Argo CD Application (prod)

