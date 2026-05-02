# Centralized Logging dengan Loki dan Grafana

Loki adalah sistem log aggregation yang dirancang oleh Grafana Labs. Loki mudah di-setup dan terintegrasi sempurna dengan Grafana yang sudah kita install.

---

## Mengapa Perlu Centralized Logging?

```
Tanpa centralized logging:
kubectl logs pod-A       ← lihat log pod A
kubectl logs pod-B       ← lihat log pod B (terpisah)
kubectl logs pod-C       ← pod C crash, log hilang!

Dengan Loki:
Grafana → query semua log dari semua Pod sekaligus
         → log disimpan bahkan setelah Pod mati
         → bisa filter, search, alert
```

---

## Arsitektur Loki Stack

```
Pods/Containers
     │ stdout/stderr
     ▼
Promtail (DaemonSet)     ← Baca log dari setiap Node
     │ push
     ▼
Loki (Store)             ← Simpan dan index log
     │ query
     ▼
Grafana                  ← Visualisasi dan search log
```

---

## Install Loki Stack dengan Helm

```bash
# Tambah repository Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki Stack (Loki + Promtail + Grafana)
# Jika sudah install Grafana sebelumnya, nonaktifkan grafana di sini
helm install loki-stack grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \       # Sudah ada dari kube-prometheus-stack
  --set prometheus.enabled=false \    # Sudah ada dari kube-prometheus-stack
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi

# Atau install Loki saja
helm install loki grafana/loki \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi

# Install Promtail (log collector)
helm install promtail grafana/promtail \
  --namespace monitoring \
  --set config.lokiAddress=http://loki:3100/loki/api/v1/push

# Verifikasi
kubectl get pods -n monitoring | grep loki
kubectl get pods -n monitoring | grep promtail
```

---

## Tambahkan Loki sebagai Data Source di Grafana

```bash
# Akses Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Di Grafana:
# 1. Configuration → Data Sources → Add Data Source
# 2. Pilih Loki
# 3. URL: http://loki:3100
# 4. Save & Test
```

---

## Query Log di Grafana (LogQL)

Grafana menyediakan halaman **Explore** untuk query log menggunakan LogQL:

```logql
# Semua log dari namespace default
{namespace="default"}

# Log dari Pod dengan label app=nginx
{namespace="default", app="nginx"}

# Filter log yang mengandung kata "error" (case-insensitive)
{namespace="default"} |= "error" or "Error" or "ERROR"

# Filter berdasarkan regex
{namespace="default"} |~ "ERROR|FATAL|PANIC"

# Exclude log tertentu
{namespace="default"} != "health check"

# Parse JSON log
{app="my-app"} | json | level="error"

# Count rate of errors per minute
sum(rate({namespace="default"} |= "ERROR" [1m]))

# Top 10 Pods dengan paling banyak error
topk(10, sum by (pod) (count_over_time({namespace="default"} |= "ERROR" [1h])))
```

---

## Ekspor Log ke Grafana Dashboard

Buat panel di Grafana untuk monitoring log:

### Panel 1: Error Rate Over Time
```logql
# Query untuk line chart
sum(rate({namespace="production"} |= "ERROR" [5m]))
```

### Panel 2: Recent Errors
```logql
# Query untuk log list
{namespace="production"} |= "ERROR" | json | line_format "{{.pod}} - {{.message}}"
```

### Panel 3: Log Volume per Service
```logql
# Query untuk pie chart
sum by (app) (count_over_time({namespace="production"} [1h]))
```

---

## Konfigurasi Promtail untuk Log Terstruktur

```yaml
# promtail-config.yaml (custom configuration)
# Tambahkan ke values saat install

config:
  snippets:
    pipelineStages:
      # Parse log JSON
      - json:
          expressions:
            level: level
            message: message
            service: service

      # Tambahkan label berdasarkan level
      - labels:
          level:
          service:

      # Filter log berdasarkan level minimum
      - match:
          selector: '{level="debug"}'
          action: drop              # Buang log debug untuk hemat storage
          drop_counter_reason: debug_logs
```

---

## Alert Berdasarkan Log

Buat alert di Grafana jika ada terlalu banyak error:

```yaml
# Di Grafana: Alerting → Alert Rules → New Alert Rule
# Query:
sum(count_over_time({namespace="production"} |= "ERROR" [5m])) > 10

# Alert jika lebih dari 10 error dalam 5 menit
```

---

## Tips Logging yang Baik

### Format Log yang Ideal (JSON structured logging)

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "user-service",
  "message": "Database connection failed",
  "error": "connection timeout after 30s",
  "request_id": "abc123",
  "user_id": "usr_456"
}
```

### Implementasi di Node.js

```javascript
const pino = require('pino');
const logger = pino({ level: 'info' });

// Structured logging
logger.error({
  service: 'user-service',
  error: err.message,
  requestId: req.id,
  userId: req.user?.id
}, 'Database connection failed');
```

### Implementasi di Python

```python
import logging
import json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "service": "my-service",
            "message": record.getMessage(),
        }
        return json.dumps(log_data)

logging.basicConfig(level=logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
```

---

## Cleanup

```bash
helm uninstall loki -n monitoring
helm uninstall promtail -n monitoring
```

---

*[Kembali ke: Monitoring dengan Prometheus & Grafana ←](./01-prometheus-grafana.md)*

*[Kembali ke: Overview Observability](./README.md)*

*[Lanjut ke: CI/CD →](../11-cicd/README.md)*
