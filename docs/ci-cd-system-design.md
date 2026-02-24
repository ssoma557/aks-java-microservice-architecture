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

\begin{table}
\begin{tabular}{|l|l|l|}
\hline
Layer & Component & Responsibility \\
\hline
SCM & GitHub app repos & Java code, Dockerfile, Helm chart, caller workflows \\
\hline
CI & Reusable GitHub Actions CI workflow & Build, test, and optionally scan Java services with Maven \\
\hline
CD (image) & Reusable GitHub Actions CD workflow & Build and push images to ACR, update Helm values in env repo \\
\hline
GitOps & Env (GitOps) repo & Holds env‑specific Helm values and Argo CD Applications \\
\hline
CD (runtime) & Argo CD on AKS & Reconciles desired state from env repo into AKS clusters \\
\hline
Registry & Azure Container Registry & Stores signed/scanned container images \\
\hline
Runtime & AKS clusters & Run microservices using Helm charts and Argo CD \\
\hline
\end{tabular}
\caption{CI/CD pipeline components}
\end{table}

### 2.2 Flow overview

\begin{enumerate}
\item Developer opens a pull request or pushes to \texttt{main} in the app repo.
\item GitHub Actions CI workflow runs: build, tests, and optional SAST.
\item On a successful push to \texttt{main}, the CD workflow builds an image, pushes it to ACR, and updates \texttt{values-<env>.yaml} in the env repo.
\item Argo CD detects changes in the env repo and syncs Helm releases into the target AKS environment.
\end{enumerate}

---

## 3. Repositories and directory structure

### 3.1 App repository layout

Example app repo:

\begin{itemize}
\item \texttt{src/} - Java source code
\item \texttt{pom.xml} - Maven project descriptor
\item \texttt{Dockerfile} - Container build definition
\item \texttt{charts/java-microservice/} - Helm chart for the service
\item \texttt{.github/workflows/app-main.yml} - Caller workflow that uses reusable CI/CD
\end{itemize}

### 3.2 Environment (GitOps) repository layout

Example env repo (\texttt{env-aks-gitops}):

\begin{itemize}
\item \texttt{java-microservice/dev/values-dev.yaml} - Dev environment Helm values
\item \texttt{java-microservice/prod/values-prod.yaml} - Prod environment Helm values
\item \texttt{java-microservice/app-dev.yaml} - Argo CD Application for dev
\item \texttt{java-microservice/app-prod.yaml} - Argo CD Application for prod
\end{itemize}

Each \texttt{values-*.yaml} file defines:

\begin{itemize}
\item \texttt{image.repository} - ACR repository URL
\item \texttt{image.tag} - Image tag (updated by CD workflow)
\item \texttt{replicaCount} - Number of pod replicas
\item \texttt{ingress.enabled} - Enable/disable ingress
\item \texttt{ingress.host} - Hostname for the service
\end{itemize}

---

## 4. Reusable CI workflow (Java/Maven)

**Purpose:** Standardized build, test, and optional SAST for all Java microservices.

**Key features:**

\begin{itemize}
\item Declared with \texttt{on: workflow\_call} so it can be reused by many repos
\item Uses minimal GitHub token permissions (\texttt{contents: read}, \texttt{security-events: write} if SAST enabled)
\item Does not require any secrets, so it is safe to run on PRs
\item Caches Maven dependencies for faster builds
\item Uploads test reports as build artifacts
\item Optional CodeQL analysis for security scanning
\end{itemize}

**Workflow structure:**

\begin{enumerate}
\item \textbf{build-test job}
  \begin{itemize}
  \item Checkout source code
  \item Set up JDK with Maven cache enabled
  \item Execute Maven goals (e.g., \texttt{mvn -B verify})
  \item Upload test reports (always runs, even on failure)
  \end{itemize}

\item \textbf{codeql job (optional)}
  \begin{itemize}
  \item Reuses official CodeQL workflow for Java
  \item Runs only if \texttt{enable-sast} input is true
  \end{itemize}
\end{enumerate}

---

## 5. Reusable CD workflow (build image + GitOps bump)

**Purpose:** Build/push image to ACR and update Helm values in the env repo so Argo CD can deploy.

**Key features:**

\begin{itemize}
\item Authenticates to Azure using GitHub OIDC (no long‑lived credentials in GitHub)
\item Builds and tags Docker image for the microservice
\item Pushes image to Azure Container Registry
\item Checks out the env (GitOps) repo
\item Updates the image tag in environment‑specific Helm values file
\item Either pushes directly for lower environments or opens a pull request for higher environments
\end{itemize}

**Inputs:**

\begin{itemize}
\item \texttt{environment} - Logical environment name (e.g., dev, prod)
\item \texttt{image-name} - Docker image name (without registry)
\item \texttt{dockerfile} - Path to Dockerfile (default: \texttt{./Dockerfile})
\item \texttt{context} - Build context directory (default: \texttt{.})
\item \texttt{env-repo} - GitHub repo that holds env manifests
\item \texttt{env-repo-branch} - Target branch in env repo (default: \texttt{main})
\item \texttt{env-path} - Path inside env repo for this app
\item \texttt{image-tag} - Optional explicit tag; defaults to short SHA
\item \texttt{bump-strategy} - \texttt{direct} for direct push or \texttt{pr} for pull request
\end{itemize}

**Secrets:**

\begin{itemize}
\item \texttt{azure-client-id} - Azure service principal client ID for OIDC
\item \texttt{azure-tenant-id} - Azure AD tenant ID
\item \texttt{azure-subscription-id} - Azure subscription ID
\item \texttt{acr-name} - Azure Container Registry name
\item \texttt{env-repo-token} - GitHub App token or PAT scoped narrowly to env repo
\end{itemize}

**Workflow structure:**

\begin{enumerate}
\item Checkout app repo
\item Azure login using OIDC (federated credentials)
\item Set image tag (use input or generate from commit SHA)
\item Build and push image to ACR
  \begin{itemize}
  \item Authenticate to ACR using \texttt{az acr login}
  \item Build Docker image with specified tag
  \item Push image to registry
  \end{itemize}
\item Checkout env repo
\item Update image tag in Helm values file
  \begin{itemize}
  \item Find \texttt{values-<env>.yaml} file
  \item Use \texttt{sed} to update \texttt{tag:} field
  \end{itemize}
\item Commit and push changes (direct strategy)
  \begin{itemize}
  \item Configure Git identity as github-actions bot
  \item Commit changes with descriptive message
  \item Push to env repo branch
  \end{itemize}
\item OR open pull request (pr strategy)
  \begin{itemize}
  \item Create dedicated branch for image bump
  \item Commit changes to branch
  \item Open PR using GitHub CLI
  \end{itemize}
\end{enumerate}

---

## 6. Caller workflow in the app repo

**Purpose:** Wire up triggers and call the reusable CI and CD workflows.

**Trigger configuration:**

\begin{itemize}
\item Pull requests against \texttt{main} branch
\item Pushes to \texttt{main} branch
\item Manual workflow dispatch
\end{itemize}

**Job structure:**

\begin{enumerate}
\item \textbf{ci job}
  \begin{itemize}
  \item Reuses central Java Maven CI workflow
  \item Specifies Java version (e.g., 17)
  \item Defines Maven goals (e.g., \texttt{verify})
  \item Enables SAST scanning
  \end{itemize}

\item \textbf{cd-dev job}
  \begin{itemize}
  \item Depends on \texttt{ci} job completion
  \item Runs only on pushes to \texttt{main} (not on PRs)
  \item Reuses central CD workflow for dev environment
  \item Uses \texttt{direct} bump strategy
  \item Passes dev-specific secrets
  \end{itemize}

\item \textbf{cd-prod job (optional)}
  \begin{itemize}
  \item Similar to cd-dev but for production
  \item Uses \texttt{pr} bump strategy for additional review
  \item Passes prod-specific secrets
  \item May require manual approval via GitHub Environments
  \end{itemize}
\end{enumerate}

---

## 7. Argo CD Applications

**Purpose:** Define how Argo CD deploys the microservice to each environment.

### 7.1 Development Application

**Configuration:**

\begin{itemize}
\item \textbf{Metadata}: Name \texttt{java-microservice-dev}, namespace \texttt{argocd}
\item \textbf{Project}: \texttt{dev-project}
\item \textbf{Source}:
  \begin{itemize}
  \item Repository: \texttt{https://github.com/org/env-aks-gitops.git}
  \item Target revision: \texttt{main}
  \item Path: \texttt{java-microservice}
  \item Helm value files: \texttt{dev/values-dev.yaml}
  \end{itemize}
\item \textbf{Destination}:
  \begin{itemize}
  \item Server: \texttt{https://kubernetes.default.svc}
  \item Namespace: \texttt{dev-namespace}
  \end{itemize}
\item \textbf{Sync policy}:
  \begin{itemize}
  \item Automated with prune and self-heal enabled
  \item Create namespace if not exists
  \item Apply out-of-sync resources only
  \end{itemize}
\end{itemize}

### 7.2 Production Application

**Configuration:**

\begin{itemize}
\item \textbf{Metadata}: Name \texttt{java-microservice-prod}, namespace \texttt{argocd}
\item \textbf{Project}: \texttt{prod-project}
\item \textbf{Source}:
  \begin{itemize}
  \item Repository: \texttt{https://github.com/org/env-aks-gitops.git}
  \item Target revision: \texttt{main}
  \item Path: \texttt{java-microservice}
  \item Helm value files: \texttt{prod/values-prod.yaml}
  \end{itemize}
\item \textbf{Destination}:
  \begin{itemize}
  \item Server: \texttt{https://kubernetes.default.svc}
  \item Namespace: \texttt{prod-namespace}
  \end{itemize}
\item \textbf{Sync policy}:
  \begin{itemize}
  \item Automated with prune and self-heal enabled
  \item Create namespace if not exists
  \item Apply out-of-sync resources only
  \end{itemize}
\end{itemize}

---

## 8. Security and operational considerations

### 8.1 Authentication and authorization

\begin{itemize}
\item \textbf{GitHub to Azure}: Uses OIDC with workload identity federation; no static service principal secrets stored in GitHub
\item \textbf{GitHub token permissions}: Minimized to \texttt{contents: read} and \texttt{id-token: write} for CD workflow
\item \textbf{Env repo access}: GitHub App token or fine-grained PAT scoped only to env repository
\item \textbf{Branch protection}: Protect \texttt{main} branch with required CI checks and reviews
\item \textbf{Environment protection}: Use GitHub Environments for prod with manual approvals
\end{itemize}

### 8.2 GitOps principles

\begin{itemize}
\item \textbf{Git as source of truth}: All runtime configuration changes go through Git commits in env repo
\item \textbf{Declarative deployments}: Argo CD keeps clusters in sync with desired state
\item \textbf{Audit trail}: Full Git history provides auditability of who changed what and when
\item \textbf{Rollback}: Revert Git commits to roll back deployments
\item \textbf{Separation of concerns}: App repo owns code and images; env repo owns deployment configuration
\end{itemize}

### 8.3 Scalability

\begin{itemize}
\item \textbf{Many services}: Reuse same CI/CD workflows across all microservices
\item \textbf{Multiple environments}: Single workflow supports dev, staging, prod with different inputs
\item \textbf{Multiple clusters}: Argo CD Applications can target different Kubernetes clusters
\item \textbf{Multi-region}: Env repo can contain values for regional deployments
\end{itemize}

### 8.4 Reliability and observability

\begin{itemize}
\item \textbf{CI logs}: GitHub Actions provides detailed logs for each pipeline run
\item \textbf{Argo CD sync history}: Track deployment status and health checks
\item \textbf{AKS health}: Monitor pod health, resource usage, and application metrics
\item \textbf{Alerts}: Configure alerts on pipeline failures, sync errors, and health degradation
\item \textbf{Safe rollouts}: Use HPA, PodDisruptionBudgets, and readiness probes for zero-downtime deployments
\end{itemize}

---
