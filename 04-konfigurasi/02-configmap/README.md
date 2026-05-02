# ConfigMap

ConfigMap adalah object Kubernetes untuk menyimpan data konfigurasi **non-sensitif** dalam bentuk key-value. Memungkinkan kamu memisahkan konfigurasi dari Docker image.

---

## Apa itu ConfigMap?

ConfigMap menyimpan konfigurasi yang bisa di-inject ke Pod sebagai:
1. **Environment variables**
2. **File/volume mount**
3. **Command-line arguments**

```
ConfigMap
┌─────────────────────────────────┐
│  DB_HOST=postgres-service       │
│  APP_PORT=8080                  │
│  LOG_LEVEL=info                 │
│  MAX_CONNECTIONS=100            │
│  nginx.conf=<file content>      │
└─────────────────────────────────┘
         │
         ├─── Sebagai env var → Container (process.env.DB_HOST)
         └─── Sebagai file  → Container (/etc/config/nginx.conf)
```

---

## Membuat ConfigMap

### Cara 1: Dari literal value (command line)

```bash
# Buat ConfigMap dari command line
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres-service \
  --from-literal=APP_PORT=8080 \
  --from-literal=LOG_LEVEL=info

# Lihat hasilnya
kubectl get configmap app-config -o yaml
```

### Cara 2: Dari file

```bash
# Buat file konfigurasi
cat > app.properties << EOF
DB_HOST=postgres-service
APP_PORT=8080
LOG_LEVEL=info
EOF

# Buat ConfigMap dari file
kubectl create configmap app-config --from-file=app.properties

# Buat ConfigMap dari direktori (semua file dalam direktori)
kubectl create configmap nginx-config --from-file=./config/
```

### Cara 3: Dari YAML manifest (cara yang disarankan)

```yaml
# Lihat contoh-configmap.yaml
```

---

## Dua Cara Menggunakan ConfigMap di Pod

### Cara 1: Sebagai Environment Variables

```yaml
spec:
  containers:
  - name: app
    env:
    # Inject satu key dari ConfigMap
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config    # Nama ConfigMap
          key: DB_HOST        # Key dalam ConfigMap
    
    # Inject semua key dari ConfigMap sekaligus
    envFrom:
    - configMapRef:
        name: app-config      # Semua key jadi env var
```

### Cara 2: Sebagai Volume Mount (File)

```yaml
spec:
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config      # ConfigMap yang di-mount

  containers:
  - name: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d   # Setiap key jadi file
```

---

## Perintah kubectl untuk ConfigMap

```bash
# Lihat semua ConfigMap
kubectl get configmaps
kubectl get cm  # shorthand

# Lihat isi ConfigMap
kubectl describe cm app-config
kubectl get cm app-config -o yaml

# Edit ConfigMap (perubahan tidak otomatis di-reload, perlu restart Pod)
kubectl edit cm app-config

# Hapus ConfigMap
kubectl delete cm app-config
```

---

## Catatan Penting: Reload ConfigMap

Ketika ConfigMap diubah:
- **Env vars:** TIDAK otomatis update — Pod perlu di-restart
- **Volume mount:** Otomatis update (dalam beberapa menit)

```bash
# Restart Pod untuk mendapatkan ConfigMap terbaru (env vars)
kubectl rollout restart deployment my-app
```

---

## Latihan

```bash
# 1. Buat ConfigMap dari YAML
kubectl apply -f contoh-configmap.yaml

# 2. Lihat isi ConfigMap
kubectl describe configmap app-config

# 3. Buat Pod yang menggunakan ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'env | grep -E "DB_HOST|APP_PORT|LOG_LEVEL"; sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
EOF

# 4. Lihat env vars di dalam Pod
kubectl logs test-configmap

# 5. Cleanup
kubectl delete pod test-configmap
kubectl delete configmap app-config
```

---

*[Lihat contoh YAML →](./contoh-configmap.yaml)*

*[Lanjut ke: Secret →](../03-secret/README.md)*

*[Kembali ke: Konfigurasi](../README.md)*
