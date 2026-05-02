# Monitoring dengan Prometheus dan Grafana

Prometheus adalah sistem monitoring dan alerting open-source yang sangat populer untuk Kubernetes. Grafana adalah tool visualisasi yang menampilkan data Prometheus dalam bentuk dashboard.

---

## Arsitektur Monitoring Stack

```
Kubernetes Cluster
┌──────────────────────────────────────────────────────┐
│                                                      │
│  ┌─────────────┐    ┌─────────────────────────────┐  │
│  │  Your App   │    │   kube-state-metrics         │  │
│  │ (exposes    │    │   (K8s object metrics)       │  │
│  │  /metrics)  │    └────────────┬────────────────┘  │
│  └──────┬──────┘                 │                   │
│         │  scrape                │ scrape             │
│         ▼                        ▼                   │
│  ┌──────────────────────────────────────────────┐    │
│  │              Prometheus Server               │    │
│  │         (scrape, store, query)               │    │
│  └──────────────────┬───────────────────────────┘    │
│                     │ data source                    │
│                     ▼                                │
│  ┌──────────────────────────────────────────────┐    │
│  │              Grafana                          │    │
│  │         (dashboard, alerting)                 │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │         Node Exporter (DaemonSet)            │    │
│  │    (CPU, Memory, Disk metrics per Node)      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## Install dengan Helm (kube-prometheus-stack)

`kube-prometheus-stack` adalah Helm chart yang install semua komponen sekaligus:
- Prometheus
- Grafana
- AlertManager
- Node Exporter
- kube-state-metrics
- Default dashboards dan alerts

### Setup

```bash
# Tambah repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Buat namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Verifikasi semua Pod running
kubectl get pods -n monitoring
# Output yang diharapkan:
# NAME                                                   READY   STATUS
# alertmanager-kube-prometheus-stack-alertmanager-0     2/2     Running
# kube-prometheus-stack-grafana-xxx-yyy                 3/3     Running
# kube-prometheus-stack-kube-state-metrics-xxx          1/1     Running
# kube-prometheus-stack-operator-xxx                    1/1     Running
# kube-prometheus-stack-prometheus-node-exporter-xxx    1/1     Running
# prometheus-kube-prometheus-stack-prometheus-0         2/2     Running
```

---

## Akses Grafana

```bash
# Dapatkan password Grafana
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
echo ""
# Output: <password>  (biasanya "prom-operator")

# Port-forward untuk akses dari browser
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Buka browser: http://localhost:3000
# Username: admin
# Password: (dari command di atas)
```

---

## Dashboard Default yang Tersedia

Setelah install, kamu sudah punya banyak dashboard bawaan di Grafana:

1. **Kubernetes / Compute Resources / Cluster** — Overview penggunaan resource cluster
2. **Kubernetes / Compute Resources / Node (Pods)** — Resource per Node
3. **Kubernetes / Compute Resources / Namespace (Pods)** — Resource per Namespace
4. **Kubernetes / Compute Resources / Pod** — Resource per Pod
5. **Node Exporter / Nodes** — Metrics detail setiap Node
6. **Kubernetes / Networking / Cluster** — Network traffic

---

## Query Prometheus (PromQL Dasar)

Prometheus menggunakan query language PromQL. Berikut query yang sering dipakai:

```promql
# CPU usage per Pod (%)
100 * sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
  / sum(container_spec_cpu_quota{namespace="default"}) by (pod) * 100

# Memory usage per Pod
container_memory_working_set_bytes{namespace="default", container!=""}

# Pod restart count (dalam 1 jam terakhir)
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Node CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# HTTP request rate
rate(http_requests_total[5m])

# Error rate (HTTP 5xx)
sum(rate(http_requests_total{status_code=~"5.."}[5m])) 
  / sum(rate(http_requests_total[5m]))
```

---

## Akses Prometheus Langsung

```bash
# Port-forward ke Prometheus UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Buka: http://localhost:9090
# Di sini kamu bisa query langsung dengan PromQL
```

---

## Konfigurasi Kustom

### Override Values saat Install

```bash
# Simpan custom values
cat > monitoring-values.yaml << EOF
grafana:
  adminPassword: "mySecurePassword123"
  persistence:
    enabled: true
    size: 5Gi

prometheus:
  prometheusSpec:
    retention: 30d          # Simpan data 30 hari
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 50Gi

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 2Gi
EOF

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring-values.yaml
```

---

## Menambahkan Custom Metrics dari Aplikasi

Agar aplikasimu bisa di-scrape Prometheus, tambahkan endpoint `/metrics`:

### Contoh di Node.js (dengan prom-client)

```javascript
// server.js
const prometheus = require('prom-client');
const express = require('express');
const app = express();

// Expose default metrics (CPU, memory, dll)
prometheus.collectDefaultMetrics();

// Custom metric
const httpRequestsTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status'],
});

// Middleware untuk hitung requests
app.use((req, res, next) => {
  res.on('finish', () => {
    httpRequestsTotal.labels(req.method, req.path, res.statusCode).inc();
  });
  next();
});

// Endpoint metrics
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.send(await prometheus.register.metrics());
});
```

```yaml
# Tambahkan annotation ke Deployment
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

---

## Setup Alerting

AlertManager mengirim notifikasi ketika ada alert. Konfigurasi untuk Slack:

```yaml
# alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: slack-alerts
  namespace: monitoring
spec:
  route:
    groupBy: ['alertname', 'cluster']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: slack-notifications

  receivers:
  - name: slack-notifications
    slackConfigs:
    - apiURL:
        secretKeyRef:
          name: slack-webhook
          key: url
      channel: '#alerts-kubernetes'
      title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
      text: '{{ range .Alerts }}*Alert:* {{ .Labels.alertname }}\n*Summary:* {{ .Annotations.summary }}\n{{ end }}'
```

---

## Cleanup

```bash
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring
```

---

*[Kembali ke: Overview Observability](./README.md)*

*[Lanjut ke: Logging →](./02-logging.md)*
