# Job dan CronJob

Job dan CronJob digunakan untuk **batch processing** — pekerjaan yang perlu dijalankan sekali atau berulang secara terjadwal, bukan berjalan terus-menerus.

---

## Job

Job membuat satu atau lebih Pod dan memastikan **jumlah Pod yang ditentukan berhasil selesai** (exit code 0). Pod yang gagal akan otomatis di-retry.

### Kapan Menggunakan Job?
- Database migration
- Seeding data awal
- Proses batch sekali jalan
- Komputasi berat (machine learning, rendering)
- Export/import data

### Perilaku Job

```
Job dibuat → Pod berjalan → Exit code 0? → Job Selesai ✓
                         → Exit code != 0 → Retry (sesuai backoffLimit)
                         → Retry habis → Job Gagal ✗
```

---

## CronJob

CronJob membuat Job secara otomatis berdasarkan jadwal (sama seperti cron Linux).

### Kapan Menggunakan CronJob?
- Backup database harian
- Kirim laporan mingguan
- Cleanup file lama
- Cek kesehatan sistem secara periodik
- Sinkronisasi data terjadwal

### Format Jadwal Cron

```
┌───────────── menit (0-59)
│ ┌───────────── jam (0-23)
│ │ ┌───────────── hari dalam bulan (1-31)
│ │ │ ┌───────────── bulan (1-12)
│ │ │ │ ┌───────────── hari dalam minggu (0-6, 0=Minggu)
│ │ │ │ │
* * * * *

Contoh jadwal:
"0 2 * * *"      → Setiap hari pukul 02:00
"*/15 * * * *"   → Setiap 15 menit
"0 9 * * 1-5"    → Setiap hari kerja pukul 09:00
"0 0 1 * *"      → Tanggal 1 setiap bulan pukul 00:00
"@daily"         → Setiap hari pukul 00:00 (alias)
"@hourly"        → Setiap jam
"@weekly"        → Setiap minggu
```

---

## Perintah kubectl untuk Job dan CronJob

```bash
# Job
kubectl get jobs
kubectl describe job my-job
kubectl logs job/my-job
kubectl delete job my-job

# Jalankan Job secara manual dari CronJob
kubectl create job manual-run --from=cronjob/my-cronjob

# CronJob
kubectl get cronjobs
kubectl get cj  # shorthand
kubectl describe cronjob my-cronjob

# Suspend CronJob (hentikan sementara)
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob my-cronjob -p '{"spec":{"suspend":false}}'
```

---

## Latihan

```bash
# 1. Buat Job sederhana
kubectl apply -f contoh-job.yaml

# 2. Monitor Job
kubectl get jobs
kubectl get pods  # Lihat Pod yang dibuat Job

# 3. Lihat output
kubectl logs job/db-migration-job

# 4. Buat CronJob
kubectl apply -f contoh-cronjob.yaml

# 5. Lihat CronJob
kubectl get cronjobs

# 6. Tunggu 1 menit (atau trigger manual)
kubectl create job test-run --from=cronjob/cleanup-cronjob

# 7. Cleanup
kubectl delete job --all
kubectl delete cronjob --all
```

---

*[Lihat contoh Job →](./contoh-job.yaml)*

*[Lihat contoh CronJob →](./contoh-cronjob.yaml)*

*[Lanjut ke: Helm →](../../07-helm/README.md)*

*[Kembali ke: Object Lanjutan](../README.md)*
