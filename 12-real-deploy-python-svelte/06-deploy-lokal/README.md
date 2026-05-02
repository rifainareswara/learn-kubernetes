# 06 - Deploy Lokal

> **Estimasi Waktu:** 30 menit
>
> **Tujuan:** Menjalankan seluruh stack aplikasi di cluster Kubernetes lokal (Minikube atau Kind)

---

## Gambaran Umum

Di sesi ini kita akan menggabungkan semua yang sudah dipelajari dan menjalankan aplikasi Todo secara end-to-end di cluster lokal.

**Urutan deployment sangat penting:**

```
1. Namespace
   ↓
2. Secrets & ConfigMaps
   ↓
3. Database (StatefulSet PostgreSQL)
   ↓
4. Backend (Deployment FastAPI)  ← Butuh database sudah siap
   ↓
5. Frontend (Deployment Svelte)
   ↓
6. Ingress
```

Mengapa urutan ini? Backend memerlukan koneksi ke database saat startup. Jika database belum siap, backend akan crash-loop.

---

## Pilih Cluster Lokal

### Minikube
- Lebih mudah di-setup
- Built-in addon untuk Ingress
- Membutuhkan VM atau Docker

### Kind (Kubernetes in Docker)
- Lebih ringan
- Cocok untuk CI/CD pipeline
- Perlu konfigurasi ekstra untuk Ingress

---

## Panduan Lengkap

Lihat [langkah-deploy.md](./langkah-deploy.md) untuk panduan step-by-step yang lengkap.

---

## Quick Start (Ringkasan)

```bash
# 1. Build images
docker build -t todolist-backend:local ./backend
docker build --build-arg VITE_API_URL=/api -t todolist-frontend:local ./frontend

# 2. Load ke cluster lokal
minikube image load todolist-backend:local
minikube image load todolist-frontend:local

# 3. Deploy semua
kubectl create namespace todolist
kubectl apply -f k8s/database/ -n todolist
kubectl wait --for=condition=ready pod -l app=postgres -n todolist --timeout=90s
kubectl apply -f k8s/backend/ -n todolist
kubectl apply -f k8s/frontend/ -n todolist
kubectl apply -f k8s/ingress/ -n todolist

# 4. Buka tunnel (di terminal terpisah)
minikube tunnel

# 5. Buka browser
open http://localhost
```

---

## Selanjutnya

- [07-deploy-cloud/README.md](../07-deploy-cloud/README.md) — Deploy ke cloud (GKE, EKS, ACK)
