# 01 - Persiapan

> **Estimasi Waktu:** 15 menit
>
> **Tujuan:** Memahami apa yang akan dibangun dan menyiapkan struktur project

---

## Apa yang Akan Kita Bangun?

Kita akan membangun **aplikasi Todo List** — aplikasi sederhana yang memungkinkan pengguna untuk:

- Melihat daftar todos (`GET /api/todos`)
- Membuat todo baru (`POST /api/todos`)
- Mengupdate todo (tandai selesai) (`PUT /api/todos/{id}`)
- Menghapus todo (`DELETE /api/todos/{id}`)

Meski terlihat sederhana, aplikasi ini mencerminkan **pola arsitektur yang digunakan di production** — tiga tier (frontend, backend, database) yang masing-masing berjalan di container terpisah dan dikelola oleh Kubernetes.

---

## Arsitektur Teknis

```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                     │
│                                                         │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │   Svelte    │    │   FastAPI    │    │PostgreSQL │  │
│  │  (nginx)   │    │  (uvicorn)   │    │           │  │
│  │  Pod x2     │    │   Pod x2     │    │  Pod x1   │  │
│  └──────┬──────┘    └──────┬───────┘    └─────┬─────┘  │
│         │                  │                  │        │
│  ┌──────┴──────┐    ┌──────┴───────┐    ┌─────┴─────┐  │
│  │   Service   │    │   Service    │    │  Service  │  │
│  │  frontend   │    │   backend    │    │ postgres  │  │
│  └──────┬──────┘    └──────┬───────┘    └───────────┘  │
│         │                  │                            │
│         └────────┬─────────┘                           │
│                  │                                      │
│          ┌───────┴───────┐                             │
│          │    Ingress     │                             │
│          │  Controller    │                             │
│          └───────┬────────┘                            │
└──────────────────┼──────────────────────────────────────┘
                   │
                Internet
```

---

## Teknologi yang Digunakan

### Backend: FastAPI (Python)

**Kenapa FastAPI?**
- **Modern & async** — Mendukung async/await secara native, performa tinggi
- **Auto documentation** — Swagger UI otomatis tersedia di `/docs` dan `/redoc`
- **Type safety** — Menggunakan Python type hints dan Pydantic untuk validasi data
- **Populer di industri** — Banyak digunakan untuk microservices dan ML API

```python
# Contoh: endpoint FastAPI yang bersih dan mudah dibaca
@app.get("/todos", response_model=List[TodoResponse])
async def get_todos(db: Session = Depends(get_db)):
    return db.query(Todo).all()
```

### Frontend: Svelte

**Kenapa Svelte?**
- **Sangat ringan** — Dikompilasi ke vanilla JavaScript, tidak ada runtime overhead
- **Bundle size kecil** — Ideal untuk aplikasi yang di-serve via nginx di Kubernetes
- **Reactive secara native** — Tanpa perlu state management library tambahan
- **Mudah dipelajari** — Syntax yang bersih, cocok untuk pemula sekalipun

### Database: PostgreSQL

**Kenapa PostgreSQL?**
- **Database relasional paling populer** di dunia open-source
- **Dukungan JSON** — Bisa menyimpan data semi-struktural juga
- **Battle-tested** — Terbukti stabil untuk production workload
- **Image Docker resmi** — Tersedia di Docker Hub dengan konfigurasi yang mudah

---

## Environment yang Dibutuhkan

### Wajib

| Tool | Versi Minimum | Cek Instalasi |
|---|---|---|
| Docker | 24.x | `docker --version` |
| kubectl | 1.28+ | `kubectl version --client` |
| Kubernetes Cluster | 1.28+ | `kubectl cluster-info` |

### Pilihan (untuk cluster lokal)

| Tool | Kegunaan | Link |
|---|---|---|
| Minikube | Cluster Kubernetes lokal di VM | [minikube.sigs.k8s.io](https://minikube.sigs.k8s.io) |
| Kind | Cluster Kubernetes lokal di Docker | [kind.sigs.k8s.io](https://kind.sigs.k8s.io) |

### Verifikasi Setup

```bash
# 1. Cek Docker
docker --version
docker ps  # pastikan Docker daemon berjalan

# 2. Cek kubectl
kubectl version --client

# 3. Cek koneksi ke cluster
kubectl cluster-info
kubectl get nodes

# 4. Jika pakai Minikube - pastikan berjalan
minikube status
```

Output yang diharapkan dari `kubectl get nodes`:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   5d    v1.29.2
```

---

## Struktur Project Lengkap

Lihat file [struktur-project.md](./struktur-project.md) untuk detail kode aplikasi dan struktur folder yang direkomendasikan.

---

## Langkah Selanjutnya

Setelah memahami gambaran besar, lanjut ke:

1. [struktur-project.md](./struktur-project.md) — Pelajari kode aplikasi dan struktur folder
2. [02-backend-python/](../02-backend-python/README.md) — Dockerize dan deploy FastAPI
3. [03-frontend-svelte/](../03-frontend-svelte/README.md) — Dockerize dan deploy Svelte

---

> **Tips:** Baca semua README terlebih dahulu sebelum mulai menjalankan perintah. Memahami gambaran besar akan membantu kamu men-debug masalah lebih cepat.

> **Perhatian:** Jangan skip bagian persiapan. Banyak masalah deployment terjadi karena environment yang belum siap dengan benar.
