# User Guide: Install Prometheus for SSD Instance

This guide explains how to safely install **Prometheus** into an existing **SSD namespace** without impacting running SSD pods.  
It is suitable for **demo, testing, and sandbox environments**.

---

## Prerequisites

- SSD is already running in a Kubernetes namespace  
  (example: `<SSD_NAMESPACE>`)
- Helm is installed and configured
- Cluster access via `kubectl`

---

## Step 1: Verify SSD is Running

```bash
kubectl get pods -n <SSD_NAMESPACE>
```

Ensure all SSD pods are in `Running` or expected states.

---

## Step 2: Add Prometheus Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## Step 3: Create Prometheus Values File

Create a file named `prometheus-values.yaml`:

```yaml
server:
  enabled: true
  persistentVolume:
    enabled: false
  service:
    type: ClusterIP
  retention: "6h"
  global:
    scrape_interval: 30s
    scrape_timeout: 10s

alertmanager:
  enabled: false

pushgateway:
  enabled: false

kube-state-metrics:
  enabled: false

nodeExporter:
  enabled: false
```

---

## Step 4: Install Prometheus into SSD Namespace

```bash
helm install prometheus-demo prometheus-community/prometheus   -n <SSD_NAMESPACE>   -f prometheus-values.yaml
```

---

## Step 5: Verify Prometheus Pods

```bash
kubectl get pods -n <SSD_NAMESPACE> | grep prometheus
```

---

## Step 6: Access Prometheus UI

```bash
kubectl port-forward svc/prometheus-demo-server -n <SSD_NAMESPACE> 9090:80
```

Open in browser:

```
http://localhost:9090
```

---

## Step 7 (Optional): Enable Metrics Scraping for SSD Pods

```bash
kubectl annotate pod -n <SSD_NAMESPACE>   -l app=ssd   prometheus.io/scrape="true"   prometheus.io/path="/metrics"   prometheus.io/port="8008"
```

To remove:

```bash
kubectl annotate pod -n <SSD_NAMESPACE> -l app=ssd prometheus.io/scrape-
```

---

## Example PromQL Queries

### CPU Usage per Pod

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="ssd-demo"}[5m])
)
```

### Memory Usage per Pod (MB)

```promql
sum by (pod) (
  container_memory_working_set_bytes{namespace="ssd-demo"}
) / 1024 / 1024
```

---

## Rollback

```bash
helm uninstall prometheus-demo -n <SSD_NAMESPACE>
```
