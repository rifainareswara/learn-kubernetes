# 05 - Ingress & Routing

> **Estimasi Waktu:** 20 menit
>
> **Tujuan:** Mengkonfigurasi Nginx Ingress Controller untuk routing traffic ke frontend dan backend

---

## Apa itu Ingress?

**Ingress** adalah resource Kubernetes yang mendefinisikan aturan routing HTTP/HTTPS dari luar cluster ke Services di dalam cluster. Ingress membutuhkan **Ingress Controller** untuk bekerja — Ingress Controller-lah yang benar-benar memproses aturan tersebut.

```
Internet
    │
    ▼
[Ingress Controller]  ← "Otak" routing (nginx, traefik, dll)
    │
    ├── /api/*  ──▶  Service: backend  (port 8000)
    └── /*      ──▶  Service: frontend (port 80)
```

**Ingress** = aturan routing (file YAML yang kita buat)
**Ingress Controller** = software yang mengimplementasikan aturan tersebut

---

## Install Nginx Ingress Controller

### Minikube (paling mudah)

```bash
# Enable addon bawaan Minikube
minikube addons enable ingress

# Verifikasi
kubectl get pods -n ingress-nginx
```

### Kind

```bash
# Apply manifest official nginx ingress untuk Kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Tunggu sampai ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Helm (untuk semua cluster)

```bash
# Tambah Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verifikasi
kubectl get svc -n ingress-nginx
```

---

## Routing Rules yang Kita Gunakan

### Path Routing

| Path Pattern | Tujuan | Keterangan |
|---|---|---|
| `/api/(.*)` | Service `backend:8000` | Path `/api` di-strip, jadi `/todos` |
| `/` | Service `frontend:80` | Semua traffic lainnya ke frontend |

### Path Rewriting

Ingress Nginx menggunakan annotation `nginx.ingress.kubernetes.io/rewrite-target` untuk me-rewrite path:

```
Request dari browser: GET /api/todos
                          ↓ (rewrite oleh Ingress)
Request ke backend:   GET /todos
```

Konfigurasi ini penting karena FastAPI kita mendefinisikan endpoint sebagai `/todos`, bukan `/api/todos`. Ingress yang "menambahkan" prefix `/api` untuk pemisahan routing.

---

## TLS/HTTPS dengan cert-manager (Opsional)

Untuk production, kamu perlu HTTPS. [cert-manager](https://cert-manager.io/) bisa mengotomasi pembuatan dan pembaruan sertifikat TLS dari Let's Encrypt.

### Install cert-manager

```bash
# Apply manifest
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Tunggu semua Pod ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s
```

### Buat ClusterIssuer (Let's Encrypt)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: kamu@example.com           # Email untuk notifikasi sertifikat
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Setelah cert-manager siap, aktifkan bagian TLS di `ingress.yaml`.

---

## File di Folder Ini

| File | Kegunaan |
|---|---|
| [ingress.yaml](./ingress.yaml) | Konfigurasi Ingress dengan routing rules |

---

## Verifikasi Ingress

```bash
# Cek status Ingress
kubectl get ingress -n myapp

# Detail Ingress (termasuk IP/hostname)
kubectl describe ingress myapp-ingress -n myapp

# Cek Ingress Controller berjalan
kubectl get pods -n ingress-nginx
```

Output yang diharapkan dari `kubectl get ingress -n myapp`:
```
NAME             CLASS   HOSTS       ADDRESS        PORTS   AGE
myapp-ingress    nginx   localhost   192.168.49.2   80      5m
```

### Test Routing

```bash
# Test akses frontend
curl http://localhost/
# atau buka browser: http://localhost

# Test akses backend API
curl http://localhost/api/health
# Output: {"status":"healthy","service":"todo-backend"}

# Test endpoint todos
curl http://localhost/api/todos
# Output: []  (kosong jika belum ada data)
```

---

> **Tips:** Jika menggunakan Minikube, jalankan `minikube tunnel` di terminal terpisah agar Ingress bisa diakses dari `localhost`.

> **Perhatian:** Untuk domain production (bukan localhost), kamu perlu mengkonfigurasi DNS record yang mengarah ke IP External dari Ingress Controller. Caranya berbeda-beda tergantung cloud provider.

---

## Selanjutnya

- [06-deploy-lokal/README.md](../06-deploy-lokal/README.md) — Panduan lengkap deploy di Minikube/Kind
