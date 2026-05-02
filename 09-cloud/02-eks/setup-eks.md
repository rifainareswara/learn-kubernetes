# Setup Amazon Elastic Kubernetes Service (EKS)

Panduan lengkap untuk membuat cluster Kubernetes di AWS menggunakan EKS.

---

## Prasyarat

- Akun AWS dengan billing aktif
- AWS CLI v2 terinstall
- `eksctl` terinstall (tool resmi untuk EKS)
- `kubectl` terinstall

---

## Step 1: Install AWS CLI dan eksctl

### Install AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verifikasi
aws --version
```

### Install eksctl

```bash
# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verifikasi
eksctl version
```

---

## Step 2: Konfigurasi AWS CLI

```bash
# Setup AWS credentials
aws configure
# Masukkan:
# AWS Access Key ID: (dari IAM user)
# AWS Secret Access Key: (dari IAM user)
# Default region name: ap-southeast-1  (Singapore)
# Default output format: json

# Verifikasi
aws sts get-caller-identity
# Output: UserId, Account, Arn
```

### Buat IAM User untuk EKS (best practice)

```bash
# Buat user khusus EKS
aws iam create-user --user-name eks-admin

# Attach policy
aws iam attach-user-policy \
  --user-name eks-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Buat access key
aws iam create-access-key --user-name eks-admin
# Simpan output! Ini hanya muncul sekali.

# Gunakan key baru
aws configure --profile eks-admin
export AWS_PROFILE=eks-admin
```

---

## Step 3: Buat EKS Cluster

### Cara 1: Menggunakan eksctl (Direkomendasikan)

```bash
# Buat cluster paling sederhana
eksctl create cluster \
  --name my-cluster \
  --region ap-southeast-1 \
  --nodes 2

# Atau dengan konfigurasi lebih detail:
eksctl create cluster \
  --name dev-cluster \
  --region ap-southeast-1 \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed                    # Managed node group (AWS yang patch OS)

# Tunggu 10-15 menit... (EKS lebih lama dari GKE)

# Lihat cluster
eksctl get cluster
```

### Cara 2: Menggunakan file konfigurasi

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: ap-southeast-1
  version: "1.28"

iam:
  withOIDC: true              # Diperlukan untuk IRSA (IAM Roles for Service Accounts)

managedNodeGroups:
- name: general-workers
  instanceType: t3.medium
  desiredCapacity: 2
  minSize: 1
  maxSize: 5
  amiFamily: AmazonLinux2
  iam:
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
      - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

- name: spot-workers            # Spot instances untuk hemat biaya
  instanceTypes: ["t3.medium", "t3.large"]
  spot: true
  desiredCapacity: 1
  minSize: 0
  maxSize: 10

addons:
- name: vpc-cni
- name: coredns
- name: kube-proxy
- name: aws-ebs-csi-driver      # Untuk PersistentVolume dengan EBS
```

```bash
eksctl create cluster -f cluster-config.yaml
```

---

## Step 4: Connect kubectl ke EKS

```bash
# Update kubeconfig secara otomatis
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name my-cluster

# Verifikasi
kubectl cluster-info
kubectl get nodes
```

---

## Step 5: Deploy Aplikasi

```bash
# Sama seperti cluster lokal!
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Contoh dengan LoadBalancer (menggunakan AWS ALB)
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx \
  --type=LoadBalancer \
  --port=80

# Tunggu LoadBalancer (bisa 2-3 menit)
kubectl get service nginx --watch
# EXTERNAL-IP akan berisi DNS AWS seperti: xxx.ap-southeast-1.elb.amazonaws.com

# Test
curl http://xxx.ap-southeast-1.elb.amazonaws.com
```

---

## Step 6: Setup AWS Load Balancer Controller

AWS Load Balancer Controller lebih powerful dari default LoadBalancer:

```bash
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Buat service account dengan IAM role
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## Step 7: Auto-scaling

```bash
# Install Cluster Autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=ap-southeast-1

# Setup HPA
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70
```

---

## Menggunakan Amazon ECR (Private Registry)

```bash
# Login ke ECR
aws ecr get-login-password --region ap-southeast-1 | \
  docker login --username AWS \
  --password-stdin ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com

# Buat repository
aws ecr create-repository \
  --repository-name my-app \
  --region ap-southeast-1

# Build dan push image
docker build -t my-app:v1 .
docker tag my-app:v1 ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/my-app:v1
docker push ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/my-app:v1
```

---

## Cleanup (PENTING!)

```bash
# Hapus semua resource Kubernetes
kubectl delete all --all -A

# Hapus PVC (data di EBS)
kubectl delete pvc --all -A

# Hapus cluster (ini juga hapus Node group dan VPC yang dibuat eksctl)
eksctl delete cluster --name my-cluster --region ap-southeast-1

# Verifikasi
eksctl get cluster --region ap-southeast-1
aws ec2 describe-instances --region ap-southeast-1 --filters "Name=tag:eks:cluster-name,Values=my-cluster"
```

---

## Biaya Estimasi

| Resource | Estimasi/Bulan |
|----------|---------------|
| EKS Control Plane | $73 |
| 2x t3.medium Node | ~$60 |
| EBS 50GB per node | ~$10 |
| Load Balancer | ~$20 |
| **Total** | **~$163** |

> Gunakan **Spot Instances** untuk hemat 60-70% biaya Node.

---

*[Kembali ke: Cloud Overview](../README.md)*
