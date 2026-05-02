# 03 - Frontend Svelte

> **Estimasi Waktu:** 25 menit
>
> **Tujuan:** Dockerize aplikasi Svelte dengan multi-stage build dan deploy ke Kubernetes

---

## Konsep: Svelte + Nginx di Kubernetes

Aplikasi Svelte **dikompilasi menjadi file HTML, CSS, dan JavaScript statis** saat build time. Di production, kita tidak memerlukan Node.js untuk menjalankannya — cukup web server seperti **Nginx** untuk melayani file-file statis ini.

```
Proses Build:                    Container yang Berjalan di Kubernetes:
┌──────────────┐                 ┌──────────────────────────────────┐
│  Node.js     │                 │  Nginx                           │
│  npm run build│  ──▶ dist/ ──▶ │  /usr/share/nginx/html/          │
│  (kompilasi) │                 │  ├── index.html                  │
└──────────────┘                 │  ├── assets/                     │
                                 │  │   ├── index-abc123.js        │
                                 │  │   └── index-def456.css       │
                                 │  └── favicon.png                 │
                                 └──────────────────────────────────┘
```

---

## Build-time vs Runtime Environment Variables

Ini adalah konsep penting yang **sering membingungkan** pemula:

### Build-time Variables (Svelte/Vite)

Svelte menggunakan Vite sebagai bundler. Environment variables dengan prefix `VITE_` diinjeksikan ke dalam kode JavaScript **saat proses `npm run build`**, bukan saat container berjalan.

```javascript
// Di kode Svelte
const API_URL = import.meta.env.VITE_API_URL;
// Nilai ini sudah "di-bake" ke dalam file .js yang dikompilasi
```

**Implikasi:** Jika kamu ingin mengubah `VITE_API_URL`, kamu harus **build ulang Docker image**. Kamu tidak bisa mengubahnya via environment variable di Kubernetes Deployment.

### Cara Kita Mengatasinya

Kita set `VITE_API_URL=/api` saat build time:

```dockerfile
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build
```

Artinya frontend selalu memanggil `/api/todos` sebagai URL API. Di Kubernetes, Ingress bertugas meneruskan request `/api/*` ke Service backend FastAPI.

```
Browser ──▶ /api/todos ──▶ Ingress ──▶ backend Service ──▶ FastAPI Pod
```

---

## Konfigurasi Nginx untuk SPA

Single Page Application (SPA) seperti Svelte menggunakan **client-side routing**. Artinya, URL seperti `/todos/123` dikelola oleh JavaScript di browser, bukan oleh server.

**Masalah:** Jika pengguna mengakses langsung `http://todolist.com/todos/123`, Nginx akan mencari file `todos/123/index.html` yang tidak ada → 404 Not Found.

**Solusi:** Konfigurasi `try_files` di Nginx:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Cara kerjanya:
1. Coba cari file `$uri` (misalnya `/todos/123`)
2. Coba cari direktori `$uri/` (misalnya `/todos/123/`)
3. Jika tidak ada, berikan `/index.html` → biarkan JavaScript yang menangani routing

---

## File `nginx.conf` Lengkap

Buat file `frontend/nginx.conf`:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression untuk performa lebih baik
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        image/svg+xml;

    # SPA fallback: semua route diteruskan ke index.html
    # Ini penting agar client-side routing Svelte berfungsi dengan benar
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache aggressive untuk asset statis (Vite menambahkan hash ke nama file)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Jangan cache index.html (agar update ter-refresh dengan cepat)
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

> **Tips:** Simpan file `nginx.conf` ini di folder `frontend/` bersama Dockerfile. Saat Docker build, file ini akan disalin ke dalam image.

---

## File di Folder Ini

| File | Kegunaan |
|---|---|
| [Dockerfile](./Dockerfile) | Instruksi build Docker image frontend |
| [deployment.yaml](./deployment.yaml) | Kubernetes Deployment untuk Svelte/Nginx |
| [service.yaml](./service.yaml) | Kubernetes Service untuk expose frontend |

---

## Cara Build Docker Image

> [!NOTE]
> Semua perintah di bawah dijalankan dari **root folder project kamu** (`my-todo-app/`).
> Jika ingin langsung mencoba dengan kode referensi yang sudah ada, gunakan:
> `cd /path/ke/repo/12-real-deploy-python-svelte/app`

```bash
# Pastikan kamu berada di root project (my-todo-app/ atau app/)
pwd
# Output yang diharapkan: .../my-todo-app  ATAU  .../app

# Masuk ke folder frontend
cd frontend/

# Build image dengan default API URL (/api)
docker build -t todolist-frontend:local .

# Atau build dengan custom API URL (untuk testing)
docker build \
  --build-arg VITE_API_URL=http://localhost:8000 \
  -t todolist-frontend:local .

# Verifikasi image
docker images | grep frontend

# Test jalankan secara lokal
docker run -p 8080:80 todolist-frontend:local

# Buka browser
open http://localhost:8080
```

---

## Verifikasi Deployment

Setelah deploy ke Kubernetes, cek apakah Nginx berjalan dengan benar:

```bash
# Cek status Pod
kubectl get pods -n todolist -l app=frontend

# Lihat logs Nginx
kubectl logs -n todolist -l app=frontend

# Port-forward untuk test lokal (tanpa Ingress)
kubectl port-forward svc/frontend 8080:80 -n todolist
# Buka: http://localhost:8080
```

---

> **Perhatian:** Pastikan kamu juga membuat file `.dockerignore` di folder `frontend/`:
> ```
> node_modules/
> dist/
> .env
> .env.local
> ```
> Ini mencegah folder `node_modules` (yang bisa sangat besar) ikut ter-copy ke Docker build context, sehingga proses build jauh lebih cepat.

---

## Selanjutnya

- [04-database/README.md](../04-database/README.md) — Deploy PostgreSQL dengan StatefulSet
