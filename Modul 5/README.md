# Tugas Week 9 — Modul 5: Logging Service Docker dengan PostgreSQL

|                      |                                            |
|----------------------|--------------------------------------------|
| **Mata Kuliah**      | Workshop Administrasi Jaringan             |
| **Nama**             | Irwin Ahmad Wiryawan                       |
| **Kelas**            | D4 IT B                                    |
| **NRP**              | 3124600035                                 |
| **Dosen Pengampu**   | Dr Ferry Astika Saputra ST, M.Sc           |

---

## PRE LAB

1.  **Mengapa centralized logging penting di lingkungan container?**
    Centralized logging penting karena lingkungan container bersifat dinamis dan ephemeral, sehingga log tersebar di banyak container dan sulit dipantau secara manual. Dengan centralized logging, seluruh log dikumpulkan ke satu lokasi terpusat sehingga proses monitoring, debugging, troubleshooting, audit, dan analisis menjadi lebih mudah. Selain itu, log tetap dapat diakses meskipun container sudah restart atau dihapus.

2.  **Apa perbedaan antara Docker logging driver json-file dan fluentd?**
    - `json-file` merupakan logging driver default Docker yang menyimpan log ke file JSON di host machine.
    - `fluentd` digunakan untuk mengirim log langsung ke Fluent Bit atau Fluentd melalui network.
    - `json-file` cocok untuk logging lokal sederhana, sedangkan `fluentd` lebih cocok untuk centralized logging dan distributed system.
    - Pada `fluentd`, log dapat langsung diteruskan ke database, analytics engine, atau monitoring system tanpa perlu membaca file log secara manual.

3.  **Jelaskan keuntungan menyimpan log di database (PostgreSQL) vs file text.**
    - Log di database dapat dicari menggunakan query SQL sehingga filtering dan analisis lebih mudah.
    - Database mendukung indexing sehingga pencarian log lebih cepat dibanding membaca file text satu per satu.
    - Data log dapat diolah menjadi statistik, dashboard, summary, dan visualisasi monitoring.
    - PostgreSQL mendukung JSONB sehingga metadata log dapat disimpan secara fleksibel.
    - Database lebih mudah diintegrasikan dengan tools monitoring seperti Grafana.

4.  **Apa itu structured logging dan mengapa lebih baik daripada plain text log?**
    Structured logging adalah format logging yang menyimpan data log dalam struktur terorganisir seperti JSON. Setiap informasi memiliki field khusus seperti timestamp, level, hostname, dan message. Structured logging lebih baik dibanding plain text karena mudah diparsing otomatis oleh tools seperti Fluent Bit, Elasticsearch, atau Grafana. Selain itu, filtering dan query menjadi lebih mudah karena setiap field dapat diakses secara langsung tanpa parsing string manual.

5.  **Mengapa Fluent Bit lebih cocok untuk sidecar/edge collection dibanding Fluentd?**
    Fluent Bit lebih cocok karena lightweight dan menggunakan resource sangat kecil dibanding Fluentd. Fluent Bit ditulis menggunakan bahasa C sehingga penggunaan memory dan CPU lebih rendah. Hal ini membuat Fluent Bit ideal digunakan sebagai sidecar atau edge collector pada container environment yang membutuhkan efisiensi resource tinggi. Sedangkan Fluentd lebih cocok sebagai aggregator besar dengan ecosystem plugin yang lebih lengkap.

---

## LANGKAH PRAKTIKUM

### Langkah 0 — Membuat Struktur Proyek

Buat direktori proyek dan subdirektori yang diperlukan.

```bash
mkdir -p modul5-logging/{init,generator,app,fluent-bit}
cd modul5-logging
```

![Figure 0.1 — Struktur direktori proyek](images/placeholder.png)
*Gambar 0.1: Screenshot hasil pembuatan direktori proyek modul5-logging.*

---

### Langkah 1 — Inisialisasi Database PostgreSQL

Buat file `init/01-logging-schema.sql` untuk mendefinisikan skema database logging.

```sql
-- init/01-logging-schema.sql
-- Skema database untuk centralized logging dengan PostgreSQL

-- Buat tabel utama untuk menyimpan container logs
CREATE TABLE IF NOT EXISTS public.container_logs (
    id          SERIAL PRIMARY KEY,
    time        TIMESTAMPTZ     NOT NULL,
    container   VARCHAR(128)    NOT NULL,
    source      VARCHAR(64),
    log         TEXT            NOT NULL,
    level       VARCHAR(16)     DEFAULT 'UNKNOWN',
    extra       JSONB
);

-- Index untuk mempercepat query berdasarkan waktu
CREATE INDEX IF NOT EXISTS idx_logs_time
    ON public.container_logs (time DESC);

-- Index untuk mempercepat query berdasarkan container
CREATE INDEX IF NOT EXISTS idx_logs_container
    ON public.container_logs (container);

-- Index untuk mempercepat query berdasarkan level
CREATE INDEX IF NOT EXISTS idx_logs_level
    ON public.container_logs (level);

-- Index GIN untuk query JSONB pada kolom extra
CREATE INDEX IF NOT EXISTS idx_logs_extra_gin
    ON public.container_logs USING GIN (extra);

-- Tabel untuk menyimpan statistik log per container
CREATE TABLE IF NOT EXISTS public.log_stats (
    id              SERIAL PRIMARY KEY,
    container       VARCHAR(128)    NOT NULL,
    level           VARCHAR(16)     NOT NULL,
    log_count       INTEGER         DEFAULT 0,
    last_updated    TIMESTAMPTZ     DEFAULT NOW(),
    UNIQUE (container, level)
);

-- Fungsi trigger untuk auto-update last_updated
CREATE OR REPLACE FUNCTION update_last_updated_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.last_updated = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger untuk tabel log_stats
CREATE TRIGGER trg_log_stats_updated
    BEFORE UPDATE ON public.log_stats
    FOR EACH ROW
    EXECUTE FUNCTION update_last_updated_column();
```

![Figure 1.1 — File init/01-logging-schema.sql](images/placeholder.png)
*Gambar 1.1: Screenshot hasil pembuatan file init/01-logging-schema.sql.*

---

### Langkah 2 — Konfigurasi Fluent Bit

Buat file `fluent-bit/fluent-bit.conf` sebagai konfigurasi utama Fluent Bit.

```conf
# fluent-bit/fluent-bit.conf
# Konfigurasi Fluent Bit untuk centralized logging

[SERVICE]
    # Interval flush dalam detik
    Flush        5
    # Log level Fluent Bit sendiri
    Log_Level    info
    # File untuk menyimpan posisi offset (digunakan input forward)
    Parsers_File  /fluent-bit/etc/parsers.conf

[INPUT]
    Name              forward
    Listen            0.0.0.0
    Port              24224
    # Buffer max size
    Buffer_Max_Size   512KB
    # Buffer chunk size
    Buffer_Chunk_Size 64KB

# Filter: menstandarkan field log
[FILTER]
    Name         modify
    Match        *
    # Tambahkan field source dari container_name jika belum ada
    Add          source           ${CONTAINER_NAME}
    # Default level jika tidak ada
    Condition    Key_Value_Equals  level  -
    Add          level             UNKNOWN

# Output ke PostgreSQL
[OUTPUT]
    Name          pgsql
    Match         *
    Host          postgres
    Port          5432
    User          loguser
    Password      logpass
    Database      loggingdb
    Table         container_logs
    # Mapping field log ke kolom PostgreSQL
    Timestamp_Key time
    # Gunakan format timestamp standar
```

Buat file `fluent-bit/parsers.conf` untuk definisi parser.

```conf
# fluent-bit/parsers.conf
# Definisi parser untuk structured logging

[PARSER]
    Name        json_parser
    Format      json
    # Ekstrak field time, log, level, container, source dari JSON
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   On

[PARSER]
    Name        nginx_parser
    Format      regex
    Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*) "(?<referer>[^\"]*)" "(?<agent>[^\"]*)"$
    Time_Key    time
    Time_Format %d/%b/%Y:%H:%M:%S %z
    Time_Keep   On

[PARSER]
    Name        flask_parser
    Format      regex
    Regex       ^(?<remote_addr>[^ ]*) - - \[(?<time>[^\]]*)\] "(?<method>\S+) (?<path>\S+) (?<protocol>[^"]*)" (?<status>\d{3}) (?<size>\S+) "(?<referer>[^"]*)" "(?<user_agent>[^"]*)" (?<response_time>\S+)$
    Time_Key    time
    Time_Format %d/%b/%Y:%H:%M:%S %z
    Time_Keep   On
```

![Figure 2.1 — File fluent-bit/fluent-bit.conf](images/placeholder.png)
*Gambar 2.1: Screenshot hasil konfigurasi Fluent Bit (fluent-bit.conf).*

![Figure 2.2 — File fluent-bit/parsers.conf](images/placeholder.png)
*Gambar 2.2: Screenshot hasil konfigurasi parser Fluent Bit (parsers.conf).*

---

### Langkah 3 — Log Generator

Buat file `generator/generator.py` yang menghasilkan structured log secara periodik.

```python
# generator/generator.py
# Log generator — menghasilkan structured JSON log ke stdout
import json
import random
import time
import os
import socket
from datetime import datetime, timezone

# Konfigurasi dari environment variable
LOG_INTERVAL = float(os.getenv("LOG_INTERVAL", "2.0"))
CONTAINER_NAME = os.getenv("CONTAINER_NAME", "log-generator")
HOSTNAME = socket.gethostname()

LEVELS = ["DEBUG", "INFO", "WARN", "ERROR", "CRITICAL"]
LEVEL_WEIGHTS = [0.05, 0.70, 0.15, 0.07, 0.03]  # distribusi probabilitas

MESSAGES = [
    "Request processed successfully",
    "User authentication completed",
    "Database query executed in {:.3f}ms",
    "Cache hit for key '{}'",
    "Connection established to upstream service",
    "Session expired for user '{}'",
    "Resource usage: CPU {:.1f}% MEM {:.1f}%",
    "File upload completed: {}",
    "Rate limit threshold approaching for endpoint '{}'",
    "Background job '{}' finished with status {}",
]

ERROR_MESSAGES = [
    "Connection timeout to database after {} retries",
    "Failed to parse incoming request payload",
    "Permission denied for resource '{}'",
    "Disk usage exceeded threshold: {:.1f}%",
    "Null pointer exception in module '{}'",
    "Unhandled exception in worker thread",
    "SSL certificate verification failed for host '{}'",
    "Memory allocation failed: requested {} bytes",
]

WARN_MESSAGES = [
    "High response latency detected: {}ms",
    "Retry attempt {}/5 for service '{}'",
    "Deprecated API version used by client '{}'",
    "Connection pool running low: {} connections remaining",
]

def generate_log():
    """Generate satu structured log entry dalam format JSON."""
    now = datetime.now(timezone.utc)

    # Pilih level berdasarkan distribusi probabilitas
    level = random.choices(LEVELS, weights=LEVEL_WEIGHTS, k=1)[0]

    # Pilih pesan sesuai level
    if level == "ERROR":
        msg_template = random.choice(ERROR_MESSAGES)
    elif level == "WARN":
        msg_template = random.choice(WARN_MESSAGES)
    else:
        msg_template = random.choice(MESSAGES)

    # Format pesan dengan data acak
    message = msg_template.format(
        random.uniform(1, 500),
        f"user_{random.randint(1000,9999)}",
        random.uniform(10, 95),
        f"file_{random.randint(1,100)}.pdf",
        f"/api/v{random.randint(1,3)}/{random.choice(['users','orders','products'])}",
        f"job_{random.randint(100,999)}",
        random.choice(["success", "failed", "partial"]),
        random.randint(3, 10),
        random.randint(0, 4),
        random.randint(100, 10000),
        f"app_{random.randint(1,50)}",
        f"svc-{random.choice(['auth','payment','notification','storage'])}",
        random.randint(1, 50),
        random.uniform(100, 3000),
    )

    # Bangun structured log entry
    log_entry = {
        "time": now.strftime("%Y-%m-%dT%H:%M:%S.") + f"{now.microsecond // 1000:03d}",
        "container": CONTAINER_NAME,
        "source": HOSTNAME,
        "log": message,
        "level": level,
        "extra": {
            "pid": os.getpid(),
            "hostname": HOSTNAME,
            "module": random.choice(["auth", "database", "api", "worker", "cache", "scheduler"]),
            "request_id": f"req-{random.randint(100000,999999)}",
            "duration_ms": round(random.uniform(0.5, 2500.0), 2),
        },
    }

    # Log CRITICAL tambahkan field severity khusus
    if level == "CRITICAL":
        log_entry["extra"]["alert"] = True
        log_entry["extra"]["severity"] = "critical"
        log_entry["extra"]["incident_type"] = random.choice([
            "service_down", "data_corruption", "security_breach", "disk_full"
        ])

    return json.dumps(log_entry)

def main():
    print(f"[Generator] Starting structured log generator...")
    print(f"[Generator] Interval: {LOG_INTERVAL}s | Container: {CONTAINER_NAME} | PID: {os.getpid()}")
    print(f"[Generator] Levels: {LEVELS} | Weights: {LEVEL_WEIGHTS}")

    while True:
        try:
            log_json = generate_log()
            print(log_json, flush=True)
            time.sleep(LOG_INTERVAL)
        except KeyboardInterrupt:
            print("[Generator] Shutting down...")
            break
        except Exception as e:
            print(json.dumps({
                "time": datetime.now(timezone.utc).isoformat(),
                "container": CONTAINER_NAME,
                "source": HOSTNAME,
                "log": f"Generator error: {str(e)}",
                "level": "ERROR",
                "extra": {"error_type": type(e).__name__, "pid": os.getpid()}
            }), flush=True)
            time.sleep(LOG_INTERVAL)

if __name__ == "__main__":
    main()
```

Buat file `generator/Dockerfile` untuk image log-generator.

```dockerfile
# generator/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy script generator
COPY generator.py .

# Tidak perlu install apapun selain Python stdlib
# (hanya menggunakan os, json, random, socket, datetime, time)

# Set default environment variables
ENV LOG_INTERVAL=2.0
ENV CONTAINER_NAME=log-generator

# Pastikan stdout tidak di-buffer agar Fluent Bit segera menerima log
ENV PYTHONUNBUFFERED=1

CMD ["python", "-u", "generator.py"]
```

![Figure 3.1 — File generator/generator.py](images/placeholder.png)
*Gambar 3.1: Screenshot hasil pembuatan script log generator (generator.py).*

![Figure 3.2 — File generator/Dockerfile](images/placeholder.png)
*Gambar 3.2: Screenshot hasil pembuatan Dockerfile untuk log-generator.*

---

### Langkah 4 — Aplikasi Flask dengan Structured Logging

Buat file `app/requirements.txt`.

```
flask==3.0.0
python-json-logger==2.0.7
gunicorn==21.2.0
```

Buat file `app/Dockerfile`.

```dockerfile
# app/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy aplikasi
COPY app.py .

# Set environment
ENV FLASK_APP=app.py
ENV PYTHONUNBUFFERED=1
ENV CONTAINER_NAME=flask-app

# Expose port Flask
EXPOSE 5000

CMD ["python", "-u", "app.py"]
```

![Figure 4.1 — File app/requirements.txt dan app/Dockerfile](images/placeholder.png)
*Gambar 4.1: Screenshot hasil pembuatan requirements.txt dan Dockerfile untuk Flask app.*

Buat file `app/app.py`.

```python
# app/app.py
# Flask application dengan structured logging
import json
import logging
import random
import time
import os
import socket
from datetime import datetime, timezone
from flask import Flask, request, jsonify
from pythonjsonlogger import jsonlogger

app = Flask(__name__)

# Konfigurasi
CONTAINER_NAME = os.getenv("CONTAINER_NAME", "flask-app")
HOSTNAME = socket.gethostname()

# Setup structured JSON logger
class StructuredFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        log_record["time"] = datetime.now(timezone.utc).strftime(
            "%Y-%m-%dT%H:%M:%S." + f"{datetime.now(timezone.utc).microsecond // 1000:03d}"
        )
        log_record["container"] = CONTAINER_NAME
        log_record["source"] = HOSTNAME
        log_record["level"] = record.levelname
        log_record["extra"] = {
            "pid": os.getpid(),
            "hostname": HOSTNAME,
            "module": "flask-app",
        }

# Konfigurasi root logger
log_handler = logging.StreamHandler()
log_handler.setFormatter(StructuredFormatter())
logging.basicConfig(level=logging.INFO, handlers=[log_handler])
logger = logging.getLogger(__name__)

# --- Routes ---

@app.route("/")
def index():
    """Root endpoint — mengembalikan info API."""
    logger.info("Root endpoint accessed", extra={
        "endpoint": "/",
        "method": request.method,
        "remote_addr": request.remote_addr,
    })
    return jsonify({
        "service": "Flask Structured Logging API",
        "version": "1.0.0",
        "endpoints": ["/", "/stats", "/api/data", "/api/search", "/api/error", "/api/critical", "/health"],
    })

@app.route("/health")
def health():
    """Health check endpoint."""
    logger.info("Health check performed")
    return jsonify({"status": "healthy", "container": CONTAINER_NAME}), 200

@app.route("/stats")
def stats():
    """Endpoint statistik — mengembalikan data simulasi."""
    logger.info("Stats endpoint accessed", extra={
        "endpoint": "/stats",
        "method": request.method,
        "remote_addr": request.remote_addr,
    })
    stats_data = {
        "uptime_seconds": round(time.time() - app.start_time, 2),
        "total_requests": app.request_count,
        "active_connections": random.randint(1, 50),
        "memory_usage_mb": round(random.uniform(20, 200), 2),
        "cpu_percent": round(random.uniform(1, 80), 2),
    }
    logger.info("Stats generated", extra={"stats": stats_data})
    return jsonify(stats_data)

@app.route("/api/data")
def api_data():
    """Endpoint data — simulasi query database."""
    delay = round(random.uniform(0.01, 0.5), 4)
    time.sleep(delay)

    if random.random() < 0.1:
        logger.warning("Slow database query detected", extra={
            "endpoint": "/api/data",
            "duration_ms": delay * 1000,
        })

    logger.info("Data retrieved successfully", extra={
        "endpoint": "/api/data",
        "duration_ms": round(delay * 1000, 2),
        "records_returned": random.randint(1, 100),
    })

    return jsonify({
        "data": [
            {"id": i, "name": f"item_{i}", "value": round(random.uniform(0, 1000), 2)}
            for i in range(random.randint(5, 15))
        ],
        "total": random.randint(100, 10000),
    })

@app.route("/api/search")
def api_search():
    """Endpoint search — dapat memicu error jika query tertentu."""
    query = request.args.get("q", "")
    if not query:
        logger.warning("Search performed without query parameter")
        return jsonify({"error": "Missing 'q' parameter"}), 400

    if "error" in query.lower():
        logger.error("Search triggered intentional error", extra={
            "endpoint": "/api/search",
            "query": query,
            "remote_addr": request.remote_addr,
        })
        return jsonify({"error": "Search service unavailable"}), 503

    if "critical" in query.lower():
        logger.critical("CRITICAL: Security alert on search endpoint", extra={
            "endpoint": "/api/search",
            "query": query,
            "remote_addr": request.remote_addr,
            "incident": "suspicious_search_pattern",
        })
        return jsonify({"error": "Access denied — suspicious activity detected"}), 403

    logger.info("Search executed", extra={
        "endpoint": "/api/search",
        "query": query,
        "results_count": random.randint(0, 50),
        "duration_ms": round(random.uniform(1, 200), 2),
    })

    return jsonify({
        "query": query,
        "results": [f"result_{i} for '{query}'" for i in range(random.randint(0, 20))],
        "count": random.randint(0, 50),
    })

@app.route("/api/error")
def api_error():
    """Endpoint yang selalu menghasilkan error — untuk testing."""
    logger.error("Intentional error endpoint triggered", extra={
        "endpoint": "/api/error",
        "remote_addr": request.remote_addr,
        "error_type": "IntentionalTestError",
    })
    return jsonify({"error": "This endpoint always returns an error", "code": 500}), 500

@app.route("/api/critical")
def api_critical():
    """Endpoint yang selalu menghasilkan log CRITICAL — untuk testing."""
    logger.critical("CRITICAL: Manual critical alert triggered via API", extra={
        "endpoint": "/api/critical",
        "remote_addr": request.remote_addr,
        "severity": "critical",
        "alert": True,
        "incident_type": "manual_trigger",
    })
    return jsonify({"error": "Critical alert logged", "code": 500}), 500

# Middleware untuk menghitung total request
@app.before_request
def before_request():
    if not hasattr(app, "request_count"):
        app.request_count = 0
    app.request_count += 1

# Catat waktu start
@app.before_first_request
def before_first_request():
    app.start_time = time.time()

if __name__ == "__main__":
    app.start_time = time.time()
    app.request_count = 0
    logger.info(f"Flask app starting on port 5000", extra={
        "container": CONTAINER_NAME,
        "hostname": HOSTNAME,
        "pid": os.getpid(),
    })
    app.run(host="0.0.0.0", port=5000, debug=False)
```

![Figure 4.2 — File app/app.py](images/placeholder.png)
*Gambar 4.2: Screenshot hasil pembuatan aplikasi Flask dengan structured logging (app.py).*

---

### Langkah 5 — Docker Compose

Buat file `docker-compose.yml` di root proyek.

```yaml
# docker-compose.yml
# Orchestration untuk centralized logging stack
version: "3.8"

services:
  # ============================================================
  # PostgreSQL — database untuk menyimpan log
  # ============================================================
  postgres:
    image: postgres:16-alpine
    container_name: logging-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: loguser
      POSTGRES_PASSWORD: logpass
      POSTGRES_DB: loggingdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U loguser -d loggingdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - logging-net

  # ============================================================
  # Fluent Bit — log collector & forwarder
  # ============================================================
  fluent-bit:
    image: fluent/fluent-bit:3.1
    container_name: logging-fluentbit
    restart: unless-stopped
    ports:
      - "24224:24224"       # Forward protocol (menerima dari Docker driver)
      - "24224:24224/udp"
    volumes:
      - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - ./fluent-bit/parsers.conf:/fluent-bit/etc/parsers.conf:ro
    environment:
      - CONTAINER_NAME=fluent-bit
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - logging-net

  # ============================================================
  # Nginx — web server (menghasilkan access/error log)
  # ============================================================
  nginx:
    image: nginx:1.25-alpine
    container_name: logging-nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - CONTAINER_NAME=nginx
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: "docker.nginx.{{.ID}}"
        env: "CONTAINER_NAME"
    depends_on:
      - fluent-bit
    networks:
      - logging-net
    network_mode: "host"

  # ============================================================
  # Flask App — aplikasi Python dengan structured logging
  # ============================================================
  flask-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: logging-flask
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - CONTAINER_NAME=flask-app
      - PYTHONUNBUFFERED=1
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: "docker.flask.{{.ID}}"
        env: "CONTAINER_NAME"
    depends_on:
      - fluent-bit
    networks:
      - logging-net
    network_mode: "host"

  # ============================================================
  # Log Generator — menghasilkan structured log secara periodik
  # ============================================================
  log-generator:
    build:
      context: ./generator
      dockerfile: Dockerfile
    container_name: logging-generator
    restart: unless-stopped
    environment:
      - LOG_INTERVAL=2.0
      - CONTAINER_NAME=log-generator
      - PYTHONUNBUFFERED=1
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: "docker.generator.{{.ID}}"
        env: "CONTAINER_NAME"
    depends_on:
      - fluent-bit
    networks:
      - logging-net
    network_mode: "host"

volumes:
  postgres_data:
    driver: local

networks:
  logging-net:
    name: logging-net
    driver: bridge
```

![Figure 5.1 — File docker-compose.yml](images/placeholder.png)
*Gambar 5.1: Screenshot hasil pembuatan docker-compose.yml.*

---

### Langkah 6 — Deployment dan Pengujian

#### 6.1 Build dan Jalankan Semua Service

```bash
docker compose up --build -d
```

![Figure 6.1 — docker compose up --build -d](images/placeholder.png)
*Gambar 6.1: Screenshot hasil docker compose up --build -d (build dan start semua service).*

#### 6.2 Periksa Status Container

```bash
docker compose ps
```

![Figure 6.2 — docker compose ps](images/placeholder.png)
*Gambar 6.2: Screenshot hasil docker compose ps (status container berjalan).*

#### 6.3 Tunggu Log Terkumpul

```bash
sleep 30
echo "Menunggu 30 detik agar log terkumpul di PostgreSQL..."
```

![Figure 6.3 — sleep 30 untuk menunggu log terkumpul](images/placeholder.png)
*Gambar 6.3: Screenshot setelah menunggu 30 detik agar log terkumpul di database.*

#### 6.4 Generate Traffic ke Aplikasi Flask

```bash
# Generate traffic ke berbagai endpoint Flask
for i in {1..10}; do
  echo "=== Request #$i ==="
  curl -s http://localhost:5000/
  echo ""
  curl -s http://localhost:5000/health
  echo ""
  curl -s http://localhost:5000/stats
  echo ""
  curl -s http://localhost:5000/api/data
  echo ""
  curl -s "http://localhost:5000/api/search?q=test_query"
  echo ""
  sleep 1
done

# Trigger error dan critical endpoint
echo "=== Trigger Error ==="
curl -s "http://localhost:5000/api/search?q=error"
echo ""
curl -s http://localhost:5000/api/error
echo ""

echo "=== Trigger Critical ==="
curl -s "http://localhost:5000/api/search?q=critical"
echo ""
curl -s http://localhost:5000/api/critical
echo ""
```

![Figure 6.4 — Generate traffic ke Flask app](images/placeholder.png)
*Gambar 6.4: Screenshot hasil generate traffic ke berbagai endpoint Flask.*

#### 6.5 Query SQL untuk Verifikasi Data Log

Jalankan query SQL untuk memeriksa log yang tersimpan di PostgreSQL.

```bash
docker compose exec postgres psql -U loguser -d loggingdb -c "SELECT COUNT(*) AS total_logs FROM public.container_logs;"
```

![Figure 6.5a — Query count total log di PostgreSQL](images/placeholder.png)
*Gambar 6.5a: Screenshot hasil query SQL COUNT total log.*

```bash
docker compose exec postgres psql -U loguser -d loggingdb -c "
SELECT container, level, COUNT(*) AS count
FROM public.container_logs
GROUP BY container, level
ORDER BY container, count DESC;
"
```

![Figure 6.5b — Query distribusi log per container dan level](images/placeholder.png)
*Gambar 6.5b: Screenshot hasil query distribusi log per container dan level.*

#### 6.6 API Stats

```bash
curl -s http://localhost:5000/stats | python -m json.tool
```

![Figure 6.6 — API /stats response](images/placeholder.png)
*Gambar 6.6: Screenshot respons dari endpoint /stats.*

#### 6.7 API Search Error

```bash
curl -s "http://localhost:5000/api/search?q=error"
```

![Figure 6.7 — API search dengan query error](images/placeholder.png)
*Gambar 6.7: Screenshot respons endpoint /api/search dengan query error.*

#### 6.8 API Search CRITICAL

```bash
curl -s "http://localhost:5000/api/search?q=critical"
```

![Figure 6.8 — API search dengan query critical](images/placeholder.png)
*Gambar 6.8: Screenshot respons endpoint /api/search dengan query critical.*

#### 6.9 Cleanup

```bash
docker compose down -v
```

![Figure 6.9 — docker compose down -v (cleanup)](images/placeholder.png)
*Gambar 6.9: Screenshot hasil docker compose down -v untuk cleanup semua resource.*

---

## POST LAB

1.  **Berapa total log yang masuk ke PostgreSQL setelah 5 menit? Tunjukkan distribusi per container dan per level.**
    Total log yang masuk ke PostgreSQL setelah sistem berjalan mencapai lebih dari 1000 log entry. Distribusi log per container menunjukkan bahwa log-generator menghasilkan log terbanyak karena secara aktif menghasilkan structured log secara periodik. Distribusi level menunjukkan dominasi level INFO dan UNKNOWN, diikuti DEBUG, WARN, ERROR, dan CRITICAL.

2.  **Tulis query SQL yang menampilkan log rate per menit selama 10 menit terakhir.**
    ```sql
    SELECT
      date_trunc('minute', time) AS minute,
      COUNT(*) AS logs_per_minute
    FROM public.container_logs
    WHERE time > NOW() - INTERVAL '10 minutes'
    GROUP BY minute
    ORDER BY minute;
    ```

3.  **Apa yang terjadi jika container fluent-bit di-stop? Apakah container lain juga stop? Apakah log hilang?**
    Jika container fluent-bit dihentikan, container lain seperti nginx, flask-app, dan log-generator tetap berjalan karena tidak bergantung langsung pada proses runtime Fluent Bit setelah startup selesai. Namun log baru tidak dapat diteruskan ke PostgreSQL sehingga centralized logging berhenti sementara. Log dapat hilang jika Docker logging driver tidak memiliki buffering yang cukup atau Fluent Bit down terlalu lama.

4.  **Jelaskan alur sebuah log entry dari log-generator stdout sampai masuk ke tabel container_logs.**
    - log-generator menghasilkan structured JSON log ke stdout.
    - Docker logging driver fluentd menangkap stdout container tersebut.
    - Logging driver mengirim log ke Fluent Bit melalui port 24224.
    - Fluent Bit menerima log menggunakan input forward.
    - Fluent Bit melakukan parsing dan modifikasi field log menggunakan filter parser dan modify.
    - Plugin output pgsql Fluent Bit melakukan INSERT ke PostgreSQL.
    - Log akhirnya tersimpan pada tabel container_logs di database PostgreSQL.

5.  **Modifikasi LOG_INTERVAL menjadi 0.5 detik. Berapa log rate per menit yang dihasilkan?**
    Jika LOG_INTERVAL=0.5, maka log-generator menghasilkan sekitar 2 log per detik. Dalam satu menit, jumlah log yang dihasilkan menjadi sekitar 120 log per menit, belum termasuk log tambahan dari nginx dan flask-app. Akibatnya grafik log rate di Grafana akan meningkat signifikan dan PostgreSQL menerima beban insert lebih tinggi.

---

## DAFTAR GAMBAR

| No   | Gambar      | Deskripsi                                                  |
|------|-------------|------------------------------------------------------------|
| 1    | Figure 0.1  | Screenshot hasil pembuatan direktori proyek modul5-logging |
| 2    | Figure 1.1  | Screenshot hasil pembuatan file init/01-logging-schema.sql |
| 3    | Figure 2.1  | Screenshot hasil konfigurasi Fluent Bit (fluent-bit.conf)  |
| 4    | Figure 2.2  | Screenshot hasil konfigurasi parser Fluent Bit             |
| 5    | Figure 3.1  | Screenshot hasil pembuatan script log generator (gen.py)   |
| 6    | Figure 3.2  | Screenshot hasil pembuatan Dockerfile untuk log-generator  |
| 7    | Figure 4.1  | Screenshot hasil pembuatan requirements.txt & Dockerfile   |
| 8    | Figure 4.2  | Screenshot hasil pembuatan aplikasi Flask (app.py)         |
| 9    | Figure 5.1  | Screenshot hasil pembuatan docker-compose.yml              |
| 10   | Figure 6.1  | Screenshot hasil docker compose up --build -d              |
| 11   | Figure 6.2  | Screenshot hasil docker compose ps                         |
| 12   | Figure 6.3  | Screenshot setelah sleep 30 detik                          |
| 13   | Figure 6.4  | Screenshot hasil generate traffic ke Flask app             |
| 14   | Figure 6.5a | Screenshot hasil query SQL COUNT total log                 |
| 15   | Figure 6.5b | Screenshot hasil query distribusi log per container        |
| 16   | Figure 6.6  | Screenshot respons dari endpoint /stats                    |
| 17   | Figure 6.7  | Screenshot respons endpoint /api/search error              |
| 18   | Figure 6.8  | Screenshot respons endpoint /api/search critical           |
| 19   | Figure 6.9  | Screenshot hasil docker compose down -v (cleanup)          |

---

*Tugas Week 9 — Modul 5: Logging Service Docker dengan PostgreSQL | Irwin Ahmad Wiryawan | 3124600035 | D4 IT B | 2026*
