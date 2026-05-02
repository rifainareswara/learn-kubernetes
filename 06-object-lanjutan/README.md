# 06 - Object Lanjutan Kubernetes

Di modul ini kamu akan mempelajari object Kubernetes yang lebih spesifik penggunaannya, namun sangat penting untuk use case production.

---

## Daftar Materi

| Object | Deskripsi | Use Case |
|--------|-----------|---------|
| [Ingress](./01-ingress/README.md) | HTTP routing berbasis domain/path | Mengekspos banyak service dengan satu IP |
| [DaemonSet](./02-daemonset/README.md) | Jalankan satu Pod di SETIAP Node | Log collector, monitoring agent |
| [StatefulSet](./03-statefulset/README.md) | Deployment untuk aplikasi stateful | Database, Kafka, Elasticsearch |
| [Job & CronJob](./04-job-cronjob/README.md) | Batch processing dan jadwal | Migration DB, report, cleanup task |

---

## Kapan Menggunakan Masing-masing?

```
Aplikasi Stateless (API, Frontend)?
→ Gunakan Deployment

Aplikasi Stateful (Database, Queue)?
→ Gunakan StatefulSet

Perlu berjalan di semua Node (monitoring, logging)?
→ Gunakan DaemonSet

Perlu HTTP routing berdasarkan domain/path?
→ Gunakan Ingress

Batch job sekali jalan?
→ Gunakan Job

Job yang perlu dijadwalkan?
→ Gunakan CronJob
```

---

## Navigasi

- [Sebelumnya: 05 - Storage](../05-storage/README.md)
- [Selanjutnya: 07 - Helm](../07-helm/README.md)
- [Kembali ke README utama](../README.md)
