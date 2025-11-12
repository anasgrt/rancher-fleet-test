# Fleet v0.13.4 GitOps Repository

## Important: Fleet v0.13.4 Targeting Rules

This repository is designed for **Fleet v0.13.4** (bundled with Rancher v2.12.3).

### Repository Structure

```
rancher-fleet-test/
├── README.md                          # This file
│
├── downstream/                        # Applications for downstream clusters
│   ├── dev/                      # Apps ONLY for dev clusters
│   │   ├── nginx-ingress/             # Ingress controller (NodePort)
│   │   │   ├── fleet.yaml
│   │   │   └── values.yaml
│   │   │
│   │   └── debug-tools/               # Debug utilities
│   │       └── fleet.yaml
│   │
│   └── prd/                      # Apps ONLY for prd clusters
│       └── production-app/            # Production-specific app
│           └── fleet.yaml
│
└── local/                             # Applications for management cluster
    └── gitrepos/
        ├── cluster-services/          # Cluster-wide services
        │   ├── fleet.yaml
        │   └── example.yaml
        │
        └── monitoring/                # Monitoring stack
            ├── fleet.yaml
            └── prometheus.yaml
```

### ⚠️ Critical Fleet v0.13.4 Limitation: Multi-Environment Targeting

**Problem:**
Fleet v0.13.4 does NOT support per-application environment targeting within a single Git repository.

**Why:**
- Multiple ClusterGroups can exist (e.g., `default` with env=dev, `production` with env=prd)
- Fleet automatically applies the **FIRST ClusterGroup alphabetically** as targetRestrictions
- You CANNOT override this in fleet.yaml (no `targets` or `targetCustomizations` support)
- All apps in a GitRepo path deploy to the same ClusterGroup

**Current Behavior:**
```
downstream/common/nginx-ingress/    → ClusterGroup 'default' (env=dev) ✅
downstream/dev/debug-tools/    → ClusterGroup 'default' (env=dev) ✅  
downstream/prd/production-app/ → ClusterGroup 'default' (env=dev) ❌ Wrong!
```

Everything deploys to `default` ClusterGroup because it's alphabetically first.

### Solutions for Multi-Environment Deployments

#### Option 1: Separate Git Repositories (Recommended for v0.13.4)

Create separate repositories for each environment:

```yaml
# vagrant-config.yml or manual Fleet configuration
fleet:
  downstream_dev:
    git_repo_url: "https://github.com/you/fleet-dev.git"
    cluster_group: default  # env=dev
    
  downstream_prd:
    git_repo_url: "https://github.com/you/fleet-prd.git" 
    cluster_group: production  # env=prd
```

**Repository Structure:**
```
fleet-dev/          # Dev-specific apps
└── apps/
    ├── nginx-ingress/
    └── debug-tools/

fleet-prd/          # Prd-specific apps
└── apps/
    ├── nginx-ingress/
    └── production-app/
```

#### Option 2: Single ClusterGroup with Conditional Logic

Remove the `production` ClusterGroup and use a single `default` ClusterGroup that matches ALL downstream clusters:

```yaml
# ClusterGroup 'default'
spec:
  selector:
    matchLabels:
      cluster-type: downstream  # Matches all downstream clusters
```

Then use Helm values or Kustomize overlays to differentiate between environments based on cluster labels.

#### Option 3: Wait for Fleet v0.14+ (Future)

Fleet v0.14+ introduces `targetCustomizations` in fleet.yaml:

```yaml
# fleet.yaml (Fleet v0.14+ only - NOT v0.13.4)
targetCustomizations:
  - name: dev-deployment
    clusterSelector:
      matchLabels:
        env: dev
    # Dev-specific configuration
    
  - name: prd-deployment
    clusterSelector:
      matchLabels:
        env: prd
    # Prd-specific configuration
```

### Current Repository Behavior

**With this structure and Fleet v0.13.4:**

| Directory | Target ClusterGroup | Clusters Receiving Deployment |
|-----------|---------------------|-------------------------------|
| `downstream/dev/nginx-ingress/` | `default` (env=dev) | Dev clusters only |
| `downstream/dev/debug-tools/` | `default` (env=dev) | Dev clusters only |
| `downstream/prd/production-app/` | `default` (env=dev) | Dev clusters only ⚠️ |
| `local/gitrepos/` | `local` (env=management) | Management cluster only |

**To properly deploy to prd clusters:**
1. Create separate Git repository for prd apps
2. Configure separate Fleet GitRepo pointing to prd repository
3. Associate with `production` ClusterGroup (env=prd)

|-----------|---------------------|-------------------------------|
| `downstream/common/` | `default` (env=dev) | Dev clusters only |
| `downstream/dev/` | `default` (env=dev) | Dev clusters only |
| `downstream/prd/` | `default` (env=dev) | Dev clusters only ⚠️ |
| `local/gitrepos/` | `local` (env=management) | Management cluster only |

**To properly deploy to prd clusters:**
1. Create separate Git repository for prd apps
2. Configure separate Fleet GitRepo pointing to prd repository
3. Associate with `production` ClusterGroup (env=prd)

### ⚠️ Critical Rules to Avoid Bundle Creation Failures

#### 1. DO NOT Mix Helm Charts with Raw Manifests

**WRONG** ❌:
```
downstream/common/nginx-ingress/
├── fleet.yaml          # Helm configuration
├── namespace.yaml      # Raw manifest - CAUSES FAILURE!
└── values.yaml
```

**CORRECT** ✅:
```
downstream/common/nginx-ingress/
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
   ls -la downstream/common/nginx-ingress/
   # If you see namespace.yaml or other .yaml files besides fleet.yaml and values*.yaml
   # DELETE the raw manifest files
   ```

2. **Using targets or targetCustomizations**
   ```bash
   # Check fleet.yaml
   grep -E 'targets:|targetCustomizations:' downstream/common/nginx-ingress/fleet.yaml
   # If found, REMOVE these fields from fleet.yaml
   ```

3. **YAML syntax errors**
   ```bash
   # Validate YAML syntax
   yq eval '.' downstream/common/nginx-ingress/fleet.yaml
   ```

### App Deployed to Wrong Clusters

**Symptom:** App appears on dev clusters when it should be prd (or vice versa)

**Root Cause:** Fleet v0.13.4 limitation - all apps in same GitRepo deploy to first ClusterGroup alphabetically

**Solution:** 
1. **Immediate**: Move prd-specific apps to separate Git repository
2. **Long-term**: Upgrade to Fleet v0.14+ for `targetCustomizations` support

### GitRepo Not Cloning

**Check GitRepo status:**
```bash
kubectl get gitrepos -n fleet-default -o yaml
```

**Check Fleet controller logs:**
```bash
kubectl logs -n cattle-fleet-system -l app=fleet-controller --tail=50
```

---

## Migration to Fleet v0.14+

When Rancher upgrades to include Fleet v0.14+:

### What Changes
- `targetCustomizations` becomes available in fleet.yaml
- Per-application environment targeting within single Git repo
- ClusterGroup pattern still works (backward compatible)

### Migration Path

**Option 1: Keep Current Structure** (simplest)
- Current ClusterGroup-based setup continues to work
- No changes needed
- Apps still deploy based on ClusterGroup selectors

**Option 2: Migrate to targetCustomizations** (more flexible)

Update fleet.yaml files:
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
        resources:
          requests:
            cpu: 100m

  - name: prd-deployment
    clusterSelector:
      matchLabels:
        env: prd
    helm:
      values:
        replicas: 3
        resources:
          requests:
            cpu: 500m

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
- [ ] ⚠️ Understand Fleet v0.13.4 limitation: All apps deploy to first ClusterGroup
- [ ] ⚠️ For true prd isolation: Use separate Git repository

---

## Support

For issues or questions:
1. Check this README's Troubleshooting section
2. Validate configuration: `/vagrant/scripts/validate_fleet_repo.sh`
3. Review Fleet controller logs: `kubectl logs -n cattle-fleet-system -l app=fleet-controller`
4. Check infrastructure repo documentation: `rancher_vagrant_environment/scripts/README.md`
