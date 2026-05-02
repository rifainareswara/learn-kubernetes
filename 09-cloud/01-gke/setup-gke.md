# Setup Google Kubernetes Engine (GKE)

Panduan lengkap untuk membuat cluster Kubernetes di Google Cloud Platform (GCP) menggunakan GKE.

---

## Prasyarat

- Akun Google Cloud (bisa daftar dengan $300 free credit untuk 90 hari)
- Kartu kredit untuk verifikasi (tidak akan dicharge jika dalam free tier)
- `gcloud` CLI terinstall

---

## Step 1: Install Google Cloud CLI (gcloud)

### macOS

```bash
# Menggunakan Homebrew
brew install google-cloud-sdk

# Atau download dari:
# https://cloud.google.com/sdk/docs/install

# Verifikasi
gcloud version
```

### Linux

```bash
# Download dan install
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Atau via apt (Debian/Ubuntu)
sudo apt-get install apt-transport-https ca-certificates gnupg curl
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | \
  sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-cli
```

---

## Step 2: Autentikasi dan Setup Project

```bash
# Login ke Google Cloud
gcloud auth login
# Akan buka browser untuk login

# Lihat semua project
gcloud projects list

# Set project default
gcloud config set project YOUR_PROJECT_ID

# Set region dan zone default (pilih yang dekat Indonesia: asia-southeast1)
gcloud config set compute/region asia-southeast1       # Singapore
gcloud config set compute/zone asia-southeast1-a

# Verifikasi konfigurasi
gcloud config list

# Aktifkan API yang diperlukan
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
```

---

## Step 3: Buat GKE Cluster

### Opsi 1: Standard Cluster (kamu kelola Node)

```bash
# Buat cluster development (hemat biaya)
gcloud container clusters create my-cluster \
  --zone asia-southeast1-a \
  --num-nodes 2 \
  --machine-type e2-standard-2 \
  --disk-size 50GB \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 3

# Atau cluster production-grade dengan lebih banyak fitur:
gcloud container clusters create production-cluster \
  --region asia-southeast1 \           # Regional cluster (HA!)
  --num-nodes 2 \                      # Node per zone (total 6 node)
  --machine-type e2-standard-4 \
  --disk-size 100GB \
  --disk-type pd-ssd \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5 \
  --enable-autorepair \                # Auto-repair Node yang bermasalah
  --enable-autoupgrade \               # Auto-upgrade Node
  --workload-pool=YOUR_PROJECT_ID.svc.id.goog \  # Workload Identity
  --enable-shielded-nodes              # Keamanan tambahan
```

### Opsi 2: Autopilot Cluster (Google kelola Node)

```bash
# GKE Autopilot: kamu tidak perlu kelola Node sama sekali!
# Hanya bayar per Pod yang berjalan, bukan per Node
gcloud container clusters create-auto autopilot-cluster \
  --region asia-southeast1

# Verifikasi
gcloud container clusters list
```

---

## Step 4: Connect kubectl ke GKE

```bash
# Generate kubeconfig untuk cluster
gcloud container clusters get-credentials my-cluster \
  --zone asia-southeast1-a \
  --project YOUR_PROJECT_ID

# Verifikasi koneksi
kubectl cluster-info
kubectl get nodes

# Output yang diharapkan:
# NAME                                    STATUS   ROLES    AGE   VERSION
# gke-my-cluster-default-pool-xxx-yyy    Ready    <none>   5m    v1.28.x
# gke-my-cluster-default-pool-xxx-zzz    Ready    <none>   5m    v1.28.x
```

---

## Step 5: Deploy Aplikasi ke GKE

```bash
# Sekarang sama seperti di cluster lokal!
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Contoh deploy nginx
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Tunggu external IP dari LoadBalancer
kubectl get service nginx --watch
# Tunggu hingga EXTERNAL-IP terisi (bukan Pending)
# NAME    TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
# nginx   LoadBalancer   10.100.1.5     34.x.x.x       80:30xxx/TCP

# Akses aplikasi
curl http://34.x.x.x
```

---

## Step 6: Menggunakan Google Container Registry (GCR) / Artifact Registry

```bash
# Konfigurasi Docker untuk push ke Artifact Registry
gcloud auth configure-docker asia-southeast1-docker.pkg.dev

# Build dan push image
docker build -t asia-southeast1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-app:v1 .
docker push asia-southeast1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-app:v1

# Deploy dari Artifact Registry
kubectl set image deployment/my-app \
  my-app=asia-southeast1-docker.pkg.dev/YOUR_PROJECT_ID/my-repo/my-app:v1
```

---

## Step 7: Auto-scaling di GKE

### Horizontal Pod Autoscaler (HPA)

```bash
# Install metrics-server (sudah ada di GKE)
kubectl top pods

# Buat HPA
kubectl autoscale deployment my-app \
  --min=2 \
  --max=10 \
  --cpu-percent=70

# Atau via YAML:
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
EOF

# Lihat status HPA
kubectl get hpa
```

---

## Monitoring di GKE

GKE terintegrasi dengan Google Cloud Monitoring secara otomatis:

```bash
# Aktifkan monitoring (sudah aktif secara default)
# Akses di: https://console.cloud.google.com/monitoring

# Atau gunakan kubectl top untuk melihat resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

---

## Cleanup (PENTING!)

> **Peringatan:** Selalu hapus cluster setelah selesai untuk menghindari biaya yang tidak perlu!

```bash
# Hapus semua deployment dulu
kubectl delete all --all

# Hapus cluster
gcloud container clusters delete my-cluster \
  --zone asia-southeast1-a \
  --quiet               # --quiet agar tidak minta konfirmasi

# Verifikasi sudah terhapus
gcloud container clusters list

# Hapus disks yang tidak dipakai
gcloud compute disks list
gcloud compute disks delete DISK_NAME --zone=ZONE
```

---

## Biaya Estimasi

| Resource | Estimasi/Bulan |
|----------|---------------|
| GKE Management fee | ~$75 (gratis untuk 1 cluster zonal!) |
| 2x e2-standard-2 Node | ~$100 |
| 50GB disk per node | ~$10 |
| LoadBalancer | ~$20 |
| **Total** | **~$130-210** |

> Untuk development, gunakan **zonal cluster** (gratis management fee) dengan **preemptible/spot nodes** (60-80% lebih murah).

---

## Troubleshooting

### Error: `PERMISSION_DENIED`

```bash
# Pastikan API sudah diaktifkan
gcloud services enable container.googleapis.com

# Cek IAM permission
gcloud projects get-iam-policy YOUR_PROJECT_ID
```

### Error: `Cluster tidak bisa dibuat - quota exceeded`

```bash
# Lihat quota
gcloud compute project-info describe --project=YOUR_PROJECT_ID
# Request quota increase di Google Cloud Console
```

---

*[Kembali ke: Cloud Overview](../README.md)*
