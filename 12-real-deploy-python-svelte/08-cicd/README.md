# 08 - CI/CD Pipeline dengan GitHub Actions

> **Estimasi Waktu:** 30 menit
>
> **Tujuan:** Mengotomasi proses build, test, dan deploy aplikasi ke Kubernetes menggunakan GitHub Actions

---

## Apa itu CI/CD?

**CI (Continuous Integration):**
Setiap kali developer push code ke repository, pipeline otomatis berjalan untuk:
- Menjalankan test
- Build Docker image
- Memastikan tidak ada yang "broken"

**CD (Continuous Deployment):**
Jika CI berhasil, pipeline otomatis:
- Push Docker image ke registry
- Update deployment di Kubernetes
- Aplikasi ter-update tanpa intervensi manual

```
Developer push code
    │
    ▼
GitHub Actions triggered
    │
    ├── Job: test
    │   ├── Run Python unit tests
    │   └── Run Svelte build check
    │
    ├── Job: build-push (jika test berhasil)
    │   ├── Build backend Docker image
    │   ├── Build frontend Docker image
    │   └── Push ke Container Registry
    │
    └── Job: deploy (jika build berhasil)
        ├── Update image tag di deployment YAML
        └── kubectl apply ke cluster
```

---

## Secrets yang Diperlukan di GitHub

Sebelum pipeline bisa berjalan, kamu perlu mengkonfigurasi secrets di repository GitHub:

Masuk ke: **Repository** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret Name | Keterangan | Contoh Nilai |
|---|---|---|
| `REGISTRY_URL` | URL container registry | `asia-southeast1-docker.pkg.dev/project-id/myapp` |
| `REGISTRY_USERNAME` | Username registry (jika diperlukan) | `_json_key` (untuk GCR) atau `AWS` (untuk ECR) |
| `REGISTRY_PASSWORD` | Password/token registry | Service account JSON atau ECR token |
| `KUBE_CONFIG` | Kubeconfig dalam format base64 | `$(cat ~/.kube/config \| base64)` |
| `DB_USER` | Username database (untuk Kubernetes Secret) | `todouser` |
| `DB_PASS` | Password database (untuk Kubernetes Secret) | `P@ssw0rd123!` |

### Cara Generate KUBE_CONFIG Secret

```bash
# Encode kubeconfig ke base64
cat ~/.kube/config | base64 | tr -d '\n'
# Salin output ini dan paste sebagai nilai secret KUBE_CONFIG
```

> **Perhatian:** Jangan pernah commit `KUBE_CONFIG` atau credentials langsung ke repository Git!

---

## File Pipeline

| File | Kegunaan |
|---|---|
| [github-actions.yaml](./github-actions.yaml) | Workflow CI/CD lengkap |

---

## Alur Kerja Pipeline

### Trigger Otomatis
- Push ke branch `main` → jalankan semua jobs (test + build + deploy)
- Push ke branch lain (feature branches) → hanya jalankan job `test`
- Pull Request → jalankan job `test` saja

### Job 1: Test
- **Backend:** Jalankan `pytest` untuk Python unit tests
- **Frontend:** Jalankan `npm run build` untuk cek Svelte compile error

### Job 2: Build & Push
- Build Docker image dengan tag unik (menggunakan git commit SHA)
- Push ke container registry
- Berjalan paralel untuk backend dan frontend

### Job 3: Deploy
- Download kubeconfig dari GitHub Secret
- Jalankan `kubectl apply` untuk update deployment
- Atau update image tag menggunakan `kubectl set image`

---

## Monitoring Pipeline

Setelah setup, kamu bisa memantau status pipeline di:
- Tab **Actions** di repository GitHub
- Klik workflow run untuk melihat detail setiap job dan step

```
✅ test (2m 15s)
  ✅ test-backend
  ✅ test-frontend

✅ build-push (5m 43s)
  ✅ build-backend
  ✅ build-frontend

✅ deploy (1m 12s)
  ✅ deploy-to-k8s
```

---

> **Tips:** Mulai dengan pipeline yang sederhana (hanya build dan push), kemudian tambahkan test dan deployment secara bertahap. Lebih baik punya pipeline yang berjalan daripada pipeline yang sempurna tapi tidak pernah selesai dibuat.

> **Perhatian:** Pastikan credentials di GitHub Secrets selalu up-to-date. Token yang kedaluwarsa adalah penyebab pipeline failure yang umum.

---

## Selanjutnya

Selamat! Kamu telah menyelesaikan modul **Project Nyata: Deploy Aplikasi Python + Svelte ke Kubernetes**.

Kembali ke [README utama](../README.md) untuk ringkasan semua yang telah dipelajari.
