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
| CI | Reusable GitHub Actions CI workflow | Build, test, and optionally scan Java services with Maven. |
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

Location: platform-ci/.github/workflows/ci-java-maven.yml

Responsibilities:

- Check out code from the calling repository.
- Set up a Java toolchain with Maven dependency caching.
- Run Maven goals (for example, mvn -B verify).
- Optionally run CodeQL or other SAST.
- Upload test reports as build artifacts.

Key security points:

- Declared with on: workflow_call so it can be reused by many repos.
- Uses minimal GitHub token permissions (contents: read, security-events: write if SAST is enabled).
- Does not require any secrets, so it is safe to run on PRs.

### 3.2 Reusable CD workflow (image build + GitOps bump)

Location: platform-ci/.github/workflows/cd-argocd-gitops.yml

Responsibilities:

- Authenticate to Azure using GitHub OIDC (no long‑lived credentials in GitHub).
- Build and tag a Docker image for the microservice.
- Push the image to ACR.
- Check out the env (GitOps) repo.
- Update the image tag in the environment‑specific Helm values file (values-<env>.yaml).
- Either push directly (bump-strategy: direct) or open a PR (bump-strategy: pr).

Inputs:

- environment (e.g., dev, prod).
- image-name (e.g., java-microservice).
- dockerfile and context (paths in the app repo).
- env-repo, env-repo-branch, and env-path for the GitOps repo.
- Optional image-tag and bump-strategy (direct or pr).

Secrets:

- Azure OIDC details (azure-client-id, azure-tenant-id, azure-subscription-id).
- acr-name for the ACR instance.
- env-repo-token (GitHub App or PAT restricted to the env repo).

---

## 4. Git repositories and structure

### 4.1 App repository layout

Example app repo:

.  
├─ src/  
├─ pom.xml  
├─ Dockerfile  
├─ charts/  
│  └─ java-microservice/           # Helm chart for the service  
└─ .github/  
   └─ workflows/  
      └─ app-main.yml              # Caller workflow that uses reusable CI/CD  

### 4.2 Env (GitOps) repository layout

Example env repo (env-aks-gitops):

env-aks-gitops/  
└─ java-microservice/  
   ├─ dev/  
   │  └─ values-dev.yaml  
   ├─ prod/  
   │  └─ values-prod.yaml  
   ├─ app-dev.yaml                 # Argo CD Application for dev  
   └─ app-prod.yaml                # Argo CD Application for prod  

Each values-*.yaml file defines:

image:  
  repository: myacrdev.azurecr.io/java-microservice  
  tag: "sha-initial"  

replicaCount: 2  
ingress:  
  enabled: true  
  host: api-dev.company.com  

---

## 5. Reusable CI workflow YAML

name: Reusable CI - Java Maven

on:  
  workflow_call:  
    inputs:  
      java-version:  
        required: false  
        type: string  
        default: '17'  
      maven-goals:  
        required: false  
        type: string  
        default: 'verify'  
      enable-sast:  
        required: false  
        type: boolean  
        default: true  

permissions:  
  contents: read  

jobs:  
  build-test:  
    runs-on: ubuntu-latest  
    permissions:  
      contents: read  
      security-events: write  
      actions: read  
    steps:  
      - name: Checkout source  
        uses: actions/checkout@v4  

      - name: Set up JDK  
        uses: actions/setup-java@v4  
        with:  
          distribution: temurin  
          java-version: ${{ inputs.java-version }}  
          cache: maven  

      - name: Build and test with Maven  
        run: mvn -B ${{ inputs.maven-goals }}  

      - name: Upload test reports  
        if: always()  
        uses: actions/upload-artifact@v4  
        with:  
          name: maven-test-reports  
          path: |  
            **/target/surefire-reports/*.xml  
            **/target/failsafe-reports/*.xml  

  codeql:  
    if: inputs.enable-sast == true  
    uses: github/codeql-action/.github/workflows/codeql-analysis.yml@v3  
    with:  
      languages: 'java'  

---

## 6. Reusable CD workflow YAML

name: Reusable CD - GitOps (Argo CD + AKS)

on:  
  workflow_call:  
    inputs:  
      environment:  
        required: true  
        type: string  
      image-name:  
        required: true  
        type: string  
      dockerfile:  
        required: false  
        type: string  
        default: './Dockerfile'  
      context:  
        required: false  
        type: string  
        default: '.'  
      env-repo:  
        required: true  
        type: string  
      env-repo-branch:  
        required: false  
        type: string  
        default: 'main'  
      env-path:  
        required: true  
        type: string  
      image-tag:  
        required: false  
        type: string  
      bump-strategy:  
        required: false  
        type: string  
        default: 'direct'  
    secrets:  
      azure-client-id:  
        required: true  
      azure-tenant-id:  
        required: true  
      azure-subscription-id:  
        required: true  
      acr-name:  
        required: true  
      env-repo-token:  
        required: true  

permissions:  
  contents: read  
  id-token: write  

jobs:  
  build-and-bump:  
    runs-on: ubuntu-latest  
    environment: ${{ inputs.environment }}  
    permissions:  
      contents: read  
      id-token: write  

    steps:  
      - name: Checkout app repo  
        uses: actions/checkout@v4  

      - name: Azure login (OIDC)  
        uses: azure/login@v2  
        with:  
          client-id: ${{ secrets.azure-client-id }}  
          tenant-id: ${{ secrets.azure-tenant-id }}  
          subscription-id: ${{ secrets.azure-subscription-id }}  

      - name: Set image tag  
        id: vars  
        run: |  
          TAG="${{ inputs.image-tag }}"  
          if [ -z "$TAG" ]; then  
            TAG="${GITHUB_SHA::7}"  
          fi  
          echo "tag=$TAG" >> $GITHUB_OUTPUT  

      - name: Build and push image to ACR  
        env:  
          ACR_NAME: ${{ secrets.acr-name }}  
        run: |  
          IMAGE="${ACR_NAME}.azurecr.io/${{ inputs.image-name }}:${{ steps.vars.outputs.tag }}"  
          echo "IMAGE=$IMAGE" >> $GITHUB_ENV  

          az acr login --name "${ACR_NAME}"  
          docker build -f "${{ inputs.dockerfile }}" -t "$IMAGE" "${{ inputs.context }}"  
          docker push "$IMAGE"  

      - name: Checkout env repo  
        uses: actions/checkout@v4  
        with:  
          repository: ${{ inputs.env-repo }}  
          token: ${{ secrets.env-repo-token }}  
          ref: ${{ inputs.env-repo-branch }}  
          path: env-repo  

      - name: Update image tag in env repo (Helm values.yaml)  
        working-directory: env-repo/${{ inputs.env-path }}  
        env:  
          IMAGE: ${{ env.IMAGE }}  
        run: |  
          FILE="values-${{ inputs.environment }}.yaml"  
          if [ ! -f "$FILE" ]; then  
            echo "Values file $FILE not found"  
            exit 1  
          fi  

          NEW_TAG="${IMAGE##*:}"  
          echo "Setting image tag to ${NEW_TAG} in $FILE"  

          sed -i -E "s/^([[:space:]]*tag:[[:space:]]*).*/\1\"${NEW_TAG}\"/" "$FILE"  

      - name: Commit changes (direct push)  
        if: ${{ inputs.bump-strategy == 'direct' }}  
        working-directory: env-repo  
        run: |  
          git config user.name "github-actions[bot]"  
          git config user.email "github-actions[bot]@users.noreply.github.com"  

          if git diff --quiet; then  
            echo "No changes to commit."  
            exit 0  
          fi  

          git commit -am "chore: bump ${{ inputs.image-name }} to ${{  
            steps.vars.outputs.tag }} (${{  
            inputs.environment }})"  
          git push origin ${{ inputs.env-repo-branch }}  

      - name: Open PR instead of direct push  
        if: ${{ inputs.bump-strategy == 'pr' }}  
        working-directory: env-repo  
        env:  
          GH_TOKEN: ${{ secrets.env-repo-token }}  
        run: |  
          BRANCH="bump-${{ inputs.image-name }}-${{ steps.vars.outputs.tag }}-${{ inputs.environment }}"  
          git config user.name "github-actions[bot]"  
          git config user.email "github-actions[bot]@users.noreply.github.com"  

          git checkout -b "$BRANCH"  
          git commit -am "chore: bump ${{ inputs.image-name }} to ${{  
            steps.vars.outputs.tag }} (${{  
            inputs.environment }})"  
          git push origin "$BRANCH"  

          gh pr create \  
            --title "Bump ${{ inputs.image-name }} to ${{  
              steps.vars.outputs.tag }} (${{  
              inputs.environment }})" \  
            --body "Automated image bump for environment \`${{  
              inputs.environment }}\`." \  
            --base "${{ inputs.env-repo-branch }}" \  
            --head "$BRANCH"  

---

## 7. Caller workflow in the app repo

name: CI/CD - Java Microservice with Argo CD GitOps

on:  
  pull_request:  
    branches: [ main ]  
  push:  
    branches: [ main ]  
  workflow_dispatch: {}  

permissions:  
  contents: read  

jobs:  
  ci:  
    uses: org/platform-ci/.github/workflows/ci-java-maven.yml@main  
    with:  
      java-version: '17'  
      maven-goals: 'verify'  
      enable-sast: true  

  cd-dev:  
    needs: ci  
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'  
    uses: org/platform-ci/.github/workflows/cd-argocd-gitops.yml@main  
    with:  
      environment: 'dev'  
      image-name: 'java-microservice'  
      dockerfile: './Dockerfile'  
      context: '.'  
      env-repo: 'org/env-aks-gitops'  
      env-repo-branch: 'main'  
      env-path: 'java-microservice'  
      bump-strategy: 'direct'  
    secrets:  
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID_DEV }}  
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}  
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}  
      acr-name: ${{ secrets.ACR_NAME_DEV }}  
      env-repo-token: ${{ secrets.ENV_REPO_TOKEN_DEV }}  

---

## 8. Argo CD Applications

# env-aks-gitops/java-microservice/app-dev.yaml
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

# env-aks-gitops/java-microservice/app-prod.yaml
apiVersion: argoproj.io/v1alpha1  
kind: Application  
metadata:  
  name: java-microservice-prod  
  namespace: argocd  
spec:  
  project: prod-project  
  source:  
    repoURL: 'https://github.com/org/env-aks-gitops.git'  
    targetRevision: main  
    path: java-microservice  
    helm:  
      valueFiles:  
        - prod/values-prod.yaml  
  destination:  
    server: 'https://kubernetes.default.svc'  
    namespace: prod-namespace  
  syncPolicy:  
    automated:  
      prune: true  
      selfHeal: true  
    syncOptions:  
      - CreateNamespace=true  
      - ApplyOutOfSyncOnly=true  
