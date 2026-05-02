# Project 01: Deploy Nginx Sederhana

Project pertama ini akan membantumu memahami alur dasar deploy aplikasi ke Kubernetes menggunakan Deployment dan Service.

---

## Arsitektur

```
                    Internet / Browser
                          │
                          │ http://localhost:30080
                          ▼
                    ┌───────────┐
                    │  Service  │
                    │ (NodePort)│
                    │ Port 30080│
                    └─────┬─────┘
                          │ load balance
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Pod     │ │  Pod     │ │  Pod     │
        │  nginx   │ │  nginx   │ │  nginx   │
        │  :80     │ │  :80     │ │  :80     │
        └──────────┘ └──────────┘ └──────────┘
        
        Deployment: nginx-deployment (replicas: 3)
```

---

## Yang Akan Dipelajari

- Membuat Deployment dengan 3 replicas
- Membuat Service NodePort
- Verifikasi load balancing antar Pod
- Melakukan rolling update
- Rollback ke versi sebelumnya
- Scale up/down

---

## Langkah-langkah Deploy

### Step 1: Pastikan cluster berjalan

```bash
kubectl cluster-info
kubectl get nodes
```

### Step 2: Deploy aplikasi

```bash
kubectl apply -f manifest.yaml
```

### Step 3: Verifikasi Deployment

```bash
# Lihat Deployment
kubectl get deployments
# Output yang diharapkan:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           30s

# Lihat Pods
kubectl get pods
# Output yang diharapkan:
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-xxx-yyy            1/1     Running   0          30s
# nginx-deployment-xxx-zzz            1/1     Running   0          30s
# nginx-deployment-xxx-aaa            1/1     Running   0          30s

# Lihat Service
kubectl get services
# NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
# nginx-service    NodePort   10.96.10.5     <none>        80:30080/TCP
```

### Step 4: Akses aplikasi

```bash
# Dengan Minikube
minikube service nginx-service --url
# Buka URL yang muncul di browser

# Atau port-forward
kubectl port-forward service/nginx-service 8080:80
# Buka http://localhost:8080

# Dengan Kind (port sudah di-forward saat buat cluster)
curl http://localhost:30080
```

### Step 5: Verifikasi Load Balancing

```bash
# Buka beberapa terminal dan jalankan request berkali-kali
for i in {1..10}; do
  curl -s http://localhost:8080 | grep "Server name"
done
# Harusnya berbeda-beda (masing-masing Pod punya hostname berbeda)
```

### Step 6: Lihat log real-time saat request masuk

```bash
# Terminal 1: watch logs semua pod
kubectl logs -l app=nginx -f --max-log-requests=10

# Terminal 2: kirim requests
curl http://localhost:8080
```

---

## Eksperimen Rolling Update

```bash
# Update ke nginx versi baru
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Pantau proses update
kubectl rollout status deployment/nginx-deployment

# Lihat history
kubectl rollout history deployment/nginx-deployment

# Rollback jika ada masalah
kubectl rollout undo deployment/nginx-deployment
```

## Eksperimen Scaling

```bash
# Scale up ke 5 replicas
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods  # Lihat 5 Pod sekarang

# Scale down ke 1 replica
kubectl scale deployment nginx-deployment --replicas=1
kubectl get pods  # Hanya 1 Pod

# Kembalikan ke 3
kubectl scale deployment nginx-deployment --replicas=3
```

## Eksperimen Self-Healing

```bash
# Hapus satu Pod secara paksa
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD

# Langsung lihat — Pod baru dibuat otomatis!
kubectl get pods -w
```

---

## Checklist Verifikasi

- [ ] `kubectl get deployments` menampilkan READY: 3/3
- [ ] `kubectl get pods` menampilkan 3 Pod dengan status Running
- [ ] Bisa akses nginx di browser atau curl
- [ ] Rolling update berhasil tanpa downtime
- [ ] Rollback berhasil kembali ke versi sebelumnya
- [ ] Self-healing: Pod dihapus → dibuat ulang otomatis

---

## Cleanup

```bash
kubectl delete -f manifest.yaml
# Atau satu per satu:
kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
```

---

*[Lihat manifest.yaml →](./manifest.yaml)*

*[Lanjut ke: Project 02 - App dengan Database →](../02-app-dengan-database/README.md)*
