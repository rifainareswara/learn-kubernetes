# Pod

Pod adalah unit terkecil dan paling dasar yang bisa di-deploy di Kubernetes. Semua container di Kubernetes berjalan di dalam Pod.

---

## Apa itu Pod?

Pod adalah **wrapper** di sekitar satu atau beberapa container yang:
- Berbagi **IP address** yang sama
- Berbagi **storage volume** yang sama
- Memiliki **lifecycle** yang sama (start dan stop bersama)

```
┌─────────────────────────────────────────────────┐
│                     POD                          │
│  IP: 10.244.1.15                                │
│                                                  │
│  ┌──────────────────┐   ┌────────────────────┐  │
│  │   Container: app │   │ Container: sidecar │  │
│  │   image: nginx   │   │ image: log-agent   │  │
│  │   port: 80       │   │ port: 9000         │  │
│  └──────────────────┘   └────────────────────┘  │
│                                                  │
│  Volume: /data (shared antara kedua container)   │
└─────────────────────────────────────────────────┘
```

---

## Kapan Menggunakan Multiple Container dalam Satu Pod?

Pola umum yang menggunakan multiple container dalam satu Pod:

### Pola Sidecar
Container helper yang mendukung container utama:
```
Container Utama (App) + Container Sidecar (Log Collector)
Keduanya berbagi filesystem → sidecar baca log dari /var/log/app
```

### Pola Init Container
Container yang berjalan sebelum container utama:
```
Init Container (DB Migration) → selesai → Container Utama (App) mulai
```

### Pola Ambassador
Proxy yang menangani komunikasi keluar:
```
Container App → Container Ambassador (proxy) → External Service
```

---

## Perbedaan Pod vs Container

| Aspek | Container (Docker) | Pod (Kubernetes) |
|-------|-------------------|------------------|
| Unit deploy | Container | Pod |
| Networking | Docker network | Kubernetes network |
| Storage | Docker volume | Kubernetes volume |
| Lifecycle | docker start/stop | kubectl apply/delete |
| Discovery | Docker DNS | Kubernetes DNS |

---

## Perintah kubectl untuk Pod

```bash
# Lihat semua Pod
kubectl get pods

# Lihat dengan informasi lebih lengkap
kubectl get pods -o wide

# Lihat Pod di semua namespace
kubectl get pods -A

# Detail Pod
kubectl describe pod <nama-pod>

# Logs Pod
kubectl logs <nama-pod>
kubectl logs <nama-pod> -f              # Follow (real-time)
kubectl logs <nama-pod> -c <container>  # Jika ada multiple container
kubectl logs <nama-pod> --previous      # Log dari container sebelumnya

# Exec ke dalam Pod
kubectl exec -it <nama-pod> -- /bin/bash
kubectl exec -it <nama-pod> -- /bin/sh  # Jika bash tidak ada

# Port forward
kubectl port-forward pod/<nama-pod> 8080:80

# Hapus Pod
kubectl delete pod <nama-pod>
```

---

## Penjelasan YAML Pod

Lihat file [contoh-pod.yaml](./contoh-pod.yaml) untuk contoh lengkap.

### Field Penting

```yaml
apiVersion: v1          # Pod menggunakan core API v1
kind: Pod
metadata:
  name: my-pod           # Nama unik dalam namespace
  labels:                # Label untuk selection
    app: my-app
    version: v1
spec:
  containers:            # Daftar container
  - name: main-app       # Nama container (unik dalam Pod)
    image: nginx:1.25    # Docker image
    ports:
    - containerPort: 80  # Port yang diekspos container
    resources:           # Batasan resource
      requests:          # Minimum yang dijamin
        memory: "64Mi"
        cpu: "250m"
      limits:            # Maksimum yang boleh dipakai
        memory: "128Mi"
        cpu: "500m"
    env:                 # Environment variables
    - name: APP_ENV
      value: "production"
    livenessProbe:       # Health check - restart jika gagal
      httpGet:
        path: /healthz
        port: 80
    readinessProbe:      # Ready check - traffic dikirim jika lulus
      httpGet:
        path: /ready
        port: 80
```

### Resources: Request vs Limit

```
Request = "Saya butuh minimal ini untuk bisa jalan"
Limit   = "Saya tidak boleh melebihi ini"

CPU: "250m" = 0.25 CPU core (m = millicores)
Memory: "128Mi" = 128 Mebibytes
```

```
Scheduler menggunakan Request untuk menentukan Node mana
Container bisa pakai lebih dari Request, tapi tidak lebih dari Limit
```

---

## Pod Lifecycle

```
Pod Lifecycle:

Pending → ContainerCreating → Running → Succeeded/Failed

Pending:          Pod sudah di-schedule ke Node, tapi container belum start
                  (mungkin sedang pull image)
ContainerCreating: Container sedang dibuat
Running:          Semua container berjalan
Succeeded:        Semua container selesai dengan exit code 0 (untuk Job)
Failed:           Setidaknya satu container exit dengan error
Unknown:          Status Pod tidak bisa diketahui (biasanya masalah Node)
```

### Restart Policy

```yaml
spec:
  restartPolicy: Always    # Default: selalu restart jika container exit
  # restartPolicy: OnFailure  # Restart hanya jika exit dengan error
  # restartPolicy: Never      # Tidak pernah restart (untuk Job)
```

---

## Pod vs Deployment: Kapan Pakai Masing-masing?

> **Spoiler:** Di production, kamu hampir tidak pernah membuat Pod secara langsung!

| Situasi | Gunakan |
|---------|---------|
| Testing cepat | Pod langsung |
| Debugging | Pod langsung |
| Aplikasi yang perlu direplikasi | Deployment |
| Database | StatefulSet |
| Log collector di setiap node | DaemonSet |

**Kenapa tidak langsung pakai Pod?**
- Pod yang mati tidak otomatis dibuat ulang (tidak seperti Deployment)
- Tidak bisa di-scale dengan mudah
- Tidak ada rolling update

---

## Latihan

1. Buat Pod nginx sederhana menggunakan file `contoh-pod.yaml`
2. Akses Pod tersebut menggunakan port-forward
3. Buka terminal baru dan masuk ke dalam Pod menggunakan `kubectl exec`
4. Lihat log nginx menggunakan `kubectl logs`
5. Hapus Pod dan lihat apa yang terjadi

```bash
# Langkah 1: Buat Pod
kubectl apply -f contoh-pod.yaml

# Langkah 2: Cek status
kubectl get pods

# Langkah 3: Port forward
kubectl port-forward pod/nginx-pod 8080:80

# Langkah 4: Test (terminal lain)
curl http://localhost:8080

# Langkah 5: Exec
kubectl exec -it nginx-pod -- /bin/bash
ls /usr/share/nginx/html  # Lihat file nginx di dalam container

# Langkah 6: Lihat log
kubectl logs nginx-pod

# Langkah 7: Hapus
kubectl delete pod nginx-pod
# Amati: Pod tidak dibuat ulang secara otomatis!
```

---

*[Lihat contoh YAML →](./contoh-pod.yaml)*

*[Lanjut ke: ReplicaSet →](../02-replicaset/README.md)*

*[Kembali ke: Object Dasar](../README.md)*
