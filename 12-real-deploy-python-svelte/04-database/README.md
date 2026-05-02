# 04 - Database PostgreSQL

> **Estimasi Waktu:** 20 menit
>
> **Tujuan:** Deploy PostgreSQL di Kubernetes menggunakan StatefulSet dan Persistent Volume

---

## Kenapa StatefulSet, Bukan Deployment?

Ini pertanyaan penting yang sering muncul. Mari kita bandingkan:

### Deployment (untuk Stateless App)
```
Pod-abc123 ──restart──▶ Pod-xyz789
(nama berubah)           (nama berubah)
(storage terpisah)       (storage baru/kosong)
```

### StatefulSet (untuk Stateful App seperti database)
```
postgres-0 ──restart──▶ postgres-0
(nama TETAP)             (nama TETAP)
(storage: pvc-postgres-0) (storage: pvc-postgres-0, data TETAP)
```

**Keuntungan StatefulSet untuk database:**

| Fitur | Deployment | StatefulSet |
|---|---|---|
| Nama Pod | Random (pod-abc123) | Urutan (postgres-0, postgres-1) |
| Storage | Baru setiap restart | Persistent, terhubung ke Pod yang sama |
| Startup Order | Bersamaan | Berurutan (0, lalu 1, lalu 2) |
| DNS stabil | Tidak | Ya (`postgres-0.postgres.todolist.svc.cluster.local`) |

---

## Persistent Storage

Data PostgreSQL disimpan di `/var/lib/postgresql/data` di dalam container. Tanpa Persistent Volume, data ini **akan hilang** setiap kali Pod di-restart.

```
Pod postgres-0
├── Container PostgreSQL
│   └── /var/lib/postgresql/data  ──mount──▶  PVC: postgres-data-0
│                                              └── PersistentVolume (disk fisik)
```

**PVC (PersistentVolumeClaim):** "Permintaan" storage ke cluster. Cluster akan menyediakan storage yang sesuai dari PersistentVolume yang tersedia.

---

## Inisialisasi Database

PostgreSQL Official Docker Image mendukung inisialisasi otomatis via environment variables:

| Variable | Kegunaan | Contoh |
|---|---|---|
| `POSTGRES_DB` | Nama database yang dibuat otomatis | `tododb` |
| `POSTGRES_USER` | Username superuser | `todouser` |
| `POSTGRES_PASSWORD` | Password superuser | `P@ssw0rd123!` |

Saat container pertama kali berjalan dan `/var/lib/postgresql/data` masih kosong, PostgreSQL akan:
1. Membuat database cluster baru
2. Membuat user dan database sesuai environment variables di atas

---

## Format Connection String

FastAPI (via SQLAlchemy) menggunakan format ini untuk koneksi ke PostgreSQL:

```
postgresql://username:password@host:port/database
```

Dalam konteks Kubernetes kita:
```
postgresql://todouser:P@ssw0rd123!@postgres:5432/tododb
```

Di mana:
- `todouser` — dari Secret `DB_USER`
- `P@ssw0rd123!` — dari Secret `DB_PASS`
- `postgres` — nama Kubernetes Service PostgreSQL (DNS internal cluster)
- `5432` — port default PostgreSQL
- `tododb` — dari ConfigMap `DB_NAME`

---

## File di Folder Ini

| File | Kegunaan |
|---|---|
| [statefulset.yaml](./statefulset.yaml) | StatefulSet untuk PostgreSQL |
| [service.yaml](./service.yaml) | Service untuk expose PostgreSQL ke backend |
| [pvc.yaml](./pvc.yaml) | PersistentVolumeClaim untuk storage data |

---

## Secret yang Dibutuhkan

Database membutuhkan Secret yang berisi kredensial. Kita bisa menggunakan **Secret yang sama** dengan backend, atau membuat yang terpisah. Disarankan menggunakan Secret yang sama agar tidak ada inkonsistensi:

```bash
# Buat Secret (pastikan sama dengan yang di 02-backend-python/configmap-secret.yaml)
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=todouser \
  --from-literal=POSTGRES_PASSWORD=P@ssw0rd123! \
  --from-literal=POSTGRES_DB=tododb \
  -n todolist
```

---

## Verifikasi Database

Setelah deploy, cek apakah PostgreSQL berjalan dengan benar:

```bash
# Cek status Pod
kubectl get pods -n todolist -l app=postgres

# Lihat logs inisialisasi database
kubectl logs -n todolist postgres-0

# Masuk ke database (untuk debug)
kubectl exec -it postgres-0 -n todolist -- psql -U todouser -d tododb

# Di dalam psql, cek tabel yang sudah dibuat oleh FastAPI
\dt
SELECT * FROM todos;
\q
```

---

## Tips Backup & Restore

```bash
# Backup database ke file
kubectl exec -n todolist postgres-0 -- \
  pg_dump -U todouser tododb > backup.sql

# Restore dari backup
kubectl exec -i -n todolist postgres-0 -- \
  psql -U todouser tododb < backup.sql
```

---

> **Perhatian:** Di production, jangan jalankan hanya satu instance PostgreSQL. Pertimbangkan menggunakan:
> - **PostgreSQL HA** dengan operator seperti [CloudNativePG](https://cloudnative-pg.io/) atau [Crunchy Data Postgres Operator](https://access.crunchydata.com/documentation/postgres-operator/)
> - **Managed database service** dari cloud provider (Cloud SQL, RDS, PolarDB) untuk menghindari kerumitan manajemen database

---

## Selanjutnya

- [05-ingress/README.md](../05-ingress/README.md) — Konfigurasi Ingress untuk routing
