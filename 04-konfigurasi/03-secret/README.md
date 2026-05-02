# Secret

Secret adalah object Kubernetes untuk menyimpan data **sensitif** seperti password, API key, token, dan certificate. Berbeda dari ConfigMap, data dalam Secret di-encode (bukan dienkripsi secara default) dan dikelola secara lebih hati-hati.

---

## Secret vs ConfigMap

| Aspek | ConfigMap | Secret |
|-------|-----------|--------|
| Tipe data | Non-sensitif | Sensitif |
| Encoding | Plain text | Base64 encoded |
| Enkripsi at rest | Tidak (default) | Bisa dikonfigurasi |
| Tampil di logs | Ya | Tidak (disensor) |
| Contoh | DB host, port | Password, API key |

> **Penting:** Secret secara default hanya di-encode Base64, BUKAN dienkripsi! Siapa pun yang bisa akses etcd atau baca Secret bisa decode nilainya. Untuk keamanan production, aktifkan **encryption at rest** atau gunakan **Vault/External Secrets**.

---

## Jenis-Jenis Secret

| Type | Gunakan untuk |
|------|--------------|
| `Opaque` | Data umum (password, API key) |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate dan private key |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/ssh-auth` | SSH credentials |
| `kubernetes.io/basic-auth` | Username + password |

---

## Membuat Secret

### Cara 1: Command line (untuk testing)

```bash
# Secret tipe Opaque
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=mysupersecretpassword \
  --from-literal=DB_USER=admin \
  --from-literal=API_KEY=abc123xyz

# Secret dari file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./server.crt \
  --from-file=tls.key=./server.key

# Docker registry secret
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=email@example.com

# TLS secret
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### Cara 2: YAML manifest

```bash
# Encode value ke base64 dulu
echo -n "mysecretpassword" | base64
# Output: bXlzZWNyZXRwYXNzd29yZA==
```

```yaml
# Lihat contoh-secret.yaml
```

---

## Menggunakan Secret di Pod

### Sebagai Environment Variable

```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret      # Nama Secret
          key: DB_PASSWORD     # Key dalam Secret
    envFrom:
    - secretRef:
        name: db-secret        # Semua key jadi env var
```

### Sebagai Volume Mount

```yaml
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400       # Permission file (read-only untuk owner)

  containers:
  - name: app
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
```

---

## Perintah kubectl untuk Secret

```bash
# Lihat semua Secret
kubectl get secrets

# Detail Secret (value tidak ditampilkan)
kubectl describe secret db-secret

# Lihat isi Secret (dalam base64)
kubectl get secret db-secret -o yaml

# Decode nilai Secret
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode

# Edit Secret
kubectl edit secret db-secret

# Hapus Secret
kubectl delete secret db-secret
```

---

## Best Practices untuk Secret

1. **Jangan simpan Secret di Git** — gunakan `.gitignore` atau sealed-secrets
2. **Gunakan RBAC** untuk batasi siapa yang bisa baca Secret
3. **Aktifkan encryption at rest** di production
4. **Pertimbangkan Vault** (HashiCorp) atau External Secrets Operator
5. **Rotate secrets secara rutin**
6. **Monitor akses** ke Secret di audit logs

---

## Latihan

```bash
# 1. Buat Secret
kubectl apply -f contoh-secret.yaml

# 2. Lihat Secret (tidak tampilkan nilai)
kubectl get secrets
kubectl describe secret db-credentials

# 3. Decode nilai manual
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
echo ""  # newline

# 4. Buat Pod yang menggunakan Secret
kubectl run test-secret --image=busybox --restart=Never \
  --env="DB_PASS=$(kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode)" \
  -- sh -c "echo Password: $DB_PASS; sleep 3600"

# Atau dengan referensi ke Secret:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-with-secret
spec:
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo DB_PASS=$DB_PASSWORD; sleep 3600']
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

kubectl logs test-with-secret

# 5. Cleanup
kubectl delete pod test-with-secret test-secret
kubectl delete -f contoh-secret.yaml
```

---

*[Lihat contoh YAML →](./contoh-secret.yaml)*

*[Lanjut ke: Storage →](../../05-storage/README.md)*

*[Kembali ke: Konfigurasi](../README.md)*
