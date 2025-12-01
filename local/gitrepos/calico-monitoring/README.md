# Calico Monitoring Configuration

This Fleet bundle configures Prometheus monitoring for Calico CNI components on the management cluster.

## What's Deployed

### 1. ServiceMonitors
- **calico-felix**: Monitors the Calico Felix agent (network policy enforcement)
- **calico-typha**: Monitors the Calico Typha datastore proxy
- **calico-kube-controllers**: Monitors the Calico Kubernetes controllers

### 2. Services
- **calico-felix-metrics**: Exposes Felix metrics on port 9091
- **calico-typha-metrics**: Exposes Typha metrics on port 9091

## Prerequisites

Before deploying this bundle, **Felix Prometheus metrics must be enabled manually**:

```bash
kubectl patch felixconfiguration default --type=merge --patch '{"spec":{"prometheusMetricsEnabled":true,"prometheusMetricsPort":9091}}'
```

This is required because the FelixConfiguration is managed by the RKE2 Calico Helm chart and cannot be managed by Fleet.

## Labels Applied

All metrics are labeled with:
- `cluster: local` - Identifies the management cluster
- `cluster_type: management` - Cluster type designation
- `environment: local` - Environment identifier

These labels ensure compatibility with Grafana dashboards that filter by cluster.

## Target Cluster

This configuration is deployed **only to the management cluster** (`env: management`).

## Metrics Available

Once deployed, Prometheus will scrape the following Calico metrics:
- Felix active endpoints
- Policy enforcement statistics
- Network traffic statistics
- CNI operations
- And many more...

## Grafana Dashboards

To view Calico metrics in Grafana:
1. Navigate to the Kubernetes dashboards
2. Select **cluster: local** (not "key")
3. Choose the **calico-system** namespace
4. View CPU, memory, and network metrics

## Files

- `fleet.yaml` - Fleet configuration and targeting
- `servicemonitors.yaml` - ServiceMonitor and Service definitions

## Notes

- **Important**: Felix Prometheus metrics must be enabled manually (see Prerequisites)
- ServiceMonitors require the `release: prometheus` label to be discovered
- Metrics become available immediately after Felix configuration is updated
