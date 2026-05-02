# 09 - Deploy ke Cloud

Setelah belajar di lokal, saatnya deploy ke cloud! Modul ini membahas tiga provider cloud utama yang populer.

---

## Pilih Cloud Provider

| Provider | Layanan K8s | Cocok untuk | CLI Tool |
|----------|------------|-------------|---------|
| **GKE** | Google Kubernetes Engine | Kubernetes asli (dibuat Google) | gcloud |
| **EKS** | Amazon Elastic Kubernetes Service | AWS ecosystem | eksctl + aws CLI |
| **ACK** | Alibaba Cloud Container Service | Asia-Pacific, terutama China | aliyun CLI |

---

## Daftar Materi

| Folder | Topik |
|--------|-------|
| [01-gke](./01-gke/README.md) | Google Kubernetes Engine |
| [02-eks](./02-eks/README.md) | Amazon Elastic Kubernetes Service |
| [03-ack](./03-ack/README.md) | Alibaba Cloud Container Service for Kubernetes |

---

## Konsep Umum di Semua Cloud

Meskipun setiap cloud berbeda, alur umumnya sama:

```
1. Install CLI tools
    ↓
2. Setup authentication (login, IAM)
    ↓
3. Buat Kubernetes cluster
    ↓
4. Connect kubectl ke cluster
    ↓
5. Deploy aplikasi (sama seperti lokal!)
    ↓
6. Setup monitoring, scaling, dll
    ↓
7. Cleanup (JANGAN lupa untuk hemat biaya!)
```

---

## Perbandingan Biaya

> **Peringatan:** Cluster cloud berbayar! Selalu jalankan cleanup setelah selesai.

Estimasi biaya untuk cluster development (tidak production-grade):
- GKE: ~$70-150/bulan (e2-standard-2 x2 node)
- EKS: ~$100-200/bulan (t3.medium x2 node + EKS fee)
- ACK: ~$50-100/bulan (ecs.c6.large x2 node)

---

## Navigasi

- [Sebelumnya: 08 - Project Latihan](../08-project-latihan/README.md)
- [Selanjutnya: 10 - Observability](../10-observability/README.md)
- [Kembali ke README utama](../README.md)
