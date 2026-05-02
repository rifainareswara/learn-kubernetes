# Install kubectl

`kubectl` adalah command-line tool (CLI) untuk berinteraksi dengan cluster Kubernetes. Ini adalah tools pertama yang harus kamu install.

---

## Apa itu kubectl?

kubectl (dibaca: "kube-control", "kube-cuddle", atau "kube-ctl") adalah cara utama kamu berkomunikasi dengan cluster Kubernetes. Hampir semua yang bisa dilakukan via Kubernetes Dashboard bisa dilakukan lebih cepat dengan kubectl.

```
Kamu → kubectl → API Server → etcd/komponen lain
```

---

## Install di macOS

### Menggunakan Homebrew (Direkomendasikan)

```bash
# Install kubectl
brew install kubectl

# Atau via kubernetes-cli
brew install kubernetes-cli

# Verifikasi
kubectl version --client
```

### Menggunakan curl

```bash
# Cari versi terbaru
KUBECTL_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)

# Download binary
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/darwin/amd64/kubectl"
# Untuk Apple Silicon (M1/M2/M3):
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/darwin/arm64/kubectl"

# Beri permission execute
chmod +x kubectl

# Pindahkan ke PATH
sudo mv kubectl /usr/local/bin/kubectl

# Verifikasi
kubectl version --client
```

---

## Install di Linux

### Menggunakan curl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Beri permission execute
chmod +x kubectl

# Pindahkan ke PATH
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verifikasi
kubectl version --client
```

### Menggunakan Package Manager

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl

# CentOS/RHEL/Fedora
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubectl
```

---

## Install di Windows

### Menggunakan Chocolatey

```powershell
choco install kubernetes-cli
```

### Menggunakan Winget

```powershell
winget install -e --id Kubernetes.kubectl
```

### Manual

1. Download kubectl dari: https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe
2. Simpan ke folder yang ada di PATH (misalnya `C:\Windows\System32`)
3. Buka Command Prompt baru dan jalankan `kubectl version --client`

---

## Verifikasi Instalasi

```bash
# Cek versi kubectl (client)
kubectl version --client

# Output yang diharapkan:
# Client Version: version.Info{Major:"1", Minor:"28", ...}
# Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

# Cek apakah kubectl bisa konek ke cluster (setelah setup cluster)
kubectl cluster-info
```

---

## Konfigurasi kubectl

kubectl menggunakan file konfigurasi yang disebut **kubeconfig**. Defaultnya ada di `~/.kube/config`.

### Struktur kubeconfig

```yaml
apiVersion: v1
kind: Config

# Daftar cluster yang bisa diakses
clusters:
- name: minikube
  cluster:
    server: https://192.168.49.2:8443
    certificate-authority: /home/user/.minikube/ca.crt

# Daftar user/credential
users:
- name: minikube
  user:
    client-certificate: /home/user/.minikube/profiles/minikube/client.crt
    client-key: /home/user/.minikube/profiles/minikube/client.key

# Context = kombinasi cluster + user + namespace
contexts:
- name: minikube
  context:
    cluster: minikube
    user: minikube
    namespace: default

# Context yang sedang aktif
current-context: minikube
```

### Perintah Manajemen Context

```bash
# Lihat semua context yang tersedia
kubectl config get-contexts

# Lihat context yang sedang aktif
kubectl config current-context

# Pindah ke context lain
kubectl config use-context <nama-context>

# Lihat isi kubeconfig
kubectl config view

# Contoh dengan banyak cluster:
# kubectl config use-context minikube         # pindah ke lokal
# kubectl config use-context gke-production   # pindah ke GKE production
```

---

## Perintah kubectl Paling Sering Dipakai

### Melihat resource

```bash
# Lihat semua Pod
kubectl get pods

# Lihat dengan lebih banyak info
kubectl get pods -o wide

# Lihat semua namespace
kubectl get pods --all-namespaces
# atau
kubectl get pods -A

# Lihat dalam namespace tertentu
kubectl get pods -n kube-system

# Watch (update otomatis)
kubectl get pods --watch
# atau
kubectl get pods -w

# Output JSON/YAML
kubectl get pod my-pod -o yaml
kubectl get pod my-pod -o json
```

### Membuat dan mengelola resource

```bash
# Apply manifest (buat atau update)
kubectl apply -f file.yaml
kubectl apply -f direktori/

# Buat resource langsung (imperatif - untuk testing)
kubectl run nginx --image=nginx
kubectl create deployment my-app --image=nginx --replicas=3

# Hapus resource
kubectl delete pod my-pod
kubectl delete -f file.yaml
kubectl delete deployment my-deployment

# Scale
kubectl scale deployment my-app --replicas=5
```

### Debugging

```bash
# Lihat detail resource
kubectl describe pod my-pod
kubectl describe node my-node

# Lihat logs
kubectl logs my-pod
kubectl logs my-pod -c container-name  # jika ada multiple container
kubectl logs my-pod -f                 # follow (real-time)
kubectl logs my-pod --previous         # log dari run sebelumnya

# Exec ke dalam container
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -c container-name -- /bin/sh

# Port forward (akses service dari lokal)
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward service/my-service 8080:80

# Lihat events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
```

---

## Setup Auto-completion (Sangat Direkomendasikan!)

Auto-completion membuat penggunaan kubectl jauh lebih cepat:

```bash
# Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Fish
kubectl completion fish | source

# Alias yang berguna (tambahkan ke ~/.bashrc atau ~/.zshrc)
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kdp='kubectl describe pod'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
```

---

## Tools Pelengkap yang Sangat Berguna

### kubectx & kubens
Mempermudah switch antara cluster dan namespace:

```bash
# Install
brew install kubectx   # macOS
# atau
sudo apt install kubectx  # Ubuntu

# Gunakan
kubectx             # Lihat semua context
kubectx minikube    # Pindah ke context minikube
kubens              # Lihat semua namespace
kubens default      # Pindah ke namespace default
```

### k9s
TUI (Terminal UI) yang sangat bagus untuk monitoring Kubernetes:

```bash
# Install
brew install k9s   # macOS
# atau download dari: https://github.com/derailed/k9s/releases

# Jalankan
k9s
```

---

## Troubleshooting Umum

**Error: `kubectl: command not found`**
```bash
# Pastikan binary ada di PATH
echo $PATH
which kubectl

# Jika tidak ada, tambahkan ke PATH
export PATH=$PATH:/usr/local/bin
```

**Error: `Unable to connect to the server`**
```bash
# Cluster belum berjalan atau kubeconfig salah
kubectl config view   # Cek konfigurasi
kubectl cluster-info  # Cek koneksi

# Jika pakai Minikube:
minikube status
minikube start
```

**Error: `error: no server found for cluster "minikube"`**
```bash
# Kubeconfig mungkin corrupt atau context salah
kubectl config get-contexts
kubectl config use-context minikube
```

---

*[Lanjut ke: Setup Minikube →](./02-setup-minikube.md)*

*[Kembali ke: Overview Setup Lokal](./README.md)*
