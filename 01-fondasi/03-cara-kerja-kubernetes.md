# Cara Kerja Kubernetes

Di file ini kita akan memahami tiga konsep fundamental yang mendasari seluruh cara kerja Kubernetes: Desired State, Reconciliation Loop, dan bagaimana API Kubernetes bekerja.

---

## Konsep 1: Desired State (Keadaan yang Diinginkan)

Ini adalah konsep paling penting yang perlu kamu pahami di Kubernetes.

### Imperatif vs Deklaratif

Ada dua pendekatan dalam mengelola sistem:

**Pendekatan Imperatif (Cara Lama):**
> "Lakukan langkah A, lalu B, lalu C."

```bash
# Pendekatan imperatif: memberi tahu HOW
ssh server1 "docker run -d nginx"
ssh server2 "docker run -d nginx"
ssh server3 "docker run -d nginx"
# Jika satu gagal, kamu harus tahu dan handle sendiri
```

**Pendekatan Deklaratif (Cara Kubernetes):**
> "Saya ingin 3 nginx berjalan. Saya tidak peduli caranya."

```yaml
# Pendekatan deklaratif: memberi tahu WHAT
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3  # ← "Saya mau 3"
  selector:
    matchLabels:
      app: nginx
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Kamu menyatakan desired state (apa yang kamu inginkan), bukan langkah-langkah untuk mencapainya.**

### Analogi Desired State

Bayangkan kamu memesan kamar hotel:

```
Desired State (Pesananmu):
- Kamar untuk 2 orang
- Lantai non-smoking
- Dengan view laut

Hotel (Kubernetes):
- Cek ketersediaan
- Assign kamar yang tepat
- Jika kamar kamu rusak → pindahkan ke kamar lain
- Kamu tidak perlu tahu detail teknisnya
```

---

## Konsep 2: Reconciliation Loop

Reconciliation Loop adalah mekanisme di mana Kubernetes terus-menerus memastikan **current state** (kondisi nyata) selalu sesuai dengan **desired state** (kondisi yang kamu inginkan).

### Visualisasi Reconciliation Loop

```
                    ┌─────────────────┐
                    │  Desired State  │
                    │  (yang kamu     │
                    │   inginkan)     │
                    └────────┬────────┘
                             │
                     dibandingkan
                             │
                    ┌────────▼────────┐
                    │  Current State  │
                    │  (kondisi nyata │
                    │   sekarang)     │
                    └────────┬────────┘
                             │
                        ada perbedaan?
                        /            \
                      YA              TIDAK
                      │                │
              ┌───────▼──────┐  ┌──────▼──────┐
              │   Lakukan    │  │  Tidak ada  │
              │   Aksi       │  │  yang perlu │
              │   Perbaikan  │  │  dilakukan  │
              └───────┬──────┘  └─────────────┘
                      │
                      └──────────── loop kembali
```

### Contoh Nyata Reconciliation

**Skenario 1: Pod Crash**
```
Desired State: 3 Pod nginx berjalan
Current State: 3 Pod nginx berjalan
Status: OK, tidak perlu aksi

-- Pod #2 crash! --

Desired State: 3 Pod nginx berjalan  (tidak berubah)
Current State: 2 Pod nginx berjalan
Perbedaan: -1 Pod
Aksi: Buat 1 Pod baru

Setelah aksi:
Current State: 3 Pod nginx berjalan
Status: OK, tidak perlu aksi
```

**Skenario 2: Node Mati**
```
Desired State: 3 Pod nginx tersebar di Node
Current State: Pod ada di Node 1 (x1), Node 2 (x1), Node 3 (x1)
Status: OK

-- Node 2 mati! --

Desired State: 3 Pod nginx berjalan
Current State: 2 Pod berjalan (Node 2 dan semua Pod-nya mati)
Perbedaan: -1 Pod
Aksi: Schedule 1 Pod baru ke Node 1 atau Node 3
```

**Skenario 3: Manual Scale**
```
kubectl scale deployment nginx --replicas=5

Desired State diubah ke: 5 Pod nginx
Current State: 3 Pod nginx berjalan
Perbedaan: -2 Pod
Aksi: Buat 2 Pod baru
```

### Controllers Menjalankan Reconciliation

Setiap jenis resource Kubernetes punya **Controller** sendiri yang menjalankan reconciliation loop-nya:

| Controller | Yang Dipantau | Aksi jika Berbeda |
|------------|--------------|-------------------|
| ReplicaSet Controller | Jumlah Pod | Tambah/hapus Pod |
| Node Controller | Status Node | Tandai tidak sehat, reschedule Pod |
| Deployment Controller | Versi aplikasi | Rolling update |
| Service Controller | Endpoint | Update load balancer |

---

## Konsep 3: Bagaimana API Kubernetes Bekerja

### Struktur API Kubernetes

Kubernetes menggunakan RESTful API. Setiap resource Kubernetes bisa diakses melalui endpoint HTTP:

```
https://<api-server>/<api-group>/<version>/namespaces/<namespace>/<resource-type>/<name>

Contoh:
GET  /api/v1/namespaces/default/pods
GET  /api/v1/namespaces/default/pods/my-pod
POST /api/v1/namespaces/default/pods
PUT  /api/v1/namespaces/default/pods/my-pod
DELETE /api/v1/namespaces/default/pods/my-pod
```

### API Groups

Kubernetes membagi API-nya menjadi beberapa grup:

```
/api/v1                    ← Core group (Pod, Service, ConfigMap, dll)
/apis/apps/v1              ← Deployment, ReplicaSet, StatefulSet, DaemonSet
/apis/batch/v1             ← Job, CronJob
/apis/networking.k8s.io/v1 ← Ingress, NetworkPolicy
/apis/storage.k8s.io/v1    ← StorageClass, PersistentVolume
```

### kubectl = HTTP Client

`kubectl` sebenarnya hanyalah sebuah HTTP client yang membuat request ke Kubernetes API Server!

```bash
# Ini:
kubectl get pods

# Sebenarnya melakukan:
GET https://api-server:6443/api/v1/namespaces/default/pods

# Kamu bisa lihat request yang dibuat:
kubectl get pods -v=8  # -v=8 menampilkan HTTP request detail
```

### Struktur Manifest YAML

Setiap YAML Kubernetes punya struktur yang konsisten:

```yaml
apiVersion: apps/v1     # ← API group + version
kind: Deployment        # ← Jenis resource
metadata:               # ← Metadata resource
  name: my-app          #   Nama resource
  namespace: default    #   Di namespace mana
  labels:               #   Label untuk pengelompokan
    app: my-app
spec:                   # ← Desired state (apa yang kamu inginkan)
  replicas: 3
  ...
status:                 # ← Current state (diisi oleh Kubernetes, bukan kamu)
  readyReplicas: 3
  ...
```

> **Penting:** Field `spec` adalah yang kamu tulis (desired state). Field `status` diisi otomatis oleh Kubernetes (current state). Reconciliation loop terus membandingkan keduanya.

---

## Alur Lengkap: Dari kubectl ke Container Berjalan

Mari kita trace secara detail apa yang terjadi ketika kamu menjalankan:

```bash
kubectl apply -f my-deployment.yaml
```

### Step 1: kubectl memproses file

```
kubectl membaca my-deployment.yaml
→ Parse YAML ke objek internal
→ Tambahkan default values yang belum ada
→ Kirim HTTP POST ke API Server
```

### Step 2: API Server memproses request

```
API Server terima request:
→ Autentikasi: "Siapa yang kirim request ini?"
→ Otorisasi: "Apakah user ini boleh create Deployment?"
→ Admission Control: "Apakah resource ini sesuai policy?"
→ Validasi: "Apakah YAML ini valid secara schema?"
→ Simpan ke etcd: "Data Deployment tersimpan"
→ Kirim konfirmasi ke kubectl
```

### Step 3: Deployment Controller bereaksi

```
Deployment Controller (watch API Server untuk perubahan):
→ "Ada Deployment baru!"
→ Buat ReplicaSet baru dengan template yang sesuai
→ Kirim request ke API Server untuk buat ReplicaSet
```

### Step 4: ReplicaSet Controller bereaksi

```
ReplicaSet Controller:
→ "Ada ReplicaSet baru dengan replicas=3!"
→ Current state: 0 Pod
→ Desired state: 3 Pod
→ Buat 3 Pod baru (tanpa Node yang di-assign)
```

### Step 5: Scheduler bereaksi

```
Scheduler (watch untuk Pod tanpa node assignment):
→ "Ada 3 Pod yang perlu dijadwalkan!"
→ Filter: Node mana yang punya resource cukup?
→ Score: Node mana yang paling cocok?
→ Bind: "Pod-1 → Node-1, Pod-2 → Node-2, Pod-3 → Node-1"
→ Update Pod spec dengan nodeName
```

### Step 6: kubelet bereaksi

```
kubelet di Node-1 (watch untuk Pod yang di-assign ke Node-1):
→ "Ada 2 Pod baru untuk saya!"
→ Pull Docker image: docker pull nginx:1.25
→ Buat container
→ Start container
→ Setup networking
→ Report status ke API Server: "Pod Running"
```

### Step 7: kube-proxy bereaksi

```
kube-proxy (watch untuk Service dan Endpoint perubahan):
→ Update iptables/IPVS rules
→ Traffic ke Service IP diteruskan ke Pod yang baru
```

---

## Watch Mechanism: Bagaimana Komponen Saling Berkomunikasi

Semua komponen Kubernetes menggunakan mekanisme **watch** untuk mendeteksi perubahan:

```
┌─────────────────────────────────────────────────────┐
│                  API Server                          │
│                                                     │
│  Menyimpan semua state di etcd                      │
│  Menyediakan "watch" endpoint                       │
│                                                     │
│  GET /api/v1/pods?watch=true                        │
│  → Stream events setiap ada perubahan Pod           │
└─────────────────────────────────────────────────────┘
         │              │               │
         │ watch        │ watch         │ watch
         ▼              ▼               ▼
   Scheduler    Controller Manager   kubelet
   "Pod baru    "ReplicaSet butuh    "Pod di-assign
   tanpa Node"  Pod baru"            ke Node saya"
```

Ini disebut **event-driven architecture** — komponen tidak polling secara aktif, tapi bereaksi terhadap event perubahan.

---

## Labels dan Selectors: Cara Kubernetes Menghubungkan Resource

Label adalah mekanisme kunci yang memungkinkan Kubernetes (dan kamu) mengelompokkan dan memfilter resource.

### Labels

```yaml
# Pod dengan labels
metadata:
  labels:
    app: frontend        # ← Label kustom
    version: v2.1
    environment: production
    tier: web
```

### Selectors

```yaml
# Service yang "memilih" Pod dengan label tertentu
spec:
  selector:
    app: frontend        # ← "Kirim traffic ke Pod yang punya label ini"
```

### Visualisasi Labels dan Selectors

```
Pods:
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Pod A         │  │  Pod B         │  │  Pod C         │
│  app: frontend │  │  app: frontend │  │  app: backend  │
│  env: prod     │  │  env: staging  │  │  env: prod     │
└────────────────┘  └────────────────┘  └────────────────┘

Service X                                Service Y
selector:                                selector:
  app: frontend    →  memilih Pod A, B     app: backend   → memilih Pod C
  env: prod        →  hanya Pod A          env: prod      → Pod C
```

---

## Latihan

1. **Simulasikan Reconciliation Loop:**
   - Buat Deployment dengan 3 replicas
   - Hapus satu Pod secara manual dengan `kubectl delete pod <nama-pod>`
   - Amati apa yang terjadi dengan `kubectl get pods -w` (flag -w untuk watch)

2. **Lihat API Server secara langsung:**
   ```bash
   kubectl proxy &
   curl http://localhost:8001/api/v1/namespaces/default/pods
   ```

3. **Debug dengan verbose:**
   ```bash
   kubectl get pods -v=6  # Level 6: tampilkan URL request
   kubectl get pods -v=8  # Level 8: tampilkan request dan response headers
   ```

4. **Watch events:**
   ```bash
   kubectl get events --watch
   ```

---

## Rangkuman

| Konsep | Penjelasan Singkat |
|--------|-------------------|
| Desired State | Kamu mendeklarasikan APA yang kamu inginkan, bukan BAGAIMANA caranya |
| Current State | Kondisi nyata cluster saat ini |
| Reconciliation Loop | Proses terus-menerus yang menyamakan current state dengan desired state |
| Controller | Komponen yang menjalankan reconciliation untuk satu jenis resource |
| Watch Mechanism | Cara komponen bereaksi terhadap perubahan tanpa polling |
| Labels & Selectors | Cara menghubungkan dan memfilter resource |

---

*[Lanjut ke: Setup Lokal →](../02-setup-lokal/README.md)*

*[Kembali ke: Arsitektur Kubernetes ←](./02-arsitektur-kubernetes.md)*
