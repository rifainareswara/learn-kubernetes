# Project Nyata: Deploy Aplikasi Python + Svelte ke Kubernetes

> **Estimasi Waktu:** 2-3 jam (pertama kali), 30-45 menit (setelah familiar)
>
> **Level:** Menengah — Pastikan kamu sudah menyelesaikan modul 01–11 sebelumnya.

---

## Deskripsi Project

Pada modul ini, kita akan men-deploy **aplikasi Todo List** lengkap ke Kubernetes. Aplikasi ini terdiri dari tiga komponen utama:

| Komponen | Teknologi | Peran |
|---|---|---|
| Backend | Python FastAPI | REST API untuk CRUD todos |
| Frontend | Svelte + Nginx | Antarmuka pengguna (SPA) |
| Database | PostgreSQL | Penyimpanan data persisten |

Ini adalah simulasi deployment aplikasi **production-ready** yang nyata — bukan contoh `hello-world` lagi. Kamu akan belajar bagaimana semua komponen saling terhubung di dalam sebuah Kubernetes cluster.

---

## Arsitektur Aplikasi

```
Internet
   │
   ▼
[Ingress Controller]
   │
   ├──/api/*──▶ [Service: backend] ──▶ [Pod: FastAPI] ──▶ [Service: postgres] ──▶ [Pod: PostgreSQL]
   │
   └──/*──────▶ [Service: frontend] ──▶ [Pod: Svelte (nginx)]
```

### Penjelasan Alur:
1. **Pengguna** membuka browser → diarahkan ke Ingress Controller
2. **Ingress** mengecek path URL:
   - Jika path dimulai dengan `/api/*` → diteruskan ke Service backend (FastAPI)
   - Jika path lainnya `/*` → diteruskan ke Service frontend (Svelte)
3. **FastAPI** memproses request, mengakses PostgreSQL jika perlu data
4. **PostgreSQL** menyimpan data di Persistent Volume (data tidak hilang saat Pod restart)

---

## Kenapa Arsitektur Ini?

- **Satu domain, dua service** — Ingress menghindarkan masalah CORS karena frontend dan backend berada di domain yang sama
- **StatefulSet untuk database** — Menjamin urutan Pod dan nama yang konsisten
- **ConfigMap + Secret** — Memisahkan konfigurasi dari kode
- **Resource limits** — Mencegah satu Pod memakan semua resource cluster

---

## Prerequisites

Pastikan kamu sudah memiliki:

- [ ] Docker terinstall dan berjalan
- [ ] `kubectl` terkonfigurasi dan terhubung ke cluster
- [ ] Cluster Kubernetes berjalan (Minikube, Kind, atau cloud)
- [ ] Pemahaman dasar Kubernetes (Deployment, Service, ConfigMap, Secret)

```bash
# Verifikasi setup kamu
docker --version
kubectl version --client
kubectl cluster-info
```

---

## Yang Akan Kamu Pelajari

Setelah menyelesaikan modul ini, kamu akan mampu:

- Dockerize aplikasi Python FastAPI dengan multi-stage build
- Dockerize aplikasi Svelte dengan Nginx sebagai web server
- Deploy database PostgreSQL menggunakan StatefulSet
- Mengelola konfigurasi dengan ConfigMap dan Secret
- Mengatur routing dengan Nginx Ingress Controller
- Deploy aplikasi ke lingkungan lokal (Minikube/Kind)
- Deploy aplikasi ke cloud (GKE, EKS, ACK)
- Membuat pipeline CI/CD dengan GitHub Actions

---

## Struktur Modul

```
12-real-deploy-python-svelte/
├── README.md                    ← Kamu ada di sini
├── app/                         ← Kode aplikasi yang bisa langsung dijalankan
│   ├── docker-compose.yml       ← Jalankan semua sekaligus (tanpa Kubernetes)
│   ├── backend/                 ← Kode Python FastAPI
│   │   ├── main.py              ← CRUD todos (siap pakai)
│   │   ├── requirements.txt
│   │   ├── .env.example         ← Template environment variable
│   │   └── Dockerfile
│   └── frontend/                ← Kode Svelte
│       ├── public/              ← Static assets
│       ├── src/
│       │   ├── App.svelte       ← UI todo (tambah, centang, hapus)
│       │   ├── main.js          ← Entry point Svelte
│       │   └── lib/
│       │       └── api.js       ← Helper fetch ke backend
│       ├── index.html
│       ├── vite.config.js
│       ├── package.json
│       ├── nginx.conf           ← Konfigurasi Nginx untuk serve SPA
│       └── Dockerfile
├── 01-persiapan/                ← Persiapan & struktur project
├── 02-backend-python/           ← Dockerize & deploy FastAPI
├── 03-frontend-svelte/          ← Dockerize & deploy Svelte
├── 04-database/                 ← Deploy PostgreSQL di Kubernetes
├── 05-ingress/                  ← Routing dengan Ingress
├── 06-deploy-lokal/             ← Panduan deploy di Minikube/Kind
├── 07-deploy-cloud/             ← Panduan deploy di GKE, EKS, ACK
└── 08-cicd/                     ← Otomatisasi dengan GitHub Actions
```

---

## Navigasi

| No | Folder | Topik | Estimasi |
|---|---|---|---|
| 01 | [01-persiapan/](./01-persiapan/README.md) | Persiapan & struktur project | 15 menit |
| 02 | [02-backend-python/](./02-backend-python/README.md) | Backend FastAPI | 30 menit |
| 03 | [03-frontend-svelte/](./03-frontend-svelte/README.md) | Frontend Svelte | 25 menit |
| 04 | [04-database/](./04-database/README.md) | Database PostgreSQL | 20 menit |
| 05 | [05-ingress/](./05-ingress/README.md) | Ingress & Routing | 20 menit |
| 06 | [06-deploy-lokal/](./06-deploy-lokal/README.md) | Deploy Lokal | 30 menit |
| 07 | [07-deploy-cloud/](./07-deploy-cloud/README.md) | Deploy ke Cloud | 45 menit |
| 08 | [08-cicd/](./08-cicd/README.md) | CI/CD Pipeline | 30 menit |

---

> **Tips:** Mulai dari [01-persiapan](./01-persiapan/README.md) dan ikuti urutan folder. Setiap folder memiliki README sendiri yang menjelaskan konsep sebelum masuk ke file konfigurasi.

> **Perhatian:** Ganti semua placeholder seperti `your-registry`, `PROJECT_ID`, `ACCOUNT_ID` dengan nilai yang sesuai dengan environment kamu.
