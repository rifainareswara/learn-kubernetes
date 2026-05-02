# 11 - CI/CD untuk Kubernetes

CI/CD (Continuous Integration / Continuous Delivery) mengotomasi proses dari push kode hingga deploy ke Kubernetes.

---

## Dua Pendekatan CI/CD untuk Kubernetes

### Push-based CI/CD (GitHub Actions)
```
Developer push → GitHub → GitHub Actions → kubectl apply → Cluster
```
- Sederhana dan familiar
- Pipeline langsung push ke cluster
- Kurang aman (CI server butuh akses cluster)

### Pull-based CI/CD / GitOps (ArgoCD)
```
Developer push → Git Repo → ArgoCD (watch) → apply ke Cluster
```
- Lebih aman (cluster pull dari Git, bukan CI push ke cluster)
- Git sebagai single source of truth
- Auto-sync dan self-healing

---

## Daftar Materi

| File | Topik |
|------|-------|
| [01 - GitHub Actions](./01-github-actions.md) | CI/CD dengan GitHub Actions |
| [02 - ArgoCD](./02-argocd.md) | GitOps dengan ArgoCD |

---

## Navigasi

- [Sebelumnya: 10 - Observability](../10-observability/README.md)
- [Kembali ke README utama](../README.md)
