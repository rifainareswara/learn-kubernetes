# CI/CD dengan GitHub Actions

GitHub Actions memungkinkan kamu mengotomasi build, test, dan deploy langsung dari GitHub repository.

---

## Konsep GitHub Actions

```
.github/workflows/deploy.yml    ← File workflow
         │
         ▼
GitHub Actions Runner
    1. Checkout kode
    2. Build Docker image
    3. Push ke registry (Docker Hub / GCR / ECR)
    4. Update Kubernetes deployment
    5. Verifikasi deploy berhasil
```

---

## Workflow Dasar: Build dan Deploy ke Kubernetes

Buat file `.github/workflows/deploy.yml` di repository kamu:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy to Kubernetes

# Trigger: jalankan workflow saat push ke branch main
on:
  push:
    branches:
      - main
  # Atau trigger manual
  workflow_dispatch:

# Environment variables global
env:
  REGISTRY: docker.io              # Docker Hub
  IMAGE_NAME: ${{ github.repository }}  # username/reponame

jobs:
  # ================================================================
  # JOB 1: Build dan Push Docker Image
  # ================================================================
  build-and-push:
    name: Build Docker Image
    runs-on: ubuntu-latest         # Runner yang digunakan
    
    # Output: image tag yang akan dipakai di job berikutnya
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build-push.outputs.digest }}

    steps:
    # Step 1: Checkout kode
    - name: Checkout repository
      uses: actions/checkout@v4

    # Step 2: Setup Docker Buildx untuk multi-platform build
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    # Step 3: Login ke registry
    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}  # GitHub Secret
        password: ${{ secrets.DOCKER_PASSWORD }}  # GitHub Secret

    # Step 4: Generate image metadata (tags, labels)
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix={{branch}}-,format=short  # main-abc1234
          type=raw,value=latest,enable={{is_default_branch}}  # latest (hanya di main)

    # Step 5: Build dan push image
    - name: Build and push Docker image
      id: build-push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true                    # Push ke registry
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha          # GitHub Actions cache
        cache-to: type=gha,mode=max

  # ================================================================
  # JOB 2: Deploy ke Kubernetes
  # ================================================================
  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-push            # Jalankan setelah job sebelumnya

    # Hanya deploy dari branch main
    if: github.ref == 'refs/heads/main'

    steps:
    # Step 1: Checkout kode (untuk mengakses manifest YAML)
    - name: Checkout repository
      uses: actions/checkout@v4

    # Step 2: Setup kubectl
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    # Step 3: Configure kubeconfig (dari GitHub Secret)
    - name: Configure kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config
        chmod 600 ~/.kube/config

    # Step 4: Update image di Deployment
    - name: Deploy to Kubernetes
      run: |
        # Dapatkan tag image yang baru di-build
        IMAGE_TAG="${{ needs.build-and-push.outputs.image-tag }}"
        SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
        FULL_IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main-${SHORT_SHA}"
        
        echo "Deploying image: $FULL_IMAGE"
        
        # Update image di deployment
        kubectl set image deployment/my-app \
          my-app=$FULL_IMAGE \
          --namespace=production
        
        # Tunggu rollout selesai
        kubectl rollout status deployment/my-app \
          --namespace=production \
          --timeout=5m

    # Step 5: Verifikasi deploy
    - name: Verify deployment
      run: |
        kubectl get pods -n production -l app=my-app
        kubectl get service -n production my-app-service

    # Step 6: Notifikasi Slack (opsional)
    - name: Notify Slack
      if: always()          # Jalankan meski ada kegagalan
      uses: slackapi/slack-github-action@v1.25.0
      with:
        channel-id: 'C0123456789'
        slack-message: |
          Deployment *${{ github.repository }}* ke production:
          Status: ${{ job.status == 'success' && '✅ Berhasil' || '❌ Gagal' }}
          Commit: ${{ github.sha }}
          Branch: ${{ github.ref_name }}
      env:
        SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## Setup GitHub Secrets

Tambahkan secrets di GitHub: **Settings → Secrets and variables → Actions**

```
DOCKER_USERNAME     : username Docker Hub
DOCKER_PASSWORD     : password Docker Hub
KUBECONFIG          : isi kubeconfig (base64 encoded)
SLACK_BOT_TOKEN     : token Slack bot (opsional)
```

```bash
# Cara encode kubeconfig ke base64:
cat ~/.kube/config | base64 | tr -d '\n'
# Copy output dan simpan sebagai GitHub Secret KUBECONFIG
```

---

## Workflow Lengkap dengan Testing

```yaml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Job 1: Test
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    - name: Install dependencies
      run: npm ci
    - name: Run tests
      run: npm test
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  # Job 2: Build (hanya jika test lulus)
  build:
    name: Build Image
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: myuser/my-app
        tags: type=sha,format=short
    - uses: docker/build-push-action@v5
      with:
        push: true
        tags: ${{ steps.meta.outputs.tags }}

  # Job 3: Deploy ke Staging
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment: staging       # Gunakan environment protection rules
    steps:
    - uses: actions/checkout@v4
    - name: Deploy to staging
      run: |
        # Sesuaikan namespace dan credentials staging
        echo "Deploying to staging..."

  # Job 4: Deploy ke Production (butuh approval manual)
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com  # URL environment di GitHub
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying to production..."
```

---

## Menggunakan Helm di GitHub Actions

```yaml
- name: Setup Helm
  uses: azure/setup-helm@v3
  with:
    version: 'v3.13.0'

- name: Deploy with Helm
  run: |
    helm upgrade --install my-app ./charts/my-app \
      --namespace production \
      --set image.tag=${{ needs.build.outputs.image-tag }} \
      --set replicaCount=3 \
      --atomic \
      --timeout 5m
```

---

## Tips dan Best Practices

1. **Selalu gunakan tag spesifik** (bukan `latest`) saat deploy production
2. **Gunakan Environment Protection** untuk production (butuh manual approval)
3. **Simpan KUBECONFIG terenkripsi** di GitHub Secrets
4. **Gunakan dedicated service account** dengan minimal permission untuk CI/CD
5. **Tambahkan smoke tests** setelah deploy untuk verifikasi

---

*[Kembali ke: Overview CI/CD](./README.md)*

*[Lanjut ke: ArgoCD →](./02-argocd.md)*
