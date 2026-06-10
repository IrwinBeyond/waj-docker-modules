# Tugas Week 9 — Modul 2: Docker Service Mount

| | |
|---|---|
| **Mata Kuliah** | Workshop Administrasi Jaringan |
| **Nama** | Irwin Ahmad Wiryawan |
| **Kelas** | D4 IT B |
| **NRP** | 3124600035 |
| **Dosen Pengampu** | Dr Ferry Astika Saputra ST, M.Sc |

---

## PRE LAB

### 1. Apa perbedaan default bridge dan user-defined bridge network?

- Default bridge dibuat otomatis oleh Docker saat instalasi, sedangkan user-defined bridge dibuat manual oleh user menggunakan `docker network create`.
- User-defined bridge mendukung DNS resolution antar container berdasarkan nama container, sedangkan default bridge tidak mendukung hal tersebut secara otomatis.
- User-defined bridge memiliki isolasi network yang lebih baik dan container dapat di-connect atau disconnect secara dinamis tanpa restart container.
- Default bridge lebih cocok untuk container sederhana, sedangkan user-defined bridge lebih cocok untuk aplikasi multi-container.

### 2. Kapan menggunakan Volume vs Bind Mount vs tmpfs?

- Volume digunakan untuk data persisten seperti database karena dikelola langsung oleh Docker dan lebih portable.
- Bind Mount digunakan untuk development workflow, misalnya source code atau file konfigurasi yang perlu sinkron langsung dengan host.
- tmpfs digunakan untuk data sementara atau sensitif seperti cache dan secret karena disimpan di RAM dan akan hilang saat container stop atau restart.

### 3. Apa yang terjadi pada named volume saat `docker compose down`? Bagaimana jika pakai flag `-v`?

Secara default, named volume tidak dihapus saat menjalankan `docker compose down` sehingga data tetap persisten. Namun jika menggunakan `docker compose down -v`, maka seluruh volume yang terkait compose tersebut ikut dihapus sehingga data di dalamnya hilang.

### 4. Apa fungsi `depends_on` dan `healthcheck` di docker-compose.yml?

- `depends_on` digunakan untuk mengatur urutan startup antar service.
- `healthcheck` digunakan untuk memeriksa apakah service benar-benar siap digunakan.
- Kombinasi keduanya memastikan container dependent, seperti aplikasi Flask, baru berjalan setelah database PostgreSQL benar-benar ready menerima koneksi.

### 5. Mengapa user-defined bridge bisa DNS resolve nama container, sedangkan default bridge tidak?

Karena user-defined bridge memiliki embedded DNS server bawaan Docker yang secara otomatis memetakan nama container menjadi alamat IP container tersebut. Default bridge tidak menyediakan fitur DNS internal sehingga container harus menggunakan alamat IP secara manual.

---

## LANGKAH PRAKTIKUM

### Langkah 1: Docker Network

#### 1.1 Eksplorasi network default

```bash
# List semua network Docker
docker network ls
```

<img src="images/1-1.png" alt="Figure 1.1 — List network Docker" width="500">
*Gambar 1.1: Screenshot hasil `docker network ls` menampilkan daftar network bawaan Docker (bridge, host, none).*

```bash
# Inspect network bridge default
docker network inspect bridge
```

<img src="images/1-2.png" alt="Figure 1.2 — Inspect network bridge default" width="700">
*Gambar 1.2: Screenshot hasil `docker network inspect bridge` menampilkan detail konfigurasi default bridge dalam format JSON.*

#### 1.2 Buat user-defined bridge network

```bash
docker network create --driver bridge --subnet 172.20.0.0/16 lab-net
```

<img src="images/1-3.png" alt="Figure 1.3 — Membuat user-defined bridge network" width="600">
*Gambar 1.3: Screenshot hasil `docker network create --driver bridge --subnet 172.20.0.0/16 lab-net` — network ID ditampilkan.*

```bash
# Verifikasi
docker network ls
```

<img src="images/1-4.png" alt="Figure 1.4 — Verifikasi network lab-net" width="600">
*Gambar 1.4: Screenshot hasil `docker network ls` setelah pembuatan — network `lab-net` sudah muncul di daftar.*

```bash
docker network inspect lab-net
```

<img src="images/1-5.png" alt="Figure 1.5 — Inspect network lab-net" width="750">
*Gambar 1.5: Screenshot hasil `docker network inspect lab-net` menampilkan konfigurasi subnet 172.20.0.0/16 dan driver bridge.*

#### 1.3 Test DNS resolution antar container

```bash
# Jalankan 2 container di network yang sama
docker run -d --name server-a --network lab-net nginx:alpine
```

<img src="images/1-6.png" alt="Figure 1.6 — Menjalankan container server-a" width="700">
*Gambar 1.6: Screenshot hasil `docker run -d --name server-a --network lab-net nginx:alpine`.*

```bash
docker run -d --name server-b --network lab-net nginx:alpine
```

<img src="images/1-7.png" alt="Figure 1.7 — Menjalankan container server-b" width="600">
*Gambar 1.7: Screenshot hasil `docker run -d --name server-b --network lab-net nginx:alpine`.*

```bash
# Test DNS — container bisa saling resolve by nama
docker exec server-a ping -c 3 server-b
```

<img src="images/1-8.png" alt="Figure 1.8 — Ping dari server-a ke server-b" width="600">
*Gambar 1.8: Screenshot hasil `docker exec server-a ping -c 3 server-b` — DNS resolution berhasil, server-b dapat dijangkau berdasarkan nama container.*

```bash
docker exec server-b ping -c 3 server-a
```

<img src="images/1-9.png" alt="Figure 1.9 — Ping dari server-b ke server-a" width="600">
*Gambar 1.9: Screenshot hasil `docker exec server-b ping -c 3 server-a` — DNS resolution berhasil dua arah.*

```bash
# Bandingkan: default bridge TIDAK bisa resolve nama
docker run -d --name server-c nginx:alpine
docker run -d --name server-d nginx:alpine
docker exec server-c ping -c 3 server-d # GAGAL!
```

<img src="images/1-10.png" alt="Figure 1.10 — Default bridge gagal DNS resolution" width="500">
*Gambar 1.10: Screenshot hasil `docker exec server-c ping -c 3 server-d` pada default bridge — ping gagal karena tidak ada DNS resolution.*

```bash
# Cleanup
docker rm -f server-a server-b server-c server-d
```

<img src="images/1-11.png" alt="Figure 1.11 — Cleanup container network test" width="600">
*Gambar 1.11: Screenshot hasil `docker rm -f server-a server-b server-c server-d` — semua container test dihapus.*

---

### Langkah 2: Docker Volume

#### 2.1 Buat dan kelola volume

```bash
docker volume create data-vol
```

<img src="images/2-1.png" alt="Figure 2.1 — Membuat volume data-vol" width="500">
*Gambar 2.1: Screenshot hasil `docker volume create data-vol` — nama volume ditampilkan.*

```bash
docker volume ls
```

<img src="images/2-2.png" alt="Figure 2.2 — List volume Docker" width="500">
*Gambar 2.2: Screenshot hasil `docker volume ls` menampilkan volume `data-vol` yang baru dibuat.*

```bash
docker volume inspect data-vol
```

<img src="images/2-3.png" alt="Figure 2.3 — Inspect volume data-vol" width="600">
*Gambar 2.3: Screenshot hasil `docker volume inspect data-vol` menampilkan mountpoint dan metadata volume dalam format JSON.*

#### 2.2 Gunakan volume di container

```bash
# Container penulis — tulis timestamp ke volume setiap 5 detik
docker run -d --name writer \
  -v data-vol:/app/data \
  alpine:3.20 sh -c "while true; do date >> /app/data/log.txt; sleep 5; done"
```

<img src="images/2-4.png" alt="Figure 2.4 — Container writer dengan volume" width="700">
*Gambar 2.4: Screenshot hasil `docker run -d --name writer -v data-vol:/app/data alpine:3.20 ...` — container writer berjalan di background.*

```bash
# Tunggu 15 detik, lalu baca dari container BERBEDA
sleep 15
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
```

<img src="images/2-5.png" alt="Figure 2.5 — Membaca data dari container berbeda" width="700">
*Gambar 2.5: Screenshot hasil `docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt` — data timestamp terbaca dari volume yang sama.*

#### 2.3 Volume persist setelah container dihapus

```bash
docker rm -f writer
```

<img src="images/2-6.png" alt="Figure 2.6 — Menghapus container writer" width="500">
*Gambar 2.6: Screenshot hasil `docker rm -f writer` — container writer dihapus.*

```bash
# Data masih ada!
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
```

<img src="images/2-7.png" alt="Figure 2.7 — Data tetap ada setelah container dihapus" width="700">
*Gambar 2.7: Screenshot hasil `docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt` — data timestamp masih terbaca, membuktikan volume bersifat persisten.*

#### 2.4 Backup dan restore volume

```bash
# BACKUP
docker run --rm \
  -v data-vol:/source:ro \
  -v $(pwd):/backup \
  alpine:3.20 tar czf /backup/data-vol-backup.tar.gz -C /source .
```

<img src="images/2-8.png" alt="Figure 2.8 — Backup volume ke file tar.gz" width="600">
*Gambar 2.8: Screenshot hasil backup volume `data-vol` ke `data-vol-backup.tar.gz` di direktori host.*

```bash
# RESTORE ke volume baru
docker volume create data-vol-restored
```

<img src="images/2-9.png" alt="Figure 2.9 — Membuat volume untuk restore" width="600">
*Gambar 2.9: Screenshot hasil `docker volume create data-vol-restored` — volume baru untuk restore dibuat.*

```bash
docker run --rm \
  -v data-vol-restored:/target \
  -v $(pwd):/backup:ro \
  alpine:3.20 tar xzf /backup/data-vol-backup.tar.gz -C /target
```

<img src="images/2-10.png" alt="Figure 2.10 — Restore backup ke volume baru" width="600">
*Gambar 2.10: Screenshot hasil restore backup ke volume `data-vol-restored`.*

```bash
# Verifikasi
docker run --rm -v data-vol-restored:/data alpine:3.20 cat /data/log.txt
```

<img src="images/2-11.png" alt="Figure 2.11 — Verifikasi data restore" width="700">
*Gambar 2.11: Screenshot hasil `docker run --rm -v data-vol-restored:/data alpine:3.20 cat /data/log.txt` — data berhasil direstore.*

---

### Langkah 3: Bind Mount

#### 3.1 Buat project

```bash
mkdir -p ~/docker-lab/web-dev/html && cd ~/docker-lab/web-dev
```

<img src="images/3-1.png" alt="Figure 3.1 — Membuat direktori project bind mount" width="600">
*Gambar 3.1: Screenshot hasil `mkdir -p ~/docker-lab/web-dev/html && cd ~/docker-lab/web-dev`.*

```bash
cat > html/index.html << 'EOF'
<html>
<body>
<h1>Hello dari Bind Mount!</h1>
<p>Timestamp: VERSI-1</p>
</body>
</html>
EOF
```

<img src="images/3-2.png" alt="Figure 3.2 — Membuat file index.html" width="600">
*Gambar 3.2: Screenshot hasil pembuatan file `html/index.html` dengan konten HTML sederhana.*

#### 3.2 Jalankan container dengan bind mount

```bash
docker run -d --name dev-server \
  -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine
```

<img src="images/3-3.png" alt="Figure 3.3 — Container dengan bind mount" width="600">
*Gambar 3.3: Screenshot hasil `docker run -d --name dev-server -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html:ro nginx:alpine` — container berjalan dengan bind mount read-only.*

#### 3.3 Test live-reload

```bash
curl http://localhost:8080
```

<img src="images/3-4.png" alt="Figure 3.4 — Curl pertama ke localhost:8080" width="600">
*Gambar 3.4: Screenshot hasil `curl http://localhost:8080` menampilkan halaman "Hello dari Bind Mount!" dengan VERSI-1.*

```bash
# Edit file di HOST → langsung terlihat di container tanpa restart!
sed -i 's/VERSI-1/VERSI-2 (diedit live)/' html/index.html
curl http://localhost:8080
```

<img src="images/3-5.png" alt="Figure 3.5 — Live-reload setelah edit file host" width="600">
*Gambar 3.5: Screenshot hasil `curl http://localhost:8080` setelah edit file — konten berubah ke VERSI-2 tanpa restart container.*

```bash
docker rm -f dev-server
```

<img src="images/3-6.png" alt="Figure 3.6 — Cleanup container bind mount" width="500">
*Gambar 3.6: Screenshot hasil `docker rm -f dev-server` — container dev-server dihapus.*

---

### Langkah 4: tmpfs Mount

```bash
docker run -d --name tmpfs-demo \
  --tmpfs /app/cache:size=64m \
  alpine:3.20 sh -c "echo 'secret-data' > /app/cache/token.txt && sleep 3600"
```

<img src="images/4-1.png" alt="Figure 4.1 — Container dengan tmpfs mount" width="600">
*Gambar 4.1: Screenshot hasil `docker run -d --name tmpfs-demo --tmpfs /app/cache:size=64m alpine:3.20 ...` — container berjalan dengan tmpfs mount berukuran 64MB.*

```bash
# Baca data
docker exec tmpfs-demo cat /app/cache/token.txt
```

<img src="images/4-2.png" alt="Figure 4.2 — Membaca data dari tmpfs" width="500">
*Gambar 4.2: Screenshot hasil `docker exec tmpfs-demo cat /app/cache/token.txt` — data 'secret-data' terbaca dari tmpfs.*

```bash
# Stop & start → data HILANG
docker stop tmpfs-demo && docker start tmpfs-demo
```

<img src="images/4-3.png" alt="Figure 4.3 — Stop dan start ulang container tmpfs" width="600">
*Gambar 4.3: Screenshot hasil `docker stop tmpfs-demo && docker start tmpfs-demo` — container di-restart.*

```bash
docker exec tmpfs-demo cat /app/cache/token.txt # File tidak ada!
```

<img src="images/4-4.png" alt="Figure 4.4 — Data tmpfs hilang setelah restart" width="500">
*Gambar 4.4: Screenshot hasil `docker exec tmpfs-demo cat /app/cache/token.txt` — error atau file tidak ditemukan, membuktikan data tmpfs hilang setelah container restart.*

```bash
docker rm -f tmpfs-demo
```

<img src="images/4-5.png" alt="Figure 4.5 — Cleanup container tmpfs" width="500">
*Gambar 4.5: Screenshot hasil `docker rm -f tmpfs-demo` — container tmpfs dihapus.*

---

### Langkah 5: Aplikasi Multi-Container dengan Docker Compose

#### 5.1 Buat project structure

```bash
mkdir -p ~/docker-lab/compose-app/{html,app} && cd ~/docker-lab/compose-app
```

<img src="images/5-1.png" alt="Figure 5.1 — Membuat direktori project compose" width="600">
*Gambar 5.1: Screenshot hasil `mkdir -p ~/docker-lab/compose-app/{html,app} && cd ~/docker-lab/compose-app`.*

#### 5.2 Buat halaman Nginx

```bash
cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8"><title>Docker Compose Lab</title>
  <style>
    body { font-family: sans-serif; max-width: 700px; margin: 40px auto; padding: 20px; }
    .card { background: white; border-radius: 10px; padding: 25px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; }
    #result { background: #263238; color: #80CBC4; padding: 15px;
              border-radius: 8px; font-family: monospace; white-space: pre-wrap; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🐳 Docker Compose Lab</h1>
    <p>Nginx → Flask → PostgreSQL</p>
    <button onclick="fetchData()">Cek Koneksi Backend</button>
    <div id="result">Klik tombol untuk test...</div>
  </div>
  <script>
    async function fetchData() {
      try {
        const r = await fetch('/api/health');
        document.getElementById('result').textContent = JSON.stringify(await r.json(), null, 2);
      } catch(e) { document.getElementById('result').textContent = 'Error: ' + e.message; }
    }
  </script>
</body></html>
EOF
```

<img src="images/5-2.png" alt="Figure 5.2 — Membuat halaman HTML Nginx" width="750">
*Gambar 5.2: Screenshot hasil pembuatan file `html/index.html` dengan UI untuk Docker Compose Lab.*

#### 5.3 Buat Flask app + Dockerfile

```bash
cat > app/requirements.txt << 'EOF'
flask==3.1.*
psycopg2-binary==2.9.*
EOF
```

<img src="images/5-3.png" alt="Figure 5.3 — Membuat requirements.txt" width="600">
*Gambar 5.3: Screenshot hasil pembuatan file `app/requirements.txt` berisi Flask dan psycopg2-binary.*

```bash
cat > app/app.py << 'PYEOF'
import os, socket, datetime
from flask import Flask, jsonify
import psycopg2

app = Flask(__name__)

@app.route("/api/health")
def health():
    result = {"status": "ok", "hostname": socket.gethostname(),
              "timestamp": datetime.datetime.now().isoformat()}
    try:
        conn = psycopg2.connect(
            host=os.environ.get("DB_HOST", "db"),
            dbname=os.environ.get("DB_NAME", "labdb"),
            user=os.environ.get("DB_USER", "labuser"),
            password=os.environ.get("DB_PASS", "labpass123"))
        cur = conn.cursor()
        cur.execute("SELECT version();")
        result["database"] = cur.fetchone()[0]
        result["db_status"] = "connected"
        cur.close(); conn.close()
    except Exception as e:
        result["db_status"] = f"error: {e}"
    return jsonify(result)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
PYEOF
```

<img src="images/5-4.png" alt="Figure 5.4 — Membuat Flask app.py" width="750">
*Gambar 5.4: Screenshot hasil pembuatan file `app/app.py` — aplikasi Flask dengan endpoint `/api/health` dan koneksi ke PostgreSQL.*

```bash
cat > app/Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

<img src="images/5-5.png" alt="Figure 5.5 — Membuat Dockerfile untuk Flask" width="600">
*Gambar 5.5: Screenshot hasil pembuatan file `app/Dockerfile` — multi-stage build untuk aplikasi Flask.*

#### 5.4 Buat Nginx config

```bash
cat > nginx.conf << 'EOF'
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    location /api/ {
        proxy_pass http://app:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

<img src="images/5-6.png" alt="Figure 5.6 — Membuat konfigurasi Nginx reverse proxy" width="700">
*Gambar 5.6: Screenshot hasil pembuatan file `nginx.conf` — konfigurasi reverse proxy `/api/` ke container Flask.*

#### 5.5 Buat `docker-compose.yml`

```bash
cat > docker-compose.yml << 'EOF'
services:
  web:
    image: nginx:alpine
    container_name: lab-web
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend
    depends_on:
      - app
    restart: unless-stopped

  app:
    build: ./app
    container_name: lab-app
    environment:
      - DB_HOST=db
      - DB_NAME=labdb
      - DB_USER=labuser
      - DB_PASS=labpass123
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: lab-db
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpass123
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pg-data:

networks:
  frontend:
  backend:
EOF
```

<img src="images/5-7.png" alt="Figure 5.7 — Membuat docker-compose.yml" width="750">
*Gambar 5.7: Screenshot hasil pembuatan file `docker-compose.yml` — definisi 3 service (web, app, db) dengan network dan volume.*

#### 5.6 Jalankan dan verifikasi

```bash
# Build dan start
docker compose up --build -d
```

<img src="images/5-8.png" alt="Figure 5.8 — Docker compose up --build -d" width="750">
*Gambar 5.8: Screenshot hasil `docker compose up --build -d` — build image Flask dan menjalankan semua service.*

```bash
# Cek status
docker compose ps
```

<img src="images/5-9.png" alt="Figure 5.9 — Docker compose ps" width="700">
*Gambar 5.9: Screenshot hasil `docker compose ps` menampilkan status ketiga service (lab-web, lab-app, lab-db) dalam keadaan Up.*

```bash
# Test
curl http://localhost:8080
```

<img src="images/5-10.png" alt="Figure 5.10 — Curl ke halaman utama" width="750">
*Gambar 5.10: Screenshot hasil `curl http://localhost:8080` menampilkan halaman HTML Docker Compose Lab.*

```bash
curl http://localhost:8080/api/health | python3 -m json.tool
```

<img src="images/5-11.png" alt="Figure 5.11 — Test endpoint API health" width="700">
*Gambar 5.11: Screenshot hasil `curl http://localhost:8080/api/health | python3 -m json.tool` — response JSON dari Flask dengan status koneksi database.*

```bash
# Cek network dan volume
docker network ls
docker volume ls
```

<img src="images/5-12.png" alt="Figure 5.12 — Cek network dan volume" width="700">
*Gambar 5.12: Screenshot hasil `docker network ls` dan `docker volume ls` — network frontend/backend dan volume pg-data terlihat.*

```bash
# Lifecycle
docker compose logs -f # follow log
```

<img src="images/5-13.png" alt="Figure 5.13 — Docker compose logs -f" width="750">
*Gambar 5.13: Screenshot hasil `docker compose logs -f` menampilkan log real-time dari semua service.*

```bash
docker compose stop # stop semua
```

<img src="images/5-14.png" alt="Figure 5.14 — Docker compose stop" width="600">
*Gambar 5.14: Screenshot hasil `docker compose stop` — semua service dihentikan.*

```bash
docker compose start # start kembali
```

<img src="images/5-15.png" alt="Figure 5.15 — Docker compose start" width="600">
*Gambar 5.15: Screenshot hasil `docker compose start` — semua service dijalankan kembali.*

```bash
docker compose down # stop + hapus container/network (volume tetap)
```

<img src="images/5-16.png" alt="Figure 5.16 — Docker compose down" width="600">
*Gambar 5.16: Screenshot hasil `docker compose down` — container dan network dihapus, volume pg-data tetap ada.*

```bash
docker compose down -v # HATI-HATI: hapus termasuk volume!
```

<img src="images/5-17.png" alt="Figure 5.17 — Docker compose down -v" width="600">
*Gambar 5.17: Screenshot hasil `docker compose down -v` — container, network, dan volume pg-data dihapus.*

---

## POST LAB

### 1. Jalankan `docker network inspect lab-frontend`. Sebutkan container dan IP masing-masing.

Container yang terhubung pada network `lab-frontend` adalah:
- `lab-web` dengan IP pada subnet frontend Docker.
- `lab-app` dengan IP pada subnet frontend Docker.

IP dapat berbeda pada setiap environment karena Docker melakukan assignment secara dinamis ketika container dibuat.

### 2. Hapus container `lab-db` lalu `docker compose up -d` lagi. Apakah data PostgreSQL masih ada? Mengapa?

Data PostgreSQL tetap ada karena database menggunakan named volume `pg-data`. Data disimpan di volume Docker, bukan di writable layer container. Saat container dibuat ulang, volume yang sama akan di-mount kembali sehingga data tetap persisten.

### 3. Tunjukkan perbedaan output `docker inspect` untuk mount type volume vs bind.

- Mount type volume memiliki `"Type": "volume"` dan source path berada di direktori internal Docker seperti `/var/lib/docker/volumes/...`.
- Mount type bind memiliki `"Type": "bind"` dan source path menunjuk langsung ke direktori host yang ditentukan user, misalnya `/home/user/project/html`.
- Volume dikelola Docker, sedangkan bind mount bergantung langsung pada filesystem host.

### 4. Jelaskan alur request dari browser ke Nginx ke Flask ke PostgreSQL.

- Browser mengirim HTTP request ke container Nginx melalui port mapping host.
- Nginx menerima request dan meneruskan request API `/api/health` ke container Flask menggunakan reverse proxy.
- Flask memproses request lalu melakukan koneksi ke PostgreSQL menggunakan hostname service `db` pada Docker network `backend`.
- PostgreSQL mengembalikan hasil query ke Flask.
- Flask mengembalikan response JSON ke Nginx, kemudian Nginx meneruskannya kembali ke browser.

### 5. Bandingkan ukuran image yang digunakan stack ini. Mana terbesar dan mengapa?

- `nginx:alpine` memiliki ukuran kecil karena menggunakan Alpine Linux yang minimalis.
- `postgres:16-alpine` lebih besar karena membawa PostgreSQL server beserta dependency database engine.
- Image terbesar biasanya adalah `python:3.11-slim` hasil build Flask app karena membawa interpreter Python, pip package, dan dependency aplikasi tambahan seperti Flask dan psycopg2.

Trade-off image besar adalah penggunaan storage dan bandwidth lebih tinggi, namun dependency aplikasi menjadi lebih lengkap dan kompatibel.

---

---

*Laporan ini dibuat sebagai bagian dari praktikum Workshop Administrasi Jaringan, Program Studi D4 Teknik Informatika, Politeknik Elektronika Negeri Surabaya (PENS), 2026.*
