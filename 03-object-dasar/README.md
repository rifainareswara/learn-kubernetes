# 03 - Object Dasar Kubernetes

Di modul ini kamu akan belajar empat object paling fundamental di Kubernetes. Kuasai ini dulu sebelum lanjut ke object yang lebih kompleks.

---

## Hierarki Object

```
Deployment
└── mengelola
    └── ReplicaSet
        └── mengelola
            └── Pod
                └── berisi
                    └── Container(s)

Service
└── mengekspos
    └── Pod(s) ← dipilih via Labels
```

---

## Daftar Materi

| Object | Deskripsi | Analogi |
|--------|-----------|---------|
| [Pod](./01-pod/README.md) | Unit terkecil, satu atau beberapa container | Satu unit apartemen |
| [ReplicaSet](./02-replicaset/README.md) | Memastikan N copy Pod berjalan | Manager yang pastikan N karyawan hadir |
| [Deployment](./03-deployment/README.md) | Mengelola ReplicaSet + rolling update | Manajer senior yang urus update pula |
| [Service](./04-service/README.md) | Mengekspos Pod ke network | Resepsionis yang terima panggilan untuk Pod |

---

## Urutan Belajar yang Disarankan

1. **Pod** — Pahami unit terkecil dulu
2. **ReplicaSet** — Pahami bagaimana Pod direplikasi
3. **Deployment** — Yang paling sering dipakai di production
4. **Service** — Cara Pod bisa diakses

---

## kubectl Cheatsheet untuk Modul Ini

```bash
# Pod
kubectl get pods
kubectl describe pod <nama>
kubectl logs <nama>
kubectl exec -it <nama> -- /bin/bash
kubectl delete pod <nama>

# Deployment
kubectl get deployments
kubectl describe deployment <nama>
kubectl rollout status deployment <nama>
kubectl rollout history deployment <nama>
kubectl rollout undo deployment <nama>
kubectl scale deployment <nama> --replicas=5

# Service
kubectl get services
kubectl describe service <nama>
```

---

## Navigasi

- [Sebelumnya: 02 - Setup Lokal](../02-setup-lokal/README.md)
- [Selanjutnya: 04 - Konfigurasi](../04-konfigurasi/README.md)
- [Kembali ke README utama](../README.md)
