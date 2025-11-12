# Fleet v0.13.4 GitOps Repository

## Important: Fleet v0.13.4 Targeting Rules

This repository is designed for **Fleet v0.13.4** (bundled with Rancher v2.12.3).

### ⚠️ Critical Rules to Avoid Bundle Creation Failures

#### 1. DO NOT Mix Helm Charts with Raw Manifests

**WRONG** ❌:
```
downstream/gitrepos/nginx-ingress/
├── fleet.yaml          # Helm configuration
├── namespace.yaml      # Raw manifest - CAUSES FAILURE!
└── values.yaml
```

**CORRECT** ✅:
```
downstream/gitrepos/nginx-ingress/
├── fleet.yaml          # Helm configuration with defaultNamespace
└── values.yaml
```

**Why?** Fleet v0.13.4 cannot process directories that contain both:
- Helm chart configurations (helm: section in fleet.yaml)
- Raw Kubernetes manifests (YAML files with apiVersion/kind)

This causes the Bundle to show **0/0 deployments** and never creates resources.

**Solution:** Use `defaultNamespace` in fleet.yaml instead of creating namespace.yaml:
```yaml
defaultNamespace: my-namespace
helm:
  chart: my-chart
  repo: https://...
```

---

#### 2. DO NOT Use `targets` or `targetCustomizations` in fleet.yaml

**WRONG** ❌:
```yaml
# fleet.yaml
targets:
  - clusterSelector:
      matchLabels:
        env: dev

# OR

targetCustomizations:
  - name: dev-cluster
    clusterSelector:
      matchLabels:
        env: dev
```

**CORRECT** ✅:
```yaml
# fleet.yaml - NO targeting fields
defaultNamespace: my-namespace
helm:
  chart: my-chart
  repo: https://...
```

**Why?** Fleet v0.13.4 automatically adds `targetRestrictions` when ClusterGroup resources exist. Adding `targets` or `targetCustomizations` creates conflicts.

**Notes:**
- `targetCustomizations` is a Fleet v0.14+ feature (not available in v0.13.4)
- `targets` conflicts with ClusterGroup-based targeting in v0.13.4
- Both cause the Bundle to show **0/0 deployments**

---

## How Fleet v0.13.4 Targeting Works

Fleet v0.13.4 uses **ClusterGroup resources** created by the infrastructure, NOT fields in fleet.yaml.

### Architecture

```
Infrastructure Code (rancher_vagrant_environment)
    ↓
Creates ClusterGroup resources with label selectors
    ↓
Fleet automatically applies targetRestrictions to Bundles
    ↓
Bundles deploy to clusters matching ClusterGroup selectors
```

### Example: ClusterGroup Resource

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: default
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      env: dev
```

This ClusterGroup automatically targets all clusters labeled `env=dev`.

### How to Control Targeting

**Option 1: Use existing ClusterGroup** (recommended)
- Infrastructure creates ClusterGroup "default" with env=dev selector
- nginx-ingress deploys only to dev cluster
- No changes needed in fleet.yaml

**Option 2: Create new ClusterGroup**
```yaml
# In infrastructure repo: kubectl apply -f -
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: production
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      env: prd
```

**Option 3: Change cluster labels**
```bash
# Label cluster to match different ClusterGroup
kubectl label cluster c-xxxxx env=prd -n fleet-default
```

---

## Directory Structure

```
rancher-fleet-test/
├── README.md                          # This file
├── downstream/                        # Applications for downstream clusters
│   └── gitrepos/
│       └── nginx-ingress/             # Example: Helm chart
│           ├── fleet.yaml             # Fleet configuration (NO targets!)
│           └── values.yaml            # Helm values
│
└── local/                             # Applications for management cluster
    └── gitrepos/
        ├── cluster-services/          # Example: Raw manifests
        │   ├── fleet.yaml             # Fleet configuration (NO targets!)
        │   └── example-service.yaml   # Kubernetes manifest
        │
        └── monitoring/                # Example: Raw manifests
            ├── fleet.yaml             # Fleet configuration (NO targets!)
            └── prometheus.yaml        # Kubernetes manifest
```

---

## fleet.yaml Templates

### For Helm Charts

```yaml
# Fleet v0.13.4 Configuration
# IMPORTANT: DO NOT add namespace.yaml or other raw manifests to this directory
# IMPORTANT: DO NOT add 'targets' or 'targetCustomizations' fields

defaultNamespace: my-namespace  # Creates namespace automatically
keepResources: false

helm:
  chart: my-chart
  repo: https://charts.example.com
  version: 1.0.0
  releaseName: my-release
  wait: false
  timeoutSeconds: 600
  valuesFiles:
    - values.yaml
```

### For Raw Kubernetes Manifests

```yaml
# Fleet v0.13.4 Configuration
# IMPORTANT: DO NOT add Helm configuration to this directory
# IMPORTANT: DO NOT add 'targets' or 'targetCustomizations' fields

defaultNamespace: my-namespace
keepResources: false
```

---

## Troubleshooting

### Bundle Shows 0/0 Deployments

**Symptom:** Bundle resource exists but shows `0/0` in status
```bash
kubectl get bundles -n fleet-default
# NAME                          BUNDLEDEPLOYMENTS-READY   STATUS
# my-app-default-12345abc       0/0
```

**Common Causes:**

1. **Mixed Helm + raw manifests** (most common)
   ```bash
   # Check directory contents
   ls -la downstream/gitrepos/nginx-ingress/
   # If you see namespace.yaml or other .yaml files besides fleet.yaml and values*.yaml
   # DELETE the raw manifest files
   ```

2. **Using targets or targetCustomizations**
   ```bash
   # Check fleet.yaml
   grep -E 'targets:|targetCustomizations:' downstream/gitrepos/nginx-ingress/fleet.yaml
   # If found, REMOVE these fields from fleet.yaml
   ```

3. **YAML syntax errors**
   ```bash
   # Validate YAML syntax
   yq eval '.' downstream/gitrepos/nginx-ingress/fleet.yaml
   ```

### GitRepo Not Cloning

**Check GitRepo status:**
```bash
kubectl get gitrepos -n fleet-default -o yaml
```

**Check Fleet controller logs:**
```bash
kubectl logs -n cattle-fleet-system -l app=fleet-controller --tail=50
```

### Bundle Created But Not Deploying

**Check ClusterGroup:**
```bash
# List ClusterGroups
kubectl get clustergroups -n fleet-default

# Check if clusters match selector
kubectl get clusters -n fleet-default --show-labels
```

**Check Bundle targeting:**
```bash
kubectl get bundle my-app-default-12345abc -n fleet-default -o yaml | grep -A 10 targetRestrictions
```

---

## Migration to Fleet v0.14+

When Rancher upgrades to include Fleet v0.14+:

### What Changes
- `targetCustomizations` becomes available in fleet.yaml
- More flexible per-application targeting
- ClusterGroup pattern still works (backward compatible)

### Migration Options

**Option 1: Keep ClusterGroup pattern** (recommended)
- No changes needed
- Infrastructure continues to create ClusterGroups
- fleet.yaml files remain simple

**Option 2: Migrate to targetCustomizations**
```yaml
# fleet.yaml with Fleet v0.14+
defaultNamespace: my-namespace

targetCustomizations:
  - name: dev-deployment
    clusterSelector:
      matchLabels:
        env: dev
    helm:
      values:
        replicas: 1

  - name: prd-deployment
    clusterSelector:
      matchLabels:
        env: prd
    helm:
      values:
        replicas: 3

helm:
  chart: my-chart
  repo: https://...
```

---

## Reference

- **Fleet v0.13.4 Documentation**: https://fleet.rancher.io (v0.13 branch)
- **Infrastructure Repository**: rancher_vagrant_environment
- **Validation Script**: `scripts/validate_fleet_repo.sh` in infrastructure repo

---

## Quick Checklist

Before pushing changes to this repository:

- [ ] ✅ NO namespace.yaml or other raw manifests in Helm chart directories
- [ ] ✅ NO `targets` field in fleet.yaml
- [ ] ✅ NO `targetCustomizations` field in fleet.yaml
- [ ] ✅ Helm charts use `defaultNamespace` instead of separate namespace.yaml
- [ ] ✅ YAML syntax is valid (`yq eval '.' fleet.yaml`)
- [ ] ✅ Run validation script: `/vagrant/scripts/validate_fleet_repo.sh`

---

## Support

For issues or questions:
1. Check this README's Troubleshooting section
2. Validate configuration: `/vagrant/scripts/validate_fleet_repo.sh`
3. Review Fleet controller logs: `kubectl logs -n cattle-fleet-system -l app=fleet-controller`
4. Check infrastructure repo documentation: `rancher_vagrant_environment/scripts/README.md`
