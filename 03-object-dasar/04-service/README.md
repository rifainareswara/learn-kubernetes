# Service

Service adalah abstraksi yang mendefinisikan cara mengakses sekumpulan Pod. Service memberikan IP dan DNS name yang stabil, meskipun Pod di belakangnya bisa berubah-ubah.

---

## Mengapa Service Dibutuhkan?

Pod bersifat **ephemeral** (sementara) — Pod bisa mati dan diganti kapan saja. Setiap Pod baru mendapat IP address baru. Jika aplikasi A perlu berkomunikasi dengan aplikasi B (3 Pod), bagaimana caranya?

```
Tanpa Service (MASALAH):
App A → Pod B (IP: 10.244.1.5)   ← IP berubah saat Pod restart!
     → Pod B (IP: 10.244.2.3)
     → Pod B (IP: 10.244.1.8)

Dengan Service (SOLUSI):
App A → Service B (IP stabil: 10.96.10.1, DNS: b-service)
              │
              ├─ Pod B (10.244.1.5)
              ├─ Pod B (10.244.2.3)  ← load balanced
              └─ Pod B (10.244.1.8)
```

---

## Jenis-Jenis Service

### 1. ClusterIP (Default)
Hanya bisa diakses dari dalam cluster:
```
Internal Service
┌────────────────────────────────────┐
│              CLUSTER               │
│                                    │
│  Pod A ──► ClusterIP Service ──► Pod B  │
│            10.96.10.1:80           │
│                                    │
│  (Tidak bisa diakses dari luar!)   │
└────────────────────────────────────┘
```

**Gunakan untuk:** Komunikasi antar service dalam cluster (database, API internal)

### 2. NodePort
Mengekspos service di setiap Node dengan port tertentu (30000-32767):
```
External Access via NodePort
                         
Internet ──► NodeIP:NodePort ──► Service ──► Pod
             (misal 192.168.1.10:30080)
```

**Gunakan untuk:** Development, testing, akses langsung ke Pod

### 3. LoadBalancer
Membuat external load balancer (di cloud provider):
```
Internet ──► Cloud LB IP ──► Service ──► Pod
             (IP publik yang diberikan cloud)
```

**Gunakan untuk:** Mengekspos aplikasi ke internet di cloud environment

### 4. ExternalName
Memetakan Service ke nama DNS eksternal:
```
Pod ──► ExternalName Service ──► external.database.com
```

**Gunakan untuk:** Akses external service menggunakan nama dalam cluster

---

## Bagaimana Service Menemukan Pod?

Service menggunakan **label selector** untuk menemukan Pod yang dilayaninya:

```yaml
# Service
spec:
  selector:
    app: nginx    # ← "Kirim traffic ke Pod dengan label ini"

# Pod
metadata:
  labels:
    app: nginx    # ← Cocok! Pod ini akan menerima traffic
```

---

## Endpoints

Ketika Service dibuat, Kubernetes otomatis membuat object **Endpoints** yang berisi IP Pod yang cocok dengan selector:

```bash
# Lihat endpoints
kubectl get endpoints
kubectl describe service my-service

# Output:
# Name:              my-service
# Namespace:         default
# Selector:          app=nginx
# IP:                10.96.10.1
# Port:              80/TCP
# TargetPort:        80/TCP
# Endpoints:         10.244.1.5:80,10.244.2.3:80,10.244.1.8:80
```

---

## DNS Service Kubernetes

Kubernetes menyediakan DNS built-in (CoreDNS). Format DNS untuk Service:

```
<service-name>.<namespace>.svc.cluster.local

Contoh:
nginx-service.default.svc.cluster.local
db-service.production.svc.cluster.local

# Dari namespace yang sama, cukup pakai nama service:
nginx-service        ← dari namespace yang sama
# Dari namespace berbeda, perlu nama lengkap:
nginx-service.default
```

---

## Perintah kubectl untuk Service

```bash
# Lihat semua Service
kubectl get services
kubectl get svc  # shorthand

# Detail Service
kubectl describe service my-service

# Buat Service dari Deployment yang sudah ada
kubectl expose deployment my-app --type=ClusterIP --port=80

# Port forward (akses dari lokal)
kubectl port-forward service/my-service 8080:80

# Hapus Service
kubectl delete service my-service
```

---

## Latihan

```bash
# 1. Buat Deployment nginx
kubectl apply -f ../03-deployment/contoh-deployment.yaml

# 2. Buat semua jenis Service
kubectl apply -f contoh-clusterip.yaml
kubectl apply -f contoh-nodeport.yaml

# 3. Lihat semua Service
kubectl get services

# 4. Test ClusterIP (dari dalam cluster)
kubectl run test-pod --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl http://nginx-clusterip-service

# 5. Test NodePort (dari luar cluster)
# Dengan Minikube:
minikube service nginx-nodeport-service --url

# 6. Lihat Endpoints
kubectl get endpoints

# 7. Lihat Pod yang dipilih Service
kubectl get pods -l app=nginx

# 8. Cleanup
kubectl delete service nginx-clusterip-service nginx-nodeport-service
kubectl delete deployment nginx-deployment
```

---

*[Lihat contoh ClusterIP →](./contoh-clusterip.yaml)*

*[Lihat contoh NodePort →](./contoh-nodeport.yaml)*

*[Lihat contoh LoadBalancer →](./contoh-loadbalancer.yaml)*

*[Lanjut ke: Konfigurasi →](../../04-konfigurasi/README.md)*

*[Kembali ke: Object Dasar](../README.md)*
