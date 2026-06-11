# Migration Guide: From Raw Manifests to Helm + ArgoCD

## What Changed

### Before
```
configmaps/
deployments/
namespaces/
secrets/
services/
statefulsets/
storage/
```
Individual YAML files organized by resource type.

### After
```
helm/renthub-chart/          # Parameterized charts
├── Chart.yaml
├── values.yaml              # Default values
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/               # Reusable templates

argocd/                      # GitOps sync
├── renthub-dev.yaml
├── renthub-staging.yaml
└── renthub-prod.yaml
```

## Key Benefits

### 1. **Single Source of Truth**
- One Helm chart for all environments
- Environment differences only in `values-*.yaml`
- Less duplication, easier to maintain

### 2. **Environment-Specific Configuration**
```yaml
# Development
replicas: 1
image: gateway:dev
resources.memory: 128Mi

# Staging
replicas: 2
image: gateway:staging
resources.memory: 256Mi

# Production
replicas: 3
image: gateway:latest
resources.memory: 512Mi
```

### 3. **GitOps Workflow**
- All state in git
- ArgoCD automatically syncs changes
- Full audit trail of all deployments

### 4. **Automated Dev Deployments**
- CI/CD updates `values-dev.yaml` with new image tags
- ArgoCD auto-syncs → dev environment always up-to-date

### 5. **Controlled Prod Deployments**
- Manual approval workflow
- Review changes before deployment
- Peace of mind for production

## Migration Steps

### Step 1: Backup Old Manifests
```bash
mkdir -p old-manifests-backup
cp -r configmaps deployments namespaces secrets services statefulsets storage old-manifests-backup/
git add old-manifests-backup/
git commit -m "Backup: old manifest structure"
```

### Step 2: Review New Structure
- All templates in `helm/renthub-chart/templates/`
- Environment-specific values in `values-*.yaml`
- ArgoCD apps in `argocd/`

### Step 3: Update Repository Configuration
- Update CI/CD to point to `helm/renthub-chart`
- Configure image tag updates for `values-dev.yaml`

### Step 4: Deploy via ArgoCD
```bash
# Create ArgoCD namespace and app
kubectl apply -f argocd/renthub-dev.yaml
kubectl apply -f argocd/renthub-staging.yaml
kubectl apply -f argocd/renthub-prod.yaml
```

### Step 5: Verify Deployments
```bash
kubectl get deployments -n renthub-dev
kubectl get statefulsets -n renthub-dev
argocd app get renthub-dev
```

### Step 6: Archive Old Manifests
Once verified in all environments:
```bash
rm -rf configmaps deployments namespaces secrets services statefulsets storage
git add -A
git commit -m "Remove old manifest structure - migrated to Helm + ArgoCD"
```

## Comparing Before and After

### ConfigMaps
**Before**: Separate file per service
**After**: Single template with values substitution
```yaml
# Before: configmaps/app-config.yaml
DB_HOST: "mysql-service.renthub.svc.cluster.local"

# After: templates/configmap.yaml + values-*.yaml
DB_HOST: "mysql-service.{{ .Values.namespace }}.svc.cluster.local"
```

### Deployments
**Before**: Static replicas for all environments
**After**: Environment-specific via values
```yaml
# Before: same for all envs
replicas: 2

# After: controlled by values
replicas: {{ .Values.services.gateway.replicas }}
# dev: 1, staging: 2, prod: 3
```

### Images
**Before**: One image tag for all
**After**: Environment-specific tags
```yaml
# Before
image: renthub/gateway:latest

# After
image: {{ .Values.services.gateway.image }}
# dev: gateway:dev, staging: gateway:staging, prod: gateway:latest
```

## Common Tasks After Migration

### Update Dev Environment
```bash
# Workflow automatically updates this
sed -i "s|renthub/gateway:.*|renthub/gateway:$NEW_TAG|" \
  helm/renthub-chart/values-dev.yaml
git push origin develop
# ArgoCD auto-syncs → done!
```

### Update Staging
```bash
# Manual edit
vim helm/renthub-chart/values-staging.yaml
git add helm/renthub-chart/values-staging.yaml
git commit -m "Update staging to new release"
git push origin main
# Manual sync via ArgoCD UI
```

### Update Production
```bash
# Careful manual edit
vim helm/renthub-chart/values-prod.yaml
git add helm/renthub-chart/values-prod.yaml
git commit -m "Update production - requires approval"
# Wait for PR approval and merge
# Manual sync via ArgoCD UI after review
```

## Validation

### Helm Validation
```bash
helm lint helm/renthub-chart/
helm template renthub-chart -f helm/renthub-chart/values-dev.yaml
```

### ArgoCD Validation
```bash
argocd app list
argocd app get renthub-dev
argocd app diff renthub-dev
```

### Manifest Comparison
```bash
# Generate manifests for comparison
helm template renthub-dev helm/renthub-chart \
  -f helm/renthub-chart/values-dev.yaml > /tmp/new-manifests.yaml

# Compare with deployed resources
kubectl get all -n renthub-dev -o yaml > /tmp/current-state.yaml
```

## Rollback Plan

If needed to rollback:
```bash
# Delete ArgoCD apps
kubectl delete application renthub-dev -n argocd

# Manually apply old manifests
kubectl apply -f old-manifests-backup/ -n renthub-dev

# Or revert git commit
git revert <commit-hash>
```

## Next Steps

1. ✅ Review new Helm structure
2. ✅ Deploy via ArgoCD
3. ✅ Monitor for issues
4. ✅ Update CI/CD pipelines
5. ✅ Archive old manifests
6. ✅ Document any custom secrets management
7. ✅ Train team on new workflow
