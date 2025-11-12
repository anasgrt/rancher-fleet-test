# Local/Management Cluster Applications

This directory contains applications that should be deployed to the management cluster (local cluster).

## Structure

```
local/
└── gitrepos/
    └── monitoring/          # Example monitoring application
        └── fleet.yaml       # Controls targeting
```

## Targeting

All applications in this directory should target the management cluster using:

```yaml
targetCustomizations:
- name: management-only
  clusterSelector:
    matchLabels:
      env: management
```

## Important Notes

- The GitRepo no longer has hard-coded targets
- Each application's `fleet.yaml` controls where it deploys
- This gives you full flexibility to:
  - Deploy some apps only to management cluster
  - Deploy some apps to both management and downstream (if needed)
  - Use different configurations per environment

## Example Use Cases

### Management-Only App
```yaml
# Monitoring, ArgoCD, cluster services
targetCustomizations:
- name: management-only
  clusterSelector:
    matchLabels:
      env: management
```

### Cross-Cluster Tool (if needed)
```yaml
# A tool that should run everywhere
targetCustomizations:
- name: all-clusters
  clusterSelector:
    matchExpressions:
    - key: env
      operator: In
      values:
      - management
      - dev
      - prd
```
