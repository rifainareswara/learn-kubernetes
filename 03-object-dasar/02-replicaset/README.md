# ReplicaSet

ReplicaSet memastikan bahwa jumlah Pod yang berjalan selalu sesuai dengan yang kamu tentukan. Ini adalah "penjaga" yang memastikan Pod tidak kurang atau tidak lebih dari yang diinginkan.

---

## Apa itu ReplicaSet?

ReplicaSet adalah controller yang:
- Memantau jumlah Pod yang berjalan
- Membuat Pod baru jika jumlahnya kurang dari yang diinginkan
- Menghapus Pod jika jumlahnya lebih dari yang diinginkan
- Menggunakan **label selector** untuk mengidentifikasi Pod yang dikelolanya

```
ReplicaSet (replicas: 3)
    │
    ├── Pod-1 (app: nginx) ← Running ✓
    ├── Pod-2 (app: nginx) ← Running ✓  
    └── Pod-3 (app: nginx) ← CRASH! → ReplicaSet buat Pod-4
```

---

## Bagaimana ReplicaSet Mengenali Pod-nya?

ReplicaSet menggunakan **label selector** untuk mengidentifikasi Pod yang dikelolanya:

```yaml
spec:
  selector:
    matchLabels:
      app: nginx      # ← "Kelola semua Pod yang punya label app=nginx"
  template:
    metadata:
      labels:
        app: nginx    # ← Pod yang dibuat harus punya label ini
```

> **Penting:** Label di `selector.matchLabels` HARUS sama dengan label di `template.metadata.labels`!

---

## Perintah kubectl untuk ReplicaSet

```bash
# Lihat semua ReplicaSet
kubectl get replicasets
kubectl get rs  # shorthand

# Detail ReplicaSet
kubectl describe rs <nama-rs>

# Scale jumlah replicas
kubectl scale rs <nama-rs> --replicas=5

# Hapus ReplicaSet (Pod ikut terhapus)
kubectl delete rs <nama-rs>

# Hapus ReplicaSet tapi biarkan Pod tetap ada
kubectl delete rs <nama-rs> --cascade=orphan
```

---

## ReplicaSet vs Deployment

> **Spoiler alert:** Dalam praktik sehari-hari, kamu hampir tidak pernah membuat ReplicaSet secara langsung!

**Kenapa?** Karena Deployment adalah "pembungkus" ReplicaSet yang menambahkan fitur:
- Rolling update
- Rollback
- History perubahan

| Fitur | ReplicaSet | Deployment |
|-------|-----------|-----------|
| Replikasi Pod | ✓ | ✓ (via ReplicaSet) |
| Self-healing | ✓ | ✓ |
| Rolling update | ✗ | ✓ |
| Rollback | ✗ | ✓ |
| History | ✗ | ✓ |

**Gunakan ReplicaSet jika:** Kamu butuh kontrol granular dan tidak perlu rolling update.

**Gunakan Deployment (hampir selalu):** Untuk semua aplikasi stateless di production.

---

## Latihan

```bash
# 1. Buat ReplicaSet
kubectl apply -f contoh-replicaset.yaml

# 2. Lihat ReplicaSet dan Pod yang dibuat
kubectl get rs
kubectl get pods -l app=nginx-rs

# 3. Hapus satu Pod, lihat apa yang terjadi
POD_NAME=$(kubectl get pods -l app=nginx-rs -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_NAME
kubectl get pods -l app=nginx-rs  # Pod baru akan muncul!

# 4. Scale up
kubectl scale rs nginx-replicaset --replicas=5
kubectl get pods

# 5. Scale down
kubectl scale rs nginx-replicaset --replicas=2
kubectl get pods

# 6. Cleanup
kubectl delete rs nginx-replicaset
```

---

*[Lihat contoh YAML →](./contoh-replicaset.yaml)*

*[Lanjut ke: Deployment →](../03-deployment/README.md)*

*[Kembali ke: Object Dasar](../README.md)*
