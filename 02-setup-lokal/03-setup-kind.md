# Setup Kind (Kubernetes in Docker)

Kind adalah tool yang menjalankan cluster Kubernetes menggunakan container Docker sebagai "node". Lebih ringan dari Minikube dan cocok untuk testing multi-node.

---

## Apa itu Kind?

Kind singkatan dari **Kubernetes IN Docker**. Setiap "node" Kubernetes sebenarnya adalah sebuah Docker container.

```
Laptop Kamu
┌──────────────────────────────────────────────────┐
│                                                  │
│  Docker Engine                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │
│  │  Container   │  │  Container   │  │  ...   │  │
│  │ "control-    │  │  "worker-    │  │        │  │
│  │  plane"      │  │   node-1"    │  │        │  │
│  │ (K8s node)   │  │ (K8s node)   │  │        │  │
│  └──────────────┘  └──────────────┘  └────────┘  │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Keunggulan Kind:**
- Lebih ringan dari Minikube (menggunakan Docker, bukan VM)
- Mudah membuat cluster multi-node
- Sangat cepat untuk membuat dan menghapus cluster
- Populer untuk testing di CI/CD pipeline

---

## Prerequisites

- Docker terinstall dan berjalan
- kubectl sudah terinstall

---

## Install Kind

### macOS

```bash
# Menggunakan Homebrew
brew install kind

# Atau download binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-amd64
# Untuk Apple Silicon:
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-darwin-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verifikasi
kind version
```

### Linux

```bash
# Download binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verifikasi
kind version
```

### Windows

```powershell
# Menggunakan Chocolatey
choco install kind

# Atau download dari:
# https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
```

---

## Membuat Cluster Kind

### Cluster Sederhana (Single Node)

```bash
# Buat cluster dengan nama default
kind create cluster

# Buat cluster dengan nama kustom
kind create cluster --name my-cluster

# Buat dengan versi Kubernetes tertentu
kind create cluster --image kindest/node:v1.28.0

# Output yang diharapkan:
# Creating cluster "kind" ...
#  ✓ Ensuring node image (kindest/node:v1.28.0) 🖼
#  ✓ Preparing nodes 📦
#  ✓ Writing configuration 📜
#  ✓ Starting control-plane 🕹️
#  ✓ Installing CNI 🔌
#  ✓ Installing StorageClass 💾
# Set kubectl context to "kind-kind"
# You can now use your cluster with:
# kubectl cluster-info --context kind-kind
```

### Cluster Multi-Node (Lebih Realistis)

Buat file konfigurasi `kind-config.yaml`:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  # Control plane node
  - role: control-plane
    # Mount port lokal ke container node
    extraPortMappings:
      - containerPort: 30000  # NodePort range awal
        hostPort: 30000
        protocol: TCP
      - containerPort: 80
        hostPort: 80
        protocol: TCP

  # Worker nodes
  - role: worker
  - role: worker
  - role: worker
```

```bash
# Buat cluster multi-node
kind create cluster --config kind-config.yaml --name dev-cluster

# Verifikasi nodes
kubectl get nodes
# NAME                       STATUS   ROLES           AGE   VERSION
# dev-cluster-control-plane  Ready    control-plane   2m    v1.28.0
# dev-cluster-worker         Ready    <none>          90s   v1.28.0
# dev-cluster-worker2        Ready    <none>          90s   v1.28.0
# dev-cluster-worker3        Ready    <none>          90s   v1.28.0
```

---

## Perintah-Perintah Kind

```bash
# Lihat semua cluster
kind get clusters

# Lihat nodes dalam cluster
kind get nodes --name my-cluster

# Dapatkan kubeconfig
kind get kubeconfig --name my-cluster

# Hapus cluster
kind delete cluster --name my-cluster

# Hapus semua cluster
kind delete clusters --all

# Load image ke dalam cluster
# (berguna agar tidak perlu push ke registry saat development)
kind load docker-image my-app:latest --name my-cluster
```

---

## Loading Docker Image ke Kind

Salah satu fitur penting Kind: kamu bisa load Docker image lokal langsung ke dalam cluster tanpa perlu push ke registry:

```bash
# Build image lokal
docker build -t my-app:dev .

# Load ke Kind cluster
kind load docker-image my-app:dev --name my-cluster

# Sekarang bisa pakai di Pod dengan imagePullPolicy: Never atau IfNotPresent
kubectl run my-app --image=my-app:dev --image-pull-policy=Never
```

---

## Konfigurasi Kind Lanjutan

### Dengan Ingress Controller

```yaml
# kind-ingress.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
# Buat cluster
kind create cluster --config kind-ingress.yaml

# Install nginx ingress controller untuk Kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Tunggu ingress controller siap
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Dengan Custom Registry

```yaml
# kind-registry.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
      endpoint = ["http://kind-registry:5000"]

nodes:
  - role: control-plane
  - role: worker
```

---

## Menggunakan Kind dengan Multiple Clusters

```bash
# Buat beberapa cluster
kind create cluster --name dev
kind create cluster --name staging

# Lihat semua cluster
kind get clusters
# dev
# staging

# Pindah antara cluster
kubectl config use-context kind-dev
kubectl config use-context kind-staging

# Atau gunakan kubectx
kubectx kind-dev
kubectx kind-staging
```

---

## Perbandingan Kind vs Minikube

| Aspek | Kind | Minikube |
|-------|------|---------|
| Driver | Docker (saja) | Docker, VM, dll |
| Multi-node | Mudah | Terbatas |
| Speed | Lebih cepat | Lebih lambat |
| Resource | Lebih ringan | Lebih berat |
| Addons | Manual | Built-in |
| Image loading | kind load | eval $(minikube docker-env) |
| Dashboard | Tidak built-in | minikube dashboard |
| Cocok untuk | Testing, CI/CD | Belajar, development |

---

## Verifikasi Setup

```bash
# 1. Buat cluster
kind create cluster --name test

# 2. Cek cluster berjalan
kubectl cluster-info --context kind-test

# 3. Lihat nodes
kubectl get nodes

# 4. Deploy nginx test
kubectl create deployment nginx --image=nginx --replicas=2

# 5. Tunggu ready
kubectl rollout status deployment/nginx

# 6. Lihat pods
kubectl get pods

# 7. Buat service
kubectl expose deployment nginx --type=NodePort --port=80

# 8. Test (via port-forward karena NodePort tidak langsung accessible di Kind)
kubectl port-forward service/nginx 8080:80 &
curl http://localhost:8080

# 9. Cleanup
kind delete cluster --name test
```

---

## Troubleshooting Umum

### Error: `failed to create cluster: running kind with rootless docker`
```bash
# Set rootful docker atau run dengan sudo
export KIND_EXPERIMENTAL_DOCKER_NETWORK=bridge
kind create cluster
```

### Error: `node(s) had untolerated taint`
```bash
# Worker node tidak bisa di-schedule
# Cek taints pada node
kubectl describe node | grep Taint

# Atau pastikan worker node sudah Ready
kubectl get nodes
```

### Image tidak ditemukan di cluster
```bash
# Load ulang image
kind load docker-image my-app:latest --name my-cluster

# Pastikan imagePullPolicy: Never di Pod spec
kubectl set image deployment/my-app my-app=my-app:latest
```

---

*[Lanjut ke: Docker Desktop →](./04-setup-docker-desktop.md)*

*[Kembali ke: Overview Setup Lokal](./README.md)*
