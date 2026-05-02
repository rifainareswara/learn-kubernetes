# Deployment

Deployment adalah object Kubernetes yang paling sering digunakan untuk menjalankan aplikasi stateless. Ini adalah cara "standar" deploy aplikasi di Kubernetes.

---

## Apa itu Deployment?

Deployment adalah controller yang:
- **Mengelola ReplicaSet** (yang pada gilirannya mengelola Pod)
- **Rolling update** — update aplikasi secara bertahap tanpa downtime
- **Rollback** — balik ke versi sebelumnya jika ada masalah
- **History** — menyimpan riwayat perubahan

```
Deployment (my-app v2 update)
│
├── ReplicaSet v1 (lama)   → scale down dari 3 ke 0
│   ├── Pod-1 (v1)  
│   ├── Pod-2 (v1)  
│   └── Pod-3 (v1)  
│
└── ReplicaSet v2 (baru)   → scale up dari 0 ke 3
    ├── Pod-4 (v2)  
    ├── Pod-5 (v2)  
    └── Pod-6 (v2)  
```

---

## Hierarki Deployment

```
Deployment
└── mengelola
    └── ReplicaSet (dibuat otomatis)
        └── mengelola
            └── Pod (dibuat otomatis)
```

Kamu tidak perlu membuat ReplicaSet secara manual — Deployment yang membuatnya!

---

## Strategi Update

### RollingUpdate (Default)
Update Pod satu per satu, sehingga selalu ada Pod yang berjalan:

```
Sebelum update (3 Pod v1):
[Pod v1] [Pod v1] [Pod v1]

Proses rolling update:
[Pod v1] [Pod v1] [Pod v2]  ← Pod v2 pertama dibuat
[Pod v1] [Pod v2] [Pod v2]  ← Pod v1 pertama dihapus
[Pod v2] [Pod v2] [Pod v2]  ← Selesai!

Tidak ada downtime!
```

### Recreate
Hapus semua Pod lama sebelum buat Pod baru (ada downtime):

```
[Pod v1] [Pod v1] [Pod v1] → semua dihapus → [Pod v2] [Pod v2] [Pod v2]
                                ↑ downtime!
```

---

## Perintah kubectl untuk Deployment

```bash
# Buat Deployment
kubectl create deployment my-app --image=nginx:1.25 --replicas=3

# Atau dari file
kubectl apply -f contoh-deployment.yaml

# Lihat Deployment
kubectl get deployments
kubectl get deploy  # shorthand

# Detail Deployment
kubectl describe deployment my-app

# Lihat status update
kubectl rollout status deployment/my-app

# Update image (trigger rolling update)
kubectl set image deployment/my-app nginx=nginx:1.26

# Atau edit langsung
kubectl edit deployment my-app

# Lihat history rollout
kubectl rollout history deployment/my-app

# Rollback ke versi sebelumnya
kubectl rollout undo deployment/my-app

# Rollback ke versi spesifik
kubectl rollout undo deployment/my-app --to-revision=2

# Pause rolling update (untuk debug)
kubectl rollout pause deployment/my-app

# Resume rolling update
kubectl rollout resume deployment/my-app

# Scale
kubectl scale deployment my-app --replicas=5

# Hapus Deployment (Pod ikut terhapus)
kubectl delete deployment my-app
```

---

## Monitoring Rolling Update

```bash
# Terminal 1: Watch pods
kubectl get pods -w

# Terminal 2: Trigger update
kubectl set image deployment/my-app nginx=nginx:1.26

# Terminal 1 akan menampilkan:
# NAME                       READY   STATUS              RESTARTS   AGE
# my-app-xxx-yyy             1/1     Running             0          5m
# my-app-xxx-yyy             1/1     Running             0          5m
# my-app-xxx-yyy             1/1     Running             0          5m
# my-app-zzz-aaa             0/1     Pending             0          0s   ← Pod baru
# my-app-zzz-aaa             0/1     ContainerCreating   0          0s
# my-app-zzz-aaa             1/1     Running             0          3s
# my-app-xxx-yyy             1/1     Terminating         0          5m   ← Pod lama
```

---

## Penjelasan Penting: maxSurge dan maxUnavailable

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Boleh ada MAKSIMAL 1 Pod LEBIH dari desired (4 Pod saat update 3→3)
    maxUnavailable: 0  # Tidak boleh ada Pod yang TIDAK AVAILABLE (zero-downtime)
```

Contoh dengan `replicas: 3, maxSurge: 1, maxUnavailable: 0`:
```
Step 1: Buat 1 Pod baru → [v1][v1][v1][v2]  (4 Pod)
Step 2: Hapus 1 Pod lama → [v1][v1][v2]     (3 Pod, tapi v2 sudah siap)
Step 3: Buat 1 Pod baru → [v1][v1][v2][v2]  (4 Pod)
Step 4: Hapus 1 Pod lama → [v1][v2][v2]     (3 Pod)
... dan seterusnya
```

---

## Latihan

```bash
# 1. Deploy aplikasi
kubectl apply -f contoh-deployment.yaml

# 2. Lihat status
kubectl get deployment
kubectl get pods
kubectl get replicasets

# 3. Coba rolling update
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# 4. Pantau proses update
kubectl rollout status deployment/nginx-deployment

# 5. Lihat history
kubectl rollout history deployment/nginx-deployment

# 6. Simulasi masalah: update ke image yang tidak ada
kubectl set image deployment/nginx-deployment nginx=nginx:versi-tidak-ada

# 7. Lihat Pod error
kubectl get pods
kubectl describe pod <pod-error>

# 8. Rollback!
kubectl rollout undo deployment/nginx-deployment

# 9. Verifikasi
kubectl get pods
kubectl rollout status deployment/nginx-deployment

# 10. Scale
kubectl scale deployment/nginx-deployment --replicas=5
kubectl get pods

# 11. Cleanup
kubectl delete deployment nginx-deployment
```

---

*[Lihat contoh YAML →](./contoh-deployment.yaml)*

*[Lanjut ke: Service →](../04-service/README.md)*

*[Kembali ke: Object Dasar](../README.md)*
