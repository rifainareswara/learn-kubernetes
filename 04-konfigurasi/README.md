# 04 - Konfigurasi Kubernetes

Di modul ini kamu akan belajar cara mengelola konfigurasi dan rahasia (secret) di Kubernetes dengan baik dan aman.

---

## Mengapa Perlu Modul Konfigurasi?

Dalam praktik modern, ada prinsip penting: **"Pisahkan konfigurasi dari kode"** (12-Factor App).

```
JANGAN lakukan ini:
┌──────────────────────────────────┐
│ const DB_HOST = "10.244.1.5";   │
│ const DB_PASS = "mysecret123";  │ ← Hardcoded di kode!
└──────────────────────────────────┘

LAKUKAN ini:
┌──────────────────────────────────┐
│ const DB_HOST = process.env     │
│   .DB_HOST;                     │ ← Dari environment variable
│ const DB_PASS = process.env     │
│   .DB_PASS;                     │ ← Dari Secret
└──────────────────────────────────┘
```

---

## Daftar Materi

| Object | Deskripsi | Untuk Apa |
|--------|-----------|-----------|
| [Namespace](./01-namespace/README.md) | Isolasi logis dalam cluster | Memisahkan dev/staging/prod |
| [ConfigMap](./02-configmap/README.md) | Data konfigurasi non-sensitif | DB host, app port, feature flags |
| [Secret](./03-secret/README.md) | Data sensitif terenkripsi | Password, API key, token |

---

## Kapan Pakai ConfigMap vs Secret?

| Data | Gunakan |
|------|---------|
| DB_HOST=postgres-service | ConfigMap |
| APP_PORT=8080 | ConfigMap |
| FEATURE_FLAG_NEW_UI=true | ConfigMap |
| DB_PASSWORD=s3cr3t | Secret |
| API_KEY=abc123xyz | Secret |
| TLS certificate | Secret |
| Docker registry credential | Secret |

**Aturan mudah:** Jika kamu malu data itu terlihat di log atau terminal, gunakan Secret!

---

## Navigasi

- [Sebelumnya: 03 - Object Dasar](../03-object-dasar/README.md)
- [Selanjutnya: 05 - Storage](../05-storage/README.md)
- [Kembali ke README utama](../README.md)
