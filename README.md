# Rancher Fleet Test Repository

Multi-cluster GitOps repository for Rancher Fleet using **environment-based labeling** (dev, prd, management).

## Repository Structure

```
├── downstream/          # Workload applications for downstream clusters
│   ├── nginx-ingress/   # Ingress controller (NodePort for Vagrant)
│   └── nginx-example/   # Simple nginx test application
└── local/               # Management cluster applications
    ├── monitoring/      # Monitoring stack examples
    └── cluster-services/# Cluster-wide services
```

## Quick Start

### Downstream Clusters (Workloads)

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

### Local Cluster (Management)

```yaml
fleet:
  local:
    enabled: true
    git_repo_url: "https://github.com/anasgrt/rancher-fleet-test.git"
    git_repo:
      paths:
        - "local"
```

## Cluster Labeling Strategy

This repository uses **simple environment labels**:

- **downstream1**: `env=dev` (development workload cluster)
- **downstream2**: `env=prd` (production workload cluster)
- **local**: `env=management` (management cluster)

## Applications

### Downstream Clusters

#### nginx-ingress
- **Type**: Helm chart v4.11.3
- **Target**: `env IN [dev, prd]` (all downstream clusters)
- **Service Type**: NodePort (30080/30443) - Vagrant optimized
- **Features**:
  - No LoadBalancer wait (faster deployment)
  - 2 replicas for HA
  - Does NOT deploy to management cluster
- **Access**: `http://<node-ip>:30080` or `https://<node-ip>:30443`

**Why not on management cluster?**
- Prevents port conflicts with Rancher's built-in ingress (80/443)
- Avoids resource duplication
- Cleaner separation of concerns

#### nginx-example
- **Type**: Kubernetes manifests
- **Target**: `env IN [dev, prd]` (all downstream clusters)
- **Purpose**: Simple test application
- **Namespace**: fleet-test

### Local Cluster (Management)

#### monitoring
- **Target**: `env=management`
- **Purpose**: Monitoring stack for management cluster
- **Namespace**: cattle-monitoring-system

#### cluster-services
- **Target**: `env=management`
- **Purpose**: Cluster-wide utilities
- **Namespace**: kube-system

## Fleet Targeting Examples

### All Downstream Clusters

```yaml
targetCustomizations:
- name: all-downstream
  clusterSelector:
    matchExpressions:
    - key: env
      operator: In
      values:
      - dev
      - prd
```

### Specific Environment

```yaml
targetCustomizations:
- name: dev-only
  clusterSelector:
    matchLabels:
      env: dev
```

### Management Cluster

```yaml
targetCustomizations:
- name: management
  clusterSelector:
    matchLabels:
      env: management
```

## Monitoring Deployments

```bash
# View all Fleet clusters with labels
kubectl get clusters.fleet.cattle.io -n fleet-default --show-labels

# Expected output:
# downstream1 ... env=dev
# downstream2 ... env=prd
# local       ... env=management (if fleet.local enabled)

# Check GitRepo resources
kubectl get gitrepos -n fleet-default

# Monitor bundle deployments
kubectl get bundles -n fleet-default

# Check specific bundle
kubectl describe bundle <bundle-name> -n fleet-default
```

## Troubleshooting

### Bundles Not Deploying

```bash
# Check cluster labels match targeting
kubectl get clusters.fleet.cattle.io -n fleet-default --show-labels

# Verify GitRepo status
kubectl get gitrepos -n fleet-default -o yaml

# Force sync
kubectl annotate gitrepo downstream-manifests -n fleet-default \
  force-sync=$(date +%s) --overwrite
```

### nginx-ingress Access Issues

The nginx-ingress uses **NodePort** for Vagrant/bare-metal:

```bash
# Get node IP
kubectl get nodes -o wide --context downstream1

# Test access
curl http://192.168.56.20:30080
```

## Development Workflow

1. Make changes to manifests/values
2. Commit and push to GitHub
3. Fleet auto-syncs (15s polling interval)
4. Monitor deployment:

```bash
watch kubectl get bundles -n fleet-default
```

## Best Practices

1. **Separate Concerns**: Management apps in `local/`, workloads in `downstream/`
2. **Use Environment Labels**: Simple `env` label for all targeting
3. **Test Locally**: Validate in Vagrant before production
4. **Version Control**: Always commit before testing
5. **Monitor**: Check bundle status regularly

## Configuration Files

All applications use `fleet.yaml` for Fleet-specific configuration:

- **Targeting**: Define which clusters receive the application
- **Customization**: Environment-specific values
- **Dependencies**: Order of deployment (if needed)

## License

Test repository for Rancher Fleet GitOps demonstrations.
