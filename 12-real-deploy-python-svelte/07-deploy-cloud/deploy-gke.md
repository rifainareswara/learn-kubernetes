# Deploy ke Google Kubernetes Engine (GKE)

> **Estimasi Waktu:** 45-60 menit (termasuk pembuatan cluster jika belum ada)
>
> **Prasyarat:** Akun Google Cloud, `gcloud` CLI terinstall, billing aktif

---

## Persiapan Google Cloud

```bash
# Login ke Google Cloud
gcloud auth login

# Set project (ganti PROJECT_ID dengan ID project GCP kamu)
gcloud config set project PROJECT_ID

# Aktifkan API yang diperlukan
gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com

# Verifikasi
gcloud config list
```

---

## Langkah 1: Push Image ke Google Artifact Registry (GAR)

Google Artifact Registry adalah penerus Google Container Registry (GCR).

### 1.1 Buat Repository di Artifact Registry

```bash
# Buat repository untuk menyimpan Docker images
gcloud artifacts repositories create myapp \
  --repository-format=docker \
  --location=asia-southeast1 \
  --description="Todo App container images"

# Verifikasi repository dibuat
gcloud artifacts repositories list --location=asia-southeast1
```

### 1.2 Konfigurasi Docker untuk Push ke GAR

```bash
# Configure Docker credentials untuk region Asia Tenggara (Jakarta)
gcloud auth configure-docker asia-southeast1-docker.pkg.dev

# Verifikasi konfigurasi
cat ~/.docker/config.json | grep asia-southeast1
```

### 1.3 Tag dan Push Image

```bash
# Pastikan kamu sudah build image (lihat langkah-deploy.md)
# Format: REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG

# ── Backend ──────────────────────────────────────────────────────────────────
docker tag todolist-backend:local \
  asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/backend:v1.0

docker push \
  asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/backend:v1.0

# ── Frontend ──────────────────────────────────────────────────────────────────
docker tag todolist-frontend:local \
  asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/frontend:v1.0

docker push \
  asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/frontend:v1.0

# Verifikasi images sudah ter-push
gcloud artifacts docker images list \
  asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp
```

---

## Langkah 2: Update Deployment YAML

Edit `k8s/backend/deployment.yaml` — ubah baris `image`:

```yaml
# Sebelum:
image: your-registry/backend:latest

# Sesudah (ganti PROJECT_ID):
image: asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/backend:v1.0
imagePullPolicy: Always
```

Edit `k8s/frontend/deployment.yaml`:

```yaml
# Sesudah (ganti PROJECT_ID):
image: asia-southeast1-docker.pkg.dev/PROJECT_ID/myapp/frontend:v1.0
imagePullPolicy: Always
```

> **Tips:** Di GKE, nodes secara otomatis memiliki akses ke Artifact Registry di project yang sama. Tidak perlu `imagePullSecrets` tambahan!

---

## Langkah 3: Buat GKE Cluster (Jika Belum Ada)

```bash
# Buat cluster GKE Autopilot (direkomendasikan, lebih mudah dikelola)
gcloud container clusters create-auto myapp-cluster \
  --location=asia-southeast1

# Atau buat cluster GKE Standard (lebih banyak kontrol)
gcloud container clusters create myapp-cluster \
  --zone=asia-southeast1-a \
  --num-nodes=2 \
  --machine-type=e2-medium \
  --disk-size=20GB

# Setelah cluster dibuat, konfigurasi kubectl
gcloud container clusters get-credentials myapp-cluster \
  --location=asia-southeast1

# Verifikasi koneksi
kubectl cluster-info
kubectl get nodes
```

---

## Langkah 4: Install Nginx Ingress Controller via Helm

```bash
# Tambah Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install Ingress Controller
# Di GKE, ini akan otomatis membuat Google Cloud Load Balancer
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2

# Tunggu sampai external IP tersedia (bisa 2-5 menit)
kubectl get svc -n ingress-nginx -w
# Tunggu kolom EXTERNAL-IP tidak lagi "<pending>"
# NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)
# ingress-nginx-controller             LoadBalancer   10.96.0.1     34.xxx.xxx.x   80:31234/TCP
```

---

## Langkah 5: Deploy Aplikasi ke GKE

Sama seperti deploy lokal, tapi tanpa step "load image":

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

## Langkah 6: Verifikasi

```bash
# Ambil External IP dari Ingress Controller
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

## Langkah 7: (Opsional) Konfigurasi Custom Domain

```bash
# 1. Buat DNS A record yang mengarah ke EXTERNAL_IP
#    Di domain registrar kamu: myapp.example.com → EXTERNAL_IP

# 2. Update ingress.yaml dengan domain kamu
#    Ubah bagian rules:
#    - host: myapp.example.com

# 3. Apply ulang ingress
kubectl apply -f k8s/ingress/ingress.yaml -n myapp

# 4. (Opsional) Install cert-manager untuk HTTPS otomatis
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s
```

---

## Cleanup (PENTING: Hindari Biaya)

```bash
# Hapus semua resource aplikasi
kubectl delete namespace myapp

# Hapus Ingress Controller (ini yang membuat Load Balancer berbayar!)
helm uninstall ingress-nginx -n ingress-nginx

# Hapus GKE cluster
gcloud container clusters delete myapp-cluster \
  --location=asia-southeast1

# Hapus Artifact Registry (opsional)
gcloud artifacts repositories delete myapp \
  --location=asia-southeast1
```

> **Perhatian:** Load Balancer GKE **terus dikenakan biaya** meski tidak ada traffic. Pastikan selalu cleanup setelah selesai belajar!

---

> **Tips:** Gunakan GKE Autopilot untuk menghindari biaya Node yang idle. Autopilot hanya mengenakan biaya untuk resource yang benar-benar digunakan Pod.
