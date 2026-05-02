# Namespace

Namespace adalah cara untuk membagi satu cluster Kubernetes menjadi beberapa "ruang kerja" virtual yang terisolasi secara logis.

---

## Mengapa Namespace Penting?

Bayangkan kamu punya tim besar yang bekerja pada banyak proyek. Tanpa namespace:
- Semua resource tercampur jadi satu
- Sulit menentukan resource mana milik proyek apa
- Developer bisa tidak sengaja menghapus resource orang lain
- Tidak bisa set resource quota per tim

Dengan namespace:
```
CLUSTER
├── Namespace: development      ← Tim dev bekerja di sini
│   ├── my-app (3 Pod)
│   └── my-db (1 Pod)
├── Namespace: staging           ← Environment testing
│   ├── my-app (2 Pod)
│   └── my-db (1 Pod)
└── Namespace: production        ← Aplikasi live
    ├── my-app (10 Pod)
    └── my-db (3 Pod)
```

---

## Namespace Default di Kubernetes

Saat cluster baru dibuat, ada beberapa namespace bawaan:

```bash
kubectl get namespaces

# OUTPUT:
# NAME              STATUS   AGE
# default           Active   10d   ← Namespace default jika tidak ditentukan
# kube-node-lease   Active   10d   ← Untuk node heartbeat
# kube-public       Active   10d   ← Resource yang bisa dibaca oleh semua
# kube-system       Active   10d   ← Komponen Kubernetes itu sendiri
```

```bash
# Lihat Pod sistem Kubernetes
kubectl get pods -n kube-system
# OUTPUT: API server, etcd, CoreDNS, kube-proxy, dll
```

---

## Perintah kubectl untuk Namespace

```bash
# Lihat semua namespace
kubectl get namespaces
kubectl get ns  # shorthand

# Buat namespace
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Hapus namespace (HATI-HATI: semua resource di dalamnya ikut terhapus!)
kubectl delete namespace development

# Jalankan perintah di namespace tertentu
kubectl get pods -n production
kubectl get pods --namespace=production

# Lihat resource di semua namespace
kubectl get pods --all-namespaces
kubectl get pods -A

# Set namespace default untuk kubectl
kubectl config set-context --current --namespace=development
# Sekarang semua perintah akan menggunakan namespace "development"

# Cek namespace aktif
kubectl config view --minify | grep namespace:
```

---

## Isolasi Resource dengan Namespace

### ResourceQuota: Batasi penggunaan resource per namespace

```yaml
# Lihat contoh-namespace.yaml untuk detail
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "10"                   # Maksimal 10 Pod
    requests.cpu: "4"            # Maksimal 4 CPU total untuk requests
    requests.memory: 8Gi         # Maksimal 8Gi RAM untuk requests
    limits.cpu: "8"              # Maksimal 8 CPU total untuk limits
    limits.memory: 16Gi          # Maksimal 16Gi RAM untuk limits
```

### LimitRange: Set default resource per Pod/Container

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - default:                    # Default LIMIT jika tidak ditentukan
      memory: 256Mi
      cpu: 500m
    defaultRequest:             # Default REQUEST jika tidak ditentukan
      memory: 128Mi
      cpu: 100m
    type: Container
```

---

## DNS Antar Namespace

Komunikasi antar namespace menggunakan DNS format lengkap:

```
Format: <service-name>.<namespace>.svc.cluster.local

Contoh:
# Dari namespace "frontend" akses database di namespace "backend":
postgresql-service.backend.svc.cluster.local

# Dari namespace yang SAMA:
postgresql-service           ← cukup nama service saja

# Dari namespace yang BERBEDA:
postgresql-service.backend   ← butuh tambahkan nama namespace
# atau lengkap:
postgresql-service.backend.svc.cluster.local
```

---

## Best Practices Namespace

1. **Pisahkan environment:** `development`, `staging`, `production`
2. **Pisahkan per tim:** `team-frontend`, `team-backend`, `team-data`
3. **Selalu set namespace saat apply:** hindari bergantung pada default
4. **Gunakan ResourceQuota** untuk mencegah satu tim monopoli resource
5. **Gunakan NetworkPolicy** untuk isolasi jaringan antar namespace

---

## Latihan

```bash
# 1. Buat namespace development
kubectl apply -f contoh-namespace.yaml

# 2. Deploy nginx di namespace development
kubectl run nginx --image=nginx -n development

# 3. Lihat Pod di namespace tertentu
kubectl get pods -n development

# 4. Lihat semua Pod di semua namespace
kubectl get pods -A

# 5. Coba akses Pod dari namespace lain
kubectl run test --image=curlimages/curl -n staging --rm -it --restart=Never -- \
  curl nginx.development.svc.cluster.local

# 6. Lihat quota usage
kubectl describe resourcequota -n development

# 7. Cleanup
kubectl delete namespace development
```

---

*[Lihat contoh YAML →](./contoh-namespace.yaml)*

*[Lanjut ke: ConfigMap →](../02-configmap/README.md)*

*[Kembali ke: Konfigurasi](../README.md)*
