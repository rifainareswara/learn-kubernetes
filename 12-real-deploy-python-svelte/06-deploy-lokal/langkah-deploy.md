# Panduan Deploy Lokal — Step by Step

> Panduan lengkap untuk menjalankan aplikasi Todo di Minikube atau Kind.

---

## Prasyarat

Sebelum mulai, pastikan:

```bash
# Minikube berjalan
minikube status
# atau Kind cluster ada
kubectl cluster-info

# Docker berjalan
docker ps

# kubectl terhubung ke cluster yang benar
kubectl config current-context
```

---

## Langkah 1: Build Docker Images

Masuk ke root direktori project kamu — ini adalah folder yang langsung berisi `backend/` dan `frontend/`:

```bash
# Jika menggunakan project kamu sendiri:
cd ~/my-todo-app

# Atau jika ingin langsung mencoba dengan kode referensi di repo ini:
cd /path/ke/repo/12-real-deploy-python-svelte/app

# Verifikasi kamu ada di tempat yang benar
ls
# Output yang diharapkan: backend/  frontend/  docker-compose.yml
```

Semua perintah build di bawah **dijalankan dari folder ini** (tidak perlu `cd` masuk-keluar):

```bash
# ── Build Backend (FastAPI) ───────────────────────────────────────────────────
docker build -t backend:local -f backend/Dockerfile backend/

# Verifikasi image berhasil dibuat
docker images | grep backend
# Output:
# backend   local   abc123def456   2 minutes ago   158MB


# ── Build Frontend (Svelte) ──────────────────────────────────────────────────
docker build \
  --build-arg VITE_API_URL=/api \
  -t frontend:local \
  -f frontend/Dockerfile frontend/

# Verifikasi
docker images | grep frontend
# Output:
# frontend  local   def456abc123   1 minute ago    27MB
```

> [!TIP]
> Flag `-f backend/Dockerfile backend/` artinya: gunakan Dockerfile dari `backend/`, dan set build context ke folder `backend/`. Cara ini lebih aman karena kamu tidak perlu bolak-balik `cd` masuk/keluar folder.

---

## Langkah 2: Load Images ke Cluster Lokal

Docker images yang ada di local machine kamu **tidak otomatis tersedia** di Minikube/Kind. Kita perlu "memasukkan" image tersebut ke dalam cluster.

### Untuk Minikube

```bash
# Load backend image
minikube image load backend:local

# Load frontend image
minikube image load frontend:local

# Verifikasi image sudah ada di Minikube
minikube image ls | grep -E "backend|frontend"
# Output:
# docker.io/library/backend:local
# docker.io/library/frontend:local
```

### Untuk Kind

```bash
# Load backend image
kind load docker-image backend:local --name <nama-cluster-kind-kamu>

# Load frontend image
kind load docker-image frontend:local --name <nama-cluster-kind-kamu>

# Cek nama cluster Kind kamu
kind get clusters

# Jika hanya ada satu cluster, tidak perlu --name
kind load docker-image backend:local
kind load docker-image frontend:local
```

---

## Langkah 3: Update Deployment YAML untuk Local Images

Saat menggunakan local image (bukan dari registry), kita perlu mengubah dua hal di file deployment:

1. **Nama image** → pakai nama local
2. **`imagePullPolicy`** → ubah ke `Never` (jangan pull dari registry)

Edit `k8s/backend/deployment.yaml`:

```yaml
# Ubah bagian ini:
containers:
  - name: backend
    image: backend:local        # Nama local image
    imagePullPolicy: Never      # Jangan pull dari registry
```

Edit `k8s/frontend/deployment.yaml`:

```yaml
# Ubah bagian ini:
containers:
  - name: frontend
    image: frontend:local       # Nama local image
    imagePullPolicy: Never      # Jangan pull dari registry
```

> **Perhatian:** Jangan lupa mengembalikan nilai ini sebelum deploy ke production! Atau buat file YAML terpisah untuk environment lokal.

---

## Langkah 4: Deploy Semua Resources

Ikuti urutan ini dengan tepat:

```bash
# ── Step 4.1: Buat Namespace ─────────────────────────────────────────────────
kubectl create namespace myapp

# Verifikasi namespace ada
kubectl get namespace myapp


# ── Step 4.2: Buat Secrets ───────────────────────────────────────────────────
# Secret untuk backend (koneksi ke database)
kubectl create secret generic backend-secret \
  --from-literal=DB_USER=todouser \
  --from-literal=DB_PASS=P@ssw0rd123! \
  -n myapp

# Secret untuk PostgreSQL (inisialisasi database)
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=todouser \
  --from-literal=POSTGRES_PASSWORD=P@ssw0rd123! \
  --from-literal=POSTGRES_DB=tododb \
  -n myapp

# Verifikasi secrets ada
kubectl get secrets -n myapp


# ── Step 4.3: Apply ConfigMap ─────────────────────────────────────────────────
kubectl apply -f k8s/backend/configmap-secret.yaml -n myapp
# Hanya apply ConfigMap dari file ini, Secret sudah dibuat di atas
# Atau buat ConfigMap terpisah:
kubectl create configmap backend-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=tododb \
  -n myapp


# ── Step 4.4: Deploy Database ─────────────────────────────────────────────────
kubectl apply -f k8s/database/statefulset.yaml -n myapp
kubectl apply -f k8s/database/service.yaml -n myapp

# Verifikasi database mulai berjalan
kubectl get pods -n myapp -l app=postgres
# Output awal (mungkin masih Pending atau ContainerCreating):
# NAME         READY   STATUS              RESTARTS   AGE
# postgres-0   0/1     ContainerCreating   0          10s


# ── Step 4.5: Tunggu Database Ready ──────────────────────────────────────────
echo "Menunggu PostgreSQL siap..."
kubectl wait --for=condition=ready pod \
  -l app=postgres \
  -n myapp \
  --timeout=120s

# Jika berhasil:
# pod/postgres-0 condition met

# Lihat logs untuk memastikan inisialisasi selesai
kubectl logs -n myapp postgres-0 | tail -5
# Output yang diharapkan:
# ...database system is ready to accept connections


# ── Step 4.6: Deploy Backend ──────────────────────────────────────────────────
kubectl apply -f k8s/backend/deployment.yaml -n myapp
kubectl apply -f k8s/backend/service.yaml -n myapp

# Tunggu backend ready
kubectl wait --for=condition=ready pod \
  -l app=backend \
  -n myapp \
  --timeout=60s

# Cek logs backend
kubectl logs -n myapp -l app=backend --tail=10
# Output yang diharapkan:
# INFO:     Started server process [1]
# INFO:     Waiting for application startup.
# INFO:     Application startup complete.
# INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)


# ── Step 4.7: Deploy Frontend ─────────────────────────────────────────────────
kubectl apply -f k8s/frontend/deployment.yaml -n myapp
kubectl apply -f k8s/frontend/service.yaml -n myapp

# Tunggu frontend ready
kubectl wait --for=condition=ready pod \
  -l app=frontend \
  -n myapp \
  --timeout=60s


# ── Step 4.8: Deploy Ingress ──────────────────────────────────────────────────
kubectl apply -f k8s/ingress/ingress.yaml -n myapp
```

---

## Langkah 5: Enable Ingress

### Untuk Minikube

```bash
# Enable addon Ingress
minikube addons enable ingress

# Tunggu Ingress Controller Pod ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Jalankan tunnel di terminal TERPISAH (biarkan berjalan)
minikube tunnel
# Jika diminta password, masukkan password admin kamu
```

> **Perhatian:** `minikube tunnel` harus tetap berjalan di terminal terpisah. Jangan tutup terminal tersebut saat testing!

### Untuk Kind

```bash
# Install Nginx Ingress Controller via kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Tunggu sampai ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

---

## Langkah 6: Verifikasi Deployment

```bash
# ── Cek semua resource ────────────────────────────────────────────────────────
kubectl get all -n myapp

# Output yang diharapkan:
# NAME                            READY   STATUS    RESTARTS   AGE
# pod/backend-7d4b9f8c6-abc12     1/1     Running   0          5m
# pod/backend-7d4b9f8c6-def34     1/1     Running   0          5m
# pod/frontend-6c5d8e7f5-ghi56    1/1     Running   0          4m
# pod/frontend-6c5d8e7f5-jkl78    1/1     Running   0          4m
# pod/postgres-0                  1/1     Running   0          8m
#
# NAME               TYPE        CLUSTER-IP       PORT(S)    AGE
# service/backend    ClusterIP   10.96.100.1      8000/TCP   5m
# service/frontend   ClusterIP   10.96.200.2      80/TCP     4m
# service/postgres   ClusterIP   10.96.150.3      5432/TCP   8m
#
# NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/backend   2/2     2            2           5m
# deployment.apps/frontend  2/2     2            2           4m
#
# NAME                                 READY   AGE
# statefulset.apps/postgres            1/1     8m


# ── Cek Ingress ───────────────────────────────────────────────────────────────
kubectl get ingress -n myapp
# Output:
# NAME             CLASS   HOSTS   ADDRESS        PORTS   AGE
# myapp-ingress    nginx   *       192.168.49.2   80      3m


# ── Test API langsung ─────────────────────────────────────────────────────────
# Test health endpoint backend
curl http://localhost/api/health
# Output: {"status":"healthy","service":"todo-backend"}

# Test get todos (harus kosong awalnya)
curl http://localhost/api/todos
# Output: []

# Test buat todo baru
curl -X POST http://localhost/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Belajar Kubernetes","description":"Modul 12 selesai!"}'
# Output: {"id":1,"title":"Belajar Kubernetes",...}

# Test get todos lagi (harus ada 1 item)
curl http://localhost/api/todos
# Output: [{"id":1,"title":"Belajar Kubernetes",...}]
```

```bash
# ── Buka di Browser ──────────────────────────────────────────────────────────
# Frontend Todo App
open http://localhost

# Swagger API Docs (untuk test API)
# (Perlu port-forward karena /docs tidak melalui Ingress)
kubectl port-forward svc/backend 8000:8000 -n myapp
open http://localhost:8000/docs
```

---

## Troubleshooting Umum

### Pod stuck di "Pending"

```bash
# Cek event untuk melihat penyebab
kubectl describe pod <nama-pod> -n myapp

# Kemungkinan penyebab:
# - Insufficient resources: naikkan resource request atau tambah Node
# - Image tidak ditemukan: pastikan sudah load image ke cluster
# - PVC tidak ter-bound: cek StorageClass tersedia
kubectl get pvc -n myapp
kubectl get storageclass
```

### Pod stuck di "CrashLoopBackOff"

```bash
# Lihat logs Pod yang crash
kubectl logs <nama-pod> -n myapp
kubectl logs <nama-pod> -n myapp --previous  # Log dari run sebelumnya

# Kemungkinan penyebab untuk backend:
# - Tidak bisa koneksi ke database: cek Secret dan Service postgres
# - Error import Python: cek requirements.txt dan Dockerfile

# Debug dengan masuk ke container
kubectl exec -it <nama-pod> -n myapp -- /bin/sh
```

### Backend tidak bisa koneksi ke database

```bash
# Verifikasi Service postgres ada
kubectl get svc postgres -n myapp

# Verifikasi Secret backend-secret ada dan benar
kubectl get secret backend-secret -n myapp -o jsonpath='{.data.DB_USER}' | base64 -d
kubectl get secret postgres-secret -n myapp -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
# Kedua output harus sama!

# Test koneksi dari dalam Pod backend
kubectl exec -it deployment/backend -n myapp -- sh -c \
  "python -c \"import psycopg2; conn = psycopg2.connect(host='postgres', dbname='tododb', user='todouser', password='P@ssw0rd123!'); print('Koneksi berhasil!')\""
```

### Ingress tidak merespons (404/502)

```bash
# Cek Ingress Controller berjalan
kubectl get pods -n ingress-nginx

# Cek Ingress configuration
kubectl describe ingress myapp-ingress -n myapp

# Cek Service endpoints
kubectl get endpoints -n myapp

# Cek logs Ingress Controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20
```

### Frontend tidak bisa memanggil API (CORS error)

```bash
# Verifikasi Ingress routing dengan curl
curl -v http://localhost/api/health

# Cek apakah backend menerima request
kubectl logs -n myapp -l app=backend --tail=20

# Pastikan path /api/ diteruskan dengan benar (bukan /api/api/)
```

---

## Cleanup

Setelah selesai belajar, bersihkan resources:

```bash
# Hapus semua resource dalam namespace myapp
kubectl delete namespace myapp

# Verifikasi namespace sudah terhapus
kubectl get namespace myapp
# Error from server (NotFound): namespaces "myapp" not found

# Hentikan Minikube tunnel (Ctrl+C di terminal tersebut)

# Opsional: hapus Minikube cluster jika tidak diperlukan lagi
# minikube delete

# Hapus local Docker images
docker rmi backend:local frontend:local
```

---

> **Tips:** Gunakan `kubectl get events -n myapp --sort-by='.lastTimestamp'` untuk melihat urutan kejadian di namespace. Sangat berguna untuk debugging masalah yang terjadi saat startup.

> **Tips:** Buat alias untuk perintah yang sering digunakan:
> ```bash
> alias k='kubectl'
> alias kn='kubectl -n myapp'
> ```

---

## Selanjutnya

- [07-deploy-cloud/README.md](../07-deploy-cloud/README.md) — Deploy aplikasi yang sama ke cloud
