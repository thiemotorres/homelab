# Monitoring and Alerting

This document describes the monitoring and alerting infrastructure using the kube-prometheus-stack.

## Overview

The homelab uses the **kube-prometheus-stack** (Prometheus + Grafana + Alertmanager) for comprehensive monitoring, providing:
- Metrics collection from all cluster components
- Custom dashboards for visualization
- Alerting with Discord notifications
- Long-term metrics storage (15 days)
- K3s-optimized configuration

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                      │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │   Node   │  │   Pods   │  │ Services │          │
│  │ Exporter │  │          │  │          │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │             │             │                 │
│       │ Metrics     │ Metrics     │ Metrics         │
│       │             │             │                 │
│       ▼             ▼             ▼                 │
│  ┌────────────────────────────────────┐            │
│  │         Prometheus                  │            │
│  │  - Scrapes metrics every 30s        │            │
│  │  - Stores 15 days                   │            │
│  │  - Evaluates alert rules            │            │
│  └──────────┬──────────────┬───────────┘            │
│             │              │                         │
│  ┌──────────▼─────┐  ┌─────▼──────────┐            │
│  │   Grafana      │  │  Alertmanager  │            │
│  │  - Dashboards  │  │  - Alert       │            │
│  │  - Queries     │  │    routing     │            │
│  └────────────────┘  └─────┬──────────┘            │
│                             │                        │
└─────────────────────────────┼────────────────────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │ alertmanager-discord │
                   │   (Webhook Adapter)  │
                   └──────────┬───────────┘
                              │
                              ▼
                   ┌──────────────────────┐
                   │   Discord Channel    │
                   │   (Notifications)    │
                   └──────────────────────┘
```

## Components

### Prometheus

**Purpose**: Time-series database and metrics collection

**Configuration**: `argocd-apps/monitoring.yaml`

```yaml
prometheus:
  enabled: true
  prometheusSpec:
    replicas: 1
    retention: 15d              # Keep metrics for 15 days
    retentionSize: "8GB"        # Max 8GB storage
    storageSpec:
      volumeClaimTemplate:
        storageClassName: local-path
        storage: 10Gi           # Total PVC size
    resources:
      requests: {memory: 512Mi, cpu: 250m}
      limits: {memory: 2Gi, cpu: 1000m}
```

**Access**:
- Not exposed externally (internal only)
- Access via port-forward:
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
  # Open http://localhost:9090
  ```

---

### Grafana

**Purpose**: Visualization and dashboarding

**Configuration**:
```yaml
grafana:
  enabled: true
  persistence:
    enabled: true
    size: 5Gi
  admin:
    existingSecret: grafana-admin  # Sealed secret
  resources:
    requests: {memory: 128Mi, cpu: 100m}
    limits: {memory: 512Mi, cpu: 500m}
  env:
    GF_SERVER_ROOT_URL: https://grafana.feto.dev
```

**Access**:
- **URL**: https://grafana.feto.dev
- **Credentials**: Stored in `grafana-admin` secret
- **TLS**: Automatic via cert-manager

**Plugins Installed**:
- `grafana-piechart-panel`
- `grafana-worldmap-panel`

---

### Alertmanager

**Purpose**: Alert routing and notification management

**Configuration**:
```yaml
alertmanager:
  enabled: true
  retention: 120h  # Keep alert history for 5 days
  storage: 5Gi
  resources:
    requests: {memory: 64Mi, cpu: 50m}
    limits: {memory: 256Mi, cpu: 200m}

  config:
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s       # Wait 10s before sending
      group_interval: 10s   # Group alerts every 10s
      repeat_interval: 12h  # Resend after 12h if still firing
      receiver: 'discord'
      routes:
      - receiver: 'discord'
        matchers:
          - severity=~"warning|critical"

    receivers:
    - name: 'discord'
      webhook_configs:
      - url: 'http://alertmanager-discord:9094'
        send_resolved: true  # Send notification when alert resolves
```

---

### alertmanager-discord

**Purpose**: Convert Alertmanager webhooks to Discord messages

**Location**: `infrastructure/monitoring/alertmanager-discord.yaml`

**Configuration**:
```yaml
deployment:
  image: benjojo/alertmanager-discord:latest
  env:
    - name: DISCORD_WEBHOOK
      valueFrom:
        secretKeyRef:
          name: discord-alerts
          key: webhook-url
```

**How it works**:
1. Alertmanager sends webhook to `http://alertmanager-discord:9094`
2. alertmanager-discord formats message
3. Posts to Discord webhook URL
4. Alert appears in Discord channel

---

## Metrics Collection

### What Gets Monitored?

#### 1. **Node Metrics** (node-exporter)
- CPU usage
- Memory usage
- Disk I/O
- Network traffic
- Filesystem usage

#### 2. **Kubernetes Metrics** (kube-state-metrics)
- Pod status and restarts
- Deployment status
- ReplicaSet status
- Service status
- PersistentVolume usage

#### 3. **Container Metrics** (cAdvisor, built-in)
- Container CPU
- Container memory
- Container network
- Container filesystem

#### 4. **Control Plane Metrics**
- API server (disabled for K3s)
- Scheduler (disabled for K3s)
- Controller manager (disabled for K3s)
- etcd (disabled for K3s)

**Why disabled?** K3s manages these components internally and doesn't expose metrics in the same way as full Kubernetes.

#### 5. **Custom Application Metrics**
- Traefik metrics (via custom scrape config)
- Any app exposing `/metrics` endpoint

---

### Scrape Configuration

Prometheus scrapes metrics every 30 seconds (default).

**ServiceMonitor** selector (collects from all):
```yaml
serviceMonitorSelector: {}
serviceMonitorNamespaceSelector: {}
podMonitorSelector: {}
podMonitorNamespaceSelector: {}
```

This means Prometheus automatically discovers and scrapes:
- Any ServiceMonitor in any namespace
- Any PodMonitor in any namespace

**Custom scrape config** for Traefik:
```yaml
additionalScrapeConfigs:
- job_name: 'traefik'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
    regex: traefik
    action: keep
  - source_labels: [__meta_kubernetes_pod_container_port_name]
    regex: metrics
    action: keep
```

---

## Alert Rules

### Enabled Alert Groups

| Rule Group | Enabled | Purpose |
|-----------|---------|---------|
| alertmanager | ✅ | Alertmanager health |
| etcd | ❌ | K3s managed (not exposed) |
| general | ✅ | General cluster alerts |
| k8s | ✅ | Kubernetes resource alerts |
| kubeApiserver | ❌ | K3s managed (basic metrics disabled) |
| kubeApiserverAvailability | ✅ | API server availability |
| kubePrometheusGeneral | ✅ | Prometheus stack health |
| kubernetesApps | ✅ | Application deployments |
| kubernetesResources | ✅ | Resource usage alerts |
| kubernetesStorage | ✅ | PV/PVC alerts |
| kubernetesSystem | ✅ | System component alerts |
| kubeScheduler | ❌ | K3s managed |
| kubeProxy | ❌ | K3s managed |
| network | ✅ | Network-related alerts |
| node | ✅ | Node health and resources |
| prometheus | ✅ | Prometheus self-monitoring |
| prometheusOperator | ✅ | Operator health |

### Common Alert Examples

#### Node Alerts
- **NodeMemoryHighUtilization**: Memory > 90%
- **NodeDiskSpaceFillingUp**: Disk filling up
- **NodeCPUHighUsage**: CPU > 95%

#### Pod Alerts
- **KubePodCrashLooping**: Pod restarting repeatedly
- **KubePodNotReady**: Pod not ready for > 15 min
- **KubeDeploymentReplicasMismatch**: Desired vs actual replicas differ

#### Storage Alerts
- **KubePersistentVolumeFillingUp**: PV > 80% full
- **KubePersistentVolumeErrors**: PV access errors

---

## Discord Notifications

### Alert Format

**Firing Alert:**
```
🔴 [FIRING] KubePodCrashLooping
Severity: warning

Pod gork-backend-xxxxx is crash looping
Namespace: gork
Pod: gork-backend-xxxxx
Container: gork-backend

Labels:
  alertname: KubePodCrashLooping
  namespace: gork
  pod: gork-backend-xxxxx
  severity: warning
```

**Resolved Alert:**
```
🟢 [RESOLVED] KubePodCrashLooping
Pod gork-backend-xxxxx has recovered
```

### Configure Discord Webhook

1. **Create Discord Webhook**:
   - Go to Discord Server Settings → Integrations
   - Create Webhook
   - Copy Webhook URL

2. **Create Sealed Secret**:
   ```bash
   kubectl create secret generic discord-alerts \
     --from-literal=webhook-url='https://discord.com/api/webhooks/...' \
     --namespace=monitoring \
     --dry-run=client -o yaml > /tmp/secret.yaml

   kubeseal --format=yaml --cert=pub-cert.pem \
     < /tmp/secret.yaml \
     > infrastructure/monitoring/sealed-secrets/discord-alerts-sealed.yaml

   rm /tmp/secret.yaml
   ```

3. **Apply and verify**:
   ```bash
   kubectl apply -f infrastructure/monitoring/sealed-secrets/discord-alerts-sealed.yaml
   kubectl get secret discord-alerts -n monitoring
   ```

---

## Dashboards

### Default Dashboards

The kube-prometheus-stack includes many pre-configured dashboards:

**Node Dashboards:**
- Node Exporter / Nodes - Detailed node metrics
- Node Exporter / USE Method / Node - Utilization, Saturation, Errors

**Kubernetes Dashboards:**
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace
- Kubernetes / Compute Resources / Pod
- Kubernetes / Kubelet

**Prometheus Dashboards:**
- Prometheus / Overview
- Alertmanager / Overview

### Custom Dashboards

**Adding custom dashboards**:

1. **Via Grafana UI** (easiest):
   - Login to Grafana
   - Create dashboard
   - Save

2. **Via ConfigMap**:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: custom-dashboard
     namespace: monitoring
     labels:
       grafana_dashboard: "1"
   data:
     my-dashboard.json: |
       {
         "dashboard": { ... }
       }
   ```

3. **Via dashboard providers** (current setup):
   ```yaml
   dashboardProviders:
     dashboardproviders.yaml:
       providers:
       - name: 'default'
         folder: ''
         type: file
         editable: true
         options:
           path: /var/lib/grafana/dashboards/default
   ```

**Recommended community dashboards:**
- **Traefik**: Dashboard ID 4475 or 11462
- **Node Exporter Full**: Dashboard ID 1860
- **Kubernetes Cluster Monitoring**: Dashboard ID 7249

---

## PromQL Query Examples

### Node Metrics

**CPU usage per node:**
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Memory usage:**
```promql
100 * (1 - ((node_memory_MemAvailable_bytes) / (node_memory_MemTotal_bytes)))
```

**Disk usage:**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})
```

### Pod Metrics

**Pod CPU usage:**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="gork",pod=~"gork-backend.*"}[5m])) by (pod)
```

**Pod memory usage:**
```promql
sum(container_memory_working_set_bytes{namespace="gork",pod=~"gork-backend.*"}) by (pod)
```

**Pod restarts:**
```promql
rate(kube_pod_container_status_restarts_total{namespace="gork"}[5m])
```

### Application Metrics

**HTTP request rate (if app exposes metrics):**
```promql
rate(http_requests_total[5m])
```

**Request duration (99th percentile):**
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## Adding ServiceMonitors

To monitor your custom applications, create ServiceMonitor resources.

### Example: gork-backend ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gork-backend
  namespace: gork
  labels:
    app: gork-backend
spec:
  selector:
    matchLabels:
      app: gork-backend
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
```

**Prerequisites:**
1. Application must expose `/metrics` endpoint
2. Metrics in Prometheus format
3. Service must have matching labels

---

## Troubleshooting

### Metrics Not Showing Up

**Check Prometheus targets:**
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090/targets
```

**Check ServiceMonitor:**
```bash
kubectl get servicemonitor -n <namespace>
kubectl describe servicemonitor <name> -n <namespace>
```

**Check if Prometheus can reach service:**
```bash
# From Prometheus pod
kubectl exec -it -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -- \
  wget -O- http://<service-name>.<namespace>:8080/metrics
```

### Alerts Not Firing

**Check Alertmanager:**
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093
# Open http://localhost:9093
```

**Check alertmanager-discord logs:**
```bash
kubectl logs -n monitoring deployment/alertmanager-discord
```

**Test Discord webhook manually:**
```bash
curl -H "Content-Type: application/json" \
  -d '{"embeds":[{"title":"Test Alert","description":"Testing webhook","color":15158332}]}' \
  https://discord.com/api/webhooks/YOUR-WEBHOOK-URL
```

### Grafana Not Loading

**Check Grafana logs:**
```bash
kubectl logs -n monitoring deployment/monitoring-kube-prometheus-grafana
```

**Check admin password:**
```bash
kubectl get secret grafana-admin -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
```

**Reset admin password:**
```bash
# Create new sealed secret with updated password
kubectl create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='newpassword' \
  --namespace=monitoring \
  --dry-run=client -o yaml | \
kubeseal --format=yaml --cert=pub-cert.pem > \
  infrastructure/monitoring/sealed-secrets/grafana-admin-sealed.yaml
```

### Prometheus Storage Full

**Check PVC usage:**
```bash
kubectl exec -it -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -- \
  df -h /prometheus
```

**Solutions:**
1. **Reduce retention**:
   ```yaml
   retention: 7d  # Change from 15d
   ```

2. **Increase PVC size**:
   ```yaml
   storage: 20Gi  # Change from 10Gi
   ```

3. **Reduce retention size**:
   ```yaml
   retentionSize: "6GB"  # Change from 8GB
   ```

---

## Performance Tuning

### Reduce Scrape Frequency

```yaml
prometheus:
  prometheusSpec:
    evaluationInterval: 30s  # How often to evaluate rules
    scrapeInterval: 30s      # How often to scrape metrics
```

### Limit Metric Retention

```yaml
prometheus:
  prometheusSpec:
    retention: 7d         # Reduce from 15d
    retentionSize: "5GB"  # Reduce from 8GB
```

### Reduce Resources

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 256Mi  # Reduce from 512Mi
        cpu: 100m      # Reduce from 250m
```

---

## Backup Monitoring Data

Prometheus data is stored in PersistentVolume but:
- Not critical (metrics regenerate)
- Large size (not practical to backup)
- Historical data can be lost without major impact

**If you need to backup:**

```bash
# Take snapshot of PVC
kubectl exec -it -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -- \
  tar czf /tmp/prometheus-backup.tar.gz /prometheus

# Copy out
kubectl cp monitoring/prometheus-monitoring-kube-prometheus-prometheus-0:/tmp/prometheus-backup.tar.gz \
  ./prometheus-backup.tar.gz
```

**Better approach**: Use remote write to external storage (Thanos, VictoriaMetrics, etc.)

---

## Best Practices

### 1. Alert Fatigue

**Avoid:**
- Too many alerts
- Alerts for non-actionable issues
- Duplicate alerts

**Do:**
- Only alert on actionable issues
- Use appropriate severity levels
- Group related alerts
- Set reasonable repeat_interval

### 2. Dashboard Organization

**Do:**
- Create dashboards per service
- Use consistent naming
- Add descriptions to panels
- Use variables for flexibility

### 3. Metric Retention

**Balance:**
- Longer retention = more historical data
- Longer retention = more disk usage
- 7-15 days is reasonable for homelab

### 4. Resource Limits

**Set:**
- Appropriate memory limits (Prometheus can OOM)
- CPU limits to prevent resource exhaustion
- Storage limits based on retention

---

## Security Considerations

### 1. Grafana Access

✅ TLS enabled via cert-manager
✅ Admin credentials in sealed secret
❌ No SSO/OAuth configured (homelab acceptable)

**Recommendations:**
- Change default admin password
- Create separate user accounts
- Enable OIDC if desired

### 2. Prometheus Access

✅ Not exposed externally
✅ Only accessible via port-forward or within cluster
❌ No authentication (Kubernetes RBAC protects access)

### 3. Metric Data

⚠️ Metrics may contain sensitive information:
- Pod names (may reveal architecture)
- Resource usage patterns
- Application behavior

**Mitigation**:
- Limit external access
- Use NetworkPolicies (recommended)
- Scrub sensitive labels if needed

---

## References

- [kube-prometheus-stack Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)

---

## Changelog

- **2025-10-26**: Documented existing monitoring stack
  - kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
  - Discord integration via alertmanager-discord
  - K3s-optimized alert rules
  - 15-day retention with 10Gi storage
