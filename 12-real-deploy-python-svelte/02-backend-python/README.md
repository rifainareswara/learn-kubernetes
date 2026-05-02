# 02 - Backend Python (FastAPI)

> **Estimasi Waktu:** 30 menit
>
> **Tujuan:** Dockerize aplikasi FastAPI dan deploy ke Kubernetes dengan konfigurasi production-ready

---

## Konsep: Mengapa Multi-Stage Build?

Saat membangun Docker image untuk Python, kita bisa langsung install semua dependensi dan jalankan aplikasi. Tapi pendekatan ini menghasilkan image yang **besar** karena menyertakan tool build yang tidak diperlukan saat runtime.

**Multi-stage build** memisahkan proses:
1. **Stage builder** — Install dependensi (butuh pip, compiler, dll)
2. **Stage runtime** — Hanya berisi aplikasi dan dependensi yang sudah terinstall

```
Stage builder (python:3.12-slim)     Stage runtime (python:3.12-slim)
┌─────────────────────────────┐       ┌───────────────────────────────┐
│  pip install requirements   │  ──▶  │  Salin dari builder           │
│  (semua tool build tersedia)│       │  (hanya hasil install)        │
│  Size: ~500MB               │       │  Size: ~150MB                 │
└─────────────────────────────┘       └───────────────────────────────┘
                                       ↑
                                   Yang di-push ke registry
                                   dan berjalan di Kubernetes
```

---

## Environment Variables

Backend FastAPI kita menggunakan environment variables berikut untuk koneksi database:

| Variable | Contoh Nilai | Sumber |
|---|---|---|
| `DB_HOST` | `postgres` | ConfigMap |
| `DB_PORT` | `5432` | ConfigMap |
| `DB_NAME` | `tododb` | ConfigMap |
| `DB_USER` | `todouser` | Secret |
| `DB_PASS` | `p@ssw0rd` | Secret |

> **Perhatian:** `DB_HOST` diisi dengan **nama Service** PostgreSQL di Kubernetes (bukan IP address). Kubernetes DNS akan me-resolve nama service ini ke IP yang benar secara otomatis.

---

## Health Check Endpoint

Kubernetes memerlukan cara untuk mengetahui apakah Pod kita **siap menerima traffic** (readiness) dan apakah Pod **masih hidup** (liveness). FastAPI kita memiliki endpoint `/health` yang digunakan untuk kedua probe ini.

```python
@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "todo-backend"}
```

**Perbedaan readinessProbe vs livenessProbe:**

| Probe | Kapan Gagal? | Efek Kegagalan |
|---|---|---|
| `readinessProbe` | App belum siap | Pod dikeluarkan dari load balancer |
| `livenessProbe` | App hang/crash | Pod di-restart |

---

## Konfigurasi CORS

Karena frontend dan backend kita berjalan di domain yang sama (via Ingress), CORS seharusnya tidak menjadi masalah. Tapi kita tetap mengkonfigurasinya untuk:
- Keamanan (memfilter origin yang diizinkan)
- Kompatibilitas saat development lokal (frontend di port 5173, backend di port 8000)

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],          # Development: izinkan semua
    # allow_origins=["https://myapp.com"],  # Production: spesifik
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> **Tips untuk Production:** Ganti `allow_origins=["*"]` dengan domain spesifik aplikasimu. Misalnya: `allow_origins=["https://myapp.com", "https://www.myapp.com"]`

---

## File di Folder Ini

| File | Kegunaan |
|---|---|
| [Dockerfile](./Dockerfile) | Instruksi build Docker image backend |
| [deployment.yaml](./deployment.yaml) | Kubernetes Deployment untuk FastAPI |
| [service.yaml](./service.yaml) | Kubernetes Service untuk expose backend |
| [configmap-secret.yaml](./configmap-secret.yaml) | Konfigurasi dan kredensial database |

---

## Cara Build Docker Image

```bash
# Masuk ke folder backend
cd backend/

# Build image
docker build -t backend:local .

# Verifikasi image berhasil dibuat
docker images | grep backend

# Test jalankan secara lokal (butuh PostgreSQL berjalan)
docker run -p 8000:8000 \
  -e DB_HOST=host.docker.internal \
  -e DB_PORT=5432 \
  -e DB_NAME=tododb \
  -e DB_USER=postgres \
  -e DB_PASS=postgres \
  backend:local

# Cek health endpoint
curl http://localhost:8000/health
# Output: {"status":"healthy","service":"todo-backend"}

# Cek Swagger docs
open http://localhost:8000/docs
```

---

## Cara Buat Secret (DB_USER & DB_PASS)

Kubernetes Secret menyimpan nilai dalam format **base64**. Bukan enkripsi, hanya encoding. Gunakan cara berikut untuk generate value base64:

```bash
# Encode username
echo -n "todouser" | base64
# Output: dG9kb3VzZXI=

# Encode password
echo -n "P@ssw0rd123" | base64
# Output: UEBzc3cwcmQxMjM=

# Verifikasi (decode kembali)
echo "dG9kb3VzZXI=" | base64 -d
# Output: todouser
```

> **Perhatian:** Flag `-n` pada `echo` penting — tanpanya, newline character `\n` ikut ter-encode dan menyebabkan error koneksi database yang sulit di-debug.

---

## Selanjutnya

- [03-frontend-svelte/README.md](../03-frontend-svelte/README.md) — Dockerize dan deploy Svelte
- [04-database/README.md](../04-database/README.md) — Deploy PostgreSQL

---

> **Tips:** Setelah membuat Secret, verifikasi nilainya dengan: `kubectl get secret backend-secret -n myapp -o jsonpath='{.data.DB_USER}' | base64 -d`
