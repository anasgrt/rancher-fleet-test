# Centralized Multi-Cluster Monitoring

This environment uses **Prometheus Federation** to provide centralized monitoring across all clusters.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Management Cluster (local-ctrl)                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Central Prometheus (with Federation)                   │ │
│  │  - Scrapes local metrics (local-ctrl node)             │ │
│  │  - Federates metrics from all downstream clusters      │ │
│  │  - Stores ALL metrics from ALL clusters                │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Central Grafana                                        │ │
│  │  - Single dashboard for ALL clusters                   │ │
│  │  - View metrics from: local-ctrl, key-ctrl,           │ │
│  │    key-worker, and any future clusters                │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │ Federates metrics via HTTP
                           │ (pulls from /federate endpoint)
                           │
┌─────────────────────────────────────────────────────────────┐
│ Key Cluster (key-ctrl + key-worker)                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Local Prometheus                                       │ │
│  │  - Scrapes local metrics (key-ctrl, key-worker)       │ │
│  │  - Exposed on NodePort 30090 for federation           │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

1. **Local Prometheus** on each cluster:
   - Collects metrics from all nodes in that cluster
   - Exposes metrics on NodePort 30090 (downstream clusters only)
   - Provides local Grafana for cluster-specific views

2. **Central Prometheus** on management cluster:
   - Scrapes its own local metrics (local-ctrl node)
   - Federates (pulls) metrics from all downstream Prometheus instances
   - Stores a complete copy of all metrics from all clusters
   - Single source of truth for multi-cluster monitoring

3. **Central Grafana** on management cluster:
   - Queries the central Prometheus
   - Shows metrics from ALL clusters in one dashboard
   - Filter by cluster label to view specific clusters

## Access URLs

### Centralized Monitoring (Management Cluster)
- **Grafana**: http://grafana.192.168.56.10.nip.io
  - Username: `admin` / Password: `admin`
  - Shows metrics from **ALL clusters** (local-ctrl, key-ctrl, key-worker)
- **Prometheus**: http://prometheus.192.168.56.10.nip.io
  - Query metrics with `cluster` label to filter by cluster

### Cluster-Specific Monitoring (Key Cluster)
- **Grafana**: http://grafana.192.168.56.20.nip.io
  - Shows only key cluster metrics (key-ctrl, key-worker)
- **Prometheus**: http://prometheus.192.168.56.20.nip.io

## Adding New Clusters

When you add a new cluster (e.g., "prd"), follow these steps to automatically include it in centralized monitoring:

### 1. Prometheus Deployment (Automatic via Fleet)

Prometheus will be **automatically deployed** to new clusters because it's in `key/gitrepos/common/prometheus/`.

Fleet deploys everything in `common/` to all downstream clusters, so any new cluster labeled with `env: <env>` will get Prometheus.

### 2. Add Federation Configuration (Manual - One Time)

Edit `local/gitrepos/prometheus/helm-release.yaml` and add your new cluster to the federation targets:

```yaml
static_configs:
  # Key cluster
  - targets: ['192.168.56.20:30090']
    labels:
      cluster: 'key'
      environment: 'key'

  # New cluster - ADD THIS
  - targets: ['<new-cluster-control-ip>:30090']
    labels:
      cluster: '<cluster-name>'
      environment: '<env-label>'
```

### 3. Update Ansible Variables (if needed)

Add the new cluster to `ansible/group_vars/all.yml`:

```yaml
downstream_clusters:
  - name: "key"
    environment: "key"
    control_plane_ip: 20
    worker_ip: 21
  - name: "prd"                # NEW CLUSTER
    environment: "prd"
    control_plane_ip: 30
    worker_ip: 31
```

### 4. Deploy

```bash
# Commit changes
git add .
git commit -m "Add prd cluster to monitoring federation"
git push

# Fleet will automatically:
# 1. Deploy Prometheus to the new cluster (via common/prometheus)
# 2. Update management cluster Prometheus with new federation target
# 3. Start collecting metrics from the new cluster

# Wait 30-60 seconds for sync
kubectl get bundles -n fleet-default
```

### 5. Verify

1. Open central Grafana: http://grafana.192.168.56.10.nip.io
2. Go to Explore → Prometheus
3. Query: `up{cluster="prd"}` (should show targets from new cluster)
4. Go to Dashboards → Kubernetes → Compute Resources
5. Select cluster filter → Should see "prd" in dropdown

## Metrics Available

All standard Kubernetes and node metrics are collected:

- **Node Metrics**: CPU, memory, disk, network (per node)
- **Pod Metrics**: CPU, memory, restarts, status (per pod)
- **Kubernetes Resources**: Deployments, services, configmaps, etc.
- **System Components**: API server, etcd, scheduler, controller manager, kubelet, CoreDNS

Filter by labels:
- `cluster="key"` - Metrics from key cluster only
- `cluster="local"` - Metrics from management cluster only
- `node="key-ctrl"` - Metrics from specific node
- `environment="key"` - Metrics from specific environment

## Important Configuration Notes

### Node Label Attachment

To ensure Grafana dashboards can filter by node name, the Helm configuration includes:

```yaml
prometheus-node-exporter:
  prometheus:
    monitor:
      attachMetadata:
        node: true
```

This attaches the Kubernetes node name as a `node` label to all node-exporter metrics, enabling proper filtering in Grafana dashboards. Without this, dashboards that filter by node name (e.g., `node="key-ctrl"`) will not display data.

**Requires**: Prometheus v2.45+ and prometheus-node-exporter chart v4.15.0+

### External Labels

Each Prometheus instance is configured with external labels for proper identification:

**Management Cluster (local)**:
```yaml
externalLabels:
  cluster: 'local'
  cluster_type: 'management'
  environment: 'local'
```

**Downstream Clusters (key)**:
```yaml
externalLabels:
  cluster: 'key'
  cluster_type: 'downstream'
  environment: 'key'
```

These labels are automatically added to all metrics scraped by that Prometheus instance, enabling multi-cluster filtering in Grafana.

## Troubleshooting

### Metrics not showing from downstream cluster

1. **Check federation target is reachable**:
   ```bash
   # From management cluster
   kubectl exec -n prometheus-monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
     wget -O- http://192.168.56.20:30090/metrics
   ```

2. **Check Prometheus targets**:
   - Open: http://prometheus.192.168.56.10.nip.io/targets
   - Look for `federate-downstream-clusters` job
   - Should show target `192.168.56.20:30090` as UP

3. **Check downstream Prometheus is running**:
   ```bash
   kubectl --kubeconfig=.shared/kubeconfig-key get pods -n prometheus-monitoring
   ```

### Adding cluster but metrics not appearing

1. Verify the control plane IP is correct in federation config
2. Check NodePort 30090 is exposed on downstream Prometheus
3. Wait 30 seconds for Prometheus to scrape (scrape_interval: 30s)
4. Check Prometheus logs: `kubectl logs -n prometheus-monitoring prometheus-prometheus-kube-prometheus-prometheus-0`

### Dashboard shows "No data" or nodes not appearing in filters

**Symptoms**: Grafana dashboards don't show any data when filtering by node name, or node names don't appear in dropdown filters.

**Root Cause**: The `node` label is not attached to node-exporter metrics.

**Solution**:

1. Verify the configuration includes node metadata attachment:
   ```bash
   kubectl get servicemonitor -n prometheus-monitoring prometheus-prometheus-node-exporter -o yaml | grep -A 2 "attachMetadata:"
   ```
   Should show:
   ```yaml
   attachMetadata:
     node: true
   ```

2. If it shows `node: false`, update your Helm values to include:
   ```yaml
   prometheus-node-exporter:
     prometheus:
       monitor:
         attachMetadata:
           node: true
   ```

3. After updating, wait 2-3 minutes for the ServiceMonitor to be updated and Prometheus to rescrape.

4. Verify the `node` label is present in metrics:
   ```bash
   kubectl exec -n prometheus-monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
     wget -qO- 'http://localhost:9090/api/v1/query?query=node_cpu_seconds_total' | grep '"node"'
   ```

5. Restart Grafana pod to refresh dashboard queries if needed:
   ```bash
   kubectl delete pod -n prometheus-monitoring -l app.kubernetes.io/name=grafana
   ```

## Benefits of This Architecture

✅ **Centralized View**: One Grafana dashboard for all clusters
✅ **Automatic Discovery**: New clusters in `common/` get Prometheus automatically
✅ **Easy to Add**: Just add one static config entry for federation
✅ **Scalable**: Can handle many clusters (limited by network/storage)
✅ **Resilient**: Local Prometheus continues working even if federation fails
✅ **Cost Efficient**: Uses existing Prometheus, no additional services needed

## Alternative: Future Improvements

For even more dynamic setup, consider:

1. **Thanos** - For long-term storage and global query view
2. **Prometheus Operator + ServiceMonitor** - Automatic service discovery across clusters
3. **Rancher Monitoring V2** - Built-in multi-cluster monitoring (if you enable it)

The current federation approach is simple, reliable, and works well for small-to-medium deployments.
