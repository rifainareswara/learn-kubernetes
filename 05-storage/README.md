# 05 - Storage di Kubernetes

Storage adalah tantangan tersendiri dalam dunia container. Container bersifat ephemeral (sementara) — semua data di dalam container akan hilang saat container restart. Kubernetes menyediakan berbagai solusi storage untuk mengatasi masalah ini.

---

## Mengapa Storage Perlu Diperhatikan?

```
Tanpa persistent storage:

Pod restart → Container mati → Data hilang!

Masalah:
- Database kehilangan semua data saat Pod restart
- File upload hilang
- State aplikasi tidak tersimpan
```

---

## Hierarki Storage di Kubernetes

```
StorageClass        ← "Template" cara membuat PV
    │
    └── PersistentVolume (PV)   ← Storage yang sebenarnya (disk fisik/cloud)
              │
              └── PersistentVolumeClaim (PVC)  ← "Klaim" storage oleh Pod
                        │
                        └── Pod  ← Menggunakan PVC sebagai Volume
```

---

## Jenis Storage di Kubernetes

| Jenis | Persistent? | Scope | Cocok untuk |
|-------|-------------|-------|------------|
| `emptyDir` | Tidak | Pod lifetime | Sharing data antar container dalam Pod |
| `hostPath` | Ya (di Node) | Node tertentu | Testing, DaemonSet |
| `PersistentVolume` | Ya | Cluster-wide | Database, file storage production |
| `ConfigMap/Secret` | Ya | Cluster-wide | Konfigurasi dan credentials |

---

## Daftar File

| File | Topik |
|------|-------|
| [01-volume-emptydir.yaml](./01-volume-emptydir.yaml) | Volume sementara untuk sharing data antar container |
| [02-persistentvolume.yaml](./02-persistentvolume.yaml) | Persistent Volume dan StorageClass |
| [03-persistentvolumeclaim.yaml](./03-persistentvolumeclaim.yaml) | PVC dan cara menggunakannya di Pod |

---

## Navigasi

- [Sebelumnya: 04 - Konfigurasi](../04-konfigurasi/README.md)
- [Selanjutnya: 06 - Object Lanjutan](../06-object-lanjutan/README.md)
- [Kembali ke README utama](../README.md)
