# Helm Dasar

Helm adalah "package manager" untuk Kubernetes. Dengan Helm, kamu bisa install aplikasi kompleks hanya dengan satu perintah, dan mengelola update/rollback dengan mudah.

---

## Konsep Utama Helm

### Chart
Chart adalah "paket" yang berisi semua template Kubernetes untuk sebuah aplikasi:

```
my-app-chart/
├── Chart.yaml          # Metadata chart (nama, versi, deskripsi)
├── values.yaml         # Default values (bisa di-override)
├── templates/          # Template YAML Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Template helper functions
└── charts/             # Chart dependencies
```

### Release
Release adalah instance dari Chart yang sudah di-install di cluster:

```bash
helm install my-nginx nginx-chart    # "my-nginx" adalah nama release
helm install production-nginx nginx-chart  # Release berbeda, chart yang sama
```

### Values
Values adalah konfigurasi yang bisa di-override saat install atau upgrade:

```yaml
# values.yaml (default)
replicas: 2
image:
  repository: nginx
  tag: "1.25"
service:
  type: ClusterIP
  port: 80
```

```bash
# Override saat install
helm install my-app ./my-chart --set replicas=5
# Atau via file
helm install my-app ./my-chart -f custom-values.yaml
```

### Repository
Repository adalah tempat koleksi Chart disimpan (seperti npm registry):

```
Helm Hub: https://artifacthub.io
Bitnami:  https://charts.bitnami.com/bitnami
```

---

## Install Helm

### macOS

```bash
# Homebrew (paling mudah)
brew install helm

# Atau script resmi
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Linux

```bash
# Script resmi
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Atau apt (Debian/Ubuntu)
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Windows

```powershell
choco install kubernetes-helm
# atau
winget install Helm.Helm
```

### Verifikasi

```bash
helm version
# Output: version.BuildInfo{Version:"v3.x.x", ...}
```

---

## Manajemen Repository

```bash
# Tambah repository populer
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add argo https://argoproj.github.io/argo-helm

# Update semua repository
helm repo update

# Lihat semua repository
helm repo list

# Cari chart
helm search repo nginx
helm search repo postgresql

# Cari di Artifact Hub (lebih lengkap)
helm search hub postgresql
```

---

## Perintah Helm Dasar

### Install Chart

```bash
# Install dengan nama release
helm install my-nginx bitnami/nginx

# Install di namespace tertentu (buat namespace jika belum ada)
helm install my-nginx bitnami/nginx --namespace webservers --create-namespace

# Install dengan override values
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=mypassword \
  --set primary.persistence.size=20Gi

# Install dari file values
helm install my-app ./my-chart -f production-values.yaml

# Dry-run: tampilkan manifest tanpa install
helm install my-app ./my-chart --dry-run

# Debug: tampilkan manifest yang akan di-apply
helm install my-app ./my-chart --dry-run --debug
```

### Lihat Release

```bash
# Daftar semua release
helm list
helm ls

# Release di semua namespace
helm list -A

# Detail release tertentu
helm status my-nginx

# Lihat values yang digunakan
helm get values my-nginx

# Lihat semua values (termasuk default)
helm get values my-nginx --all

# Lihat manifest yang di-deploy
helm get manifest my-nginx
```

### Upgrade Release

```bash
# Upgrade ke versi chart baru
helm upgrade my-nginx bitnami/nginx

# Upgrade dengan perubahan values
helm upgrade my-nginx bitnami/nginx --set service.type=NodePort

# Install jika belum ada, upgrade jika sudah ada
helm upgrade --install my-nginx bitnami/nginx

# Rollback otomatis jika upgrade gagal
helm upgrade my-nginx bitnami/nginx --atomic --timeout 5m
```

### Rollback

```bash
# Lihat history release
helm history my-nginx

# Rollback ke revision sebelumnya
helm rollback my-nginx

# Rollback ke revision tertentu
helm rollback my-nginx 2
```

### Uninstall

```bash
# Hapus release (resource Kubernetes ikut terhapus)
helm uninstall my-nginx

# Hapus tapi simpan history
helm uninstall my-nginx --keep-history
```

---

## Contoh Praktis: Install Nginx Ingress Controller

```bash
# Tambah repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Lihat semua values yang bisa dikonfigurasi
helm show values ingress-nginx/ingress-nginx

# Install
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Verifikasi
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx
```

## Contoh Praktis: Install PostgreSQL

```bash
# Lihat values PostgreSQL
helm show values bitnami/postgresql | less

# Install dengan konfigurasi
helm install my-postgres bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --set auth.postgresPassword=supersecret \
  --set auth.database=myappdb \
  --set primary.persistence.size=10Gi \
  --set primary.resources.requests.memory=256Mi

# Akses PostgreSQL
kubectl run psql-client --rm --tty -i --restart=Never \
  --namespace database \
  --image bitnami/postgresql:15 \
  --env="PGPASSWORD=supersecret" \
  --command -- psql --host my-postgres-postgresql -U postgres -d myappdb
```

---

## Membuat Chart Sendiri

### Struktur Chart

```bash
# Buat skeleton chart
helm create my-app-chart

# Struktur yang dibuat:
# my-app-chart/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     ├── hpa.yaml
#     ├── serviceaccount.yaml
#     ├── NOTES.txt
#     └── _helpers.tpl
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2          # Versi API Helm (v2 untuk Helm 3)
name: my-app-chart      # Nama chart
description: A Helm chart for My App

type: application       # "application" atau "library"
version: 0.1.0          # Versi chart (SemVer)
appVersion: "1.0.0"     # Versi aplikasi

dependencies:           # Chart lain yang diperlukan
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled   # Bisa disable via values
```

### values.yaml

```yaml
# values.yaml - Default values
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    memory: 128Mi
    cpu: 100m
  limits:
    memory: 256Mi
    cpu: 500m

postgresql:
  enabled: true
  auth:
    postgresPassword: changeme
    database: myappdb
```

### templates/deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app-chart.fullname" . }}   # ← Template function
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}             # ← Nilai dari values.yaml
  selector:
    matchLabels:
      {{- include "my-app-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### Test Chart

```bash
# Lint chart (cek error)
helm lint my-app-chart

# Dry-run untuk lihat output
helm install test my-app-chart --dry-run --debug

# Install
helm install my-release my-app-chart

# Install dengan values kustom
helm install my-release my-app-chart \
  --set replicaCount=5 \
  --set image.tag=1.26 \
  -f custom-values.yaml

# Package chart untuk distribusi
helm package my-app-chart
# Output: my-app-chart-0.1.0.tgz
```

---

## Tips dan Best Practices

1. **Selalu gunakan `--atomic`** untuk upgrade production:
   ```bash
   helm upgrade --install my-app ./chart --atomic --timeout 5m
   ```

2. **Simpan values di Git** — jangan override via `--set` di production

3. **Gunakan secrets manager** untuk values sensitif (tidak hardcode di values.yaml)

4. **Pin versi chart** untuk reproducibility:
   ```bash
   helm install my-app bitnami/nginx --version 15.1.0
   ```

5. **Gunakan helm test** untuk verifikasi setelah install:
   ```bash
   helm test my-app
   ```

---

## Troubleshooting

```bash
# Lihat semua release termasuk yang gagal
helm list -A --all

# Debug install yang gagal
helm install my-app ./chart --debug

# Lihat history dengan status
helm history my-app

# Rollback jika upgrade gagal
helm rollback my-app

# Force upgrade (ganti resource yang konflik)
helm upgrade my-app ./chart --force
```

---

*[Kembali ke: Overview Helm](./README.md)*

*[Lanjut ke: Project Latihan →](../08-project-latihan/README.md)*
