# Kargo Fleet Manifests

This directory contains Fleet-managed manifests for deploying Kargo to the management cluster.

## Files

- **fleet.yaml** - Fleet targeting configuration (cluster-type: local) with diff settings
- **namespace.yaml** - kargo namespace (created by HelmChart)
- **helm-release.yaml** - Kargo Helm chart v1.0.0 deployment via HelmChart CRD
- **kargo-controller-sa.yaml** - ServiceAccount fix for Helm chart bug
- **project.yaml** - hello-kargo namespace + Kargo Project CRD
- **rbac.yaml** - RoleBinding for kargo-controller to access secrets
- **warehouse.yaml** - Git repository monitoring configuration (will retry until secret exists)
- **promotion-task.yaml** - GitOps promotion workflow definition
- **stages.yaml** - acc and prd stage definitions

## Deployment Order

Fleet applies resources in parallel, but the design is resilient to ordering:

1. **HelmChart** creates `kargo` namespace + installs Kargo (pods, CRDs, etc.)
2. **Project** creates `hello-kargo` namespace + Project CRD
3. **RBAC** creates RoleBinding (may wait for SA from Helm)
4. **Warehouse/Stages** are created (will be in error state until secret exists)
5. **Vagrant provisioning** runs `kargo_secrets.sh` → creates git-credentials secret
6. **Warehouse** auto-retries and succeeds once secret is available

## Secret Management

**IMPORTANT**: The `git-credentials` secret is NOT included in these manifests.

### Why?
- SSH private keys should NEVER be committed to Git repositories
- Even in private repos, secrets in Git are a security risk
- The key is in `.gitignore` and managed by Vagrant provisioning

### How it works
```
Fleet → Deploys manifests → Kargo installed, Warehouse in pending state
         ↓
Vagrant → Waits for Kargo ready → Creates secret from /vagrant/argocd_deploy_key
         ↓
Kargo → Warehouse detects secret → Connects to Git → Success
```

### Automatic retry behavior
- Warehouse has `interval: 5m` - retries every 5 minutes
- Kargo controller watches for secret creation
- No manual intervention needed

### Manual secret creation (if needed)
```bash
kubectl create secret generic git-credentials \
  --from-file=sshPrivateKey=/vagrant/argocd_deploy_key \
  --from-literal=repoURL=git@github.com:anasgrt/rancher-argocd-test.git \
  --namespace=hello-kargo \
  --dry-run=client -o yaml | \
  kubectl label --local -f - kargo.akuity.io/cred-type=git --dry-run=client -o yaml | \
  kubectl apply -f -
```

## Deployment Steps

### 1. Copy manifests to Fleet Git repository
```bash
cd /path/to/rancher-fleet-test
cp -r /Users/anasalhamd/training/rancher_vagrant_environment/fleet-manifests/local/kargo local/
git add local/kargo/
git commit -m "Add Kargo Fleet manifests v1.0.0"
git push
```

### 2. Provision Vagrant environment
```bash
# Fresh deployment
vagrant up

# OR re-provision if already running
vagrant provision local-worker2
```

### 3. Verify deployment
```bash
# Check Fleet picked up the GitRepo
kubectl get gitrepo -n fleet-local

# Check Fleet bundle status
kubectl get bundles -n fleet-local

# Check Helm chart deployed
kubectl get helmchart kargo -n kube-system
kubectl get helmchart kargo -n kube-system -o jsonpath='{.status.jobName}'

# Check Kargo pods running
kubectl get pods -n kargo
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kargo -n kargo --timeout=300s

# Check Kargo CRDs installed
kubectl get crd | grep kargo

# Check Project and resources
kubectl get project -n hello-kargo
kubectl get warehouse -n hello-kargo
kubectl get stages -n hello-kargo

# Check secret created by provisioning
kubectl get secret git-credentials -n hello-kargo
kubectl get secret git-credentials -n hello-kargo -o jsonpath='{.metadata.labels}'

# Check Warehouse status (should be healthy after secret creation)
kubectl get warehouse hello-kargo -n hello-kargo -o jsonpath='{.status}'
```

### 4. Access Kargo UI
```bash
# Port-forward from your local machine
kubectl port-forward -n kargo svc/kargo-api 31444:80

# Open browser: http://localhost:31444
# Login: admin / admin
```

### 5. Test promotion workflow
```bash
# Make a commit to main branch in rancher-argocd-test repo
echo "test" >> test.txt
git add test.txt
git commit -m "Test Kargo promotion"
git push

# Watch Warehouse detect the change
kubectl get warehouse hello-kargo -n hello-kargo -w

# Promote in Kargo UI: acc → prd
# Verify stage branches created:
# - stage/acc
# - stage/prd
```

## Troubleshooting

### Warehouse in error state
```bash
# Check if secret exists
kubectl get secret git-credentials -n hello-kargo

# Check secret has correct label
kubectl get secret git-credentials -n hello-kargo -o jsonpath='{.metadata.labels}'

# Check Warehouse logs
kubectl logs -n kargo -l app.kubernetes.io/component=controller --tail=100 | grep warehouse

# Manually trigger reconciliation
kubectl annotate warehouse hello-kargo -n hello-kargo \
  kargo.akuity.io/refresh="$(date +%s)" --overwrite
```

### Kargo pods not starting
```bash
# Check HelmChart status
kubectl get helmchart kargo -n kube-system -o yaml

# Check Helm controller logs
kubectl logs -n kube-system -l app=helm-controller --tail=100

# Check pod events
kubectl get events -n kargo --sort-by='.lastTimestamp'
```

### RBAC issues
```bash
# Check RoleBinding
kubectl get rolebinding kargo-controller-secrets -n hello-kargo -o yaml

# Check ServiceAccount
kubectl get sa kargo-controller -n kargo

# Test permissions
kubectl auth can-i get secrets --as=system:serviceaccount:kargo:kargo-controller -n hello-kargo
```
