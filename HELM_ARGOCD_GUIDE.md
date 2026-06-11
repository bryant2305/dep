# RentHub Infrastructure - Helm & ArgoCD Structure

This repository contains the Kubernetes infrastructure manifests for the RentHub project using Helm charts and ArgoCD for GitOps-based deployments.

## 📁 Repository Structure

```
renthub-infrastructure/
├── helm/
│   └── renthub-chart/
│       ├── Chart.yaml                 # Helm chart metadata
│       ├── values.yaml                # Default values
│       ├── values-dev.yaml            # Development environment (auto-updated)
│       ├── values-staging.yaml        # Staging environment (manual updates)
│       ├── values-prod.yaml           # Production environment (manual + approved)
│       └── templates/
│           ├── namespace.yaml         # Namespace definition
│           ├── configmap.yaml         # ConfigMaps
│           ├── secrets.yaml           # Secrets
│           ├── storage.yaml           # Storage classes & PVCs
│           ├── services.yaml          # Kubernetes Services
│           ├── statefulsets.yaml      # Database StatefulSets
│           └── deployments.yaml       # Microservice Deployments
│
└── argocd/
    ├── renthub-dev.yaml              # Dev ArgoCD Application (auto-sync)
    ├── renthub-staging.yaml          # Staging ArgoCD Application (manual)
    └── renthub-prod.yaml             # Prod ArgoCD Application (manual)
```

## 🚀 Getting Started

### Prerequisites
- Kubernetes cluster (1.21+)
- Helm 3.x
- ArgoCD 2.x
- kubectl configured with cluster access

### Installation

1. **Install ArgoCD** (if not already installed)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. **Deploy Applications via ArgoCD**
```bash
# Deploy all environments
kubectl apply -f argocd/renthub-dev.yaml
kubectl apply -f argocd/renthub-staging.yaml
kubectl apply -f argocd/renthub-prod.yaml

# Or deploy individually
kubectl apply -f argocd/renthub-dev.yaml -n argocd
```

3. **Access ArgoCD UI**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Login: admin / $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

## 📋 Environment Management

### Development (`values-dev.yaml`)
- **Auto-synced** by ArgoCD workflows
- **Minimal resources** (1 replica for services)
- **Tag**: `dev`
- **Log level**: debug
- **Domain**: renthub-dev.local

### Staging (`values-staging.yaml`)
- **Manual sync** - requires deliberate deployments
- **Production-like resources** (2 replicas for services)
- **Tag**: `staging`
- **Log level**: info
- **Domain**: renthub-staging.local

### Production (`values-prod.yaml`)
- **Manual sync** - requires review/approval before deployment
- **Production resources** (3 replicas for services)
- **Tag**: `latest`
- **Log level**: warn
- **Domain**: renthub.com

## 🔧 Deployment Workflow

### Using Helm Directly
```bash
# Development (with auto-update)
helm upgrade --install renthub-dev helm/renthub-chart \
  -f helm/renthub-chart/values-dev.yaml \
  -n renthub-dev --create-namespace

# Staging (manual)
helm upgrade --install renthub-staging helm/renthub-chart \
  -f helm/renthub-chart/values-staging.yaml \
  -n renthub-staging --create-namespace

# Production (manual)
helm upgrade --install renthub-prod helm/renthub-chart \
  -f helm/renthub-chart/values-prod.yaml \
  -n renthub-prod --create-namespace
```

### Using ArgoCD (Recommended)
1. ArgoCD monitors the repository
2. Changes to `values-*.yaml` trigger ArgoCD sync
3. **Dev**: Auto-syncs immediately
4. **Staging & Prod**: Manual sync required via UI or CLI

```bash
# Manually sync via CLI
argocd app sync renthub-dev
argocd app sync renthub-staging
argocd app sync renthub-prod
```

## 📝 Modifying Values

### For Development (Auto-Updated)
- Workflows can auto-update `values-dev.yaml`
- Changes are reflected on next ArgoCD sync
- Example: CI/CD pipeline updates image tags

```yaml
services:
  gateway:
    image: renthub/gateway:dev  # Updated by CI
    replicas: 1
```

### For Staging (Manual)
- Edit `values-staging.yaml` directly
- Create PR for review
- Merge to main branch
- Trigger ArgoCD sync manually

### For Production (Manual + Approval)
- Edit `values-prod.yaml` with extreme caution
- Require team approval before merge
- Use branch protection rules
- Manual ArgoCD sync after approval

## 🔐 Secrets Management

**⚠️ DO NOT commit actual secrets to git!**

### Options for Secret Management:

1. **Sealed Secrets**
```bash
kubeseal -f secrets.yaml -w sealed-secrets.yaml
```

2. **External Secrets Operator**
- Reference secrets from AWS Secrets Manager, HashiCorp Vault, etc.

3. **ArgoCD Secrets**
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_PASSWORD=actual-password \
  -n renthub-dev
```

4. **Kustomize SecretGenerator**
- Use in production overlays

## 📊 Checking Deployment Status

### Via ArgoCD UI
- Visit: http://localhost:8080
- See sync status, diffs, and logs

### Via kubectl
```bash
# Check applications
kubectl get applications -n argocd

# Check resources in namespaces
kubectl get all -n renthub-dev
kubectl get all -n renthub-staging
kubectl get all -n renthub-prod

# View logs
kubectl logs -n renthub-dev deployment/gateway
```

### Via ArgoCD CLI
```bash
argocd app list
argocd app get renthub-dev
argocd app sync renthub-dev
```

## 🔄 CI/CD Integration

### Updating Dev Environment Automatically
```yaml
# Example GitHub Actions workflow
name: Update Dev Deployment
on:
  push:
    branches: [develop]

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Update image tag
        run: |
          sed -i "s|renthub/gateway:dev|renthub/gateway:${{ github.sha }}|g" \
            helm/renthub-chart/values-dev.yaml
      - name: Commit and push
        run: |
          git config user.email "ci@renthub.com"
          git config user.name "RentHub CI"
          git add helm/renthub-chart/values-dev.yaml
          git commit -m "Update dev image tags"
          git push
```

## 🛠️ Troubleshooting

### ArgoCD not syncing
```bash
# Check sync status
argocd app get renthub-dev
argocd app logs renthub-dev

# Force re-sync
argocd app sync renthub-dev --force
```

### Pod not starting
```bash
# Check pod status
kubectl describe pod -n renthub-dev deployment/gateway

# View logs
kubectl logs -n renthub-dev -l app=gateway
```

### Helm validation issues
```bash
# Validate Helm chart
helm lint helm/renthub-chart

# Dry-run to see generated manifests
helm template renthub-dev helm/renthub-chart \
  -f helm/renthub-chart/values-dev.yaml
```

## 📚 Additional Resources

- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/)

## 🤝 Contributing

1. Create feature branch from `develop`
2. Make changes to values files or templates
3. Test locally: `helm template renthub-chart -f values-*.yaml`
4. Submit PR for review
5. Merge and deploy via ArgoCD
