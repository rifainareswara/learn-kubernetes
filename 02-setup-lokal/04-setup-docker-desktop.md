# Setup Kubernetes di Docker Desktop

Docker Desktop menyediakan Kubernetes built-in yang bisa diaktifkan dengan mudah. Ini adalah pilihan termudah jika kamu sudah menggunakan Docker Desktop.

---

## Prasyarat

- Docker Desktop versi 2.0+ terinstall
- macOS atau Windows (Docker Desktop untuk Linux tidak punya fitur ini)
- RAM minimal 4GB untuk Docker Desktop

---

## Mengaktifkan Kubernetes di Docker Desktop

### macOS

1. Buka **Docker Desktop** (icon di menu bar)
2. Klik **Settings** (gear icon)
3. Pilih tab **Kubernetes**
4. Centang **Enable Kubernetes**
5. Klik **Apply & Restart**
6. Tunggu hingga status Kubernetes menjadi **Running** (bintang hijau di sudut kiri bawah)

```
Docker Desktop Settings
└── Kubernetes
    ├── [✓] Enable Kubernetes
    ├── [✓] Show system containers (advanced)
    └── [Apply & Restart]
```

### Windows

Prosesnya sama seperti macOS melalui Docker Desktop Settings.

---

## Verifikasi

```bash
# Setelah Docker Desktop restart, cek context kubectl
kubectl config get-contexts
# Harusnya ada entry "docker-desktop"

# Aktifkan context docker-desktop
kubectl config use-context docker-desktop

# Cek cluster
kubectl cluster-info

# Lihat nodes
kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   5m    v1.28.2
```

---

## Mengatur Resource

Docker Desktop Kubernetes berbagi resource dengan Docker Engine. Kamu bisa atur dari Settings:

1. **Settings** → **Resources**
2. Atur **CPUs**, **Memory**, **Swap**, **Disk image size**

**Rekomendasi minimum untuk K8s di Docker Desktop:**
- CPU: 4 core
- Memory: 6-8 GB
- Disk: 60 GB

---

## Kelebihan dan Kekurangan

| Kelebihan | Kekurangan |
|-----------|------------|
| Setup paling mudah | Hanya single-node |
| Terintegrasi dengan Docker | Berbagi resource dengan Docker |
| Tidak perlu tools tambahan | Hanya untuk macOS/Windows |
| Reset mudah via GUI | Versi K8s terbatas |

---

## Reset Kubernetes

Jika cluster bermasalah, kamu bisa reset dengan mudah:

1. **Settings** → **Kubernetes**
2. Klik **Reset Kubernetes Cluster**
3. Klik **Reset**

> **Peringatan:** Reset akan menghapus semua resource Kubernetes!

---

## Menggunakan Docker Image Lokal

Keuntungan Docker Desktop Kubernetes: Docker image yang kamu build langsung tersedia di Kubernetes tanpa perlu load manual!

```bash
# Build image seperti biasa
docker build -t my-app:latest .

# Langsung bisa digunakan di K8s
kubectl run my-app --image=my-app:latest --image-pull-policy=Never
```

---

## Pindah Antara Minikube dan Docker Desktop

Jika kamu punya keduanya, gunakan kubectl context untuk berpindah:

```bash
# Pindah ke Minikube
kubectl config use-context minikube

# Pindah ke Docker Desktop
kubectl config use-context docker-desktop

# Atau dengan kubectx
kubectx minikube
kubectx docker-desktop
```

---

## Troubleshooting

### Kubernetes tidak mau start

```bash
# Reset cluster dari Settings
# Atau uninstall dan install ulang Docker Desktop

# Cek apakah port konflik
lsof -i :6443  # Port default API server
```

### Context tidak muncul setelah enable

```bash
# Restart kubectl config
kubectl config get-contexts

# Jika tidak ada, cek kubeconfig
cat ~/.kube/config
```

---

*[Kembali ke: Overview Setup Lokal](./README.md)*

*[Lanjut ke: Object Dasar →](../03-object-dasar/README.md)*
