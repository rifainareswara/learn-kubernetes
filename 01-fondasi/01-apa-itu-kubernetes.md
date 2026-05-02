# Apa itu Kubernetes?

Sebelum kita bicara tentang Kubernetes, mari kita pahami dulu masalah apa yang mendorong terciptanya tools ini.

---

## Masalah Sebelum Ada Kubernetes

### Era Tradisional (2000-2010): Satu Server, Satu Aplikasi

Bayangkan kamu punya sebuah toko online. Di era ini, cara umum menjalankan aplikasi adalah:

```
┌─────────────────────────────────┐
│          Server Fisik            │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Operating System          │  │
│  │                            │  │
│  │  ┌──────────────────────┐  │  │
│  │  │  Aplikasi Toko Online │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
└─────────────────────────────────┘
```

**Masalahnya:**
- Server sering "nganggur" (utilisasi 10-20%)
- Satu server crash = aplikasi mati
- Scale up = beli server baru yang mahal
- Update = downtime

### Era Virtualisasi (2010-2015): Virtual Machines

Lalu muncul virtual machine (VM) yang memungkinkan satu server fisik menjalankan banyak "server virtual":

```
┌──────────────────────────────────────────┐
│              Server Fisik                 │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │    VM 1   │  │    VM 2   │  │  VM 3  │  │
│  │           │  │           │  │        │  │
│  │  OS       │  │  OS       │  │  OS    │  │
│  │  App A    │  │  App B    │  │  App C │  │
│  └──────────┘  └──────────┘  └────────┘  │
│                                          │
│         Hypervisor (VMware/KVM)          │
└──────────────────────────────────────────┘
```

**Lebih baik, tapi masih ada masalah:**
- Setiap VM butuh OS sendiri → boros resource
- Start VM butuh waktu menit, bukan detik
- Masih susah di-scale dengan cepat

### Era Container (2013-sekarang): Docker

Docker mempopulerkan container, yang lebih ringan dari VM:

```
┌──────────────────────────────────────────────┐
│               Server Fisik                    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │Container A│  │Container B│  │ Container C│  │
│  │           │  │           │  │            │  │
│  │  App A    │  │  App B    │  │   App C    │  │
│  └──────────┘  └──────────┘  └────────────┘  │
│                                              │
│          Container Runtime (Docker)          │
│          Operating System (Linux)            │
└──────────────────────────────────────────────┘
```

**Container jauh lebih baik:**
- Ringan, start dalam detik
- Isolasi antar aplikasi
- Konsisten di environment mana pun
- Efisien dalam penggunaan resource

### Masalah Baru: "Siapa yang Mengelola Ratusan Container?"

Perusahaan seperti Google, Netflix, atau Tokopedia bisa punya **ribuan container** yang berjalan sekaligus. Muncul pertanyaan baru:

- Container mana yang crash dan perlu di-restart?
- Bagaimana mendistribusikan container ke server yang tersedia?
- Bagaimana scale up/down otomatis saat traffic naik?
- Bagaimana update aplikasi tanpa downtime?
- Bagaimana container di server A bisa komunikasi dengan container di server B?

**Inilah masalah yang diselesaikan Kubernetes!**

---

## Kubernetes Adalah Solusinya

### Definisi Resmi

Kubernetes (disingkat K8s, karena ada 8 huruf antara K dan s) adalah:

> "Sistem open-source untuk otomasi deployment, scaling, dan manajemen aplikasi container."

Asal kata: **Kubernetes** berasal dari bahasa Yunani yang berarti "juru kemudi" atau "pilot" kapal.

### Apa yang Dilakukan Kubernetes

| Masalah | Solusi Kubernetes |
|---------|-------------------|
| Container crash | Auto-restart container yang mati |
| Traffic naik mendadak | Auto-scale jumlah container |
| Server mati | Pindahkan container ke server lain |
| Update aplikasi | Rolling update tanpa downtime |
| Konfigurasi beda-beda | Standardisasi via ConfigMap/Secret |
| Container saling komunikasi | Built-in service discovery & networking |

---

## Analogi: Kubernetes sebagai Direktur Orkestra

Mari kita bayangkan Kubernetes seperti **direktur orkestra** sebuah pertunjukan musik besar:

```
                    ┌──────────────────┐
                    │   Direktur       │
                    │   (Kubernetes)   │
                    └────────┬─────────┘
                             │ mengarahkan
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  Biola x10   │ │  Piano x2    │ │  Drum x3     │
    │  (Pod A)     │ │  (Pod B)     │ │  (Pod C)     │
    └──────────────┘ └──────────────┘ └──────────────┘
```

**Analogi yang lebih detail:**

- **Partitur musik** = YAML manifest (konfigurasi yang kamu tulis)
- **Direktur** = Kubernetes control plane
- **Musisi** = Container/Pod
- **Panggung** = Node (server)
- **Penonton** = User aplikasimu

Direktur orkestra:
- Memastikan jumlah musisi yang tepat dimainkan (ReplicaSet)
- Jika musisi tidak bisa main (crash), cari pengganti (self-healing)
- Mengatur siapa bermain di mana (scheduling)
- Memulai bagian baru setelah bagian sebelumnya selesai (rolling update)

**Kamu** sebagai developer hanya perlu bilang: *"Aku ingin 3 biola, 2 piano, dan 3 drum."* — Kubernetes yang mengurus semua detailnya!

---

## Analogi Lain: Kubernetes seperti Manajer Gudang Canggih

Bayangkan sebuah gudang e-commerce saat Harbolnas:

```
Kamu (Developer) bilang ke Manajer:
"Aku butuh 10 pekerja packing, 5 kasir, dan 3 driver."

Manajer (Kubernetes):
- Mengatur jadwal dan posisi setiap pekerja
- Jika pekerja sakit (crash), panggil pengganti
- Saat pesanan naik 10x, tambah pekerja otomatis
- Saat sepi, kurangi pekerja untuk hemat biaya
```

---

## Kubernetes vs Docker: Apa Bedanya?

Ini adalah pertanyaan yang paling sering ditanyakan oleh pemula:

```
Docker                          Kubernetes
──────                          ──────────
Membuat & menjalankan           Mengorkestrasi/mengatur
container di SATU mesin         container di BANYAK mesin

Seperti supir taksi             Seperti dispatcher taksi
yang bisa mengemudi             yang mengatur armada
satu mobil                      ratusan taksi
```

**Hubungan Docker dan Kubernetes:**
- Docker membuat container (image building)
- Kubernetes menjalankan dan mengelola container tersebut di skala besar
- Mereka saling melengkapi, bukan bersaing

> **Catatan:** Kubernetes sebenarnya bisa menggunakan container runtime selain Docker (seperti containerd atau CRI-O), tapi Docker tetap yang paling populer untuk development.

---

## Siapa yang Menggunakan Kubernetes?

Kubernetes awalnya dibuat oleh Google (berdasarkan sistem internal mereka bernama "Borg") dan sekarang dikelola oleh Cloud Native Computing Foundation (CNCF).

**Perusahaan yang menggunakan Kubernetes:**
- Google (tentu saja!)
- Spotify — mengelola ribuan microservices
- Airbnb — handling jutaan request per hari
- NYTimes — deployment cepat dengan zero downtime
- Tokopedia, Gojek, dan banyak unicorn Indonesia

---

## Kapan Menggunakan Kubernetes?

Kubernetes bukan solusi untuk semua masalah. Ada trade-off yang perlu dipertimbangkan:

### Kubernetes cocok jika:
- Aplikasi terdiri dari banyak microservices
- Butuh high availability (99.9%+ uptime)
- Traffic tidak menentu dan perlu auto-scaling
- Tim DevOps yang sudah ada
- Aplikasi di-deploy ke banyak environment

### Kubernetes mungkin terlalu kompleks jika:
- Aplikasi masih kecil (1-2 service)
- Tim masih sangat kecil dan belum ada DevOps
- Butuh solusi cepat dengan biaya minimal
- Aplikasi monolith yang jarang di-update

> **Tips:** Mulai dengan memahami Kubernetes meski aplikasimu masih kecil. Investasi belajar ini akan sangat berharga seiring pertumbuhan aplikasimu!

---

## Ekosistem Kubernetes

Kubernetes hanyalah inti dari ekosistem yang lebih besar:

```
                    ┌─────────────┐
                    │  Kubernetes │  ← Kita fokus di sini dulu
                    └─────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │   Helm   │   │ Istio/   │   │Prometheus│
    │(Package  │   │ Linkerd  │   │+Grafana  │
    │ Manager) │   │(Service  │   │(Monitor) │
    └──────────┘   │  Mesh)   │   └──────────┘
                   └──────────┘
```

---

## Latihan

1. **Diskusi:** Bayangkan aplikasi sederhana yang pernah kamu buat. Masalah apa yang akan muncul jika traffic-nya naik 100x lipat?

2. **Riset:** Cari satu studi kasus perusahaan yang migrasi ke Kubernetes. Apa manfaat dan tantangannya?

3. **Refleksi:** Berdasarkan kriteria di atas, apakah aplikasi yang sedang kamu kerjakan membutuhkan Kubernetes?

---

## Rangkuman

- Kubernetes muncul sebagai solusi untuk mengelola aplikasi container di skala besar
- Kubernetes mengotomasi deployment, scaling, dan self-healing container
- Kubernetes bekerja seperti direktur orkestra — kamu mendefinisikan "keinginan", Kubernetes mengeksekusi
- Kubernetes melengkapi Docker, bukan menggantikannya

---

*[Lanjut ke: Arsitektur Kubernetes →](./02-arsitektur-kubernetes.md)*

*[Kembali ke: Overview Fondasi](./README.md)*
