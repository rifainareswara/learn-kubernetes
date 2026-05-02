# StatefulSet

StatefulSet adalah workload controller untuk aplikasi yang membutuhkan **identitas yang stabil** — seperti database, message queue, dan sistem terdistribusi.

---

## Mengapa StatefulSet Ada?

Deployment tidak cocok untuk database karena:
- Pod name random (my-db-xyz123) → berubah setiap restart
- Tidak ada urutan start/stop yang dijamin
- Semua Pod identik — tidak ada "master" atau "slave"

StatefulSet menyelesaikan ini:
- Pod name terurut dan stabil (my-db-0, my-db-1, my-db-2)
- Start berurutan (my-db-0 dulu, baru my-db-1, dst.)
- Setiap Pod punya storage sendiri yang persistent

---

## Perbedaan Deployment vs StatefulSet

| Aspek | Deployment | StatefulSet |
|-------|-----------|------------|
| Pod naming | Random (app-xyz123) | Berurutan (app-0, app-1) |
| Pod identity | Tidak ada | Stabil dan unik |
| Urutan start | Tidak ada | Berurutan (0, 1, 2, ...) |
| Urutan stop | Tidak ada | Terbalik (N, N-1, ..., 0) |
| Storage | Shared atau tidak ada | Setiap Pod punya PVC sendiri |
| Scaling up | Parallel | Berurutan |
| Cocok untuk | Stateless | Stateful (DB, queue) |

---

## Contoh Use Case

```
MySQL Cluster (1 Master + 2 Slave):

mysql-0  ← Master (bisa baca & tulis)
  │
  ├── mysql-1  ← Slave (replikasi dari master)
  └── mysql-2  ← Slave (replikasi dari master)

Setiap Pod punya:
- Nama stabil: mysql-0, mysql-1, mysql-2
- DNS stabil: mysql-0.mysql-service
- Storage sendiri: PVC mysql-data-mysql-0, mysql-data-mysql-1, dst.
```

---

## DNS untuk StatefulSet

StatefulSet membutuhkan **Headless Service** untuk DNS individual Pod:

```
Format: <pod-name>.<service-name>.<namespace>.svc.cluster.local

Contoh:
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

---

## Perintah kubectl untuk StatefulSet

```bash
# Lihat semua StatefulSet
kubectl get statefulsets
kubectl get sts  # shorthand

# Detail StatefulSet
kubectl describe sts my-db

# Scale (hati-hati: ada urutan!)
kubectl scale sts my-db --replicas=5

# Update image
kubectl set image sts/my-db db=postgres:15

# Status rollout
kubectl rollout status sts/my-db

# Rollback
kubectl rollout undo sts/my-db
```

---

## Kapan TIDAK Menggunakan StatefulSet?

- Aplikasi yang bisa di-scale horizontal tanpa state (gunakan Deployment)
- Jika menggunakan managed database di cloud (RDS, Cloud SQL) — tidak perlu StatefulSet, cukup Service

---

*[Lihat contoh YAML →](./contoh-statefulset.yaml)*

*[Lanjut ke: Job & CronJob →](../04-job-cronjob/README.md)*

*[Kembali ke: Object Lanjutan](../README.md)*
