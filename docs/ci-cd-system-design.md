System Design: Secure CI/CD and GitOps for AKS Java Microservices
1. Problem statement
Design a reusable, secure CI/CD system for Java microservices deployed on AKS that:

Provides consistent build, test, and scan for all services using GitHub Actions.

Uses GitOps with Argo CD and Helm to deploy to AKS clusters (dev/prod).!

Centralizes pipeline logic in reusable workflows with least‑privilege security.

Ensures deployments are auditable and repeatable, with Git as the single source of truth.

Assumptions:

Source code hosted on GitHub.

Microservices are Java (Maven) with Dockerfiles and Helm charts.

AKS clusters exist for dev and prod.

ACR is used for container images.

Argo CD runs inside or alongside AKS.

| Layer        | Component                           | Responsibility                                                            |
| ------------ | ----------------------------------- | ------------------------------------------------------------------------- |
| SCM          | GitHub app repos                    | Java code, Dockerfile, Helm chart, caller workflows.docs.github+1         |
| CI           | Reusable GitHub Actions CI workflow | Build/test/scan Java services with Maven.docs.github+1                    |
| CD (image)   | Reusable GitHub Actions CD workflow | Build/push image to ACR, bump Helm values in env repo.ricofritzsche+1     |
| GitOps       | Env repo (GitOps repo)              | Holds env‑specific Helm values and Argo CD Applications.learn.microsoft+1 |
| CD (runtime) | Argo CD on AKS                      | Reconciles desired state from env repo into AKS.argo-cd.readthedocs+1     |
| Registry     | Azure Container Registry            | Stores signed/scanned images.learn.microsoft+1                            |
| Runtime      | AKS clusters                        | Run microservices using Helm charts and Argo CD.learn.microsoft+1         |


2.2 Flow overview
Dev pushes to main or opens PR in app repo.

GitHub Actions CI runs: build + tests + SAST.

On successful push to main, CD workflow builds image, pushes to ACR, and updates values-<env>.yaml in env repo.

Argo CD detects env repo changes and syncs Helm releases into AKS.

3. Detailed workflow design
3.1 Reusable CI workflow (Java/Maven)
Location: platform-ci/.github/workflows/ci-java-maven.yml

Responsibilities:

Checkout code.

Setup JDK and cache Maven dependencies.

Run Maven build/test (mvn verify).

Optionally run CodeQL or other SAST.

Upload test reports.

Key security points:

on: workflow_call to be reused by multiple repos.

Minimal permissions (contents: read, security-events: write only when needed).

No secrets used; safe for PRs.

YAML (with inline comments) is the same as you already have; this document references it as a design artifact.

3.2 Reusable CD workflow (image build + GitOps bump)
Location: platform-ci/.github/workflows/cd-argocd-gitops.yml

Responsibilities:

Authenticate to Azure using GitHub OIDC (no long‑lived secrets).

Build and push container image to ACR.

Checkout env repo and update values-<env>.yaml Helm file with new image tag.

Either:

Push directly (bump-strategy: direct) for lower environments.

Or create a PR for higher environments (e.g. prod) requiring review.

Important inputs:

environment: logical env (dev, prod).

image-name: microservice image name.

env-repo, env-repo-branch, env-path: GitOps repo location.

bump-strategy: direct or pr.

Secrets:

azure-client-id, azure-tenant-id, azure-subscription-id, acr-name for Azure auth.

env-repo-token (GitHub App or PAT) with narrow permissions to env repo only.

Security controls:

permissions: id-token: write for OIDC; no SP password in GitHub.

Use GitHub Environments so environment: prod can require approval.

Images tagged with commit SHA for immutability.

4. Git repositories and structure
4.1 App repo structure
Example:
java-microservice-app/
├─ src/
├─ pom.xml
├─ Dockerfile
├─ charts/
│  └─ java-microservice/           # Helm chart
└─ .github/workflows/
   └─ app-main.yml                 # Calls reusable workflows
app-main.yml:

Triggers on PR and push to main.

Calls ci-java-maven.yml for CI.

On push to main, calls cd-argocd-gitops.yml for dev (and optionally prod on tags).

4.2 Env (GitOps) repo structure
env-aks-gitops/
└─ java-microservice/
   ├─ dev/
   │  └─ values-dev.yaml
   ├─ prod/
   │  └─ values-prod.yaml
   ├─ app-dev.yaml                 # Argo CD Application for dev
   └─ app-prod.yaml                # Argo CD Application for prod
Each values-*.yaml defines:

image.repository and image.tag.

Environment‑specific overrides (replicas, ingress host, etc.).

5. Argo CD and Helm design
5.1 Argo CD deployment model
Argo CD installed in AKS (argocd namespace).

AppProjects created:

dev-project for dev apps and namespaces.

prod-project for prod.

Argo CD Application objects:
Dev:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: java-microservice-dev
  namespace: argocd
spec:
  project: dev-project
  source:
    repoURL: 'https://github.com/org/env-aks-gitops.git'
    targetRevision: main
    path: java-microservice
    helm:
      valueFiles:
        - dev/values-dev.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
Prod is identical but uses prod-project, prod-namespace, and prod/values-prod.yaml.

5.2 Helm patterns
Helm chart resides in app repo, but Argo CD reads it via env repo reference (Git or Helm repo, depending on how you package) following Argo CD Helm integration.

Environment‑specific values kept separate for clean promotion flows.

6. Security and compliance
6.1 GitHub Actions
OIDC for Azure: No service principal secrets in GitHub; tokens are short‑lived and scoped.

Permissions: Set permissions explicitly at job level. Default is contents: read, id-token: write only where needed.

Branch protection: Require CI job (build/test/SAST) to pass before merging to main.

Reusable workflows: Centralized to enforce uniform security policies and easier updates.

6.2 Argo CD and GitOps
RBAC and SSO: Integrate Argo CD with Azure AD; map groups to roles via argocd-rbac-cm.

AppProjects: Limit which repos, namespaces, clusters, and kinds each project can touch.

Secrets management: Use External Secrets Operator, Sealed Secrets, or Key Vault integration; never commit plain secrets.

Network policies: Restrict Argo CD API/UI access and enforce namespace isolation.

7. Non‑functional requirements
Scalability: CI workflows run per service; Argo CD handles many Applications and clusters using ApplicationSets if necessary.

Reliability: GitHub Actions retries, Argo CD self‑heal, and AKS health checks maintain system stability.

Observability:

Actions logs and artifacts for pipeline runs.

Argo CD audit logs for syncs and drift.

AKS telemetry via Azure Monitor/Log Analytics.

8. How to implement
Create platform-ci repo and add:

ci-java-maven.yml

cd-argocd-gitops.yml

Update app repos to:

Add Helm charts (if not present).

Add app-main.yml callers using uses: org/platform-ci/....

Create env repo (env-aks-gitops):

Add values-dev.yaml, values-prod.yaml and Argo CD Applications.

Install and configure Argo CD on AKS.

Configure Azure federated credentials and GitHub environment secrets.

Protect main and prod environments with approvals and policy.
