# 07 - Helm: Package Manager untuk Kubernetes

Helm adalah package manager untuk Kubernetes — seperti apt/yum untuk Linux, atau npm untuk Node.js. Helm memudahkan install, upgrade, dan manajemen aplikasi Kubernetes.

---

## Daftar Materi

| File | Topik |
|------|-------|
| [01 - Helm Dasar](./01-helm-dasar.md) | Install, konsep, perintah, buat chart sendiri |

---

## Mengapa Helm?

**Tanpa Helm:**
```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
# ... 10+ file untuk satu aplikasi
```

**Dengan Helm:**
```bash
helm install my-app stable/nginx     # Satu perintah!
helm upgrade my-app stable/nginx --set replicas=5
helm rollback my-app 1
helm uninstall my-app
```

---

## Navigasi

- [Sebelumnya: 06 - Object Lanjutan](../06-object-lanjutan/README.md)
- [Selanjutnya: 08 - Project Latihan](../08-project-latihan/README.md)
- [Kembali ke README utama](../README.md)
