# =============================================================================
# Multi-stage Dockerfile untuk Svelte Frontend
# =============================================================================
# Stage 1 (builder): Kompilasi Svelte menggunakan Node.js
# Stage 2 (runtime): Serve file statis hasil kompilasi menggunakan Nginx
#
# Hasilnya: image ~25MB (dibanding ~600MB jika pakai Node.js sebagai server)
# =============================================================================

# ─── Stage 1: Builder ─────────────────────────────────────────────────────────
# Gunakan Node.js Alpine untuk ukuran image builder yang lebih kecil
FROM node:20-alpine AS builder

WORKDIR /app

# Salin package files terlebih dahulu (manfaatkan Docker layer caching)
# Layer ini hanya di-rebuild jika package.json atau package-lock.json berubah
COPY package*.json ./

# Gunakan 'npm ci' bukan 'npm install' untuk:
# - Instalasi yang lebih cepat dan konsisten
# - Menggunakan exact versions dari package-lock.json
# - Ideal untuk CI/CD pipeline
RUN npm ci

# Salin semua kode sumber
COPY . .

# VITE_API_URL: URL API backend yang akan digunakan oleh frontend
# Default: /api (diteruskan oleh Ingress ke backend service)
# Bisa di-override saat build: docker build --build-arg VITE_API_URL=http://api.example.com
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL

# Kompilasi Svelte ke file HTML/CSS/JS statis
# Output akan berada di folder /app/dist
RUN npm run build


# ─── Stage 2: Runtime (Nginx) ─────────────────────────────────────────────────
# Gunakan Nginx Alpine — image yang sangat kecil (~5MB) untuk serve file statis
FROM nginx:alpine

# Label metadata
LABEL maintainer="team@myapp.com"
LABEL description="Todo App Frontend - Svelte + Nginx"
LABEL version="1.0.0"

# Salin hasil build dari stage builder ke direktori web Nginx
COPY --from=builder /app/dist /usr/share/nginx/html

# Salin konfigurasi Nginx kustom
# File ini menangani SPA routing (try_files fallback ke index.html)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80 (port default HTTP Nginx)
EXPOSE 80

# Jalankan Nginx di foreground (daemon off)
# Penting: Kubernetes memerlukan proses di foreground untuk mendeteksi status container
CMD ["nginx", "-g", "daemon off;"]
