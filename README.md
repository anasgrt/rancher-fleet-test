# Fleet v0.13.4 GitOps Repository

## Overview

This repository contains Fleet-managed Kubernetes applications for the Rancher multi-cluster environment.

**Fleet Version**: v0.13.4 (bundled with Rancher v2.12.3)
**Infrastructure**: rancher_vagrant_environment

## 📊 Deployed Applications

### Management Cluster (local-ctrl)

The following applications are automatically deployed to the management cluster via Fleet:

| Application | Purpose | Namespace | Access URL |
|-------------|---------|-----------|------------|
| **ArgoCD** | Continuous Delivery | `argocd` | <http://argocd.192.168.56.10.nip.io> |
| **Kargo** | Promotion Workflow | `kargo` | <https://kargo.192.168.56.10.nip.io> |
| **Prometheus** | Metrics & Monitoring | `prometheus-monitoring` | <http://prometheus.192.168.56.10.nip.io> |
| **Grafana** | Metrics Visualization | `prometheus-monitoring` | <http://grafana.192.168.56.10.nip.io> |

### Downstream Clusters (key-ctrl, key-worker)

The following applications are automatically deployed to downstream clusters via Fleet:

| Application | Purpose | Namespace | Access URL |
|-------------|---------|-----------|------------|
| **NGINX Ingress** | Ingress Controller | `ingress-nginx` | N/A |
| **Prometheus** | Metrics & Monitoring | `prometheus-monitoring` | <http://prometheus.192.168.56.20.nip.io> |
| **Grafana** | Metrics Visualization | `prometheus-monitoring` | <http://grafana.192.168.56.20.nip.io> |

### Access Credentials

**ArgoCD:**

- Username: `admin`
- Password: Stored in `.shared/argocd_password` (created by Ansible)

**Kargo:**

- Username: `admin`
- Password: `admin123` (default)

**Grafana:**

- Username: `admin`
- Password: `admin`

**Prometheus:**

- No authentication required (monitoring interface)

## Repository Structure

```
rancher-fleet-test/
├── README.md
│
├── key/                          # Key (downstream) clusters
│   └── gitrepos/
│       ├── common/               # Apps for ALL key clusters
│       │   └── nginx-ingress/    # Ingress controller
│       │       ├── fleet.yaml
│       │       └── values.yaml
│       │
│       ├── dev/                  # Apps ONLY for dev clusters
│       │   └── debug-tools/      # Debug utilities
│       │       ├── deployment.yaml
│       │       └── fleet.yaml
│       │
│       └── prd/                  # Apps ONLY for prd clusters
│           └── prd-app/          # Production application
│               ├── deployment.yaml
│               └── fleet.yaml
│
└── local/
    └── gitrepos/                 # Apps for management cluster
        └── ...
```

## How Targeting Works in Fleet v0.13.4

### The Three-GitRepo Pattern

This repository is deployed using **three separate Fleet GitRepos**, each targeting different clusters:

| GitRepo Name | Path | Targets | Deploys To |
|--------------|------|---------|------------|
| `common` | `key/gitrepos/common` | ClusterGroup: `default` | All clusters (dev + prd) |
| `dev` | `key/gitrepos/dev` | clusterSelector: `env=dev` | Dev clusters only |
| `prd` | `key/gitrepos/prd` | clusterSelector: `env=prd` | Prd clusters only |

### GitRepo Configuration

The GitRepos are configured in the infrastructure repository (`rancher_vagrant_environment/scripts/fleet_init.sh`):

```yaml
# common GitRepo
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: common
  namespace: fleet-default
spec:
  repo: https://github.com/anasgrt/rancher-fleet-test.git
  branch: main
  paths: [key/gitrepos/common]
  targets:
    - clusterGroup: default  # Matches both dev and prd

---
# dev GitRepo
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: dev
  namespace: fleet-default
spec:
  repo: https://github.com/anasgrt/rancher-fleet-test.git
  branch: main
  paths: [key/gitrepos/dev]
  targets:
    - clusterSelector:
        matchLabels:
          env: dev

---
# prd GitRepo
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: prd
  namespace: fleet-default
spec:
  repo: https://github.com/anasgrt/rancher-fleet-test.git
  branch: main
  paths: [key/gitrepos/prd]
  targets:
    - clusterSelector:
        matchLabels:
          env: prd
```

### ClusterGroup Configuration

The `default` ClusterGroup includes both dev and prd clusters:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: default
  namespace: fleet-default
spec:
  selector:
    matchExpressions:
    - key: env
      operator: In
      values: [dev, prd]
```

## Bundle Naming

Fleet creates bundle names using the pattern: `<gitrepo-name>-<path-with-dashes>-<app-name>`

With our structure, bundle names will be:
- `common-key-gitrepos-common-nginx-ingress` (deploys to both dev and prd)
- `dev-key-gitrepos-dev-debug-tools` (deploys to dev only)
- `prd-key-gitrepos-prd-prd-app` (deploys to prd only)

## Important: fleet.yaml Simplification

### What Does NOT Work in Fleet v0.13.4

**❌ targetCustomizations in fleet.yaml**

Fleet v0.13.4 documentation mentions `targetCustomizations`, but this feature does NOT work reliably for cluster filtering. Even when specified in fleet.yaml, Fleet ignores these settings and uses GitRepo-level targets instead.

```yaml
# ❌ This does NOT work in Fleet v0.13.4
targetCustomizations:
- name: dev
  clusterSelector:
    matchLabels:
      env: dev
```

### What DOES Work

**✅ GitRepo-level targets**

Targeting is controlled entirely by the GitRepo resource's `spec.targets` field, NOT by fleet.yaml.

```yaml
# ✅ fleet.yaml should be simple
defaultNamespace: my-namespace
keepResources: false

helm:  # Only if using Helm
  chart: my-chart
  repo: https://...
```

## fleet.yaml Templates

### For Helm Charts

```yaml
# Targeting: Controlled by GitRepo, not this file
# This file only configures the application itself

defaultNamespace: my-namespace
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
# Targeting: Controlled by GitRepo, not this file
# This file only configures the application itself

defaultNamespace: my-namespace
keepResources: false
```

## Critical Rules

### 1. DO NOT Mix Helm Charts with Raw Manifests

**❌ WRONG:**
```
nginx-ingress/
├── fleet.yaml          # Helm configuration
├── namespace.yaml      # Raw manifest - CAUSES FAILURE!
└── values.yaml
```

**✅ CORRECT:**
```
nginx-ingress/
├── fleet.yaml          # Uses defaultNamespace
└── values.yaml
```

### 2. DO NOT Add Targeting to fleet.yaml

**❌ WRONG:**
```yaml
# fleet.yaml
targetCustomizations:  # Ignored in v0.13.4
- name: dev
  clusterSelector:
    matchLabels:
      env: dev
```

**✅ CORRECT:**
```yaml
# fleet.yaml
defaultNamespace: my-namespace
# No targeting fields - controlled by GitRepo
```

### 3. Cluster Labeling

Clusters MUST have the `env` label for targeting to work:

```bash
# Verify cluster labels
kubectl get clusters.fleet.cattle.io -n fleet-default --show-labels

# Expected output:
# NAME       LABELS
# c-lfkjx    env=dev,...
# c-hq9t8    env=prd,...
```

## Troubleshooting

### Bundle Shows 0/0 Deployments

**Check for mixed content:**
```bash
# Look for unwanted raw manifests in Helm directories
ls -la key/gitrepos/common/nginx-ingress/
# Should only see: fleet.yaml, values.yaml
# Remove any namespace.yaml or other .yaml files
```

**Check for targeting in fleet.yaml:**
```bash
# These fields should NOT exist
grep -E 'targetCustomizations:|targets:' key/gitrepos/*/*/fleet.yaml
```

### App Deployed to Wrong Clusters

**Verify GitRepo targets:**
```bash
kubectl get gitrepos -n fleet-default -o custom-columns='NAME:.metadata.name,PATHS:.spec.paths,TARGETS:.spec.targets'
```

**Verify cluster labels:**
```bash
kubectl get clusters.fleet.cattle.io -n fleet-default -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels'
```

### GitRepo Not Syncing

**Check GitRepo status:**
```bash
kubectl get gitrepos -n fleet-default
kubectl describe gitrepo common -n fleet-default
```

**Check Fleet controller logs:**
```bash
kubectl logs -n cattle-fleet-system -l app=fleet-controller --tail=50
```

## Deployment Verification

### Check Current Deployments

```bash
# View all bundles and their deployment status
kubectl get bundles -n fleet-default

# Expected output:
# NAME                                       READY   STATUS
# common-key-gitrepos-common-nginx-ingress   2/2     # Both dev and prd
# dev-key-gitrepos-dev-debug-tools           1/1     # Dev only
# prd-key-gitrepos-prd-prd-app               1/1     # Prd only
```

### Verify Pods on Clusters

```bash
# Dev cluster
export KUBECONFIG=/vagrant/kubeconfig1
kubectl get pods -A | grep -E 'nginx|netshoot'

# Prd cluster  
export KUBECONFIG=/vagrant/kubeconfig2
kubectl get pods -A | grep -E 'nginx|prd-app'
```

## Adding New Applications

### To ALL Clusters

1. Create directory under `key/gitrepos/common/`
2. Add fleet.yaml and application files
3. Commit and push
4. GitRepo `common` will deploy to all clusters

### To DEV Clusters Only

1. Create directory under `key/gitrepos/dev/`
2. Add fleet.yaml and application files  
3. Commit and push
4. GitRepo `dev` will deploy to dev clusters only

### To PRD Clusters Only

1. Create directory under `key/gitrepos/prd/`
2. Add fleet.yaml and application files
3. Commit and push
4. GitRepo `prd` will deploy to prd clusters only

## Reference

- **Fleet Documentation**: https://fleet.rancher.io/
- **Infrastructure Repo**: https://github.com/anasgrt/rancher_vagrant_environment
- **Fleet Init Script**: `scripts/fleet_init.sh` in infrastructure repo

## Quick Checklist

Before committing changes:

- [ ] ✅ NO namespace.yaml in Helm chart directories
- [ ] ✅ NO `targetCustomizations` in fleet.yaml
- [ ] ✅ NO `targets` in fleet.yaml
- [ ] ✅ Helm charts use `defaultNamespace` 
- [ ] ✅ Application in correct directory (key/gitrepos/{common,dev,prd})
- [ ] ✅ YAML syntax is valid

Remember: **Targeting is controlled by GitRepos, not fleet.yaml files.**
