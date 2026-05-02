# Ingress

Ingress adalah object Kubernetes yang mengelola akses HTTP/HTTPS dari luar cluster ke Service di dalam cluster. Ingress bisa melakukan routing berdasarkan URL path atau hostname.

---

## Mengapa Ingress?

**Tanpa Ingress:** Setiap Service yang perlu diakses dari luar butuh LoadBalancer (biayanya mahal di cloud):
```
Internet → LoadBalancer 1 (bayar!) → Service A
         → LoadBalancer 2 (bayar!) → Service B
         → LoadBalancer 3 (bayar!) → Service C
```

**Dengan Ingress:** Satu LoadBalancer untuk semua Service:
```
Internet → LoadBalancer (1 IP, bayar sekali)
               ↓
           Ingress Controller
               ↓
    /api     → Service A
    /admin   → Service B
    app.com  → Service C
    api.com  → Service D
```

---

## Komponen Ingress

Ada dua komponen penting:

1. **Ingress Resource** — Definisi aturan routing (yang kamu buat)
2. **Ingress Controller** — Software yang mengimplementasikan aturan tersebut

> **Penting:** Ingress tidak bekerja sendiri! Kamu perlu install Ingress Controller terlebih dahulu.

---

## Install Ingress Controller

### Nginx Ingress Controller (Paling Populer)

```bash
# Untuk Minikube
minikube addons enable ingress

# Verifikasi
kubectl get pods -n ingress-nginx

# Untuk cluster lain (via Helm)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Untuk Kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Verifikasi controller sudah berjalan
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Dapatkan IP Ingress

```bash
# Di Minikube
minikube ip
# Output: 192.168.49.2

# Di cloud (LoadBalancer)
kubectl get service ingress-nginx-controller -n ingress-nginx
# Lihat EXTERNAL-IP

# Di Kind / lokal lainnya
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

---

## Jenis-Jenis Routing Ingress

### 1. Routing Berdasarkan Path

```yaml
# contoh-ingress.yaml
rules:
- http:
    paths:
    - path: /api      → backend-service:8080
    - path: /         → frontend-service:80
```

### 2. Routing Berdasarkan Host (Virtual Hosting)

```yaml
rules:
- host: app.example.com    → frontend-service
- host: api.example.com    → backend-service
- host: admin.example.com  → admin-service
```

### 3. HTTPS dengan TLS

```yaml
tls:
- hosts:
  - app.example.com
  secretName: tls-secret    # Secret berisi certificate
```

---

## Perintah kubectl untuk Ingress

```bash
# Lihat semua Ingress
kubectl get ingress
kubectl get ing  # shorthand

# Detail Ingress
kubectl describe ingress my-ingress

# Hapus Ingress
kubectl delete ingress my-ingress
```

---

## Testing Lokal dengan /etc/hosts

Untuk testing routing berdasarkan hostname di lokal:

```bash
# Dapatkan IP Minikube atau cluster lokal
CLUSTER_IP=$(minikube ip)

# Tambahkan entry ke /etc/hosts (butuh sudo)
echo "$CLUSTER_IP app.local api.local" | sudo tee -a /etc/hosts

# Sekarang bisa akses:
curl http://app.local
curl http://api.local
```

---

## Annotations Nginx Ingress yang Berguna

```yaml
metadata:
  annotations:
    # Redirect HTTP ke HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # Ukuran request maksimal
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    
    # Rewrite path (/api/* → /*)
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
```

---

## Latihan

```bash
# 1. Enable Ingress di Minikube
minikube addons enable ingress

# 2. Deploy aplikasi
kubectl create deployment frontend --image=nginx --replicas=2
kubectl expose deployment frontend --port=80 --name=frontend-service

kubectl create deployment backend --image=nginx --replicas=2
kubectl expose deployment backend --port=80 --name=backend-service

# 3. Apply Ingress
kubectl apply -f contoh-ingress.yaml

# 4. Dapatkan IP
MINIKUBE_IP=$(minikube ip)
echo "Minikube IP: $MINIKUBE_IP"

# 5. Test routing (tambahkan ke /etc/hosts atau gunakan curl dengan header)
curl -H "Host: myapp.local" http://$MINIKUBE_IP/
curl -H "Host: myapp.local" http://$MINIKUBE_IP/api/

# 6. Lihat Ingress
kubectl describe ingress myapp-ingress

# 7. Cleanup
kubectl delete ingress myapp-ingress
kubectl delete service frontend-service backend-service
kubectl delete deployment frontend backend
```

---

*[Lihat contoh YAML →](./contoh-ingress.yaml)*

*[Lanjut ke: DaemonSet →](../02-daemonset/README.md)*

*[Kembali ke: Object Lanjutan](../README.md)*
