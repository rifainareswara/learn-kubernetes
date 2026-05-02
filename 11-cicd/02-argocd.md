# GitOps dengan ArgoCD

ArgoCD adalah declarative GitOps continuous delivery tool untuk Kubernetes. Dengan ArgoCD, Git repository menjadi sumber kebenaran (single source of truth) untuk state cluster.

---

## Apa itu GitOps?

GitOps adalah praktik di mana seluruh infrastruktur dan konfigurasi aplikasi didefinisikan sebagai kode di Git, dan sistem secara otomatis menerapkan perubahan tersebut ke cluster.

```
TANPA GitOps (Push-based):
Developer → CI/CD Pipeline → kubectl apply → Cluster
(CI server butuh akses langsung ke cluster)

DENGAN GitOps (Pull-based):
Developer → Git Push → Repository
                          ↑ sync
                       ArgoCD → Cluster
(Cluster yang "menarik" perubahan dari Git)
```

---

## Mengapa ArgoCD?

- **Git sebagai sumber kebenaran** — perubahan ke cluster HANYA melalui Git
- **Auto-sync** — cluster selalu sesuai dengan yang ada di Git
- **Drift detection** — jika ada yang berubah di cluster tanpa lewat Git, ArgoCD tahu
- **UI yang bagus** — visualisasi semua resource dan statusnya
- **RBAC** — kontrol akses yang granular
- **Multi-cluster** — kelola banyak cluster dari satu ArgoCD

---

## Install ArgoCD

```bash
# Buat namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verifikasi Pod berjalan
kubectl get pods -n argocd

# Output yang diharapkan:
# NAME                                              READY   STATUS
# argocd-application-controller-xxx                 1/1     Running
# argocd-applicationset-controller-xxx              1/1     Running
# argocd-dex-server-xxx                             1/1     Running
# argocd-notifications-controller-xxx               1/1     Running
# argocd-redis-xxx                                  1/1     Running
# argocd-repo-server-xxx                            1/1     Running
# argocd-server-xxx                                 1/1     Running
```

---

## Akses ArgoCD UI

```bash
# Method 1: Port forward (untuk testing)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Buka https://localhost:8080
# Username: admin

# Dapatkan password awal
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""

# Method 2: Expose via LoadBalancer
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Method 3: Expose via Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF
```

---

## Install ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# Ganti password
argocd account update-password
```

---

## Struktur Repository untuk GitOps

Ada dua pendekatan struktur repo:

### Pendekatan 1: Monorepo (semua dalam satu repo)

```
my-infrastructure-repo/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── service.yaml
├── namespaces/
│   ├── production.yaml
│   └── staging.yaml
└── argocd/
    └── applications.yaml        ← Definisi ArgoCD Application
```

### Pendekatan 2: App of Apps

```
infrastructure-repo/
└── argocd/
    ├── app-of-apps.yaml         ← Satu Application yang manage semua
    └── applications/
        ├── frontend-app.yaml
        ├── backend-app.yaml
        └── database-app.yaml
```

---

## Membuat ArgoCD Application

### Via UI
1. Buka ArgoCD UI
2. **+ NEW APP**
3. Isi form:
   - **Application Name:** my-app
   - **Project:** default
   - **Repository URL:** https://github.com/myusername/my-k8s-manifests
   - **Path:** apps/my-app
   - **Cluster:** (cluster lokal)
   - **Namespace:** production
4. **Create**

### Via YAML Manifest

```yaml
# my-app-argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd          # Application resource selalu di namespace argocd

spec:
  project: default           # ArgoCD project

  # Source: dari mana manifest diambil
  source:
    repoURL: https://github.com/myusername/my-k8s-manifests
    targetRevision: HEAD      # Branch/tag yang digunakan
    path: apps/my-app         # Path dalam repository

    # Jika menggunakan Helm:
    # helm:
    #   valueFiles:
    #   - values.yaml
    #   - values-production.yaml
    #   parameters:
    #   - name: image.tag
    #     value: v1.2.3

    # Jika menggunakan Kustomize:
    # kustomize:
    #   images:
    #   - myapp=myregistry/myapp:v1.2.3

  # Destination: deploy ke mana
  destination:
    server: https://kubernetes.default.svc    # Cluster lokal
    namespace: production

  # Sync policy
  syncPolicy:
    automated:                  # Auto-sync: langsung sync saat ada perubahan di Git
      prune: true               # Hapus resource yang tidak ada di Git
      selfHeal: true            # Sync kembali jika ada perubahan di cluster di luar Git

    syncOptions:
    - CreateNamespace=true      # Buat namespace jika belum ada
    - PrunePropagationPolicy=foreground  # Hapus resource anak sebelum induk
```

```bash
# Apply Application
kubectl apply -f my-app-argocd.yaml

# Sync manual (jika auto-sync dimatikan)
argocd app sync my-app

# Lihat status
argocd app get my-app
```

---

## App of Apps Pattern

Satu Application yang mengelola banyak Application lain:

```yaml
# apps-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myusername/my-k8s-manifests
    targetRevision: HEAD
    path: argocd/applications    # Path yang berisi semua Application YAML
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Alur Lengkap GitOps dengan ArgoCD

```
1. Developer push kode baru ke GitHub
           ↓
2. GitHub Actions build image baru
   → Docker image: myapp:main-abc1234
           ↓
3. GitHub Actions update image tag di k8s-manifests repo
   → Edit deployment.yaml: image: myapp:main-abc1234
   → Commit dan push ke k8s-manifests repo
           ↓
4. ArgoCD deteksi ada perubahan di Git
   → Compare current state vs desired state
           ↓
5. ArgoCD sync (apply) perubahan ke cluster
   → kubectl apply deployment.yaml
           ↓
6. Rolling update berjalan di cluster
           ↓
7. ArgoCD tampilkan status "Synced" dan "Healthy"
```

### Otomasi Update Manifest dengan GitHub Actions

```yaml
# Tambahkan step ini ke workflow GitHub Actions setelah push image:

- name: Update Kubernetes manifest
  run: |
    SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
    NEW_IMAGE="myuser/my-app:main-${SHORT_SHA}"
    
    # Clone manifest repo
    git clone https://x-access-token:${{ secrets.MANIFEST_REPO_TOKEN }}@github.com/myusername/k8s-manifests
    cd k8s-manifests
    
    # Update image tag menggunakan sed atau yq
    # Menggunakan yq (lebih aman dari sed):
    yq e ".spec.template.spec.containers[0].image = \"${NEW_IMAGE}\"" \
      -i apps/my-app/deployment.yaml
    
    # Commit dan push
    git config user.email "actions@github.com"
    git config user.name "GitHub Actions"
    git add .
    git commit -m "chore: update my-app to ${SHORT_SHA}"
    git push
```

---

## Rollback dengan ArgoCD

```bash
# Lihat history aplikasi
argocd app history my-app

# Rollback ke revision tertentu
argocd app rollback my-app <revision-number>

# Atau via Git: revert commit di manifest repo
git revert HEAD
git push
# ArgoCD akan auto-sync kembali ke state sebelumnya
```

---

## Monitoring ArgoCD

ArgoCD sudah menyediakan metrics untuk Prometheus:

```bash
# ArgoCD Metrics
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082
curl http://localhost:8082/metrics

# Application controller metrics
kubectl port-forward svc/argocd-application-controller-metrics -n argocd 8084:8084
```

---

## Cleanup

```bash
# Hapus semua Application
argocd app delete my-app
argocd app delete apps-of-apps

# Uninstall ArgoCD
kubectl delete -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

---

*[Kembali ke: GitHub Actions ←](./01-github-actions.md)*

*[Kembali ke: Overview CI/CD](./README.md)*

*[Kembali ke: README Utama →](../README.md)*
