# Belajar Kubernetes: Dari Nol Sampai Deploy

Selamat datang di panduan belajar Kubernetes lengkap dalam Bahasa Indonesia! Panduan ini dirancang untuk membawa kamu dari pemula total hingga mampu melakukan deployment aplikasi ke production menggunakan Kubernetes.

---

## Kenapa Belajar Kubernetes?

Kubernetes telah menjadi standar industri untuk menjalankan aplikasi container di skala besar. Dengan menguasai Kubernetes, kamu bisa:

- Menjalankan aplikasi yang sangat tersedia (highly available) tanpa downtime
- Menskalakan aplikasi secara otomatis sesuai beban
- Mengelola puluhan hingga ribuan container dengan mudah
- Bekerja di perusahaan teknologi modern yang menggunakan cloud-native stack

---

## Peta Belajar (Learning Roadmap)

```
FASE 1 - Fondasi (Minggu 1-2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [00] Prasyarat          → Docker, CLI, Networking, YAML
  [01] Fondasi Kubernetes → Konsep, Arsitektur, Cara Kerja
  [02] Setup Lokal        → kubectl, Minikube, Kind

FASE 2 - Object Dasar (Minggu 3-4)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [03] Object Dasar       → Pod, ReplicaSet, Deployment, Service
  [04] Konfigurasi        → Namespace, ConfigMap, Secret
  [05] Storage            → Volume, PV, PVC

FASE 3 - Object Lanjutan (Minggu 5-6)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [06] Object Lanjutan    → Ingress, DaemonSet, StatefulSet, Job

FASE 4 - Tooling & Praktik (Minggu 7-8)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [07] Helm               → Package manager untuk Kubernetes
  [08] Project Latihan    → 3 project nyata dari mudah ke sulit

FASE 5 - Cloud & Production (Minggu 9-12)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [09] Cloud Provider     → GKE, EKS, ACK
  [10] Observability      → Monitoring & Logging
  [11] CI/CD              → GitHub Actions & ArgoCD
```

---

## Cara Menggunakan Repo Ini

1. **Ikuti urutan nomor folder** - Materi dirancang bertahap, jangan melompat
2. **Praktikkan setiap YAML** - Teori tanpa praktik tidak cukup
3. **Kerjakan latihan** di setiap akhir materi
4. **Jangan malu bertanya** - Setiap konsep sulit wajar untuk diulang

### Prasyarat Sebelum Mulai

Sebelum masuk ke materi Kubernetes, pastikan kamu sudah familiar dengan:
- Docker (wajib!)
- Perintah dasar Linux/terminal
- Konsep dasar jaringan (IP, port, DNS)
- Format YAML

Lihat [00-prasyarat/README.md](./00-prasyarat/README.md) untuk checklist lengkap.

---

## Navigasi Cepat

| No | Topik | Deskripsi | Estimasi |
|----|-------|-----------|----------|
| 00 | [Prasyarat](./00-prasyarat/README.md) | Checklist persiapan sebelum mulai | 1 minggu |
| 01 | [Fondasi](./01-fondasi/README.md) | Konsep dasar & arsitektur Kubernetes | 1 minggu |
| 02 | [Setup Lokal](./02-setup-lokal/README.md) | Install tools & cluster lokal | 2-3 hari |
| 03 | [Object Dasar](./03-object-dasar/README.md) | Pod, ReplicaSet, Deployment, Service | 1 minggu |
| 04 | [Konfigurasi](./04-konfigurasi/README.md) | Namespace, ConfigMap, Secret | 3-4 hari |
| 05 | [Storage](./05-storage/README.md) | Volume, PersistentVolume, PVC | 2-3 hari |
| 06 | [Object Lanjutan](./06-object-lanjutan/README.md) | Ingress, DaemonSet, StatefulSet, Job | 1 minggu |
| 07 | [Helm](./07-helm/README.md) | Package manager Kubernetes | 3-4 hari |
| 08 | [Project Latihan](./08-project-latihan/README.md) | 3 project nyata end-to-end | 1-2 minggu |
| 09 | [Cloud](./09-cloud/README.md) | Deploy ke GKE, EKS, dan ACK | 1 minggu |
| 10 | [Observability](./10-observability/README.md) | Monitoring & logging dengan Prometheus/Grafana | 3-4 hari |
| 11 | [CI/CD](./11-cicd/README.md) | Otomasi deploy dengan GitHub Actions & ArgoCD | 3-4 hari |

---

## Tools yang Dibutuhkan

```bash
# Wajib
- kubectl      # CLI utama Kubernetes
- Docker       # Container runtime
- Minikube     # Cluster lokal (pilihan 1)
  ATAU
- kind         # Cluster lokal (pilihan 2)

# Opsional tapi sangat berguna
- Helm         # Package manager
- k9s          # TUI untuk Kubernetes
- kubectx      # Switch antar context mudah
```

---

## Konvensi dalam Panduan Ini

Sepanjang panduan ini, kamu akan menemukan beberapa penanda:

> **Catatan:** Informasi tambahan yang penting

> **Peringatan:** Hal yang perlu diwaspadai

> **Tips:** Trik dan best practice

```bash
# Perintah yang bisa langsung dijalankan
kubectl get pods
```

```yaml
# Contoh file YAML
apiVersion: v1
kind: Pod
```

---

## Status Materi

Semua materi dalam repo ini sudah lengkap dan siap digunakan. Jika menemukan kesalahan atau ingin berkontribusi, silakan buat issue atau pull request.

---

*Selamat belajar! Kubernetes memang kompleks di awal, tapi semakin kamu praktik, semakin jelas gambarannya. Tetap semangat!*
