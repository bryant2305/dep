# Quick Start Guide - RentHub Infrastructure

## 🚀 Get Started in 5 Minutes

### 1. Prerequisites Check
```bash
# Verify you have the required tools
helm version          # Should be 3.x
kubectl version       # Should be 1.21+
argocd version        # If ArgoCD is installed
```

### 2. Deploy via Helm (Direct)
```bash
# Deploy dev environment
helm install renthub-dev ./helm/renthub-chart \
  -f ./helm/renthub-chart/values-dev.yaml \
  -n renthub-dev --create-namespace

# Verify deployment
kubectl get pods -n renthub-dev
```

### 3. Deploy via ArgoCD (Recommended)
```bash
# Apply ArgoCD applications
kubectl apply -f argocd/renthub-dev.yaml -n argocd

# Check sync status
argocd app get renthub-dev
```

### 4. Access Services
```bash
# Port-forward to gateway
kubectl port-forward -n renthub-dev svc/gateway-service 3333:3333

# Test gateway health
curl http://localhost:3333/health
```

## 📊 Check Status

```bash
# All pods running?
kubectl get pods -n renthub-dev

# Services accessible?
kubectl get svc -n renthub-dev

# ArgoCD sync status
argocd app list
```

## 🔄 Make Changes

### Update Development (Auto-Deployed)
1. Modify image tag in `helm/renthub-chart/values-dev.yaml`
2. Push to develop branch
3. GitHub Actions auto-updates and pushes changes
4. ArgoCD auto-syncs → done! ✓

### Update Staging or Production (Manual)
1. Edit `helm/renthub-chart/values-staging.yaml` or `values-prod.yaml`
2. Create pull request
3. Get team review
4. Merge to main
5. Manual sync via: `argocd app sync renthub-staging` or UI

## 🛠️ Useful Commands

```bash
# View Helm values
helm show values ./helm/renthub-chart/

# Dry-run a deployment
helm template renthub-dev ./helm/renthub-chart \
  -f ./helm/renthub-chart/values-dev.yaml

# Scale a deployment
kubectl scale deployment gateway -n renthub-dev --replicas=3

# View logs
kubectl logs -n renthub-dev -l app=gateway

# Update deployment
helm upgrade renthub-dev ./helm/renthub-chart \
  -f ./helm/renthub-chart/values-dev.yaml \
  -n renthub-dev
```

## 📚 Learn More

- Full guide: See [HELM_ARGOCD_GUIDE.md](HELM_ARGOCD_GUIDE.md)
- Migration details: See [MIGRATION_GUIDE.md](MIGRATION_GUIDE.md)
- Helm docs: https://helm.sh/docs/
- ArgoCD docs: https://argo-cd.readthedocs.io/

## ❓ Troubleshooting

**Pod not starting?**
```bash
kubectl describe pod -n renthub-dev <pod-name>
kubectl logs -n renthub-dev <pod-name>
```

**ArgoCD not syncing?**
```bash
argocd app get renthub-dev
argocd app sync renthub-dev --force
```

**Need to rollback?**
```bash
helm history renthub-dev -n renthub-dev
helm rollback renthub-dev <revision> -n renthub-dev
```

## 🎯 Next Steps

- [ ] Deploy dev environment
- [ ] Test services are accessible
- [ ] Deploy staging environment
- [ ] Setup staging approvals in GitOps workflow
- [ ] Configure secrets management
- [ ] Setup monitoring and logging
- [ ] Deploy to production

Happy deploying! 🎉
