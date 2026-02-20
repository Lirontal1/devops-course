# CI/CD and GitOps Documentation

This document describes the complete CI/CD pipeline and GitOps workflow for the QuakeWatch application.

## Table of Contents

1. [Git Branching Strategy](#git-branching-strategy)
2. [GitHub Actions Workflows](#github-actions-workflows)
3. [Helm Chart](#helm-chart)
4. [ArgoCD GitOps](#argocd-gitops)
5. [Setup Instructions](#setup-instructions)

## Git Branching Strategy

### Branch Structure

We follow a simplified Git Flow strategy:

- **main**: Production-ready code
- **develop**: Integration branch for features
- **feature/**: Feature branches (e.g., feature/add-logging)
- **bugfix/**: Bug fix branches (e.g., bugfix/fix-api-error)
- **hotfix/**: Emergency fixes for production (e.g., hotfix/critical-bug)

### Workflow

```
feature/new-feature → develop → main
                                  ↓
                            (Production Deploy)
```

### Branch Protection Rules

For **main** branch:
- Require pull request reviews
- Require status checks to pass (CI workflow)
- No direct commits
- Merge via Pull Request only

### Development Workflow

1. Create feature branch from main:
   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/your-feature
   ```

2. Make changes and commit:
   ```bash
   git add .
   git commit -m "Add: your feature description"
   ```

3. Push to remote:
   ```bash
   git push origin feature/your-feature
   ```

4. Create Pull Request to main
   - CI workflow runs automatically
   - Request code review
   - Merge when approved

5. After merge to main:
   - CD workflow triggers automatically
   - Docker image built and pushed to DockerHub
   - ArgoCD detects changes and syncs

## GitHub Actions Workflows

### CI Workflow (ci.yml)

**Trigger**: Push to any branch except main

**Purpose**: Validate code quality and ensure Docker image builds successfully

**Steps**:
1. Checkout code
2. Set up Python 3.11
3. Install dependencies (pip, pylint, requirements.txt)
4. Run pylint on all Python files
5. Set up Docker Buildx
6. Build Docker image
7. Test Docker image (run container and verify accessibility)

**Example**:
```bash
# Triggered on:
git push origin feature/new-feature
```

**Configuration**:
```yaml
on:
  push:
    branches-ignore:
      - main
```

### CD Workflow (cd.yml)

**Trigger**: Pull Request merged to main branch

**Purpose**: Build and push production Docker image to DockerHub

**Steps**:
1. Checkout code
2. Set up Docker Buildx
3. Login to DockerHub (using secrets)
4. Generate version tag (YYYYMMDD-githash)
5. Build and push Docker image with two tags:
   - `liron29/quakewatch:YYYYMMDD-githash`
   - `liron29/quakewatch:latest`
6. Use layer caching for faster builds

**Example**:
```bash
# Triggered when PR is merged to main
```

**Configuration**:
```yaml
on:
  pull_request:
    types:
      - closed
    branches:
      - main
```

### Required GitHub Secrets

Add these secrets in GitHub repository settings (Settings → Secrets and variables → Actions):

1. **DOCKERHUB_USERNAME**: Your DockerHub username (e.g., liron29)
2. **DOCKERHUB_TOKEN**: DockerHub access token

**Creating DockerHub Token**:
```
1. Login to DockerHub
2. Account Settings → Security → New Access Token
3. Name: github-actions
4. Permissions: Read, Write, Delete
5. Copy token and add to GitHub secrets
```

## Helm Chart

### Chart Structure

```
helm/
├── Chart.yaml           # Chart metadata
├── values.yaml          # Default configuration values
├── templates/           # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── ...
└── quakewatch-1.0.0.tgz # Packaged chart
```

### Key Configuration (values.yaml)

```yaml
replicaCount: 3
image:
  repository: liron29/quakewatch
  tag: latest
service:
  type: LoadBalancer
  port: 80
  targetPort: 5000
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Packaging and Publishing

```bash
# Lint chart
helm lint helm/

# Package chart
helm package helm/

# Test installation locally
helm install quakewatch helm/ --dry-run --debug

# Install chart
helm install quakewatch helm/

# Upgrade chart
helm upgrade quakewatch helm/

# Uninstall
helm uninstall quakewatch
```

### Publishing to DockerHub (OCI Registry)

```bash
# Login to DockerHub
helm registry login registry-1.docker.io -u liron29

# Package chart
helm package helm/

# Push to DockerHub
helm push quakewatch-1.0.0.tgz oci://registry-1.docker.io/liron29

# Pull from DockerHub
helm pull oci://registry-1.docker.io/liron29/quakewatch --version 1.0.0

# Install from DockerHub
helm install quakewatch oci://registry-1.docker.io/liron29/quakewatch --version 1.0.0
```

## ArgoCD GitOps

### What is ArgoCD?

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It monitors Git repositories for changes and automatically syncs the desired state to your cluster.

### ArgoCD Application Configuration

Location: `argocd/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: quakewatch
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/QuakeWatch.git
    targetRevision: main
    path: helm
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Sync when cluster state drifts
      allowEmpty: false
```

### Installing ArgoCD

```bash
# Create argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access ArgoCD UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Or expose via LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Deploying QuakeWatch with ArgoCD

```bash
# Update the repoURL in argocd/application.yaml with your GitHub repository

# Apply ArgoCD application
kubectl apply -f argocd/application.yaml

# Check application status
kubectl get applications -n argocd

# View in ArgoCD UI
# Navigate to http://localhost:8080 (if using port-forward)
# Login: admin / <password from above>
```

### ArgoCD CLI Commands

```bash
# Install ArgoCD CLI
brew install argocd  # macOS
# or download from: https://argo-cd.readthedocs.io/en/stable/cli_installation/

# Login to ArgoCD
argocd login localhost:8080

# List applications
argocd app list

# Get application details
argocd app get quakewatch

# Sync application manually
argocd app sync quakewatch

# View application logs
argocd app logs quakewatch

# Delete application
argocd app delete quakewatch
```

### Making Changes and Observing ArgoCD

**Scenario**: Update replica count

1. Edit `helm/values.yaml`:
   ```yaml
   replicaCount: 5  # Changed from 3
   ```

2. Commit and push:
   ```bash
   git add helm/values.yaml
   git commit -m "Update: increase replicas to 5"
   git push origin main
   ```

3. ArgoCD detects change:
   - Auto-syncs after ~3 minutes (default)
   - Or click "Sync" in UI immediately

4. Verify in cluster:
   ```bash
   kubectl get deployments
   # Should show 5 replicas
   ```

### ArgoCD Sync Policies

**Manual Sync** (default):
```yaml
syncPolicy: {}
```

**Automated Sync**:
```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources not in Git
    selfHeal: true   # Revert manual changes
```

**Self-Heal Demonstration**:
```bash
# Manually scale deployment
kubectl scale deployment quakewatch --replicas=10

# ArgoCD detects drift and reverts to Git state (3 replicas)
# Check logs: argocd app logs quakewatch
```

## Setup Instructions

### Prerequisites

- GitHub account
- DockerHub account
- Kubernetes cluster (Docker Desktop)
- kubectl installed
- helm installed
- ArgoCD installed

### Step 1: Fork and Clone Repository

```bash
# Fork the repository on GitHub

# Clone your fork
git clone https://github.com/YOUR_USERNAME/QuakeWatch.git
cd QuakeWatch
```

### Step 2: Configure GitHub Secrets

1. Go to GitHub repository → Settings → Secrets and variables → Actions
2. Add secrets:
   - DOCKERHUB_USERNAME
   - DOCKERHUB_TOKEN

### Step 3: Test CI Workflow

```bash
# Create feature branch
git checkout -b feature/test-ci

# Make a change
echo "# Test" >> README.md

# Commit and push
git add README.md
git commit -m "Test: CI workflow"
git push origin feature/test-ci

# Check GitHub Actions tab for running workflow
```

### Step 4: Test CD Workflow

```bash
# Create Pull Request from feature/test-ci to main
# Merge the PR
# Check GitHub Actions tab for CD workflow
# Verify image on DockerHub
```

### Step 5: Deploy with ArgoCD

```bash
# Update argocd/application.yaml with your repo URL

# Install ArgoCD (if not installed)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Deploy application
kubectl apply -f argocd/application.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Step 6: Verify Deployment

```bash
# Check pods
kubectl get pods

# Check service
kubectl get svc

# Access application
kubectl port-forward svc/quakewatch 5000:80
# Navigate to http://localhost:5000
```

## Complete Workflow Diagram

```
┌─────────────┐
│  Developer  │
└──────┬──────┘
       │
       │ 1. git push feature/branch
       ▼
┌─────────────────────┐
│   GitHub Actions    │
│   (CI Workflow)     │
│  - Pylint           │
│  - Build Docker     │
│  - Test             │
└─────────────────────┘
       │
       │ 2. Create PR → Merge to main
       ▼
┌─────────────────────┐
│   GitHub Actions    │
│   (CD Workflow)     │
│  - Build & Push     │
│  - DockerHub        │
└──────┬──────────────┘
       │
       │ 3. Image pushed
       ▼
┌─────────────────────┐
│     DockerHub       │
│  liron29/quakewatch │
│  - latest           │
│  - version tag      │
└─────────────────────┘
       │
       │ 4. Helm chart in Git
       │    (monitored by ArgoCD)
       ▼
┌─────────────────────┐
│      ArgoCD         │
│  - Monitors Git     │
│  - Auto Sync        │
│  - Self Heal        │
└──────┬──────────────┘
       │
       │ 5. Deploy to cluster
       ▼
┌─────────────────────┐
│   Kubernetes        │
│   - Deployment      │
│   - Service         │
│   - HPA             │
└─────────────────────┘
```

## Troubleshooting

### CI Workflow Fails

```bash
# Check pylint errors
# View GitHub Actions logs
# Fix issues locally:
pylint *.py --disable=C0114,C0115,C0116,R0903
```

### CD Workflow Not Triggering

```bash
# Verify secrets are set correctly
# Check PR was merged (not closed without merge)
# View GitHub Actions logs
```

### ArgoCD Not Syncing

```bash
# Check application status
kubectl describe application quakewatch -n argocd

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Manual sync
argocd app sync quakewatch
```

### Helm Chart Issues

```bash
# Lint chart
helm lint helm/

# Dry run
helm install quakewatch helm/ --dry-run --debug

# Check values
helm get values quakewatch
```

## Best Practices

1. **Always create feature branches** - Never commit directly to main
2. **Write meaningful commit messages** - Use conventional commits format
3. **Test locally first** - Run pylint and docker build before pushing
4. **Keep Helm values configurable** - Use values.yaml for environment-specific settings
5. **Monitor ArgoCD sync status** - Ensure automated sync is working
6. **Use semantic versioning** - Tag releases properly
7. **Document changes** - Update documentation with code changes
8. **Review PR carefully** - CD triggers on merge, ensure quality

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
