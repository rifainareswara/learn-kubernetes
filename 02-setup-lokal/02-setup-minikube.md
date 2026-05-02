# Setup Minikube

Minikube adalah tools yang menjalankan cluster Kubernetes single-node di dalam VM atau container di laptopmu. Ini adalah pilihan terbaik untuk belajar Kubernetes.

---

## Apa itu Minikube?

```
Laptop Kamu
┌──────────────────────────────────────────────┐
│                                              │
│  ┌───────────────────────────────────────┐   │
│  │            Minikube VM/Container       │   │
│  │                                        │   │
│  │  ┌──────────────┐  ┌────────────────┐  │   │
│  │  │ Control Plane │  │  Worker Node   │  │   │
│  │  │ (API Server,  │  │  (kubelet,     │  │   │
│  │  │  etcd, dll)   │  │   kube-proxy)  │  │   │
│  │  └──────────────┘  └────────────────┘  │   │
│  │           (berjalan dalam 1 VM/Node)    │   │
│  └───────────────────────────────────────┘   │
│                                              │
└──────────────────────────────────────────────┘
```

Minikube menjalankan Kubernetes "all-in-one" di satu node, sehingga cocok untuk:
- Belajar dan eksperimen
- Development lokal
- Testing manifest sebelum deploy ke production

---

## Prerequisites

- Docker terinstall (jika menggunakan Docker driver)
- Atau VirtualBox/HyperKit/HyperV (jika menggunakan VM driver)
- RAM minimal 2GB yang bisa dialokasikan
- CPU minimal 2 core
- 20GB disk space

---

## Install Minikube

### macOS

```bash
# Menggunakan Homebrew (paling mudah)
brew install minikube

# Atau download binary langsung
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
# Untuk Apple Silicon:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64

sudo install minikube-darwin-amd64 /usr/local/bin/minikube
# Atau untuk Apple Silicon:
sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# Verifikasi
minikube version
```

### Linux

```bash
# Download dan install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verifikasi
minikube version
```

### Windows

```powershell
# Menggunakan Chocolatey
choco install minikube

# Menggunakan winget
winget install Kubernetes.minikube

# Atau download installer dari:
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
```

---

## Menjalankan Minikube

### Start Cluster

```bash
# Start dengan driver default (Docker jika tersedia)
minikube start

# Start dengan resource yang ditentukan
minikube start --cpus=4 --memory=8192

# Start dengan versi Kubernetes tertentu
minikube start --kubernetes-version=v1.28.0

# Start dengan driver spesifik
minikube start --driver=docker       # Menggunakan Docker (direkomendasikan)
minikube start --driver=virtualbox   # Menggunakan VirtualBox
minikube start --driver=hyperkit     # macOS (built-in)
minikube start --driver=hyperv       # Windows (built-in)
```

### Output yang diharapkan saat start pertama:

```
😄  minikube v1.32.0 on Darwin 14.0 (arm64)
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

---

## Perintah-Perintah Penting Minikube

```bash
# Lihat status cluster
minikube status

# Hentikan cluster (simpan state)
minikube stop

# Hapus cluster (semua data hilang)
minikube delete

# Hapus semua profil cluster
minikube delete --all

# Masuk ke VM Minikube via SSH
minikube ssh

# Lihat IP cluster
minikube ip

# Lihat dashboard (web UI)
minikube dashboard

# Lihat semua addons yang tersedia
minikube addons list

# Enable addon
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Lihat logs
minikube logs
```

---

## Mengakses Aplikasi di Minikube

Karena Minikube berjalan di dalam VM/container, akses ke aplikasi sedikit berbeda dari cluster biasa.

### Cara 1: minikube service (paling mudah)

```bash
# Buka service di browser secara otomatis
minikube service my-service

# Dapatkan URL service
minikube service my-service --url
```

### Cara 2: kubectl port-forward

```bash
kubectl port-forward service/my-service 8080:80
# Akses di: http://localhost:8080
```

### Cara 3: NodePort dengan IP Minikube

```bash
# Dapatkan IP Minikube
MINIKUBE_IP=$(minikube ip)
echo "Akses di: http://${MINIKUBE_IP}:<NodePort>"
```

---

## Profil Minikube (Multiple Clusters)

Minikube mendukung beberapa cluster sekaligus via profil:

```bash
# Buat cluster dengan profil baru
minikube start --profile=cluster-dev
minikube start --profile=cluster-staging

# Lihat semua profil
minikube profile list

# Pindah ke profil lain
minikube profile cluster-dev

# Hapus profil tertentu
minikube delete --profile=cluster-dev
```

---

## Menggunakan Docker dalam Minikube

Untuk build Docker image dan langsung tersedia di Minikube (tanpa perlu push ke registry):

```bash
# Arahkan Docker CLI ke Docker daemon dalam Minikube
eval $(minikube docker-env)

# Sekarang docker build akan langsung tersedia di Minikube
docker build -t my-app:latest .

# Gunakan di Pod (imagePullPolicy: Never agar tidak pull dari registry)
kubectl run my-app --image=my-app:latest --image-pull-policy=Never

# Kembalikan Docker ke daemon lokal
eval $(minikube docker-env --unset)
```

---

## Verifikasi Setup Lengkap

```bash
# 1. Cek status Minikube
minikube status
# Output: host, kubelet, apiserver → Running

# 2. Cek koneksi kubectl ke cluster
kubectl cluster-info
# Output: Kubernetes control plane is running at https://...

# 3. Lihat node
kubectl get nodes
# Output: minikube   Ready   control-plane   ...

# 4. Jalankan Pod test
kubectl run test-nginx --image=nginx --port=80
kubectl get pods

# 5. Tunggu Pod Running
kubectl wait --for=condition=Ready pod/test-nginx --timeout=60s

# 6. Test akses
kubectl port-forward pod/test-nginx 8080:80 &
curl http://localhost:8080
# Output: <!DOCTYPE html> ... nginx default page

# 7. Cleanup
kill %1  # hentikan port-forward
kubectl delete pod test-nginx
```

---

## Troubleshooting Umum

### Error: `Exiting due to PROVIDER_DOCKER_NOT_RUNNING`
```bash
# Docker tidak berjalan
# Solusi: start Docker terlebih dahulu
open -a Docker      # macOS
sudo systemctl start docker  # Linux
```

### Error: `minikube start` terlalu lama / hang
```bash
# Coba dengan driver lain
minikube delete
minikube start --driver=docker

# Atau tambah timeout
minikube start --wait-timeout=10m
```

### Error: `Unable to pull image`
```bash
# Cek koneksi internet
# Coba dengan mirror registry
minikube start --image-mirror-country=cn  # Jika di China

# Atau set proxy
minikube start --docker-env HTTP_PROXY=http://proxy:port \
               --docker-env HTTPS_PROXY=http://proxy:port
```

### Error: `Error response from daemon: Conflict`
```bash
# Container lama masih ada
minikube delete
minikube start
```

### Pod stuck di Pending
```bash
# Cek apakah node punya resource cukup
kubectl describe node minikube
kubectl describe pod <pod-name>

# Cek events
kubectl get events
```

### Minikube sangat lambat
```bash
# Tambah resource
minikube stop
minikube config set cpus 4
minikube config set memory 6144
minikube start
```

---

## Konfigurasi Default Minikube

```bash
# Lihat semua konfigurasi
minikube config view

# Set konfigurasi default
minikube config set driver docker
minikube config set cpus 2
minikube config set memory 4096
minikube config set disk-size 30g
minikube config set kubernetes-version v1.28.0
```

---

*[Lanjut ke: Setup Kind →](./03-setup-kind.md)*

*[Kembali ke: Overview Setup Lokal](./README.md)*
