# DaemonSet

DaemonSet memastikan bahwa **setiap Node** dalam cluster menjalankan satu copy dari Pod tertentu. Saat Node baru ditambahkan ke cluster, Pod otomatis ditambahkan ke Node tersebut.

---

## Apa itu DaemonSet?

```
Cluster dengan 3 Node:

Tanpa DaemonSet:
Node 1: [App A] [App B]
Node 2: [App A]
Node 3: [App A] [App B] [App C]
(Pod tersebar tidak merata)

Dengan DaemonSet (log-collector):
Node 1: [App A] [App B] [log-collector]  ← 1 log-collector per Node
Node 2: [App A]          [log-collector]
Node 3: [App A] [App B]  [log-collector]
```

Saat Node 4 ditambahkan:
```
Node 4: [log-collector]  ← Otomatis ditambahkan!
```

---

## Kapan Menggunakan DaemonSet?

DaemonSet cocok untuk:
- **Log collectors** — Fluentd, Filebeat yang perlu baca log dari setiap Node
- **Monitoring agents** — Node Exporter, Datadog Agent
- **Network plugins** — CNI plugins (Calico, Flannel) sudah pakai DaemonSet
- **Storage daemons** — Ceph, GlusterFS
- **Security agents** — Antivirus, IDS/IPS

---

## DaemonSet vs Deployment

| Aspek | DaemonSet | Deployment |
|-------|-----------|-----------|
| Jumlah Pod | 1 per Node | N (bebas) |
| Scheduling | Di setiap Node | Di Node manapun |
| Scale | Auto (ikut jumlah Node) | Manual atau HPA |
| Use case | System daemon, agent | Aplikasi |

---

## Perintah kubectl untuk DaemonSet

```bash
# Lihat semua DaemonSet
kubectl get daemonsets
kubectl get ds  # shorthand

# Detail DaemonSet
kubectl describe ds log-collector

# Lihat Pod DaemonSet berjalan di Node mana
kubectl get pods -l app=log-collector -o wide

# Update DaemonSet (rolling update)
kubectl set image ds/log-collector log-collector=fluent/fluentd:v1.16

# Lihat status update
kubectl rollout status ds/log-collector

# Rollback
kubectl rollout undo ds/log-collector
```

---

## Update Strategy DaemonSet

```yaml
updateStrategy:
  type: RollingUpdate          # RollingUpdate atau OnDelete
  rollingUpdate:
    maxUnavailable: 1          # Maksimal 1 Node tidak tersedia saat update
    maxSurge: 0                # DaemonSet tidak support surge (tidak seperti Deployment)
```

---

## Latihan

```bash
# 1. Apply DaemonSet
kubectl apply -f contoh-daemonset.yaml

# 2. Lihat DaemonSet
kubectl get ds

# 3. Lihat Pod berjalan di Node mana
kubectl get pods -l app=node-exporter -o wide

# 4. Jumlah Pod harus sama dengan jumlah Node
kubectl get nodes | wc -l  # Jumlah Node
kubectl get pods -l app=node-exporter | wc -l  # Jumlah Pod (sama!)

# 5. Cleanup
kubectl delete ds node-exporter-ds
```

---

*[Lihat contoh YAML →](./contoh-daemonset.yaml)*

*[Lanjut ke: StatefulSet →](../03-statefulset/README.md)*

*[Kembali ke: Object Lanjutan](../README.md)*
