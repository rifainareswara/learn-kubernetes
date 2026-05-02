# 00 - Prasyarat Sebelum Belajar Kubernetes

Sebelum memulai perjalanan belajar Kubernetes, ada beberapa pengetahuan dasar yang perlu kamu miliki. Panduan ini akan membantumu mengevaluasi kesiapanmu dan memberikan sumber belajar untuk area yang masih perlu diperkuat.

---

## Checklist Prasyarat

### 1. Docker (WAJIB)

Docker adalah fondasi dari Kubernetes. Kamu harus sudah familiar dengan Docker sebelum belajar Kubernetes.

**Checklist Docker:**
- [ ] Bisa menginstall dan menjalankan Docker
- [ ] Memahami konsep image vs container
- [ ] Bisa menjalankan `docker run`, `docker build`, `docker ps`
- [ ] Bisa membuat `Dockerfile` sederhana
- [ ] Memahami konsep port mapping (`-p 8080:80`)
- [ ] Memahami konsep volume mounting (`-v`)
- [ ] Bisa menggunakan Docker Compose untuk multi-container

**Uji dirimu — bisakah kamu menjawab pertanyaan ini?**

1. Apa perbedaan antara Docker Image dan Docker Container?
2. Apa itu Dockerfile dan untuk apa digunakan?
3. Bagaimana cara melihat semua container yang sedang berjalan?
4. Apa perintah untuk menghentikan semua container yang berjalan?

**Contoh soal praktik:**
```bash
# Bisakah kamu menjalankan nginx di Docker dan mengaksesnya di browser?
docker run -d -p 8080:80 --name my-nginx nginx

# Verifikasi: buka http://localhost:8080
```

**Sumber belajar Docker:**
- [Dokumentasi resmi Docker](https://docs.docker.com/get-started/)
- [Docker untuk Pemula (YouTube)](https://www.youtube.com/results?search_query=belajar+docker+bahasa+indonesia)
- [Play with Docker](https://labs.play-with-docker.com/) — belajar gratis di browser

---

### 2. Command Line Interface / Linux CLI (WAJIB)

Kubernetes dioperasikan 90% melalui terminal. Kamu perlu nyaman dengan perintah-perintah Linux dasar.

**Checklist CLI:**
- [ ] Navigasi direktori: `cd`, `ls`, `pwd`, `mkdir`
- [ ] Membaca dan mengedit file: `cat`, `less`, `nano`/`vim`
- [ ] Manajemen proses: `ps`, `kill`, `top`/`htop`
- [ ] Jaringan: `curl`, `wget`, `ping`, `netstat`/`ss`
- [ ] Pipe dan redirection: `|`, `>`, `>>`, `<`
- [ ] Variabel environment: `export`, `echo $VAR`
- [ ] Permissions: `chmod`, `chown`
- [ ] SSH dasar

**Uji dirimu:**

1. Bagaimana cara mencari teks "error" di dalam file log?
2. Bagaimana cara melihat port mana saja yang sedang terbuka di sistem?
3. Apa perbedaan `>` dan `>>` dalam shell?

**Contoh perintah yang sering dipakai:**
```bash
# Perintah yang SERING digunakan saat bekerja dengan Kubernetes
grep "Error" /var/log/app.log          # Cari teks di file
tail -f /var/log/app.log               # Lihat log real-time
curl -X GET http://localhost:8080      # Test HTTP request
export KUBECONFIG=~/.kube/config       # Set environment variable
```

**Sumber belajar CLI:**
- [Linux Command (website)](https://linuxcommand.org/)
- [The Art of Command Line (GitHub)](https://github.com/jlevy/the-art-of-command-line)

---

### 3. Konsep Jaringan Dasar (PENTING)

Kubernetes sangat bergantung pada konsep jaringan. Kamu tidak perlu jadi network engineer, tapi perlu memahami konsep dasar.

**Checklist Networking:**
- [ ] Memahami IP address (IPv4: 192.168.x.x, 10.x.x.x)
- [ ] Memahami port dan protokol (HTTP:80, HTTPS:443, SSH:22)
- [ ] Memahami DNS (domain name → IP address)
- [ ] Memahami konsep load balancing (distribusi traffic)
- [ ] Memahami konsep firewall/network policy dasar
- [ ] Memahami HTTP request/response (status code: 200, 404, 500)

**Uji dirimu:**

1. Apa perbedaan antara IP private (192.168.x.x) dan IP public?
2. Apa itu DNS dan bagaimana cara kerjanya?
3. Apa artinya port 8080:80 dalam Docker?
4. Apa itu load balancer?

**Konsep jaringan penting dalam Kubernetes:**
```
Client → Load Balancer → Service → Pod → Container
         (external)     (K8s)    (K8s)
```

**Sumber belajar Networking:**
- [Computer Networking Course - Kunal Kushwaha](https://www.youtube.com/watch?v=IPvYjXCsTg8)
- [Networking concepts for Kubernetes](https://www.youtube.com/watch?v=5cNrTU6o3Fw)

---

### 4. Format YAML (WAJIB)

Hampir semua konfigurasi Kubernetes ditulis dalam format YAML. Kamu harus memahaminya dengan baik.

**Checklist YAML:**
- [ ] Memahami struktur key-value
- [ ] Memahami indentasi (YAML sangat sensitif terhadap spasi!)
- [ ] Memahami tipe data: string, number, boolean, list, object
- [ ] Bisa membaca dan menulis YAML tanpa error syntax
- [ ] Memahami perbedaan `-` (list) vs `key:` (object)

**Contoh YAML yang perlu kamu pahami:**
```yaml
# Ini adalah komentar di YAML

# Key-value sederhana
nama: "Kubernetes"
versi: 1.28
aktif: true

# List (array)
bahasa-pemrograman:
  - Python
  - Go
  - Java

# Object bersarang (nested)
server:
  host: localhost
  port: 8080
  ssl:
    enabled: true
    cert: /path/to/cert

# List of objects
pods:
  - name: web-pod
    image: nginx:1.25
    port: 80
  - name: api-pod
    image: node:18
    port: 3000
```

**Kesalahan YAML yang paling sering terjadi:**
```yaml
# SALAH - menggunakan tab
server:
	port: 8080  # Tab tidak boleh digunakan!

# BENAR - menggunakan spasi
server:
  port: 8080  # Gunakan 2 spasi

# SALAH - indentasi tidak konsisten
metadata:
  name: test
   labels:  # 3 spasi, tidak konsisten!
    app: test

# BENAR
metadata:
  name: test
  labels:
    app: test
```

**Tools untuk validasi YAML:**
- [YAML Lint Online](https://www.yamllint.com/)
- Extension VSCode: "YAML" by Red Hat
- `python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"` — validasi di terminal

**Sumber belajar YAML:**
- [Learn YAML in Y Minutes](https://learnxinyminutes.com/docs/yaml/)
- [YAML Tutorial (TutorialsPoint)](https://www.tutorialspoint.com/yaml/index.htm)

---

### 5. Pemahaman Dasar tentang Container & Microservices (DIREKOMENDASIKAN)

**Checklist:**
- [ ] Memahami perbedaan container vs virtual machine
- [ ] Memahami konsep microservices vs monolith
- [ ] Memahami mengapa container lebih efisien dari VM
- [ ] Memahami konsep "immutable infrastructure"

**Analogi sederhana:**
```
Virtual Machine = Rumah dengan fondasi, dinding, atap sendiri
Container       = Apartemen - berbagi fondasi & infrastruktur, tapi ruangan terpisah
```

---

## Evaluasi Kesiapanmu

Jawab jujur pertanyaan-pertanyaan ini:

### Pertanyaan Self-Assessment

**Docker (skor maks 5):**
1. Apa perintah untuk melihat semua Docker image yang ada di mesin kamu? *(1 poin)*
2. Apa perbedaan `CMD` dan `ENTRYPOINT` di Dockerfile? *(1 poin)*
3. Bagaimana cara menghubungkan dua container agar bisa saling berkomunikasi? *(1 poin)*
4. Apa itu Docker registry dan apa perbedaan Docker Hub vs private registry? *(1 poin)*
5. Bagaimana cara membuat image dari container yang sudah berjalan? *(1 poin)*

**Linux CLI (skor maks 5):**
1. Bagaimana cara mencari file bernama "config.yaml" di seluruh sistem? *(1 poin)*
2. Apa perintah untuk melihat penggunaan disk per direktori? *(1 poin)*
3. Bagaimana cara menjalankan proses di background? *(1 poin)*
4. Apa perbedaan `sudo` dan `su`? *(1 poin)*
5. Bagaimana cara melihat semua variabel environment yang aktif? *(1 poin)*

**Networking (skor maks 5):**
1. Apa range IP address untuk subnet 192.168.1.0/24? *(1 poin)*
2. Apa perbedaan TCP dan UDP? *(1 poin)*
3. Bagaimana cara mengecek apakah port 80 terbuka di sebuah server? *(1 poin)*
4. Apa itu NAT (Network Address Translation)? *(1 poin)*
5. Bagaimana cara DNS resolver bekerja? *(1 poin)*

### Interpretasi Skor

| Skor Total | Kesiapan |
|------------|----------|
| 12-15 | Siap mulai Kubernetes sekarang! |
| 8-11 | Hampir siap, perkuat 1-2 area yang lemah |
| 5-7 | Perlu 1-2 minggu perkuat fondasi dulu |
| 0-4 | Mulai dari Docker dan Linux CLI dulu |

---

## Rencana Belajar yang Disarankan

### Jika kamu pemula total (belum familiar Docker):
```
Minggu 1: Belajar Docker basics
Minggu 2: Belajar Linux CLI + YAML
Minggu 3: Mulai Kubernetes dari modul 01
```

### Jika kamu sudah familiar Docker tapi belum pernah pakai Kubernetes:
```
Hari 1-2: Review checklist prasyarat
Hari 3: Mulai modul 01-fondasi
```

### Jika kamu sudah pernah pakai Kubernetes:
```
Langsung ke bagian yang belum dikuasai
Gunakan sebagai referensi cepat
```

---

## Setup Environment Awal

Sebelum lanjut, pastikan laptop/komputermu memenuhi spesifikasi minimum:

**Spesifikasi Minimum:**
- RAM: 8 GB (16 GB direkomendasikan untuk Minikube)
- CPU: 2 core (4 core direkomendasikan)
- Storage: 20 GB free space
- OS: macOS, Linux, atau Windows dengan WSL2

**Verifikasi Docker sudah terinstall:**
```bash
docker --version
# Output yang diharapkan: Docker version 24.x.x, build xxxxxxx

docker run hello-world
# Output yang diharapkan: "Hello from Docker!"
```

---

Jika semua checklist di atas sudah hijau, kamu siap melanjutkan ke **[01 - Fondasi Kubernetes](../01-fondasi/README.md)**!

---

*[Kembali ke README utama](../README.md)*
