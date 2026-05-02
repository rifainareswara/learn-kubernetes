# 10 - Observability: Monitoring dan Logging

Observability adalah kemampuan untuk memahami kondisi sistem dari output eksternalnya. Di Kubernetes, ini mencakup tiga pilar: **Metrics**, **Logging**, dan **Tracing**.

---

## Tiga Pilar Observability

```
┌─────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                         │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Metrics    │  │   Logging    │  │   Tracing    │   │
│  │              │  │              │  │              │   │
│  │ "Berapa CPU  │  │ "Apa yang    │  │ "Request ini │   │
│  │  dipakai?"   │  │  terjadi?"   │  │  lewat mana?"│   │
│  │              │  │              │  │              │   │
│  │ Prometheus   │  │ Loki/ELK     │  │ Jaeger/Tempo │   │
│  │ +Grafana     │  │ +Grafana     │  │ +Grafana     │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Daftar Materi

| File | Topik |
|------|-------|
| [01 - Prometheus & Grafana](./01-prometheus-grafana.md) | Setup monitoring stack lengkap |
| [02 - Logging](./02-logging.md) | Centralized logging dengan Loki |

---

## Navigasi

- [Sebelumnya: 09 - Cloud](../09-cloud/README.md)
- [Selanjutnya: 11 - CI/CD](../11-cicd/README.md)
- [Kembali ke README utama](../README.md)
