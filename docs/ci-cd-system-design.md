# System Design: Secure CI/CD and GitOps for AKS Java Microservices

## 1. Problem statement

Design a reusable, secure CI/CD system for Java microservices deployed on AKS that:

- Provides consistent build, test, and scan for all services using GitHub Actions.
- Uses GitOps with Argo CD and Helm to deploy to AKS clusters (dev and prod).
- Centralizes pipeline logic in reusable workflows with least‑privilege security.
- Ensures deployments are auditable and repeatable, with Git as the single source of truth.

Assumptions:

- Source code is hosted on GitHub.
- Microservices are Java (Maven) with Dockerfiles and Helm charts.
- AKS clusters exist for `dev` and `prod`.
- Azure Container Registry (ACR) is used for container images.
- Argo CD runs inside or alongside AKS.

---

## 2. High‑level architecture

### 2.1 Components

| Layer | Component | Responsibility |
| --- | --- | --- |
| SCM | GitHub app repos | Java code, Dockerfile, Helm chart, caller workflows. |
| CI | Reusable GitHub Actions CI workflow | Build, test, and (optionally) scan Java services with Maven. |
| CD (image) | Reusable GitHub Actions CD workflow | Build and push images to ACR, update Helm values in env repo. |
| GitOps | Env (GitOps) repo | Holds env‑specific Helm values and Argo CD Applications. |
| CD (runtime) | Argo CD on AKS | Reconciles desired state from env repo into AKS clusters. |
| Registry | Azure Container Registry | Stores signed/scanned container images. |
| Runtime | AKS clusters | Run microservices using Helm charts and Argo CD. |

### 2.2 Flow overview

1. Developer opens a pull request or pushes to `main` in the app repo.  
2. GitHub Actions CI workflow runs: build, tests, and optional SAST.  
3. On a successful push to `main`, the CD workflow builds an image, pushes it to ACR, and updates `values-<env>.yaml` in the env repo.  
4. Argo CD detects changes in the env repo and syncs Helm releases into the target AKS environment.

---

## 3. Detailed workflow design

### 3.1 Reusable CI workflow (Java/Maven)

**Location:** `platform-ci/.github/workflows/ci-java-maven.yml`

**Responsibilities:**

- Check out code from the calling repository.
- Set up a Java toolchain with Maven dependency caching.
- Run Maven goals (for example, `mvn -B verify`).
- Optionally run CodeQL or other SAST.
- Upload test reports as build artifacts.

**Key security points:**

- Declared with `on: workflow_call` so it can be reused by many repos.
- Uses minimal GitHub token permissions (`contents: read`, `security-events: write` if SAST is enabled).
- Does not require any secrets, so it is safe to run on PRs.

**High‑level behavior (YAML lives in the workflow file, not here):**

- One job `build-test` that:
  - Uses `actions/checkout` to fetch code.
  - Uses `actions/setup-java` with Maven cache enabled.
  - Executes Maven goals passed as inputs.
  - Uploads test reports on success or failure.
- Optional `codeql` job that reuses the official CodeQL workflow for Java.

### 3.2 Reusable CD workflow (image build + GitOps bump)

**Location:** `platform-ci/.github/workflows/cd-argocd-gitops.yml`

**Responsibilities:**

- Authenticate to Azure using GitHub OIDC (no long‑lived credentials in GitHub).
- Build and tag a Docker image for the microservice.
- Push the image to ACR.
- Check out the env (GitOps) repo.
- Update the image tag in the environment‑specific Helm values file (`values-<env>.yaml`).
- Either:
  - Push directly for lower environments (`bump-strategy: direct`), or
  - Open a pull request for higher environments (`bump-strategy: pr`).

**Inputs:**

- `environment` (e.g., `dev`, `prod`).
- `image-name` (e.g., `java-microservice`).
- `dockerfile` and `context` (paths in the app repo).
- `env-repo`, `env-repo-branch`, and `env-path` for the GitOps repo.
- Optional `image-tag` override and `bump-strategy` (`direct` or `pr`).

**Secrets:**

- Azure OIDC details (`azure-client-id`, `azure-tenant-id`, `azure-subscription-id`).
- `acr-name` for the ACR instance.
- `env-repo-token` (GitHub App or PAT restricted to the env repo).

**Key behavior:**

- By default, tags images with the short commit SHA for immutability.
- Uses `az acr login` and `docker build` / `docker push` to publish images.
- Uses a small script (e.g. `sed`) to update the `tag:` field in Helm values.
- Commits and pushes changes to env repo, or opens a PR for review.

---

## 4. Git repositories and structure

### 4.1 App repository layout

Example app repo (`aks-java-microservice-architecture` or similar):

```text
.
├─ src/
├─ pom.xml
├─ Dockerfile
├─ charts/
│  └─ java-microservice/           # Helm chart for the service
└─ .github/
   └─ workflows/
      └─ app-main.yml             # Caller workflow that uses reusable CI/CD
