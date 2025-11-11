# Rancher Fleet Test Repository

Multi-cluster GitOps repository for Rancher Fleet with separate configurations for local (management) and downstream (workload) clusters.

## Repository Structure

```
├── downstream/          # Workload applications for downstream clusters
│   ├── nginx-ingress/   # Ingress controller (NodePort for Vagrant)
│   ├── nginx-example/   # Simple nginx test application
│   └── my-app/          # Custom application example
└── local/               # Management cluster applications
    ├── monitoring/      # Monitoring stack examples
    └── cluster-services/# Cluster-wide services
```

## Quick Start

### Option 1: Downstream Clusters Only (Default)

Update `vagrant-config.yml`:

```yaml
fleet:
  enabled: true
  downstream:
    enabled: true
    git_repo_url: "https://github.com/anasgrt/rancher-fleet-test.git"
    git_repo:
      paths:
        - "downstream"
```

### Option 2: Both Local and Downstream Clusters

```yaml
fleet:
  enabled: true
  
  downstream:
    enabled: true
    git_repo_url: "https://github.com/anasgrt/rancher-fleet-test.git"
    git_repo:
      paths:
        - "downstream"
  
  local:
    enabled: true
    git_repo_url: "https://github.com/anasgrt/rancher-fleet-test.git"
    git_repo:
      paths:
        - "local"
```

## Applications

### Downstream Clusters

#### nginx-ingress
- **Type**: Helm chart
- **Version**: 4.11.3
- **Target**: **DOWNSTREAM CLUSTERS ONLY** (cluster-type=downstream)
- **Service Type**: NodePort (30080/30443) - Optimized for Vagrant/bare-metal
- **Features**: 
  - No LoadBalancer wait (faster deployment in Vagrant)
  - DefaultBackend disabled (reduced resource usage)
  - 2 replicas for high availability
  - Does NOT deploy to management cluster (prevents conflicts with Rancher's ingress)
- **Access**: `http://<downstream-node-ip>:30080` or `https://<downstream-node-ip>:30443`

**Why downstream only?**
The management cluster runs Rancher's built-in ingress controller. Deploying nginx-ingress there would:
- Create port conflicts (80/443 already in use)
- Waste resources (duplicate ingress controllers)
- Complicate troubleshooting (which ingress is handling requests?)

The `cluster-type: downstream` label in `fleet.yaml` ensures this ingress controller is deployed **only to workload clusters**.

#### nginx-example
- **Type**: Kubernetes manifests
- **Purpose**: Simple test application
- **Namespace**: fleet-test

#### my-app
- **Type**: Kubernetes manifests with ingress
- **Purpose**: Template for custom applications
- **Features**:
  - Multi-environment targeting (dev/prod)
  - Ingress configuration example
  - Resource limits configured

### Local Cluster

#### monitoring
- **Purpose**: Monitoring stack placeholder
- **Namespace**: cattle-monitoring-system
- **Note**: Example configuration - replace with actual Prometheus stack

#### cluster-services
- **Purpose**: Cluster-wide utilities
- **Namespace**: kube-system
- **Note**: Example ConfigMap - customize as needed

## Fleet Targeting

All applications use the new `cluster-type` label for precise targeting:

### Downstream Clusters
```yaml
targetCustomizations:
- name: downstream-clusters
  clusterSelector:
    matchLabels:
      cluster-type: downstream
```

### Local Cluster
```yaml
targetCustomizations:
- name: local-cluster
  clusterSelector:
    matchLabels:
      cluster-type: local
```

### Environment-Specific Targeting
```yaml
targetCustomizations:
- name: dev-clusters
  clusterSelector:
    matchLabels:
      cluster-type: downstream
      env: dev
```

## Monitoring Deployments

```bash
# View all Fleet clusters with labels
kubectl get clusters.fleet.cattle.io -n fleet-default --show-labels

# Check GitRepo resources
kubectl get gitrepos -n fleet-default

# Monitor bundle deployments
kubectl get bundles -n fleet-default

# Check specific bundle details
kubectl get bundle <bundle-name> -n fleet-default -o yaml

# View bundle deployment status
kubectl get bundledeployments -n fleet-default
```

## Troubleshooting

### Bundle Stuck in "WaitApplied"
```bash
# Check bundle status
kubectl describe bundle <bundle-name> -n fleet-default

# Check Helm release (if using Helm chart)
kubectl get secrets -n <namespace> -l owner=helm

# Force reconciliation
kubectl annotate gitrepo <repo-name> -n fleet-default force-sync=$(date +%s) --overwrite
```

### Clusters Not Labeled
```bash
# Check cluster labels
kubectl get clusters.fleet.cattle.io -n fleet-default --show-labels

# Manually label downstream cluster
kubectl label clusters.fleet.cattle.io <cluster-name> -n fleet-default \
  cluster-type=downstream env=dev --overwrite

# Manually label local cluster
kubectl label clusters.fleet.cattle.io local -n fleet-default \
  cluster-type=local env=management --overwrite
```

### nginx-ingress Issues in Vagrant

The nginx-ingress is configured with NodePort to work in Vagrant/bare-metal environments:

- **HTTP**: http://<node-ip>:30080
- **HTTPS**: https://<node-ip>:30443

Access example:
```bash
# Get node IP
kubectl get nodes -o wide

# Test ingress
curl http://192.168.56.20:30080
```

## Development Workflow

1. **Make changes** to manifests/values
2. **Commit and push** to GitHub
3. **Fleet auto-syncs** (15s polling interval)
4. **Monitor deployment**:
   ```bash
   watch kubectl get bundles -n fleet-default
   ```

## Migration from Old Structure

The repository has been refactored from:
```
clusters/
├── nginx-ingress/
└── nginx-example/
```

To:
```
downstream/
├── nginx-ingress/  # Updated with NodePort
├── nginx-example/  # Updated targeting
└── my-app/         # New example
local/
├── monitoring/     # New examples
└── cluster-services/
```

**Action Required**: Update your `vagrant-config.yml` to use `paths: ["downstream"]` instead of `paths: ["clusters"]`.

## Best Practices

1. **Separate Concerns**: Keep management apps in `local/`, workloads in `downstream/`
2. **Use Labels**: Target clusters precisely with `cluster-type` and `env` labels
3. **Test Locally**: Use Vagrant environment for testing before production
4. **Version Control**: Always commit and push changes to trigger Fleet sync
5. **Monitor**: Regularly check bundle status and logs

## Contributing

When adding new applications:

1. Create appropriate directory under `downstream/` or `local/`
2. Add `fleet.yaml` with correct targeting
3. Include all manifests or Helm values
4. Update this README with application description
5. Test in Vagrant before production deployment

## License

This is a test repository for Rancher Fleet GitOps demonstrations.
