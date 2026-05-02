# =============================================================================
# Multi-stage Dockerfile untuk Python FastAPI
# =============================================================================
# Menggunakan multi-stage build untuk menghasilkan image yang lebih kecil.
# Stage builder: install semua dependensi Python
# Stage runtime: hanya berisi aplikasi + dependensi yang sudah terinstall
# =============================================================================

# ─── Stage 1: Builder ─────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Install dependensi sistem yang mungkin dibutuhkan oleh beberapa paket Python
# (misalnya psycopg2 butuh libpq-dev, tapi kita pakai psycopg2-binary jadi tidak perlu)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Salin requirements terlebih dahulu (manfaatkan Docker layer caching)
# Jika requirements.txt tidak berubah, layer ini akan di-cache
COPY requirements.txt .

# Install ke --user path agar bisa disalin ke stage berikutnya
RUN pip install --no-cache-dir --user -r requirements.txt


# ─── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM python:3.12-slim

# Label untuk metadata image (opsional, tapi bagus untuk dokumentasi)
LABEL maintainer="team@myapp.com"
LABEL description="Todo App Backend - FastAPI"
LABEL version="1.0.0"

WORKDIR /app

# Salin hasil install dari stage builder
COPY --from=builder /root/.local /root/.local

# Salin kode aplikasi
COPY . .

# Tambahkan path instalasi user ke PATH
ENV PATH=/root/.local/bin:$PATH

# Nonaktifkan penulisan file .pyc (menghemat disk dan mempercepat startup)
ENV PYTHONDONTWRITEBYTECODE=1

# Nonaktifkan buffering output (log langsung muncul, penting untuk debug di Kubernetes)
ENV PYTHONUNBUFFERED=1

# Port yang akan di-expose (harus sama dengan --port di CMD)
EXPOSE 8000

# Jalankan uvicorn sebagai ASGI server
# --host 0.0.0.0: terima koneksi dari semua interface (penting di container)
# --port 8000: port yang digunakan
# Catatan: JANGAN gunakan --reload di production
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
