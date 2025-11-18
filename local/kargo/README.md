# Kargo Fleet Manifests

This directory contains Fleet-managed manifests for deploying Kargo to the management cluster.

## Files

- **fleet.yaml** - Fleet targeting configuration (cluster-type: local)
- **namespace.yaml** - kargo namespace
- **helm-release.yaml** - Kargo Helm chart deployment via HelmChart CRD
- **kargo-controller-sa.yaml** - ServiceAccount fix for Helm chart bug
- **project.yaml** - Kargo Project and hello-kargo namespace
- **rbac.yaml** - RoleBinding for kargo-controller to access secrets
- **warehouse.yaml** - Git repository monitoring configuration
- **promotion-task.yaml** - GitOps promotion workflow definition
- **stages.yaml** - acc and prd stage definitions

## Secret Management

**IMPORTANT**: The `git-credentials` secret is NOT included in these manifests.

### Why?
- SSH private keys should never be committed to Git repositories
- Even in private repos, secrets in Git are a security risk
- The key is already in `.gitignore` and managed by Vagrant

### How it works
1. Fleet deploys these manifests → creates namespaces and Kargo installation
2. Vagrant provisioning runs `scripts/kargo_secrets.sh` → creates secret from `/vagrant/argocd_deploy_key`
3. Secret is created with proper labels: `kargo.akuity.io/cred-type=git`
4. Kargo can now access the Git repository

### Manual creation (if needed)
```bash
kubectl create secret generic git-credentials \
  --from-file=sshPrivateKey=/vagrant/argocd_deploy_key \
  --from-literal=repoURL=git@github.com:anasgrt/rancher-argocd-test.git \
  --namespace=hello-kargo \
  --dry-run=client -o yaml | \
  kubectl label --local -f - kargo.akuity.io/cred-type=git --dry-run=client -o yaml | \
  kubectl apply -f -
```

## Deployment

1. Copy these manifests to your Fleet Git repository:
   ```bash
   cp -r fleet-manifests/local/kargo /path/to/rancher-fleet-test/local/
   ```

2. Commit and push:
   ```bash
   cd /path/to/rancher-fleet-test
   git add local/kargo/
   git commit -m "Add Kargo Fleet manifests"
   git push
   ```

3. Verify deployment:
   ```bash
   # Check Fleet picked up the changes
   kubectl get gitrepo -n fleet-local
   
   # Check Helm chart deployed
   kubectl get helmchart kargo -n kube-system
   
   # Check Kargo pods
   kubectl get pods -n kargo
   
   # Check Kargo resources
   kubectl get project,warehouse,stages,promotiontask -n hello-kargo
   
   # Check secret was created by provisioning
   kubectl get secret git-credentials -n hello-kargo
   ```

4. Access Kargo UI:
   ```bash
   kubectl port-forward -n kargo svc/kargo-api 31444:80
   # Open: http://localhost:31444
   # Login: admin / admin
   ```
