# Tugas Week 9 — Modul 3: Web Service Docker

| | |
|---|---|
| **Mata Kuliah** | Workshop Administrasi Jaringan |
| **Nama** | Irwin Ahmad Wiryawan |
| **Kelas** | D4 IT B |
| **NRP** | 3124600035 |
| **Dosen Pengampu** | Dr Ferry Astika Saputra ST, M.Sc |

---

## PRE LAB

### 1. Apa keuntungan menjalankan web server di container dibandingkan langsung di host?

- Konfigurasi antar aplikasi menjadi terisolasi sehingga tidak saling konflik dependency maupun port.
- Deployment menjadi lebih konsisten karena environment container sama di setiap mesin.
- Proses scaling, backup, dan redeploy menjadi lebih mudah.
- Container lebih ringan dibanding virtual machine sehingga penggunaan resource lebih efisien.
- Web server dapat dipindahkan atau direplikasi dengan cepat menggunakan image Docker.

### 2. Jelaskan perbedaan document root Apache (/usr/local/apache2/htdocs/) vs Nginx (/usr/share/nginx/html/).

- Apache menggunakan default document root di `/usr/local/apache2/htdocs/` untuk menyimpan file website yang akan disajikan ke client.
- Nginx menggunakan default document root di `/usr/share/nginx/html/`.
- Keduanya memiliki fungsi sama sebagai lokasi file HTML, CSS, JavaScript, dan asset web lainnya, tetapi berbeda struktur direktori sesuai implementasi image resmi masing-masing web server.

### 3. Apa itu SSL Termination dan mengapa dilakukan di reverse proxy?

SSL Termination adalah proses dekripsi koneksi HTTPS pada reverse proxy sebelum request diteruskan ke backend service. SSL dilakukan di reverse proxy agar backend tidak perlu menangani enkripsi sendiri sehingga konfigurasi SSL lebih terpusat, performa lebih efisien, dan manajemen certificate menjadi lebih mudah.

### 4. Apa perbedaan name-based dan IP-based virtual hosting?

- Name-based virtual hosting membedakan website berdasarkan hostname atau domain pada header Host, sehingga beberapa website dapat menggunakan satu alamat IP yang sama.
- IP-based virtual hosting menggunakan alamat IP berbeda untuk setiap website.
- Name-based virtual hosting lebih umum digunakan karena lebih hemat penggunaan IP address.

### 5. Mengapa self-signed certificate menghasilkan warning di browser?

Karena self-signed certificate tidak ditandatangani oleh Certificate Authority (CA) terpercaya. Browser tidak dapat memverifikasi identitas pemilik certificate sehingga menganggap koneksi berpotensi tidak aman dan menampilkan warning keamanan.

---

## LANGKAH PRAKTIKUM

### Langkah 0: Persiapan Project

```bash
mkdir -p ~/docker-lab/web-service/{apache/{sites,html-site1,html-site2},nginx,flask,certs,logs}
cd ~/docker-lab/web-service
```

![Figure 0.1 — Membuat struktur direktori project](images/placeholder.png)
*Gambar 0.1: Screenshot hasil `mkdir -p` membuat struktur direktori project web-service.*

---

### Langkah 1: Konfigurasi Apache2 dengan Virtual Host

#### 1.1 Buat halaman web untuk dua virtual host

```bash
# Site 1: Company Profile
cat > apache/html-site1/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><title>Site 1 - Company</title>
<style>body{font-family:sans-serif;text-align:center;padding:50px;background:#1a237e;color:white;}
.box{background:rgba(255,255,255,0.1);padding:30px;border-radius:12px;max-width:500px;margin:0 auto;}</style>
</head>
<body><div class="box">
<h1>🏢 Site 1 — Company Profile</h1>
<p>Virtual Host: <strong>site1.lab</strong></p>
<p>Server: Apache httpd di Docker</p>
</div></body></html>
EOF
```

![Figure 1.1 — Membuat halaman Site 1](images/placeholder.png)
*Gambar 1.1: Screenshot hasil pembuatan file `apache/html-site1/index.html` untuk Company Profile.*

```bash
# Site 2: Blog
cat > apache/html-site2/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head><meta charset="UTF-8"><title>Site 2 - Blog</title>
<style>body{font-family:sans-serif;text-align:center;padding:50px;background:#1b5e20;color:white;}
.box{background:rgba(255,255,255,0.1);padding:30px;border-radius:12px;max-width:500px;margin:0 auto;}</style>
</head>
<body><div class="box">
<h1>📝 Site 2 — Blog</h1>
<p>Virtual Host: <strong>site2.lab</strong></p>
<p>Server: Apache httpd di Docker</p>
</div></body></html>
EOF
```

![Figure 1.2 — Membuat halaman Site 2](images/placeholder.png)
*Gambar 1.2: Screenshot hasil pembuatan file `apache/html-site2/index.html` untuk Blog.*

#### 1.2 Buat konfigurasi Apache Virtual Host

```bash
cat > apache/sites/vhosts.conf << 'EOF'
# ============================================
# Apache Virtual Host Configuration (Docker)
# ============================================

# --- Site 1: site1.lab ---
<VirtualHost *:80>
    ServerName site1.lab
    ServerAlias www.site1.lab
    DocumentRoot /usr/local/apache2/htdocs/site1
    <Directory /usr/local/apache2/htdocs/site1>
        AllowOverride None
        Require all granted
    </Directory>
    # Per-site logging
    ErrorLog /var/log/apache2/site1-error.log
    CustomLog /var/log/apache2/site1-access.log combined
</VirtualHost>

# --- Site 2: site2.lab ---
<VirtualHost *:80>
    ServerName site2.lab
    ServerAlias www.site2.lab
    DocumentRoot /usr/local/apache2/htdocs/site2
    <Directory /usr/local/apache2/htdocs/site2>
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog /var/log/apache2/site2-error.log
    CustomLog /var/log/apache2/site2-access.log combined
</VirtualHost>
EOF
```

![Figure 1.3 — Membuat konfigurasi Virtual Host Apache](images/placeholder.png)
*Gambar 1.3: Screenshot hasil pembuatan file `apache/sites/vhosts.conf` berisi dua Virtual Host.*

#### 1.3 Buat Dockerfile Apache

```bash
cat > apache/Dockerfile << 'EOF'
FROM httpd:2.4-alpine

# Aktifkan module yang dibutuhkan
RUN sed -i \
    -e 's/#LoadModule vhost_alias_module/LoadModule vhost_alias_module/' \
    -e 's/#LoadModule rewrite_module/LoadModule rewrite_module/' \
    /usr/local/apache2/conf/httpd.conf

# Include virtual host config
RUN echo "Include conf/extra/vhosts.conf" >> /usr/local/apache2/conf/httpd.conf

# Buat direktori log
RUN mkdir -p /var/log/apache2

# Copy virtual host config
COPY sites/vhosts.conf /usr/local/apache2/conf/extra/vhosts.conf

# Copy site files
COPY html-site1/ /usr/local/apache2/htdocs/site1/
COPY html-site2/ /usr/local/apache2/htdocs/site2/

EXPOSE 80
EOF
```

![Figure 1.4 — Membuat Dockerfile Apache](images/placeholder.png)
*Gambar 1.4: Screenshot hasil pembuatan `apache/Dockerfile` untuk build image Apache dengan Virtual Host.*

---

### Langkah 2: Generate Self-Signed SSL Certificate

```bash
# Generate SSL certificate untuk *.lab (wildcard)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/C=ID/ST=Jawa Timur/L=Surabaya/O=PENS Lab/CN=*.lab" \
  -addext "subjectAltName=DNS:*.lab,DNS:site1.lab,DNS:site2.lab,DNS:app.lab"
```

![Figure 2.1 — Generate self-signed SSL certificate](images/placeholder.png)
*Gambar 2.1: Screenshot hasil `openssl req -x509` menghasilkan self-signed certificate untuk wildcard `*.lab`.*

```bash
# Verifikasi certificate
openssl x509 -in certs/server.crt -noout -text | head -20
```

![Figure 2.2 — Verifikasi certificate](images/placeholder.png)
*Gambar 2.2: Screenshot hasil `openssl x509 -in certs/server.crt -noout -text | head -20` menampilkan detail certificate.*

```bash
ls -la certs/
```

![Figure 2.3 — List file certificate](images/placeholder.png)
*Gambar 2.3: Screenshot hasil `ls -la certs/` menampilkan file `server.key` dan `server.crt` beserta permission-nya.*

---

### Langkah 3: Konfigurasi Nginx sebagai Reverse Proxy + SSL Termination

```bash
cat > nginx/default.conf << 'EOF'
# ============================================
# Nginx Reverse Proxy + SSL Configuration
# ============================================

# --- Upstream definitions ---
upstream apache_backend {
    server apache-web:80;
}

upstream flask_backend {
    server flask-app:5000;
}

# --- HTTP: Redirect ke HTTPS ---
server {
    listen 80;
    server_name site1.lab site2.lab app.lab;
    return 301 https://$host$request_uri;
}

# --- HTTPS: site1.lab → Apache ---
server {
    listen 443 ssl;
    server_name site1.lab;

    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/site1-access.log;
    error_log /var/log/nginx/site1-error.log;
}

# --- HTTPS: site2.lab → Apache ---
server {
    listen 443 ssl;
    server_name site2.lab;

    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/site2-access.log;
    error_log /var/log/nginx/site2-error.log;
}

# --- HTTPS: app.lab → Flask ---
server {
    listen 443 ssl;
    server_name app.lab;

    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://flask_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/app-access.log;
    error_log /var/log/nginx/app-error.log;
}

# --- Default: catch-all ---
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    return 444;
}
EOF
```

![Figure 3.1 — Membuat konfigurasi Nginx Reverse Proxy + SSL](images/placeholder.png)
*Gambar 3.1: Screenshot hasil pembuatan file `nginx/default.conf` berisi konfigurasi reverse proxy Nginx dengan SSL termination.*

---

### Langkah 4: Buat Flask Backend App

```bash
cat > flask/requirements.txt << 'EOF'
flask==3.1.*
psycopg2-binary==2.9.*
EOF
```

![Figure 4.1 — Membuat requirements.txt Flask](images/placeholder.png)
*Gambar 4.1: Screenshot hasil pembuatan file `flask/requirements.txt` berisi dependensi Flask dan psycopg2.*

```bash
cat > flask/app.py << 'PYEOF'
import os, socket, datetime
from flask import Flask, jsonify, request
import psycopg2

app = Flask(__name__)

def get_db():
    return psycopg2.connect(
        host=os.environ.get("DB_HOST", "db"),
        dbname=os.environ.get("DB_NAME", "labdb"),
        user=os.environ.get("DB_USER", "labuser"),
        password=os.environ.get("DB_PASS", "labpass123"))

@app.route("/")
def index():
    return jsonify({
        "service": "Flask Backend API",
        "hostname": socket.gethostname(),
        "timestamp": datetime.datetime.now().isoformat(),
        "client_ip": request.headers.get("X-Real-IP", request.remote_addr),
        "proto": request.headers.get("X-Forwarded-Proto", "http")
    })

@app.route("/api/health")
def health():
    result = {"status": "ok", "database": "unknown"}
    try:
        conn = get_db()
        cur = conn.cursor()
        cur.execute("SELECT version();")
        result["database"] = cur.fetchone()[0]
        result["db_status"] = "connected"
        cur.close(); conn.close()
    except Exception as e:
        result["db_status"] = f"error: {e}"
    return jsonify(result)

@app.route("/api/visitors", methods=["POST"])
def add_visitor():
    try:
        conn = get_db(); cur = conn.cursor()
        cur.execute("""
            CREATE TABLE IF NOT EXISTS visitors (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100),
                visited_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )""")
        name = request.json.get("name", "anonymous")
        cur.execute("INSERT INTO visitors (name) VALUES (%s) RETURNING id, visited_at", (name,))
        row = cur.fetchone()
        conn.commit(); cur.close(); conn.close()
        return jsonify({"id": row[0], "name": name, "visited_at": str(row[1])}), 201
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route("/api/visitors", methods=["GET"])
def get_visitors():
    try:
        conn = get_db(); cur = conn.cursor()
        cur.execute("SELECT id, name, visited_at FROM visitors ORDER BY id DESC LIMIT 20")
        rows = [{"id": r[0], "name": r[1], "visited_at": str(r[2])} for r in cur.fetchall()]
        cur.close(); conn.close()
        return jsonify(rows)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
PYEOF
```

![Figure 4.2 — Membuat aplikasi Flask](images/placeholder.png)
*Gambar 4.2: Screenshot hasil pembuatan file `flask/app.py` berisi kode API Flask dengan endpoint visitors CRUD.*

```bash
cat > flask/Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

![Figure 4.3 — Membuat Dockerfile Flask](images/placeholder.png)
*Gambar 4.3: Screenshot hasil pembuatan `flask/Dockerfile` untuk build image Flask backend.*

---

### Langkah 5: Buat Docker Compose

```bash
cat > docker-compose.yml << 'EOF'
services:

  # --- Nginx Reverse Proxy + SSL Termination ---
  proxy:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
      - nginx-logs:/var/log/nginx
    networks:
      - web-net
    depends_on:
      - apache-web
      - flask-app
    restart: unless-stopped

  # --- Apache Web Server (Virtual Hosts) ---
  apache-web:
    build: ./apache
    container_name: apache-web
    volumes:
      - apache-logs:/var/log/apache2
    networks:
      - web-net
    restart: unless-stopped

  # --- Flask Backend API ---
  flask-app:
    build: ./flask
    container_name: flask-app
    environment:
      - DB_HOST=db
      - DB_NAME=labdb
      - DB_USER=labuser
      - DB_PASS=labpass123
    networks:
      - web-net
      - db-net
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  # --- PostgreSQL Database ---
  db:
    image: postgres:16-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpass123
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - db-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pg-data:
  nginx-logs:
  apache-logs:

networks:
  web-net:
  db-net:
EOF
```

![Figure 5.1 — Membuat docker-compose.yml](images/placeholder.png)
*Gambar 5.1: Screenshot hasil pembuatan file `docker-compose.yml` berisi 4 service: proxy, apache-web, flask-app, dan db.*

---

### Langkah 6: Deploy dan Testing

#### 6.1 Tambahkan DNS lokal

```bash
# Tambahkan entry ke /etc/hosts
echo "127.0.0.1 site1.lab site2.lab app.lab" | sudo tee -a /etc/hosts
```

![Figure 6.1 — Menambahkan DNS lokal ke /etc/hosts](images/placeholder.png)
*Gambar 6.1: Screenshot hasil menambahkan entry `127.0.0.1 site1.lab site2.lab app.lab` ke `/etc/hosts`.*

#### 6.2 Build dan jalankan

```bash
docker compose up --build -d
```

![Figure 6.2 — Docker compose up build](images/placeholder.png)
*Gambar 6.2: Screenshot hasil `docker compose up --build -d` — semua service berhasil di-build dan dijalankan di background.*

```bash
docker compose ps
```

![Figure 6.3 — Docker compose ps](images/placeholder.png)
*Gambar 6.3: Screenshot hasil `docker compose ps` menampilkan status semua container (proxy, apache-web, flask-app, db).*

#### 6.3 Test Virtual Host Apache via Nginx Proxy

```bash
# Test HTTP → HTTPS redirect
curl -I http://site1.lab:8080
```

![Figure 6.4 — Test redirect HTTP ke HTTPS](images/placeholder.png)
*Gambar 6.4: Screenshot hasil `curl -I http://site1.lab:8080` menampilkan response HTTP 301 redirect ke HTTPS.*

```bash
# Test Site 1 (Apache via Nginx proxy, skip SSL verify)
curl -k https://site1.lab:8443
# Harus tampil: "Site 1 — Company Profile"
```

![Figure 6.5 — Test Site 1 via HTTPS](images/placeholder.png)
*Gambar 6.5: Screenshot hasil `curl -k https://site1.lab:8443` menampilkan halaman "Site 1 — Company Profile" dari Apache.*

```bash
# Test Site 2
curl -k https://site2.lab:8443
# Harus tampil: "Site 2 — Blog"
```

![Figure 6.6 — Test Site 2 via HTTPS](images/placeholder.png)
*Gambar 6.6: Screenshot hasil `curl -k https://site2.lab:8443` menampilkan halaman "Site 2 — Blog" dari Apache.*

```bash
# Test Flask API
curl -k https://app.lab:8443
curl -k https://app.lab:8443/api/health | python3 -m json.tool
```

![Figure 6.7 — Test Flask API](images/placeholder.png)
*Gambar 6.7: Screenshot hasil `curl -k https://app.lab:8443` dan `curl -k https://app.lab:8443/api/health` menampilkan response JSON dari Flask backend.*

#### 6.4 Test API CRUD

```bash
# Tambah visitor
curl -k -X POST https://app.lab:8443/api/visitors \
  -H "Content-Type: application/json" \
  -d '{"name": "Mahasiswa PENS"}'
```

![Figure 6.8 — POST visitor ke API](images/placeholder.png)
*Gambar 6.8: Screenshot hasil POST ke `/api/visitors` — response JSON berisi id, name, dan visited_at dari visitor baru.*

```bash
# Lihat daftar visitor
curl -k https://app.lab:8443/api/visitors | python3 -m json.tool
```

![Figure 6.9 — GET daftar visitor](images/placeholder.png)
*Gambar 6.9: Screenshot hasil GET `/api/visitors` menampilkan daftar visitor dalam format JSON yang terformat rapi.*

#### 6.5 Cek SSL Certificate

```bash
# Lihat detail certificate
echo | openssl s_client -connect site1.lab:8443 -servername site1.lab 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates
```

![Figure 6.10 — Cek SSL certificate](images/placeholder.png)
*Gambar 6.10: Screenshot hasil `openssl s_client` menampilkan subject, issuer, dan masa berlaku certificate self-signed.*

#### 6.6 Analisis Log

```bash
# Log Nginx
docker exec nginx-proxy cat /var/log/nginx/site1-access.log
```

![Figure 6.11 — Log akses Nginx Site 1](images/placeholder.png)
*Gambar 6.11: Screenshot hasil `docker exec nginx-proxy cat /var/log/nginx/site1-access.log` menampilkan log akses ke site1.lab.*

```bash
docker exec nginx-proxy cat /var/log/nginx/app-access.log
```

![Figure 6.12 — Log akses Nginx App](images/placeholder.png)
*Gambar 6.12: Screenshot hasil `docker exec nginx-proxy cat /var/log/nginx/app-access.log` menampilkan log akses ke app.lab.*

```bash
# Log Apache
docker exec apache-web cat /var/log/apache2/site1-access.log
```

![Figure 6.13 — Log akses Apache Site 1](images/placeholder.png)
*Gambar 6.13: Screenshot hasil `docker exec apache-web cat /var/log/apache2/site1-access.log` menampilkan log akses Apache.*

```bash
# Atau akses via volume
docker run --rm -v $(docker compose config --volumes | head -1):/logs alpine ls /logs
```

![Figure 6.14 — Akses log via Docker volume](images/placeholder.png)
*Gambar 6.14: Screenshot hasil akses log container melalui Docker volume dengan container sementara Alpine.*

---

## POST LAB

### 1. Bandingkan response header dari Apache vs Nginx. Header apa yang menunjukkan software web server?

Header yang menunjukkan software web server adalah header `Server`.
- Apache biasanya menampilkan header seperti `Server: Apache/2.4.x`.
- Nginx biasanya menampilkan header seperti `Server: nginx/1.x`.
Header tersebut menunjukkan software web server yang menangani request client.

![Figure PostLab 1 — Perbandingan response header Apache vs Nginx](images/placeholder.png)
*Gambar PostLab 1: Screenshot perbandingan response header dari `curl -I` ke Apache langsung dan melalui Nginx proxy, menunjukkan header `Server` yang berbeda.*

### 2. Jika Nginx proxy down, apakah Apache masih bisa diakses langsung? Bagaimana cara testnya?

Apache masih dapat diakses langsung jika port Apache diekspos ke host atau dilakukan port forwarding manual. Pengujian dapat dilakukan menggunakan curl langsung ke port Apache container, misalnya:

```bash
docker run -p 8081:80 apache-web
curl http://localhost:8081
```

Jika halaman tetap muncul, berarti Apache tetap berjalan meskipun reverse proxy Nginx down.

![Figure PostLab 2 — Test Apache langsung tanpa Nginx](images/placeholder.png)
*Gambar PostLab 2: Screenshot hasil `curl http://localhost:8081` mengakses Apache langsung tanpa melalui reverse proxy Nginx.*

### 3. Tunjukkan bahwa X-Real-IP header diteruskan dengan benar dari Nginx ke Flask.

Header `X-Real-IP` diteruskan oleh Nginx menggunakan konfigurasi:

```
proxy_set_header X-Real-IP $remote_addr;
```

Pada Flask, nilai header dapat dilihat menggunakan:

```python
request.headers.get("X-Real-IP")
```

Jika response Flask menampilkan alamat IP client asli, maka forwarding header berhasil.

![Figure PostLab 3 — Verifikasi X-Real-IP header forwarding](images/placeholder.png)
*Gambar PostLab 3: Screenshot response JSON dari Flask endpoint `/` yang menampilkan `client_ip` sesuai dengan IP client asli, membuktikan `X-Real-IP` berhasil diteruskan.*

### 4. Jelaskan mengapa Flask app perlu terhubung ke dua network (web-net dan db-net).

Flask app perlu berada di dua network karena berfungsi sebagai penghubung antara frontend dan database.
- `web-net` digunakan agar Flask dapat menerima request dari Nginx reverse proxy.
- `db-net` digunakan agar Flask dapat berkomunikasi dengan PostgreSQL.
Dengan pemisahan network, database menjadi lebih terisolasi dan tidak langsung terekspos ke frontend.

![Figure PostLab 4 — Ilustrasi network isolation](images/placeholder.png)
*Gambar PostLab 4: Screenshot hasil `docker network inspect web-net` dan `docker network inspect db-net` menunjukkan container yang terhubung ke masing-masing network.*

### 5. Apa yang terjadi jika file server.key atau server.crt dihapus saat container running?

Jika file certificate atau private key dihapus saat container sedang berjalan, koneksi HTTPS dapat gagal ketika Nginx melakukan reload atau restart konfigurasi. Nginx membutuhkan kedua file tersebut untuk proses SSL handshake. Jika salah satu file hilang, service HTTPS tidak dapat berjalan dengan benar dan biasanya menghasilkan error SSL atau gagal start.

![Figure PostLab 5 — Error SSL saat certificate dihapus](images/placeholder.png)
*Gambar PostLab 5: Screenshot error yang muncul saat mencoba akses HTTPS setelah file `server.key` atau `server.crt` dihapus dari container.*

---

## DAFTAR GAMBAR

| Figure | Deskripsi |
|--------|-----------|
| 0.1 | Membuat struktur direktori project |
| 1.1 | Membuat halaman Site 1 |
| 1.2 | Membuat halaman Site 2 |
| 1.3 | Membuat konfigurasi Virtual Host Apache |
| 1.4 | Membuat Dockerfile Apache |
| 2.1 | Generate self-signed SSL certificate |
| 2.2 | Verifikasi certificate |
| 2.3 | List file certificate |
| 3.1 | Membuat konfigurasi Nginx Reverse Proxy + SSL |
| 4.1 | Membuat requirements.txt Flask |
| 4.2 | Membuat aplikasi Flask |
| 4.3 | Membuat Dockerfile Flask |
| 5.1 | Membuat docker-compose.yml |
| 6.1 | Menambahkan DNS lokal ke /etc/hosts |
| 6.2 | Docker compose up build |
| 6.3 | Docker compose ps |
| 6.4 | Test redirect HTTP ke HTTPS |
| 6.5 | Test Site 1 via HTTPS |
| 6.6 | Test Site 2 via HTTPS |
| 6.7 | Test Flask API |
| 6.8 | POST visitor ke API |
| 6.9 | GET daftar visitor |
| 6.10 | Cek SSL certificate |
| 6.11 | Log akses Nginx Site 1 |
| 6.12 | Log akses Nginx App |
| 6.13 | Log akses Apache Site 1 |
| 6.14 | Akses log via Docker volume |
| PostLab 1 | Perbandingan response header Apache vs Nginx |
| PostLab 2 | Test Apache langsung tanpa Nginx |
| PostLab 3 | Verifikasi X-Real-IP header forwarding |
| PostLab 4 | Ilustrasi network isolation |
| PostLab 5 | Error SSL saat certificate dihapus |

---

*Laporan ini dibuat sebagai bagian dari praktikum Workshop Administrasi Jaringan, Program Studi D4 Teknik Informatika, Politeknik Elektronika Negeri Surabaya (PENS), 2026.*
