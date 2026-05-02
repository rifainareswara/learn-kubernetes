# Deploy ke Amazon Elastic Kubernetes Service (EKS)

> **Estimasi Waktu:** 45-60 menit
>
> **Prasyarat:** Akun AWS, `aws` CLI terinstall dan terkonfigurasi, `eksctl` terinstall

---

## Persiapan AWS CLI

```bash
# Pastikan AWS CLI sudah terkonfigurasi
aws configure
# Masukkan: Access Key ID, Secret Access Key, Region (ap-southeast-1 untuk Singapore)

# Verifikasi
aws sts get-caller-identity
# Output: {"UserId": "...", "Account": "123456789012", "Arn": "..."}

# Set variabel untuk kemudahan
export AWS_REGION=ap-southeast-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"
echo "Region: $AWS_REGION"
```

---

## Langkah 1: Push Image ke Amazon ECR

Amazon Elastic Container Registry (ECR) adalah private container registry AWS.

### 1.1 Buat Repository di ECR

```bash
# Buat repository untuk backend
aws ecr create-repository \
  --repository-name todolist/backend \
  --region $AWS_REGION

# Buat repository untuk frontend
aws ecr create-repository \
  --repository-name todolist/frontend \
  --region $AWS_REGION

# Lihat semua repository
aws ecr describe-repositories --region $AWS_REGION
```

### 1.2 Login Docker ke ECR

```bash
# Get token login dan pipe ke docker login
aws ecr get-login-password --region $AWS_REGION | \
  docker login \
  --username AWS \
  --password-stdin \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
# Output: Login Succeeded
```

### 1.3 Tag dan Push Image

```bash
# ── Backend ──────────────────────────────────────────────────────────────────
docker tag todolist-backend:local \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/todolist/backend:v1.0

docker push \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/todolist/backend:v1.0

# ── Frontend ──────────────────────────────────────────────────────────────────
docker tag todolist-frontend:local \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/todolist/frontend:v1.0

docker push \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/todolist/frontend:v1.0

# Verifikasi images
aws ecr describe-images \
  --repository-name todolist/backend \
  --region $AWS_REGION
```

---

## Langkah 2: Update Deployment YAML

Edit `k8s/backend/deployment.yaml`:

```yaml
image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/todolist/backend:v1.0
```

Edit `k8s/frontend/deployment.yaml`:

```yaml
image: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/todolist/frontend:v1.0
```

> **Penting:** Di EKS, Node memerlukan IAM role yang memiliki permission `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, dll untuk pull image dari ECR. Jika menggunakan `eksctl`, ini biasanya dikonfigurasi otomatis.

---

## Langkah 3: Buat EKS Cluster (Jika Belum Ada)

```bash
# Install eksctl jika belum ada
# macOS:
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Buat cluster EKS (proses ini memakan waktu 15-20 menit)
eksctl create cluster \
  --name todolist-cluster \
  --region $AWS_REGION \
  --nodegroup-name todolist-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# Konfigurasi kubectl otomatis setelah cluster dibuat
# eksctl secara otomatis update ~/.kube/config

# Verifikasi
kubectl cluster-info
kubectl get nodes
```

---

## Langkah 4: Install AWS Load Balancer Controller

AWS Load Balancer Controller mengelola ALB/NLB untuk Kubernetes. Ini diperlukan agar Ingress bisa menggunakan AWS Application Load Balancer.

```bash
# 4.1 Buat IAM Policy untuk Load Balancer Controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 4.2 Buat IAM Service Account
eksctl create iamserviceaccount \
  --cluster=todolist-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=$AWS_REGION

# 4.3 Install menggunakan Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=todolist-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verifikasi
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Langkah 5: Install Nginx Ingress Controller

Sebagai alternatif yang lebih sederhana dari AWS Load Balancer Controller, kita bisa tetap pakai Nginx Ingress:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"

# Tunggu External hostname tersedia (EKS menggunakan hostname, bukan IP)
kubectl get svc -n ingress-nginx -w
# Kolom EXTERNAL-IP akan berisi AWS hostname:
# xxxxx.elb.ap-southeast-1.amazonaws.com
```

---

## Langkah 6: Deploy Aplikasi ke EKS

```bash
# Buat namespace dan secrets
kubectl create namespace todolist

kubectl create secret generic backend-secret \
  --from-literal=DB_USER=todouser \
  --from-literal=DB_PASS=P@ssw0rd123! \
  -n todolist

kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=todouser \
  --from-literal=POSTGRES_PASSWORD=P@ssw0rd123! \
  --from-literal=POSTGRES_DB=tododb \
  -n todolist

kubectl create configmap backend-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=tododb \
  -n todolist

# Deploy
kubectl apply -f k8s/database/ -n todolist
kubectl wait --for=condition=ready pod -l app=postgres -n todolist --timeout=120s

kubectl apply -f k8s/backend/ -n todolist
kubectl wait --for=condition=ready pod -l app=backend -n todolist --timeout=60s

kubectl apply -f k8s/frontend/ -n todolist
kubectl wait --for=condition=ready pod -l app=frontend -n todolist --timeout=60s

kubectl apply -f k8s/ingress/ -n todolist
```

---

## Langkah 7: Verifikasi

```bash
# Ambil External hostname Ingress Controller
EXTERNAL_HOSTNAME=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "External Hostname: $EXTERNAL_HOSTNAME"

# Test (mungkin perlu tunggu 2-3 menit agar DNS propagasi)
curl http://$EXTERNAL_HOSTNAME/api/health

# Buka browser
open http://$EXTERNAL_HOSTNAME
```

---

## Menggunakan Amazon RDS (Rekomendasi Production)

Untuk production, disarankan menggunakan **Amazon RDS** untuk PostgreSQL alih-alih menjalankan PostgreSQL di Pod:

```bash
# Buat RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier todolist-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16.1 \
  --master-username todouser \
  --master-user-password P@ssw0rd123! \
  --db-name tododb \
  --allocated-storage 20 \
  --no-multi-az \
  --publicly-accessible false \
  --region $AWS_REGION

# Setelah RDS ready (5-10 menit), ambil endpoint
aws rds describe-db-instances \
  --db-instance-identifier todolist-postgres \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
```

Update `DB_HOST` di ConfigMap dengan endpoint RDS dan hapus StatefulSet PostgreSQL dari cluster.

---

## Cleanup (PENTING)

```bash
# Hapus aplikasi
kubectl delete namespace todolist

# Hapus Ingress Controller dan Load Balancer
helm uninstall ingress-nginx -n ingress-nginx

# Hapus EKS cluster (ini bisa memakan waktu 10-15 menit)
eksctl delete cluster --name todolist-cluster --region $AWS_REGION

# Hapus ECR repositories (jika tidak diperlukan)
aws ecr delete-repository --repository-name todolist/backend --force --region $AWS_REGION
aws ecr delete-repository --repository-name todolist/frontend --force --region $AWS_REGION

# Hapus RDS (jika dibuat)
aws rds delete-db-instance \
  --db-instance-identifier todolist-postgres \
  --skip-final-snapshot
```

> **Perhatian:** EKS cluster Node (EC2 instances) terus ditagih per jam. Selalu hapus cluster setelah selesai belajar!
