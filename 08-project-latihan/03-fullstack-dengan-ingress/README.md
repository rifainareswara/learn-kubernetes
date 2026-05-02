# Project 03: Fullstack dengan Ingress

Project paling lengkap — deploy aplikasi fullstack dengan routing berbasis Ingress, menggunakan satu domain dengan path berbeda untuk frontend dan backend.

---

## Arsitektur

```
                         Internet
                            │
                     myapp.local:80
                            │
                    ┌───────▼────────┐
                    │  Ingress       │
                    │  (nginx-ingress)│
                    └───────┬────────┘
                            │
               ┌────────────┴────────────┐
               │                         │
        /  (frontend)             /api (backend)
               │                         │
    ┌──────────▼──────────┐   ┌──────────▼──────────┐
    │   frontend-service  │   │   backend-service    │
    │   ClusterIP:80      │   │   ClusterIP:3000     │
    └──────────┬──────────┘   └──────────┬──────────┘
               │                         │
    ┌──────────▼──────────┐   ┌──────────▼──────────┐
    │  Frontend Pods (x3) │   │  Backend Pods (x2)   │
    │  nginx (static)     │   │  node.js API         │
    └─────────────────────┘   └──────────┬───────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  PostgreSQL          │
                              │  (StatefulSet)       │
                              │  + PVC               │
                              └─────────────────────┘
```

---

## Prerequisites

Pastikan Ingress Controller sudah terinstall:

```bash
# Minikube
minikube addons enable ingress

# Atau dengan Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Verifikasi
kubectl get pods -n ingress-nginx
```

---

## Deploy

```bash
# 1. Apply semua manifest
kubectl apply -f manifest.yaml

# 2. Tunggu semua Pod running
kubectl get pods -w

# 3. Dapatkan IP Ingress
# Minikube:
INGRESS_IP=$(minikube ip)

# Cloud:
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Ingress IP: $INGRESS_IP"

# 4. Tambahkan ke /etc/hosts untuk testing lokal
echo "$INGRESS_IP myapp.local" | sudo tee -a /etc/hosts
```

---

## Verifikasi

```bash
# Test frontend (path /)
curl http://myapp.local/

# Test backend API (path /api/)
curl http://myapp.local/api/health

# Lihat Ingress
kubectl describe ingress fullstack-ingress

# Lihat semua resource
kubectl get all
```

---

## Checklist

- [ ] Semua Pod berstatus Running
- [ ] Ingress bisa diakses di http://myapp.local
- [ ] Path `/` menampilkan halaman frontend
- [ ] Path `/api/` mengembalikan response JSON dari backend
- [ ] Backend bisa konek ke database PostgreSQL

---

## Cleanup

```bash
kubectl delete -f manifest.yaml
sudo sed -i '/myapp.local/d' /etc/hosts  # Hapus dari /etc/hosts
```

---

*[Lihat manifest.yaml →](./manifest.yaml)*

*[Kembali ke: Project Latihan](../README.md)*

*[Lanjut ke: Cloud →](../../09-cloud/README.md)*
