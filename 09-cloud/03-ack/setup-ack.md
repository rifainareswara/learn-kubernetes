# Setup Alibaba Cloud Container Service for Kubernetes (ACK)

Panduan setup Kubernetes di Alibaba Cloud menggunakan ACK — tersedia di data center Jakarta (ap-southeast-5).

---

## Prasyarat

- Akun Alibaba Cloud
- Aliyun CLI terinstall
- `kubectl` terinstall

---

## Step 1: Install Aliyun CLI

### macOS

```bash
brew install aliyun-cli

# Verifikasi
aliyun version
```

### Linux

```bash
# Download CLI
curl -o aliyun-cli-linux-latest-amd64.tgz \
  https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz

tar xzvf aliyun-cli-linux-latest-amd64.tgz
sudo mv aliyun /usr/local/bin/

# Verifikasi
aliyun version
```

---

## Step 2: Konfigurasi Aliyun CLI

```bash
# Konfigurasi credentials
aliyun configure
# Masukkan:
# Access Key ID: (dari Alibaba Cloud Console → IAM → AccessKey)
# Access Key Secret: (dari IAM)
# Default Region ID: ap-southeast-5  (Jakarta)
# Default output format: json

# Verifikasi
aliyun sts GetCallerIdentity
```

### Buat RAM User (best practice)

```bash
# Buat user via Console:
# Alibaba Cloud Console → RAM → Users → Create User
# Attach policy: AdministratorAccess (development) atau custom policy (production)
# Buat AccessKey untuk user tersebut
```

---

## Step 3: Buat ACK Cluster

### Via Alibaba Cloud Console (Cara Paling Mudah)

1. Login ke https://cs.console.aliyun.com
2. Klik **Create Kubernetes Cluster**
3. Pilih **Managed Kubernetes** (ACK Pro rekomendasi)
4. Konfigurasi:
   - Region: **AP-Southeast 5 (Jakarta)**
   - Worker Node: ECS.c6.large (2 vCPU, 4GB) minimal 2 node
   - VPC: Buat baru atau pilih yang ada
   - CIDR: 192.168.0.0/16 (Pod), 172.21.0.0/20 (Service)
5. Klik **Create**
6. Tunggu 5-10 menit

### Via Aliyun CLI

```bash
# Buat cluster via CLI
aliyun cs POST /clusters \
  --header "Content-Type=application/json" \
  --body '{
    "name": "my-cluster",
    "cluster_type": "ManagedKubernetes",
    "region_id": "ap-southeast-5",
    "kubernetes_version": "1.28.3-aliyun.1",
    "vpcid": "vpc-xxxxx",
    "worker_vswitch_ids": ["vsw-xxxxx"],
    "worker_instance_types": ["ecs.c6.large"],
    "num_of_nodes": 2,
    "worker_data_disks": [{"category": "cloud_essd", "size": "100"}],
    "snat_entry": true,
    "endpoint_public_access": true
  }'

# Lihat status cluster
aliyun cs GET /clusters
```

---

## Step 4: Connect kubectl ke ACK

### Via Console

1. Di ACK Console, buka cluster detail
2. Klik **Connection Information**
3. Download kubeconfig file
4. Simpan ke `~/.kube/config`

### Via Aliyun CLI

```bash
# Dapatkan cluster ID
CLUSTER_ID=$(aliyun cs GET /clusters | python3 -c \
  "import sys,json; clusters=json.load(sys.stdin); \
   print([c['cluster_id'] for c in clusters if c['name']=='my-cluster'][0])")

# Download kubeconfig
aliyun cs GET /k8s/$CLUSTER_ID/user_config > kubeconfig.yaml

# Merge dengan kubeconfig yang ada
export KUBECONFIG=~/.kube/config:$(pwd)/kubeconfig.yaml
kubectl config view --merge --flatten > ~/.kube/config_merged
mv ~/.kube/config_merged ~/.kube/config

# Verifikasi
kubectl cluster-info
kubectl get nodes
```

---

## Step 5: Deploy Aplikasi

```bash
# Deploy sama seperti cluster lokal
kubectl apply -f deployment.yaml

# Expose dengan LoadBalancer (Alibaba SLB)
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Tunggu external IP
kubectl get service nginx --watch
# EXTERNAL-IP akan berupa IP dari Alibaba SLB

# Test
curl http://EXTERNAL_IP
```

---

## Menggunakan Container Registry (ACR)

```bash
# Login ke ACR
docker login registry.ap-southeast-5.aliyuncs.com \
  -u YOUR_ALIYUN_USERNAME \
  -p YOUR_ALIYUN_PASSWORD

# Build dan push image
docker build -t registry.ap-southeast-5.aliyuncs.com/NAMESPACE/my-app:v1 .
docker push registry.ap-southeast-5.aliyuncs.com/NAMESPACE/my-app:v1

# Buat image pull secret
kubectl create secret docker-registry acr-secret \
  --docker-server=registry.ap-southeast-5.aliyuncs.com \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD

# Gunakan di Deployment
# spec.template.spec.imagePullSecrets:
# - name: acr-secret
```

---

## ACK Storage: Alibaba Cloud Disk

```yaml
# StorageClass untuk Alibaba Cloud Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-essd
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  type: cloud_essd    # ESSD (Enhanced SSD) - performa terbaik
  # cloud_efficiency   = Standard efficiency disk (murah)
  # cloud_ssd          = SSD disk
  # cloud_essd         = ESSD (paling cepat)
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Auto-scaling di ACK

```bash
# Install metrics-server (biasanya sudah ada di ACK)
kubectl top nodes
kubectl top pods

# HPA
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70

# Cluster Autoscaler sudah built-in di ACK
# Konfigurasi di ACK Console → Cluster → Node Pools → Enable Auto Scaling
```

---

## Cleanup

```bash
# Hapus resource Kubernetes
kubectl delete all --all
kubectl delete pvc --all

# Hapus cluster via Console:
# ACK Console → Clusters → Delete

# Atau via CLI:
aliyun cs DELETE /clusters/$CLUSTER_ID
```

---

## Biaya Estimasi (Jakarta Region)

| Resource | Estimasi/Bulan |
|----------|---------------|
| ACK Pro (managed control plane) | ~$65 |
| 2x ecs.c6.large (2C4G) | ~$60 |
| 2x Cloud Disk 100GB ESSD | ~$20 |
| SLB (Load Balancer) | ~$15 |
| **Total** | **~$160** |

> ACK Pro gratis untuk beberapa bulan pertama — cek promo di console.

---

*[Kembali ke: Cloud Overview](../README.md)*
