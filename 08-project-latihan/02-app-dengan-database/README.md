# Project 02: Aplikasi dengan Database

Project ini mensimulasikan arsitektur nyata: frontend React + backend Node.js API + database PostgreSQL.

---

## Arsitektur

```
                    Browser / Client
                          │
                          │ :30090 (frontend) | :30091 (backend API)
                          ▼
                ┌─────────────────────────────┐
                │         NodePort Services    │
                │  frontend-svc  backend-svc   │
                └──────────┬─────────┬─────────┘
                           │         │
                    ┌──────▼──┐  ┌───▼──────┐
                    │Frontend │  │ Backend  │
                    │(nginx)  │  │(node.js) │
                    │ x2 Pod  │  │ x2 Pod   │
                    └─────────┘  └────┬─────┘
                                      │ ClusterIP
                                      │ (postgres-service:5432)
                               ┌──────▼──────┐
                               │  PostgreSQL  │
                               │  (StatefulSet│
                               │  1 replica)  │
                               │ + PVC (10Gi) │
                               └─────────────┘

Konfigurasi:
├── ConfigMap: database host, port, name
└── Secret: database password
```

---

## File yang Diperlukan

| File | Isi |
|------|-----|
| `configmap-secret.yaml` | Konfigurasi dan credentials |
| `database.yaml` | PostgreSQL (StatefulSet + Service + PVC) |
| `backend.yaml` | Backend API (Deployment + Service) |
| `frontend.yaml` | Frontend (Deployment + Service) |

---

## Langkah-langkah Deploy

### Step 1: Buat semua resource (urutan penting!)

```bash
# 1. Buat konfigurasi dan secret dulu
kubectl apply -f configmap-secret.yaml

# 2. Deploy database
kubectl apply -f database.yaml

# 3. Tunggu database siap
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=120s

# 4. Deploy backend
kubectl apply -f backend.yaml

# 5. Deploy frontend
kubectl apply -f frontend.yaml
```

### Step 2: Atau deploy semua sekaligus

```bash
kubectl apply -f .
# (apply semua file di direktori saat ini)

# Tunggu semua Pod running
kubectl get pods -w
```

### Step 3: Verifikasi setiap komponen

```bash
# Database
kubectl get statefulset postgres
kubectl get pods -l app=postgres
kubectl logs postgres-0

# Backend
kubectl get deployment backend-deployment
kubectl get pods -l app=backend

# Frontend
kubectl get deployment frontend-deployment
kubectl get pods -l app=frontend

# Services
kubectl get services
```

### Step 4: Test koneksi antar service

```bash
# Masuk ke Pod backend, test koneksi ke database
BACKEND_POD=$(kubectl get pods -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $BACKEND_POD -- /bin/sh

# Di dalam Pod:
# env | grep DB    ← lihat environment variables database
# nc -z postgres-service 5432 && echo "DB connected!" ← test koneksi
# exit
```

### Step 5: Akses dari luar

```bash
# Frontend
minikube service frontend-service --url
# Buka URL di browser

# Backend API
API_URL=$(minikube service backend-service --url)
curl $API_URL/api/health
curl $API_URL/api/users

# Atau dengan port-forward
kubectl port-forward service/frontend-service 3000:80 &
kubectl port-forward service/backend-service 8080:3000 &

curl http://localhost:3000
curl http://localhost:8080/api/health
```

---

## Verifikasi Database

```bash
# Masuk ke PostgreSQL
kubectl exec -it postgres-0 -- psql -U appuser -d myappdb

# Di dalam psql:
# \l          ← list databases
# \dt         ← list tables
# SELECT * FROM users LIMIT 5;
# \q          ← keluar
```

---

## Checklist Verifikasi

- [ ] PostgreSQL Pod berstatus Running
- [ ] Backend Pod berstatus Running
- [ ] Frontend Pod berstatus Running
- [ ] Backend bisa konek ke database (cek logs)
- [ ] Frontend bisa diakses di browser
- [ ] API backend merespons request
- [ ] Data tersimpan di database setelah restart backend (karena PVC)

---

## Test Persistensi Data

```bash
# 1. Insert data via API atau masuk ke database
kubectl exec -it postgres-0 -- psql -U appuser -d myappdb -c \
  "INSERT INTO users (name, email) VALUES ('Test User', 'test@example.com');"

# 2. Restart backend Pod
kubectl rollout restart deployment/backend-deployment

# 3. Cek data masih ada
kubectl exec -it postgres-0 -- psql -U appuser -d myappdb -c \
  "SELECT * FROM users;"
# Data harus masih ada!

# 4. Bahkan hapus Pod PostgreSQL pun data tetap ada (karena PVC)
kubectl delete pod postgres-0
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=60s
kubectl exec -it postgres-0 -- psql -U appuser -d myappdb -c \
  "SELECT * FROM users;"
# Data masih ada!
```

---

## Cleanup

```bash
kubectl delete -f .
# Atau
kubectl delete deployment frontend-deployment backend-deployment
kubectl delete statefulset postgres
kubectl delete service frontend-service backend-service postgres-service
kubectl delete configmap app-config
kubectl delete secret app-secret
kubectl delete pvc postgres-data-postgres-0

# Cek tidak ada yang tersisa
kubectl get all
kubectl get pvc
```

---

*[Lanjut ke: Project 03 - Fullstack dengan Ingress →](../03-fullstack-dengan-ingress/README.md)*

*[Kembali ke: Project Latihan](../README.md)*
