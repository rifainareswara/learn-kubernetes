# 07 - Deploy ke Cloud

> **Estimasi Waktu:** 45 menit
>
> **Tujuan:** Memahami perbedaan deploy ke cloud vs lokal, dan menjalankan aplikasi di cloud Kubernetes

---

## Perbedaan Deploy Cloud vs Lokal

| Aspek | Lokal (Minikube/Kind) | Cloud (GKE/EKS/ACK) |
|---|---|---|
| **Container Registry** | Load image langsung | Push ke registry (GCR, ECR, ACR) |
| **Load Balancer** | `minikube tunnel` | Cloud Load Balancer (otomatis, IP real) |
| **Persistent Storage** | `hostPath` (disk VM) | Cloud Disk (SSD yang reliable, backup) |
| **DNS** | `localhost` atau IP lokal | Domain real dengan DNS global |
| **TLS/HTTPS** | Opsional, tidak kritis | WAJIB untuk production |
| **Biaya** | Gratis | Berbayar (per resource) |
| **Skalabilitas** | Terbatas | Bisa auto-scale |

---

## Langkah Umum untuk Semua Cloud

Terlepas dari cloud provider yang digunakan, langkah dasarnya sama:

```
1. Build Docker image di local
   ↓
2. Push image ke Container Registry cloud
   ↓
3. Update deployment.yaml dengan URL image registry
   ↓
4. Apply Kubernetes manifests ke cluster cloud
   ↓
5. Install Ingress Controller (jika belum ada)
   ↓
6. Konfigurasi DNS → IP External Ingress
   ↓
7. (Opsional) Setup TLS dengan cert-manager
```

---

## Pilih Cloud Provider

| File | Provider | Keterangan |
|---|---|---|
| [deploy-gke.md](./deploy-gke.md) | Google Kubernetes Engine | Cocok untuk pengguna GCP |
| [deploy-eks.md](./deploy-eks.md) | Amazon Elastic Kubernetes Service | Cocok untuk pengguna AWS |
| [deploy-ack.md](./deploy-ack.md) | Alibaba Container Service for Kubernetes | Cocok untuk pengguna Alibaba Cloud |

---

## Tips Umum untuk Production

### 1. Gunakan Namespace yang Bermakna

```bash
# Pisahkan environment dengan namespace
kubectl create namespace myapp-production
kubectl create namespace myapp-staging
```

### 2. Resource Requests dan Limits Selalu Ada

Di cloud, kamu bayar untuk resource yang kamu gunakan. Tanpa limits, satu Pod bisa memakan resource berlebih dan meningkatkan biaya.

### 3. Konfigurasi HorizontalPodAutoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 4. Backup Database Secara Berkala

Untuk PostgreSQL di production, gunakan:
- Cloud-managed database (Cloud SQL, RDS, PolarDB) — lebih mudah dan reliable
- Atau CronJob Kubernetes untuk backup berkala ke Object Storage

### 5. Monitoring & Alerting

Pertimbangkan install:
- **Prometheus + Grafana** — untuk metrics dan dashboard
- **Loki** — untuk log aggregation
- Atau gunakan managed monitoring dari cloud provider

---

> **Perhatian:** Jangan lupa menghapus/mematikan cluster cloud setelah selesai belajar untuk menghindari biaya yang tidak perlu!

---

## Selanjutnya

- [08-cicd/README.md](../08-cicd/README.md) — Otomasi deployment dengan GitHub Actions
