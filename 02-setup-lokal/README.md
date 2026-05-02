# 02 - Setup Lokal Kubernetes

Di modul ini kamu akan menyiapkan environment Kubernetes lokal di laptopmu sehingga bisa mulai praktik.

---

## Tujuan Pembelajaran

Setelah modul ini, kamu akan:
- Punya `kubectl` terinstall dan terkonfigurasi
- Punya cluster Kubernetes lokal yang berjalan (Minikube atau Kind)
- Bisa menjalankan perintah-perintah kubectl dasar

---

## Pilih Tools yang Sesuai

Ada beberapa pilihan untuk menjalankan Kubernetes lokal:

| Tools | Cocok untuk | Kebutuhan Resource | Kemudahan |
|-------|------------|-------------------|-----------|
| **Minikube** | Belajar, single-node | RAM 2GB+ | Mudah |
| **Kind** | Testing, CI/CD, multi-node | RAM 1GB+ | Sedang |
| **Docker Desktop** | Pengguna Windows/Mac yang sudah pakai Docker | RAM 4GB+ | Sangat mudah |
| **k3d** | Ringan, cepat | RAM 512MB+ | Sedang |

**Rekomendasi untuk pemula:** Mulai dengan **Minikube** (paling lengkap dokumentasinya) atau **Docker Desktop** (jika sudah terinstall).

---

## Daftar Materi

| File | Topik |
|------|-------|
| [01 - Install kubectl](./01-install-kubectl.md) | CLI utama Kubernetes — wajib diinstall |
| [02 - Setup Minikube](./02-setup-minikube.md) | Cluster lokal single-node, direkomendasikan untuk pemula |
| [03 - Setup Kind](./03-setup-kind.md) | Kubernetes in Docker, bagus untuk multi-node lokal |
| [04 - Docker Desktop](./04-setup-docker-desktop.md) | Enable Kubernetes di Docker Desktop |

---

## Urutan Setup

```
Langkah 1: Install kubectl (WAJIB)
    ↓
Langkah 2: Install salah satu cluster lokal:
    - Minikube (direkomendasikan untuk pemula)
    - Kind
    - Docker Desktop (jika sudah ada)
    ↓
Langkah 3: Verifikasi setup
    ↓
Langkah 4: Siap praktik!
```

---

## Quick Verification

Setelah setup, jalankan perintah ini untuk memverifikasi semuanya berjalan:

```bash
# Cek kubectl
kubectl version --client

# Cek koneksi ke cluster
kubectl cluster-info

# Lihat nodes
kubectl get nodes

# Jalankan Pod pertama
kubectl run nginx --image=nginx
kubectl get pods
```

---

## Navigasi

- [Sebelumnya: 01 - Fondasi](../01-fondasi/README.md)
- [Selanjutnya: 03 - Object Dasar](../03-object-dasar/README.md)
- [Kembali ke README utama](../README.md)
