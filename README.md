# ArgoCD Applications

This directory contains ArgoCD Application manifests for managing deployments using the GitOps methodology.

## 📋 Overview

These manifests implement the **App of Apps** pattern, allowing you to manage multiple applications through a single root application. ArgoCD will automatically sync your applications from Git repositories to your Kubernetes cluster.

## 📁 Files

### 1. `argocd-app.yaml`
**Purpose**: ArgoCD self-management application

This manifest deploys ArgoCD itself using Helm, enabling ArgoCD to manage its own configuration through GitOps.

**Key Features:**
- Deploys ArgoCD via Helm chart (version 7.7.11)
- Configures LoadBalancer service for external access
- Sets up resource limits for cost optimization
- Configures repository connections
- Enables automated sync and self-healing

**Configuration:**
- **Server Replicas**: 2 (for high availability)
- **Controller Replicas**: 1
- **Repo Server Replicas**: 2
- **Service Type**: LoadBalancer
- **Insecure Mode**: Disabled (uses HTTPS)

### 2. `root-app.yaml`
**Purpose**: Root application implementing the App of Apps pattern

This is the parent application that manages all other applications. It monitors a Git repository and automatically deploys any application manifests it finds.

**Key Features:**
- Implements App of Apps pattern
- Monitors Git repository for application definitions
- Automatically syncs child applications
- Enables GitOps workflow

**Configuration:**
- **Repository**: `https://github.com/Himansri21/Argocd-Gitops`
- **Branch**: `main`
- **Path**: Root directory (`.`)
- **Includes**: `website-app.yaml`

### 3. `website-app.yaml`
**Purpose**: Example application deployment (Portfolio website)

This manifest deploys your portfolio/website application to the Kubernetes cluster.

**Key Features:**
- Deploys application from Git repository
- Creates dedicated namespace (`website`)
- Automated sync with retry logic
- Handles image updates gracefully
- Implements proper cleanup on deletion

**Configuration:**
- **Repository**: `https://github.com/Himansri21/gitops`
- **Branch**: `master`
- **Manifests Path**: `k8s/`
- **Namespace**: `website`
- **Retry Attempts**: 5 with exponential backoff

## 🚀 Quick Start

### Prerequisites

1. ArgoCD installed on your GKE cluster (handled by Terraform)
2. kubectl configured to access your cluster
3. Git repositories set up with your application manifests

### Deployment Order

#### Step 1: Deploy ArgoCD (Optional - if not using Terraform)

If you didn't deploy ArgoCD via Terraform, apply this manifest:

```bash
kubectl apply -f argocd-app.yaml
```

Wait for ArgoCD to be ready:
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

#### Step 2: Deploy Root App of Apps

```bash
kubectl apply -f root-app.yaml
```

This creates the root application that will manage all child applications.

#### Step 3: Verify Deployment

Check ArgoCD applications:
```bash
kubectl get applications -n argocd
```

Expected output:
```
NAME                STATUS   HEALTH   SYNC
root-app-of-apps    Synced   Healthy  Auto
portfolio           Synced   Healthy  Auto
```

## ⚙️ Configuration

### Customizing for Your Environment

Before deploying, update the following in each file:

#### `argocd-app.yaml`
```yaml
# Update domain (line 13)
domain: argocd.yourdomain.com

# Update repository URLs (lines 60-65)
repositories:
  - url: https://github.com/YOUR_USERNAME/YOUR_INFRA_REPO
    type: git
  - url: https://github.com/YOUR_USERNAME/YOUR_APP_REPO
    type: git
```

#### `root-app.yaml`
```yaml
# Update repository URL (line 11)
repoURL: https://github.com/YOUR_USERNAME/argocd-apps-repo

# Update branch if needed (line 12)
targetRevision: main  # or master, develop, etc.

# Update path if manifests are in subdirectory (line 13)
path: apps/  # e.g., "apps/", "manifests/", etc.
```

#### `website-app.yaml`
```yaml
# Update repository URL (line 11)
repoURL: https://github.com/YOUR_USERNAME/your-app-repo

# Update branch (line 12)
targetRevision: main  # or master

# Update manifests path (line 13)
path: k8s  # or manifests, helm, etc.

# Update namespace (line 17)
namespace: your-app-namespace
```

## 📊 Application Structure

### App of Apps Pattern

```
root-app-of-apps (root-app.yaml)
│
└── portfolio (website-app.yaml)
    ├── Deployment
    ├── Service
    ├── Ingress
    └── ConfigMap
```

### How It Works

1. **Root App** monitors the Git repository for application definitions
2. When it finds `website-app.yaml`, it creates the Portfolio application
3. **Portfolio App** syncs Kubernetes manifests from its repository
4. ArgoCD continuously monitors both repositories for changes
5. Any Git commit triggers automatic synchronization

## 🔄 GitOps Workflow

### Making Changes

1. **Update your application code**
2. **Build and push new container image**
3. **Update Kubernetes manifests** in Git (e.g., new image tag)
4. **ArgoCD automatically detects** the change
5. **ArgoCD syncs** the new version to the cluster

### Manual Sync (if needed)

```bash
# Sync specific application
kubectl patch application portfolio -n argocd \
  --type merge -p '{"operation":{"sync":{}}}'

# Or use ArgoCD CLI
argocd app sync portfolio
```

## 📝 Sync Policies

### Automated Sync
All applications have automated sync enabled:
- **Auto-prune**: Remove resources deleted from Git
- **Self-heal**: Revert manual changes to match Git

### Retry Policy
`website-app.yaml` includes retry logic:
- **Initial delay**: 5 seconds
- **Factor**: 2 (exponential backoff)
- **Max duration**: 3 minutes
- **Max attempts**: 5

## 🔍 Monitoring

### Check Application Status

```bash
# List all applications
kubectl get applications -n argocd

# Detailed status
kubectl describe application portfolio -n argocd

# Check sync status
kubectl get application portfolio -n argocd -o jsonpath='{.status.sync.status}'
```

### View in ArgoCD UI

1. Get ArgoCD URL:
   ```bash
   kubectl get svc argocd-server -n argocd
   ```

2. Get admin password:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o jsonpath="{.data.password}" | base64 -d
   ```

3. Access UI at `http://<EXTERNAL-IP>`

## 🐛 Troubleshooting

### Application Not Syncing

**Check application status:**
```bash
kubectl describe application portfolio -n argocd
```

**Common issues:**
- Repository URL incorrect
- Branch/path doesn't exist
- Authentication required (for private repos)
- Manifest validation errors

**Solution:**
```bash
# Force refresh
argocd app get portfolio --refresh

# Manual sync
argocd app sync portfolio --force
```

### Repository Connection Failed

**Error:** `Failed to load target state: authentication required`

**Solution - Add SSH key or access token:**
```bash
# For private repositories
argocd repo add https://github.com/username/repo \
  --username YOUR_USERNAME \
  --password YOUR_GITHUB_TOKEN
```

### Sync Hanging or Timeout

**Check sync operation:**
```bash
kubectl get application portfolio -n argocd -o yaml | grep -A 20 operation
```

**Cancel stuck sync:**
```bash
argocd app terminate-op portfolio
```

### Health Check Failures

**View health status:**
```bash
kubectl get application portfolio -n argocd -o jsonpath='{.status.health}'
```

**Common causes:**
- Pod not starting (check logs)
- Service misconfigured
- Resource limits too low

## 🔐 Security Best Practices

### Private Repositories

For private repositories, configure authentication:

**Option 1: SSH Keys**
```bash
argocd repo add git@github.com:username/repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

**Option 2: Access Token**
```bash
argocd repo add https://github.com/username/repo \
  --username <username> \
  --password <token>
```

### HTTPS with TLS

Update `argocd-app.yaml` for production:
```yaml
configs:
  params:
    server.insecure: false  # Enforce HTTPS
```

Then configure Ingress with TLS certificate.

## 📚 Advanced Patterns

### Multiple Applications

Add more applications to your root app:

1. Create new application manifest (e.g., `backend-app.yaml`)
2. Place it in the repository monitored by `root-app.yaml`
3. Root app automatically detects and deploys it

### Application Sets

For managing multiple similar applications (e.g., multi-tenant):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-tenant
```

### Sync Waves

Control deployment order using annotations:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploys first
```

## 🎯 Best Practices

1. **Use separate repositories**
   - Infrastructure code (Terraform, ArgoCD config)
   - Application code (K8s manifests)

2. **Branch strategy**
   - `main`/`master` → Production
   - `develop` → Staging
   - Feature branches → Development

3. **Resource management**
   - Always set resource limits
   - Use horizontal pod autoscaling
   - Monitor resource usage

4. **Version control**
   - Tag releases in Git
   - Use semantic versioning
   - Keep detailed commit messages

5. **Testing**
   - Validate manifests before committing
   - Use staging environment
   - Implement rollback procedures

## 📖 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://www.gitops.tech/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

## 🤝 Contributing

To add new applications:

1. Create application manifest (e.g., `new-app.yaml`)
2. Follow the structure of `website-app.yaml`
3. Update repository URLs and paths
4. Commit to the root app's monitored repository
5. ArgoCD automatically deploys it

## 💡 Examples

### Example: Adding a Backend Application

Create `backend-app.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Himansri21/backend-api
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: backend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Example: Multi-Environment Setup

```yaml
# production-app.yaml
metadata:
  name: app-production
spec:
  source:
    targetRevision: main
    path: k8s/overlays/production

# staging-app.yaml
metadata:
  name: app-staging
spec:
  source:
    targetRevision: develop
    path: k8s/overlays/staging
```

## 🔄 Update Workflow

### Standard Update Process

1. **Develop locally**
   ```bash
   # Make changes to your app
   git checkout -b feature/new-feature
   ```

2. **Update manifests**
   ```bash
   # Update image tag in deployment
   sed -i 's/image: app:v1.0/image: app:v2.0/' k8s/deployment.yaml
   ```

3. **Commit and push**
   ```bash
   git add k8s/
   git commit -m "feat: update to v2.0"
   git push origin feature/new-feature
   ```

4. **Merge to main**
   ```bash
   # Create PR, review, merge
   # ArgoCD automatically deploys
   ```

5. **Monitor deployment**
   ```bash
   argocd app get portfolio --watch
   ```

## ⚡ Quick Commands Reference

```bash
# List all applications
kubectl get apps -n argocd

# Application details
argocd app get <app-name>

# Sync application
argocd app sync <app-name>

# View logs
argocd app logs <app-name>

# Rollback
argocd app rollback <app-name> <revision>

# Delete application
kubectl delete application <app-name> -n argocd

# Refresh without sync
argocd app get <app-name> --refresh
```

---

**Note**: These manifests are designed to work with the GKE cluster provisioned by the Terraform configuration in the parent directory.
