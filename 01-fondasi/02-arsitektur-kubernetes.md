# Arsitektur Kubernetes

Sekarang kita akan membedah "mesin" di balik Kubernetes. Memahami arsitektur ini penting agar kamu bisa troubleshoot masalah dan mengerti kenapa sesuatu bekerja seperti itu.

---

## Gambaran Besar: Cluster Kubernetes

Kubernetes berjalan sebagai sebuah **cluster** — sekumpulan mesin (bisa fisik atau virtual) yang bekerja bersama.

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    CONTROL PLANE                         │    │
│  │  ┌────────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  │    │
│  │  │ API Server │  │   etcd   │  │Scheduler│  │Control │  │    │
│  │  │            │  │          │  │         │  │Manager │  │    │
│  │  └────────────┘  └──────────┘  └────────┘  └────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                   │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │     WORKER NODE 1    │    │     WORKER NODE 2    │            │
│  │                      │    │                      │            │
│  │  ┌──────────────┐    │    │  ┌──────────────┐    │           │
│  │  │    kubelet   │    │    │  │    kubelet   │    │           │
│  │  └──────────────┘    │    │  └──────────────┘    │           │
│  │  ┌──────────────┐    │    │  ┌──────────────┐    │           │
│  │  │  kube-proxy  │    │    │  │  kube-proxy  │    │           │
│  │  └──────────────┘    │    │  └──────────────┘    │           │
│  │                      │    │                      │            │
│  │  ┌─────┐  ┌─────┐    │    │  ┌─────┐  ┌─────┐   │           │
│  │  │ Pod │  │ Pod │    │    │  │ Pod │  │ Pod │   │           │
│  │  └─────┘  └─────┘    │    │  └─────┘  └─────┘   │           │
│  └─────────────────────┘    └─────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

Cluster Kubernetes terdiri dari dua bagian utama:
1. **Control Plane** — "Otak" yang mengatur segalanya
2. **Worker Node** — "Otot" yang menjalankan aplikasi

---

## Control Plane

Control Plane adalah otak dari cluster Kubernetes. Ia bertanggung jawab untuk membuat keputusan global tentang cluster (misalnya, scheduling), dan mendeteksi serta merespons event cluster.

### 1. API Server (kube-apiserver)

```
┌────────────────────────────────────────────┐
│                API Server                   │
│                                            │
│  kubectl ──────► REST API ──────► Validasi │
│  Dashboard ────►             ──────► etcd  │
│  Controller ───►             ◄────── etcd  │
└────────────────────────────────────────────┘
```

API Server adalah **pintu gerbang** ke cluster Kubernetes. Semua komunikasi melewati komponen ini.

**Fungsi utama:**
- Menerima dan memproses semua request (dari kubectl, dashboard, dll)
- Memvalidasi request (apakah YAML kamu valid?)
- Menyimpan dan mengambil data dari etcd
- Mengautentikasi dan mengotorisasi siapa yang boleh melakukan apa

**Analogi:** API Server seperti **resepsionis kantor** — semua tamu (request) harus melewatinya, dicek identitasnya, dan diarahkan ke bagian yang tepat.

### 2. etcd

```
┌────────────────────────────────────────────┐
│                   etcd                      │
│                                            │
│  Distributed Key-Value Store               │
│                                            │
│  /registry/pods/default/my-pod            │
│  /registry/services/default/my-service    │
│  /registry/deployments/default/my-app     │
└────────────────────────────────────────────┘
```

etcd adalah **database Kubernetes** — menyimpan semua state/kondisi cluster.

**Fungsi utama:**
- Menyimpan semua konfigurasi cluster
- Konsisten dan highly available (data tidak hilang)
- Hanya API Server yang boleh berbicara langsung dengan etcd

**Analogi:** etcd seperti **buku besar (ledger)** yang mencatat semua yang ada di perusahaan.

> **Peringatan:** Kehilangan data etcd = kehilangan seluruh konfigurasi cluster! Selalu backup etcd di production.

### 3. Scheduler (kube-scheduler)

```
┌────────────────────────────────────────────────────┐
│                    Scheduler                        │
│                                                    │
│  "Ada Pod baru yang belum punya Node!"             │
│                                                    │
│  Pertimbangan:                                     │
│  - Resource yang tersedia (CPU, Memory)            │
│  - Affinity/Anti-affinity rules                    │
│  - Taints dan tolerations                          │
│  - Node yang sedang bermasalah                     │
│                                                    │
│  Keputusan: "Pod ini cocok di Node 2!"             │
└────────────────────────────────────────────────────┘
```

Scheduler bertugas memutuskan **Pod berjalan di Node mana**.

**Fungsi utama:**
- Memantau Pod yang belum di-assign ke Node
- Memilih Node terbaik berdasarkan berbagai faktor
- Tidak menjalankan Pod sendiri — hanya "menjadwalkan"

**Analogi:** Scheduler seperti **manajer HR** yang menentukan karyawan baru (Pod) bekerja di departemen (Node) mana berdasarkan keahlian dan ketersediaan tempat.

### 4. Controller Manager (kube-controller-manager)

```
┌─────────────────────────────────────────────────────┐
│              Controller Manager                      │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │  Node Controller                             │   │
│  │  "Node 2 tidak respond 5 menit, tandai NotReady" │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │  ReplicaSet Controller                       │   │
│  │  "Harusnya 3 Pod, tapi cuma 2 yang jalan!"   │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │  Endpoint Controller                         │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Controller Manager menjalankan semua **proses kontroler** yang memastikan cluster berada dalam desired state.

**Kontroler yang ada di dalamnya:**
- **Node Controller** — Memantau kesehatan Node
- **ReplicaSet Controller** — Memastikan jumlah Pod sesuai
- **Deployment Controller** — Mengelola rolling update
- **Service Controller** — Mengelola load balancer di cloud
- Dan masih banyak lagi...

**Analogi:** Controller Manager seperti **tim supervisor** yang masing-masing mengawasi aspek berbeda dari bisnis dan memastikan semuanya berjalan sesuai rencana.

### 5. Cloud Controller Manager (opsional)

Komponen ini ada jika cluster berjalan di cloud provider (GKE, EKS, AKS). Fungsinya menghubungkan Kubernetes dengan API cloud provider untuk membuat load balancer, storage, dll.

---

## Worker Node

Worker Node adalah mesin (VM atau fisik) yang benar-benar **menjalankan** aplikasimu. Setiap node memiliki komponen berikut:

### 1. kubelet

```
┌────────────────────────────────────────────┐
│                  kubelet                    │
│                                            │
│  API Server ──► kubelet ──► Container      │
│                              Runtime       │
│                                            │
│  Tugas:                                    │
│  - Terima instruksi dari API Server        │
│  - Pastikan container berjalan sesuai spec │
│  - Laporkan status kembali ke API Server   │
└────────────────────────────────────────────┘
```

kubelet adalah **agen** di setiap Node yang berkomunikasi dengan Control Plane.

**Fungsi utama:**
- Menerima PodSpec dari API Server
- Memastikan container yang ada di PodSpec berjalan dan sehat
- Melaporkan status Pod dan Node ke API Server
- Menjalankan health check (liveness/readiness probe)

**Analogi:** kubelet seperti **kepala departemen** yang menerima instruksi dari manajemen pusat dan memastikan karyawan di departemennya bekerja sesuai arahan.

### 2. kube-proxy

```
┌────────────────────────────────────────────┐
│                kube-proxy                   │
│                                            │
│  Manages iptables/IPVS rules               │
│                                            │
│  Client ──► Service IP ──► Pod IP          │
│                                            │
│  Memastikan traffic ke Service             │
│  diteruskan ke Pod yang tepat              │
└────────────────────────────────────────────┘
```

kube-proxy adalah komponen yang mengatur **networking** di setiap Node.

**Fungsi utama:**
- Mengimplementasikan konsep "Service" di level jaringan
- Mengatur iptables/IPVS rules untuk routing traffic
- Memastikan Service IP bisa diakses dari mana saja di cluster

**Analogi:** kube-proxy seperti **operator telepon** yang meneruskan panggilan masuk ke ekstensi yang tepat.

### 3. Container Runtime

```
┌────────────────────────────────────────────┐
│            Container Runtime               │
│                                            │
│  kubelet ──► CRI ──► containerd / CRI-O   │
│                          │                 │
│                          ▼                 │
│                    Container berjalan      │
└────────────────────────────────────────────┘
```

Container Runtime adalah software yang benar-benar **menjalankan container**.

**Opsi Container Runtime:**
| Runtime | Keterangan |
|---------|------------|
| containerd | Paling umum, digunakan oleh Docker juga |
| CRI-O | Lightweight, dikembangkan Red Hat |
| Docker Engine | Dulu default, sekarang sudah deprecated di K8s |

---

## Visualisasi Lengkap Alur Kerja

Mari kita lihat apa yang terjadi ketika kamu menjalankan `kubectl apply -f deployment.yaml`:

```
1. kubectl
   │
   │  kubectl apply -f deployment.yaml
   ▼
2. API Server
   │
   │  Validasi YAML
   │  Autentikasi user
   │  Simpan ke etcd
   ▼
3. etcd
   │
   │  Data tersimpan: "Deployment my-app, 3 replicas"
   ▼
4. Controller Manager (Deployment Controller)
   │
   │  "Ada Deployment baru! Buat ReplicaSet!"
   ▼
5. API Server (lagi)
   │
   │  ReplicaSet dibuat, simpan ke etcd
   ▼
6. Controller Manager (ReplicaSet Controller)
   │
   │  "Ada ReplicaSet baru! Buat 3 Pod!"
   ▼
7. Scheduler
   │
   │  "Pod belum di-assign ke Node mana pun"
   │  "Node 1 punya CPU dan Memory cukup"
   │  "Assign Pod 1 ke Node 1"
   ▼
8. kubelet di Node 1
   │
   │  "Ada Pod baru untukku!"
   │  Pull image dari registry
   │  Jalankan container
   ▼
9. Container berjalan!
   │
   │  kubelet lapor ke API Server: "Pod Running"
   ▼
10. Status Pod: Running
```

---

## Pods: Unit Terkecil Kubernetes

Pod adalah unit terkecil yang bisa di-deploy di Kubernetes. Satu Pod berisi satu atau beberapa container yang berbagi:
- Network namespace (IP address yang sama)
- Storage (volume yang sama)
- Lifecycle (start dan stop bersama)

```
┌─────────────────────────────────────────────┐
│                    POD                       │
│                                             │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │  Container 1  │    │   Container 2    │   │
│  │  (nginx)     │    │  (log-sidecar)   │   │
│  └──────────────┘    └──────────────────┘   │
│                                             │
│  IP: 10.244.1.5                             │
│  Port: 80, 9000                             │
│  Volume: /data (shared)                     │
└─────────────────────────────────────────────┘
```

**Kenapa bukan langsung Container seperti Docker?**

Pod adalah abstraksi yang memungkinkan Kubernetes mengelola container dengan lebih fleksibel. Misalnya, pola "sidecar" — container utama (aplikasi) dan container helper (logging, proxy) berjalan dalam satu Pod dan berbagi network + storage.

---

## Namespace: Isolasi Logis

Namespace adalah cara untuk mempartisi satu cluster Kubernetes menjadi beberapa "virtual cluster":

```
┌────────────────────────────────────────────────┐
│                KUBERNETES CLUSTER               │
│                                                │
│  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │ Namespace  │  │ Namespace  │  │Namespace │  │
│  │  default   │  │ production │  │ staging  │  │
│  │            │  │            │  │          │  │
│  │  my-app    │  │  my-app    │  │  my-app  │  │
│  │  my-db     │  │  my-db     │  │  my-db   │  │
│  └────────────┘  └────────────┘  └──────────┘  │
└────────────────────────────────────────────────┘
```

---

## Perbandingan: Control Plane vs Worker Node

| Aspek | Control Plane | Worker Node |
|-------|--------------|-------------|
| Fungsi utama | Manajemen cluster | Menjalankan workload |
| Komponen utama | API Server, etcd, Scheduler, Controller Manager | kubelet, kube-proxy, Container Runtime |
| Apakah ada Pod? | Biasanya tidak (atau hanya system Pod) | Ya, Pod aplikasi berjalan di sini |
| Jumlah | Minimal 1 (production: 3 untuk HA) | Bisa banyak (skala sesuai kebutuhan) |
| Kerentanan | Kalau mati, cluster tidak bisa di-manage | Kalau mati, Pod di-reschedule ke Node lain |

---

## Managed vs Self-Managed Kubernetes

### Self-Managed
Kamu install dan kelola semua komponen sendiri:
```
Kamu urus:
- Install etcd
- Install API Server
- Install Scheduler
- Konfigurasi networking
- Update semua komponen
- Backup etcd
```

### Managed Kubernetes (GKE, EKS, AKS, ACK)
Cloud provider yang mengurus Control Plane:
```
Cloud provider urus:  │  Kamu urus:
- etcd                │  - Worker Nodes
- API Server          │  - Aplikasimu
- Scheduler           │  - Network policies
- Backup & Update     │  - Biaya cloud
```

> **Tips untuk pemula:** Mulai dengan managed Kubernetes (GKE/EKS) untuk production. Self-managed lebih cocok untuk belajar atau organisasi dengan kebutuhan sangat spesifik.

---

## Latihan

1. **Gambar ulang** diagram arsitektur ini dari ingatan. Ini latihan yang sangat efektif!

2. **Identifikasi:** Komponen mana yang bertanggung jawab jika:
   - Pod crash dan perlu di-restart?
   - Pod baru perlu di-assign ke Node?
   - User kirim perintah `kubectl apply`?
   - Node mati mendadak?

3. **Diskusi:** Kenapa etcd hanya boleh diakses oleh API Server? Apa risikonya jika semua komponen bisa langsung menulis ke etcd?

---

## Rangkuman

| Komponen | Lokasi | Fungsi |
|----------|--------|--------|
| API Server | Control Plane | Pintu gerbang semua request |
| etcd | Control Plane | Database cluster |
| Scheduler | Control Plane | Assign Pod ke Node |
| Controller Manager | Control Plane | Jaga desired state |
| kubelet | Worker Node | Jalankan Pod sesuai instruksi |
| kube-proxy | Worker Node | Atur networking/Service |
| Container Runtime | Worker Node | Jalankan container |

---

*[Lanjut ke: Cara Kerja Kubernetes →](./03-cara-kerja-kubernetes.md)*

*[Kembali ke: Apa itu Kubernetes ←](./01-apa-itu-kubernetes.md)*
