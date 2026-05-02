# 01 - Fondasi Kubernetes

Selamat datang di modul pertama! Di sini kamu akan membangun pemahaman yang kuat tentang apa itu Kubernetes, bagaimana arsitekturnya, dan bagaimana cara kerjanya.

---

## Tujuan Pembelajaran

Setelah menyelesaikan modul ini, kamu akan:
- Memahami masalah apa yang diselesaikan oleh Kubernetes
- Mengetahui komponen-komponen utama dalam cluster Kubernetes
- Memahami konsep "desired state" dan reconciliation loop
- Siap untuk mulai praktik dengan cluster lokal

---

## Daftar Materi

| File | Topik | Estimasi Waktu |
|------|-------|----------------|
| [01 - Apa itu Kubernetes?](./01-apa-itu-kubernetes.md) | Problem statement, solusi K8s, analogi | 30-45 menit |
| [02 - Arsitektur Kubernetes](./02-arsitektur-kubernetes.md) | Control Plane, Worker Node, semua komponen | 45-60 menit |
| [03 - Cara Kerja Kubernetes](./03-cara-kerja-kubernetes.md) | Desired state, reconciliation, API workflow | 30-45 menit |

---

## Ringkasan Konsep Kunci

### Kubernetes dalam satu kalimat:
> Kubernetes adalah sistem orkestrasi container yang secara otomatis mengelola deployment, scaling, dan ketersediaan aplikasi container kamu.

### Tiga konsep yang paling penting:
1. **Desired State** — Kamu mendeklarasikan "keadaan yang diinginkan", Kubernetes yang memastikannya tercapai
2. **Self-healing** — Kubernetes otomatis memperbaiki masalah (container crash → restart otomatis)
3. **Declarative Configuration** — Konfigurasi ditulis sebagai YAML, bukan perintah imperatif

---

## Navigasi

- [Sebelumnya: 00 - Prasyarat](../00-prasyarat/README.md)
- [Selanjutnya: 02 - Setup Lokal](../02-setup-lokal/README.md)
- [Kembali ke README utama](../README.md)
