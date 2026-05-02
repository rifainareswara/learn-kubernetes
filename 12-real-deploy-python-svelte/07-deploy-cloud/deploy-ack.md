# Deploy ke Alibaba Container Service for Kubernetes (ACK)

> **Estimasi Waktu:** 45-60 menit
>
> **Prasyarat:** Akun Alibaba Cloud, `aliyun` CLI terinstall dan terkonfigurasi

---

## Persiapan Alibaba Cloud CLI

```bash
# Install Alibaba Cloud CLI (jika belum)
# macOS:
brew install aliyun-cli

# Konfigurasi credentials
aliyun configure
# Masukkan: AccessKey ID, AccessKey Secret, Region (id-jakarta untuk Indonesia)

# Verifikasi
aliyun sts GetCallerIdentity
```

---

## Langkah 1: Push Image ke Alibaba Container Registry (ACR)

Alibaba Container Registry (ACR) tersedia dalam dua tier:
- **ACR Personal Edition** — Gratis, cocok untuk belajar
- **ACR Enterprise Edition** — Berbayar, untuk production dengan fitur lengkap

### 1.1 Buat Namespace dan Repository di ACR

Konfigurasi melalui Alibaba Cloud Console:
1. Buka [cr.console.aliyun.com](https://cr.console.aliyun.com)
2. Pilih region **Indonesia (Jakarta)** atau yang terdekat
3. Buat **Namespace** baru (misalnya: `mycompany-myapp`)
4. Buat **Repository** untuk `backend` dan `frontend`

### 1.2 Login Docker ke ACR

```bash
# Login ke ACR Personal Edition (registry.cn-REGION.aliyuncs.com)
docker login \
  --username=YOUR_ALIBABA_ACCOUNT_EMAIL \
  registry.cn-jakarta.aliyuncs.com

# Masukkan password Alibaba Cloud Console (bukan AccessKey!)

# Verifikasi login berhasil
# Output: Login Succeeded
```

> **Tips:** Jika kamu menggunakan RAM User (sub-account), gunakan format: `username@company.id` untuk field username.

### 1.3 Tag dan Push Image

```bash
# Format: registry.cn-REGION.aliyuncs.com/NAMESPACE/REPOSITORY:TAG

# ── Backend ──────────────────────────────────────────────────────────────────
docker tag todolist-backend:local \
  registry.cn-jakarta.aliyuncs.com/mycompany-myapp/backend:v1.0

docker push \
  registry.cn-jakarta.aliyuncs.com/mycompany-myapp/backend:v1.0

# ── Frontend ──────────────────────────────────────────────────────────────────
docker tag todolist-frontend:local \
  registry.cn-jakarta.aliyuncs.com/mycompany-myapp/frontend:v1.0

docker push \
  registry.cn-jakarta.aliyuncs.com/mycompany-myapp/frontend:v1.0
```

---

## Langkah 2: Update Deployment YAML

Edit `k8s/backend/deployment.yaml`:

```yaml
image: registry.cn-jakarta.aliyuncs.com/mycompany-myapp/backend:v1.0
imagePullPolicy: Always
```

Edit `k8s/frontend/deployment.yaml`:

```yaml
image: registry.cn-jakarta.aliyuncs.com/mycompany-myapp/frontend:v1.0
imagePullPolicy: Always
```

### Buat Image Pull Secret (Jika Repository Private)

Jika repository ACR kamu bersifat private, Kubernetes perlu credentials untuk pull image:

```bash
kubectl create secret docker-registry acr-credentials \
  --docker-server=registry.cn-jakarta.aliyuncs.com \
  --docker-username=YOUR_ALIBABA_ACCOUNT_EMAIL \
  --docker-password=YOUR_CONSOLE_PASSWORD \
  -n myapp
```

Kemudian tambahkan di deployment.yaml:

```yaml
spec:
  imagePullSecrets:
    - name: acr-credentials
  containers:
    - name: backend
      image: registry.cn-jakarta.aliyuncs.com/mycompany-myapp/backend:v1.0
```

---

## Langkah 3: Buat ACK Cluster (Jika Belum Ada)

### Melalui Alibaba Cloud Console

1. Buka [cs.console.aliyun.com](https://cs.console.aliyun.com)
2. Klik **Create Cluster** → **Standard Managed Cluster**
3. Konfigurasi:
   - **Cluster Name:** `myapp-cluster`
   - **Region:** Indonesia (Jakarta)
   - **Kubernetes Version:** 1.28+
   - **Node Type:** ecs.c6.xlarge (4 vCPU, 8 GB RAM) — minimum untuk belajar
   - **Node Count:** 2
   - **Container Runtime:** containerd
4. Klik **Create Cluster** (proses 10-15 menit)

### Download Kubeconfig

```bash
# Setelah cluster siap, download kubeconfig dari Console
# Atau menggunakan CLI:
aliyun cs GET /k8s/CLUSTER_ID/user_config --output yaml > ~/.kube/config-ack

# Set kubeconfig
export KUBECONFIG=~/.kube/config-ack

# Verifikasi
kubectl cluster-info
kubectl get nodes
```

---

## Langkah 4: Install Nginx Ingress Controller

ACK menyediakan Nginx Ingress Controller sebagai addon, atau kita bisa install manual:

### Melalui ACK Console (Direkomendasikan)

1. Di Console ACK, buka cluster kamu
2. Klik **Add-ons** → **Nginx Ingress Controller**
3. Klik Install

### Atau via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2

# Tunggu External IP dari Alibaba SLB (Server Load Balancer) tersedia
kubectl get svc -n ingress-nginx -w
# EXTERNAL-IP akan berisi IP publik SLB Alibaba
```

---

## Langkah 5: Deploy Aplikasi ke ACK

```bash
# Buat namespace
kubectl create namespace myapp

# Buat Secrets
kubectl create secret generic backend-secret \
  --from-literal=DB_USER=todouser \
  --from-literal=DB_PASS=P@ssw0rd123! \
  -n myapp

kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=todouser \
  --from-literal=POSTGRES_PASSWORD=P@ssw0rd123! \
  --from-literal=POSTGRES_DB=tododb \
  -n myapp

kubectl create configmap backend-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=tododb \
  -n myapp

# Deploy semua resources
kubectl apply -f k8s/database/ -n myapp
kubectl wait --for=condition=ready pod -l app=postgres -n myapp --timeout=120s

kubectl apply -f k8s/backend/ -n myapp
kubectl wait --for=condition=ready pod -l app=backend -n myapp --timeout=60s

kubectl apply -f k8s/frontend/ -n myapp
kubectl wait --for=condition=ready pod -l app=frontend -n myapp --timeout=60s

kubectl apply -f k8s/ingress/ -n myapp
```

---

## Langkah 6: Konfigurasi Alibaba SLB untuk Ingress

Saat Nginx Ingress di-install, Alibaba akan otomatis membuat **Server Load Balancer (SLB)**:

```bash
# Ambil External IP dari SLB
EXTERNAL_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "External IP: $EXTERNAL_IP"

# Test akses
curl http://$EXTERNAL_IP/api/health
# Output: {"status":"healthy","service":"todo-backend"}

# Buka browser
open http://$EXTERNAL_IP
```

---

## Menggunakan Alibaba ApsaraDB for RDS (Rekomendasi Production)

Untuk production, gunakan **ApsaraDB RDS for PostgreSQL** alih-alih menjalankan PostgreSQL di Pod:

1. Buka [rdsnext.console.aliyun.com](https://rdsnext.console.aliyun.com)
2. Buat instance **RDS for PostgreSQL**:
   - Region: Indonesia (Jakarta)
   - Engine: PostgreSQL 16
   - Specification: 1 vCPU, 2 GB RAM (untuk belajar)
3. Buat database `tododb` dan user `todouser`
4. Whitelist IP Node ACK agar bisa mengakses RDS
5. Update `DB_HOST` di ConfigMap dengan endpoint RDS

---

## Menggunakan Alibaba Cloud Disk sebagai StorageClass

Untuk production, konfigurasi StorageClass yang menggunakan Alibaba Cloud Disk:

```yaml
# Tambahkan di statefulset.yaml
spec:
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: alicloud-disk-ssd   # StorageClass Alibaba
        resources:
          requests:
            storage: 20Gi
```

StorageClass yang tersedia di ACK:
- `alicloud-disk-ssd` — SSD Cloud Disk (direkomendasikan untuk database)
- `alicloud-disk-efficiency` — Basic Cloud Disk (lebih murah, untuk non-kritis)
- `alicloud-disk-essd` — Enhanced SSD (performa tertinggi)

---

## Cleanup (PENTING)

```bash
# Hapus aplikasi
kubectl delete namespace myapp

# Hapus Ingress Controller (ini yang membuat SLB berbayar!)
helm uninstall ingress-nginx -n ingress-nginx

# Hapus ACK cluster melalui Console:
# cs.console.aliyun.com → pilih cluster → Delete Cluster
# ATAU via CLI:
aliyun cs DELETE /clusters/CLUSTER_ID
```

> **Perhatian:** SLB Alibaba **terus dikenakan biaya** meski tidak ada traffic. Pastikan selalu hapus SLB setelah selesai belajar. Cek di [slb.console.aliyun.com](https://slb.console.aliyun.com) untuk memastikan tidak ada SLB yang tertinggal.

> **Tips:** Gunakan **Alibaba Cloud Budget Alert** untuk mendapatkan notifikasi jika biaya melebihi threshold yang kamu set.
