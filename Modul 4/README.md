# Tugas Week 9 — Modul 4: Database PostgreSQL

| | |
|---|---|
| **Mata Kuliah** | Workshop Administrasi Jaringan |
| **Nama** | Irwin Ahmad Wiryawan |
| **Kelas** | D4 IT B |
| **NRP** | 3124600035 |
| **Dosen Pengampu** | Dr Ferry Astika Saputra ST, M.Sc |

---

## PRE LAB

### 1. Apa fungsi file/folder `/docker-entrypoint-initdb.d/` di image PostgreSQL?

Folder `/docker-entrypoint-initdb.d/` digunakan untuk menyimpan file inisialisasi seperti `.sql` atau `.sh` yang akan dijalankan otomatis saat PostgreSQL container pertama kali dibuat dengan volume kosong. Folder ini biasanya digunakan untuk membuat schema, tabel, user, insert sample data, atau konfigurasi awal database secara otomatis.

### 2. Mengapa `POSTGRES_PASSWORD` wajib diset? Apa risikonya jika tidak ada password?

`POSTGRES_PASSWORD` wajib diset untuk mengamankan akses superuser PostgreSQL. Tanpa password, database dapat diakses tanpa autentikasi yang aman sehingga meningkatkan risiko unauthorized access, data leakage, dan pengambilalihan database oleh pihak tidak sah. PostgreSQL image resmi juga secara default menolak startup tanpa password untuk mencegah insecure deployment.

### 3. Jelaskan perbedaan antara `pg_dump` format custom (`-Fc`) dan format SQL plain text.

- Format custom (`-Fc`) menghasilkan file backup binary terkompresi yang lebih kecil dan mendukung selective restore menggunakan `pg_restore`.
- Format SQL plain text menghasilkan file `.sql` berisi statement SQL yang dapat dibaca manusia dan dijalankan langsung menggunakan `psql`.
- Format custom lebih fleksibel dan efisien untuk backup besar, sedangkan plain SQL lebih mudah untuk inspeksi manual atau migrasi sederhana.

### 4. Apa itu `shared_buffers` dan mengapa perlu disesuaikan untuk container?

`shared_buffers` adalah memory cache internal PostgreSQL yang digunakan untuk menyimpan halaman data yang sering diakses agar query lebih cepat. Pada container, nilai `shared_buffers` perlu disesuaikan dengan resource host karena container biasanya memiliki RAM terbatas. Jika terlalu besar, container dapat mengalami memory pressure atau OOM kill, sedangkan jika terlalu kecil performa query dapat menurun.

### 5. Mengapa data PostgreSQL harus disimpan di Docker Volume, bukan di container layer?

Karena filesystem container bersifat ephemeral sehingga data akan hilang ketika container dihapus atau dibuat ulang. Docker Volume menyediakan persistent storage yang tetap ada meskipun container restart, recreate, atau update image. Volume juga lebih aman dan optimal untuk workload database dibanding writable layer container.

---

## LANGKAH PRAKTIKUM

### Langkah 0: Persiapan Project

```bash
# Buat direktori project
mkdir -p ~/docker-lab/postgresql/{init,config,backup}

# Masuk ke direktori project
cd ~/docker-lab/postgresql
```

![Figure 0.1 — Membuat direktori project PostgreSQL](images/placeholder.png)
*Gambar 0.1: Screenshot hasil `mkdir -p ~/docker-lab/postgresql/{init,config,backup}` dan `cd ~/docker-lab/postgresql`.*

---

### Langkah 1: Deploy PostgreSQL dengan Docker Compose

#### 1.1 Buat init script untuk inisialisasi database

```bash
cat > init/01-create-schema.sql << 'EOF'
-- ============================================================
-- PostgreSQL Init Script: Database Akademik
-- Module 4 — Workshop Administrasi Jaringan
-- ============================================================

-- 1. Buat user aplikasi (non-superuser)
CREATE USER labuser WITH PASSWORD 'labpass123';
ALTER USER labuser CREATEDB;

-- 2. Buat user read-only untuk monitoring / reporting
CREATE USER app_reader WITH PASSWORD 'readerpass123';

-- 3. Buat database akademik
CREATE DATABASE akademik OWNER labuser;

-- 4. Connect ke database akademik
\c akademik

-- 5. Buat schema aplikasi
CREATE SCHEMA IF NOT EXISTS app AUTHORIZATION labuser;

-- 6. Set search_path default untuk labuser
ALTER USER labuser SET search_path TO app, public;

-- 7. Buat tabel mahasiswa
CREATE TABLE app.mahasiswa (
    id          SERIAL PRIMARY KEY,
    nim         VARCHAR(15)  NOT NULL UNIQUE,
    nama        VARCHAR(100) NOT NULL,
    jurusan     VARCHAR(50)  NOT NULL,
    angkatan    INT          NOT NULL,
    email       VARCHAR(100),
    created_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP
);

-- 8. Buat tabel mata_kuliah
CREATE TABLE app.mata_kuliah (
    id       SERIAL PRIMARY KEY,
    kode     VARCHAR(10)  NOT NULL UNIQUE,
    nama     VARCHAR(100) NOT NULL,
    sks      INT          NOT NULL CHECK (sks > 0),
    semester INT          NOT NULL
);

-- 9. Buat tabel nilai (relasi many-to-many)
CREATE TABLE app.nilai (
    id              SERIAL PRIMARY KEY,
    mahasiswa_id    INT NOT NULL REFERENCES app.mahasiswa(id) ON DELETE CASCADE,
    mata_kuliah_id  INT NOT NULL REFERENCES app.mata_kuliah(id) ON DELETE CASCADE,
    nilai           CHAR(2),
    grade           VARCHAR(2),
    semester_ambil  INT,
    UNIQUE (mahasiswa_id, mata_kuliah_id, semester_ambil)
);

-- 10. Insert sample data mahasiswa
INSERT INTO app.mahasiswa (nim, nama, jurusan, angkatan, email) VALUES
    ('3124600035', 'Irwin Ahmad Wiryawan', 'Teknik Informatika', 2024, 'irwin@pens.ac.id'),
    ('3124600036', 'Budi Santoso',       'Teknik Informatika', 2024, 'budi@pens.ac.id'),
    ('3124600037', 'Siti Nurhaliza',     'Sistem Informasi',   2024, 'siti@pens.ac.id'),
    ('3124600038', 'Ahmad Fauzi',        'Teknik Informatika', 2023, 'fauzi@pens.ac.id'),
    ('3124600039', 'Dewi Lestari',       'Sistem Informasi',   2023, 'dewi@pens.ac.id'),
    ('3124600040', 'Rudi Hartono',       'Teknik Komputer',    2024, 'rudi@pens.ac.id');

-- 11. Insert sample data mata_kuliah
INSERT INTO app.mata_kuliah (kode, nama, sks, semester) VALUES
    ('WAJ101', 'Workshop Administrasi Jaringan', 3, 4),
    ('BDD201', 'Basis Data Lanjut',              4, 4),
    ('PBO301', 'Pemrograman Berorientasi Objek', 3, 4),
    ('JAR401', 'Jaringan Komputer',              3, 3),
    ('SIS501', 'Sistem Operasi',                 2, 3);

-- 12. Insert sample data nilai
INSERT INTO app.nilai (mahasiswa_id, mata_kuliah_id, nilai, grade, semester_ambil) VALUES
    (1, 1, '85', 'A',  4),
    (1, 2, '78', 'B+', 4),
    (1, 3, '92', 'A',  4),
    (2, 1, '70', 'B',  4),
    (2, 2, '65', 'B-', 4),
    (3, 1, '88', 'A',  4),
    (3, 4, '75', 'B+', 3),
    (4, 4, '60', 'C+', 3),
    (4, 5, '80', 'A-', 3);

-- 13. Grant privileges
-- labuser sudah jadi owner schema app, jadi punya full access
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA app TO labuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA app TO labuser;

-- app_reader: read-only access
GRANT USAGE ON SCHEMA app TO app_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_reader;
ALTER DEFAULT PRIVILEGES FOR ROLE labuser IN SCHEMA app
    GRANT SELECT ON TABLES TO app_reader;

-- 14. Verifikasi data
SELECT 'Schema dan sample data berhasil dibuat!' AS status;
SELECT COUNT(*) AS jumlah_mahasiswa FROM app.mahasiswa;
SELECT COUNT(*) AS jumlah_mata_kuliah FROM app.mata_kuliah;
SELECT COUNT(*) AS jumlah_nilai FROM app.nilai;
EOF
```

![Figure 1.1 — Membuat init script 01-create-schema.sql](images/placeholder.png)
*Gambar 1.1: Screenshot hasil pembuatan file `init/01-create-schema.sql` dengan heredoc.*

#### 1.2 Buat custom PostgreSQL configuration

```bash
cat > config/custom-postgresql.conf << 'EOF'
# ============================================================
# Custom PostgreSQL Configuration
# Dioptimasi untuk container dengan resource terbatas (~512MB RAM)
# ============================================================

# --- Memory ---
shared_buffers = 128MB          # 25% dari RAM container (~512MB)
work_mem = 4MB                  # Per-operation sort memory
maintenance_work_mem = 32MB     # Untuk VACUUM, CREATE INDEX, dll
effective_cache_size = 384MB    # Estimasi cache OS untuk query planner

# --- Connection ---
max_connections = 50            # Disesuaikan untuk container
superuser_reserved_connections = 3

# --- WAL (Write-Ahead Log) ---
wal_buffers = 4MB
min_wal_size = 80MB
max_wal_size = 256MB
checkpoint_completion_target = 0.9

# --- Query Planner ---
random_page_cost = 1.1          # SSD-optimized
effective_io_concurrency = 200

# --- Logging ---
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_statement = 'ddl'           # Log semua DDL
log_min_duration_statement = 1000  # Log query > 1 detik
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p-%l] %q%u@%d '

# --- Autovacuum ---
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min

# --- Statistik ---
track_activities = on
track_counts = on
track_functions = pl
EOF
```

![Figure 1.2 — Membuat custom PostgreSQL config](images/placeholder.png)
*Gambar 1.2: Screenshot hasil pembuatan file `config/custom-postgresql.conf`.*

#### 1.3 Buat docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
# ============================================================
# Docker Compose: PostgreSQL + pgAdmin
# Module 4 — Workshop Administrasi Jaringan
# ============================================================

version: "3.9"

services:
  # ---- PostgreSQL Database Server ----
  db:
    image: postgres:16-alpine
    container_name: postgres-db
    restart: unless-stopped

    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-adminpass456}
      PGDATA: /var/lib/postgresql/data/pgdata

    ports:
      - "5432:5432"

    volumes:
      # Persistent data volume
      - pg-data:/var/lib/postgresql/data
      # Init scripts (dieksekusi hanya saat volume kosong / pertama kali)
      - ./init:/docker-entrypoint-initdb.d
      # Custom config
      - ./config/custom-postgresql.conf:/etc/postgresql/postgresql.conf

    # Terapkan custom config
    command:
      - "postgres"
      - "-c"
      - "config_file=/etc/postgresql/postgresql.conf"

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

    networks:
      - postgres-net

  # ---- pgAdmin Web Interface ----
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin-web
    restart: unless-stopped

    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL:-admin@pens.ac.id}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD:-pgadmin123}
      PGADMIN_CONFIG_SERVER_MODE: "False"
      PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"

    ports:
      - "8080:80"

    volumes:
      - pgadmin-data:/var/lib/pgadmin

    depends_on:
      db:
        condition: service_healthy

    networks:
      - postgres-net

volumes:
  pg-data:
    name: postgres-pg-data
  pgadmin-data:
    name: postgres-pgadmin-data

networks:
  postgres-net:
    name: postgres-network
    driver: bridge
EOF
```

![Figure 1.3 — Membuat docker-compose.yml](images/placeholder.png)
*Gambar 1.3: Screenshot hasil pembuatan file `docker-compose.yml`.*

#### 1.4 Deploy services

```bash
# Jalankan semua service di background
docker compose up -d
```

![Figure 1.4 — docker compose up -d](images/placeholder.png)
*Gambar 1.4: Screenshot hasil `docker compose up -d` — container PostgreSQL dan pgAdmin berhasil dijalankan.*

```bash
# Cek status container
docker compose ps
```

![Figure 1.5 — docker compose ps](images/placeholder.png)
*Gambar 1.5: Screenshot hasil `docker compose ps` menampilkan container postgres-db dan pgadmin-web dengan status Up.*

```bash
# Lihat log database (pastikan init script berhasil)
docker compose logs db
```

![Figure 1.6 — docker compose logs db](images/placeholder.png)
*Gambar 1.6: Screenshot hasil `docker compose logs db` menampilkan log startup PostgreSQL dan eksekusi init script.*

---

### Langkah 2: Koneksi dan Verifikasi Database

#### 2.1 Koneksi dengan `psql` client (host)

```bash
# Install PostgreSQL client (jika belum ada)
sudo apt update && sudo apt install -y postgresql-client
```

![Figure 2.1 — Instalasi postgresql-client](images/placeholder.png)
*Gambar 2.1: Screenshot hasil `sudo apt install -y postgresql-client` menunjukkan package berhasil diinstal.*

```bash
# Koneksi ke PostgreSQL sebagai superuser (postgres)
PGPASSWORD=adminpass456 psql -h localhost -U postgres -d akademik
```

![Figure 2.2 — Koneksi psql ke database akademik](images/placeholder.png)
*Gambar 2.2: Screenshot hasil koneksi `psql` ke database `akademik` sebagai user `postgres`.*

```sql
-- Lihat semua database di server
\l
```

![Figure 2.3 — psql \l list database](images/placeholder.png)
*Gambar 2.3: Screenshot hasil perintah `\l` menampilkan daftar database termasuk `akademik`.*

```sql
-- Lihat semua schema
\dn
```

![Figure 2.4 — psql \dn list schema](images/placeholder.png)
*Gambar 2.4: Screenshot hasil perintah `\dn` menampilkan schema `app` dan `public`.*

```sql
-- Lihat semua tabel di schema app
\dt app.*
```

![Figure 2.5 — psql \dt app.*](images/placeholder.png)
*Gambar 2.5: Screenshot hasil perintah `\dt app.*` menampilkan tabel `mahasiswa`, `mata_kuliah`, dan `nilai`.*

```sql
-- Lihat detail struktur tabel mahasiswa
\d+ app.mahasiswa
```

![Figure 2.6 — psql \d+ app.mahasiswa](images/placeholder.png)
*Gambar 2.6: Screenshot hasil `\d+ app.mahasiswa` menampilkan kolom, tipe data, constraint, dan index tabel mahasiswa.*

```sql
-- Query: tampilkan seluruh data mahasiswa
SELECT * FROM app.mahasiswa;

-- Query: tampilkan mahasiswa dengan nilai mata kuliah
SELECT
    m.nim,
    m.nama,
    mk.nama AS mata_kuliah,
    n.nilai,
    n.grade
FROM app.mahasiswa m
JOIN app.nilai n ON m.id = n.mahasiswa_id
JOIN app.mata_kuliah mk ON n.mata_kuliah_id = mk.id
ORDER BY m.nim, mk.nama;
```

![Figure 2.7 — Query SELECT data mahasiswa dan join](images/placeholder.png)
*Gambar 2.7: Screenshot hasil query `SELECT` data mahasiswa dan `JOIN` dengan tabel nilai dan mata kuliah.*

```sql
-- Keluar dari psql
\q
```

#### 2.2 Koneksi langsung via `docker exec`

```bash
# Koneksi ke PostgreSQL di dalam container (tanpa perlu install psql di host)
docker exec -it postgres-db psql -U labuser -d akademik
```

![Figure 2.8 — docker exec psql ke container](images/placeholder.png)
*Gambar 2.8: Screenshot hasil `docker exec -it postgres-db psql -U labuser -d akademik` — masuk ke psql di dalam container.*

```sql
-- Query gabungan: mahasiswa dengan total SKS yang diambil
SELECT
    m.nim,
    m.nama,
    m.jurusan,
    COUNT(n.id) AS jumlah_mk,
    COALESCE(SUM(mk.sks), 0) AS total_sks
FROM app.mahasiswa m
LEFT JOIN app.nilai n ON m.id = n.mahasiswa_id
LEFT JOIN app.mata_kuliah mk ON n.mata_kuliah_id = mk.id
GROUP BY m.id, m.nim, m.nama, m.jurusan
ORDER BY total_sks DESC;

\q
```

![Figure 2.9 — Query join mahasiswa dengan total SKS](images/placeholder.png)
*Gambar 2.9: Screenshot hasil query gabungan mahasiswa dengan jumlah mata kuliah dan total SKS.*

#### 2.3 Akses pgAdmin via browser

1. Buka browser ke http://localhost:8080
2. Login dengan:
   - **Email:** `admin@pens.ac.id`
   - **Password:** `pgadmin123`
3. Tambahkan server baru:
   - Klik kanan **Servers** → **Register** → **Server...**
   - Tab **General**:
     - Name: `PostgreSQL-Lab`
   - Tab **Connection**:
     - Host name/address: `db` (nama service di docker-compose)
     - Port: `5432`
     - Maintenance database: `akademik`
     - Username: `labuser`
     - Password: `labpass123`
4. Klik **Save**
5. Jelajahi: **Servers → PostgreSQL-Lab → Databases → akademik → Schemas → app → Tables**

![Figure 2.10 — pgAdmin login dan koneksi ke server](images/placeholder.png)
*Gambar 2.10: Screenshot pgAdmin: (kiri) halaman login pgAdmin, (kanan) dialog Register Server dengan konfigurasi koneksi ke PostgreSQL container.*

---

### Langkah 3: Operasi CRUD SQL

```sql
-- ============================================================
-- Operasi CRUD pada database akademik
-- ============================================================

-- A. CREATE — Tambah data baru
INSERT INTO app.mahasiswa (nim, nama, jurusan, angkatan, email)
VALUES ('3124600041', 'Linda Kusuma', 'Teknik Informatika', 2024, 'linda@pens.ac.id');

INSERT INTO app.nilai (mahasiswa_id, mata_kuliah_id, nilai, grade, semester_ambil)
VALUES (6, 1, '90', 'A', 4);

-- B. READ — Baca data
SELECT * FROM app.mahasiswa WHERE nim = '3124600041';

-- C. UPDATE — Ubah data
UPDATE app.mahasiswa
SET email = 'linda.kusuma@pens.ac.id'
WHERE nim = '3124600041';

-- Verifikasi update
SELECT nim, nama, email FROM app.mahasiswa WHERE nim = '3124600041';

-- D. DELETE — Hapus data
DELETE FROM app.nilai
WHERE mahasiswa_id = (SELECT id FROM app.mahasiswa WHERE nim = '3124600041');

DELETE FROM app.mahasiswa WHERE nim = '3124600041';

-- E. Verifikasi akhir — data kembali seperti semula
SELECT COUNT(*) AS jumlah_mahasiswa FROM app.mahasiswa;
SELECT COUNT(*) AS jumlah_nilai FROM app.nilai;
```

![Figure 3.1 — Hasil operasi CRUD](images/placeholder.png)
*Gambar 3.1: Screenshot hasil seluruh operasi CRUD: INSERT data baru, SELECT verifikasi, UPDATE email, dan DELETE mengembalikan data seperti semula.*

---

### Langkah 4: Backup dan Restore

#### 4.1 Backup database

```bash
# Backup format custom (-Fc) — compressed, mendukung pg_restore
docker exec postgres-db pg_dump \
  -U postgres \
  -Fc \
  -d akademik \
  > backup/akademik-backup.custom

# Backup format SQL plain text — readable, bisa di-psql langsung
docker exec postgres-db pg_dump \
  -U postgres \
  -Fp \
  -d akademik \
  > backup/akademik-backup.sql

# Bandingkan ukuran file
ls -lh backup/
```

![Figure 4.1 — Backup database format custom dan SQL](images/placeholder.png)
*Gambar 4.1: Screenshot hasil backup database: (atas) perintah `pg_dump` format custom dan plain SQL, (bawah) perbandingan ukuran file backup.*

#### 4.2 Restore database

```bash
# Buat database baru untuk pengujian restore
docker exec postgres-db psql -U postgres -c "CREATE DATABASE akademik_restore OWNER labuser;"
```

![Figure 4.2 — Membuat database untuk restore](images/placeholder.png)
*Gambar 4.2: Screenshot hasil `CREATE DATABASE akademik_restore` untuk pengujian restore.*

```bash
# Restore dari format custom menggunakan pg_restore
docker exec -i postgres-db pg_restore \
  -U postgres \
  -d akademik_restore \
  < backup/akademik-backup.custom

# Verifikasi data berhasil direstore
docker exec postgres-db psql -U postgres -d akademik_restore \
  -c "SELECT COUNT(*) FROM app.mahasiswa;"
docker exec postgres-db psql -U postgres -d akademik_restore \
  -c "SELECT COUNT(*) FROM app.nilai;"

# Cleanup: hapus database restore
docker exec postgres-db psql -U postgres -c "DROP DATABASE akademik_restore;"
```

![Figure 4.3 — Restore database dan verifikasi](images/placeholder.png)
*Gambar 4.3: Screenshot hasil restore database: `pg_restore` ke `akademik_restore`, verifikasi jumlah data, dan drop database.*

#### 4.3 Automated backup script

```bash
cat > backup/auto-backup.sh << 'EOF'
#!/bin/bash
# ============================================================
# Auto Backup Script for PostgreSQL Docker
# Usage: ./auto-backup.sh [retention_days]
# ============================================================

set -euo pipefail

# --- Konfigurasi ---
CONTAINER_NAME="postgres-db"
BACKUP_DIR="./backup"
DB_USER="postgres"
DB_NAME="akademik"
RETENTION_DAYS="${1:-7}"           # Default: simpan 7 hari
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# --- Buat direktori backup jika belum ada ---
mkdir -p "$BACKUP_DIR"

# --- Backup format custom ---
echo "[$(date)] Memulai backup database $DB_NAME ..."
docker exec "$CONTAINER_NAME" pg_dump \
  -U "$DB_USER" \
  -Fc \
  -d "$DB_NAME" \
  > "$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.custom"

# --- Kompresi dengan gzip (opsional, custom sudah terkompresi) ---
gzip -f "$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.custom"

echo "[$(date)] Backup selesai: ${DB_NAME}_${TIMESTAMP}.custom.gz"

# --- Rotasi: hapus backup lebih lama dari RETENTION_DAYS ---
echo "[$(date)] Menghapus backup lebih lama dari $RETENTION_DAYS hari..."
find "$BACKUP_DIR" -name "${DB_NAME}_*.custom.gz" -mtime +"$RETENTION_DAYS" -delete

# --- Tampilkan daftar backup ---
echo "[$(date)] Daftar backup saat ini:"
ls -lh "$BACKUP_DIR"/*.custom.gz 2>/dev/null || echo "  (belum ada backup)"

echo "[$(date)] Selesai."
EOF

chmod +x backup/auto-backup.sh
```

![Figure 4.4 — Membuat auto-backup script](images/placeholder.png)
*Gambar 4.4: Screenshot hasil pembuatan file `backup/auto-backup.sh` dan `chmod +x`.*

```bash
# Jalankan script backup
./backup/auto-backup.sh 30
```

![Figure 4.5 — Menjalankan auto-backup.sh](images/placeholder.png)
*Gambar 4.5: Screenshot hasil eksekusi `./backup/auto-backup.sh 30` — backup berhasil dibuat dan rotasi dijalankan.*

```bash
# Lihat file backup yang dihasilkan
ls -lh backup/
```

![Figure 4.6 — ls -lh backup/](images/placeholder.png)
*Gambar 4.6: Screenshot hasil `ls -lh backup/` menampilkan file-file backup termasuk hasil script otomatis.*

---

### Langkah 5: Monitoring PostgreSQL

#### 5.1 Statistik database

```sql
-- Query 1: Statistik koneksi aktif
SELECT
    pid,
    usename  AS username,
    application_name,
    client_addr,
    state,
    query_start,
    LEFT(query, 80) AS query_preview
FROM pg_stat_activity
WHERE state IS NOT NULL
  AND pid <> pg_backend_pid()
ORDER BY query_start DESC;
```

![Figure 5.1 — Query statistik koneksi aktif](images/placeholder.png)
*Gambar 5.1: Screenshot hasil query `pg_stat_activity` menampilkan koneksi aktif ke database.*

```sql
-- Query 2: Statistik tabel (ukuran, rows, sequential vs index scan)
SELECT
    schemaname,
    relname                                              AS table_name,
    n_live_tup                                           AS estimated_rows,
    pg_size_pretty(pg_total_relation_size(relid))        AS total_size,
    pg_size_pretty(pg_relation_size(relid))              AS data_size,
    pg_size_pretty(pg_total_relation_size(relid)
                   - pg_relation_size(relid))             AS index_size,
    seq_scan,
    idx_scan,
    n_tup_ins                                            AS inserts,
    n_tup_upd                                            AS updates,
    n_tup_del                                            AS deletes
FROM pg_stat_user_tables
WHERE schemaname = 'app'
ORDER BY pg_total_relation_size(relid) DESC;
```

![Figure 5.2 — Query statistik tabel](images/placeholder.png)
*Gambar 5.2: Screenshot hasil query statistik tabel di schema `app` — ukuran, estimated rows, index usage, dan operasi DML.*

#### 5.2 Monitoring log PostgreSQL

```bash
# Lihat log PostgreSQL di dalam container
docker exec postgres-db ls /var/log/postgresql/

# Lihat isi log terbaru
docker exec postgres-db tail -50 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log
```

![Figure 5.3 — Log PostgreSQL di container](images/placeholder.png)
*Gambar 5.3: Screenshot hasil `docker exec` untuk melihat log PostgreSQL — menampilkan koneksi, query yang lambat, dan aktivitas autovacuum.*

---

## POST LAB

### 1. Jalankan `docker compose down` lalu `docker compose up -d`. Apakah data mahasiswa masih ada? Buktikan.

Data mahasiswa masih ada karena database menggunakan named volume `pg-data`. Saat `docker compose down`, container dihapus tetapi volume tidak ikut dihapus. Setelah `docker compose up -d`, PostgreSQL menggunakan volume yang sama sehingga seluruh tabel dan data tetap persisten. Hal ini dapat dibuktikan dengan query:

```sql
SELECT * FROM app.mahasiswa;
```

yang masih menampilkan data sebelumnya.

![Figure PostLab 1 — Verifikasi data setelah docker compose down dan up](images/placeholder.png)
*Gambar PostLab 1: Screenshot hasil `docker compose down`, `docker compose up -d`, lalu query `SELECT * FROM app.mahasiswa` — data masih lengkap.*

### 2. Jalankan `docker compose down -v` lalu `docker compose up -d`. Apa yang terjadi? Apakah init script dijalankan ulang?

Perintah `docker compose down -v` menghapus container sekaligus seluruh volume terkait. Akibatnya seluruh data database hilang. Saat `docker compose up -d` dijalankan kembali, PostgreSQL membuat volume baru sehingga init script pada `/docker-entrypoint-initdb.d/` dijalankan ulang untuk membuat schema, tabel, user, dan sample data dari awal.

![Figure PostLab 2 — docker compose down -v dan recreate](images/placeholder.png)
*Gambar PostLab 2: Screenshot hasil `docker compose down -v`, `docker compose up -d`, dan log yang menunjukkan init script dieksekusi ulang.*

### 3. Bandingkan ukuran file backup format custom vs SQL. Mana yang lebih kecil dan mengapa?

Backup format custom (`-Fc`) biasanya lebih kecil dibanding SQL plain text karena menggunakan kompresi internal PostgreSQL. Format SQL menyimpan seluruh query dalam bentuk teks sehingga ukuran file lebih besar terutama untuk database dengan banyak data.

![Figure PostLab 3 — Perbandingan ukuran backup custom vs SQL](images/placeholder.png)
*Gambar PostLab 3: Screenshot perbandingan ukuran file `akademik-backup.custom` vs `akademik-backup.sql` menggunakan `ls -lh`.*

### 4. Buat query yang menampilkan mahasiswa yang belum memiliki nilai di semester apapun.

```sql
SELECT m.*
FROM app.mahasiswa m
LEFT JOIN app.nilai n
    ON m.id = n.mahasiswa_id
WHERE n.id IS NULL;
```

Query tersebut menggunakan `LEFT JOIN` untuk mencari mahasiswa yang tidak memiliki pasangan data pada tabel nilai.

![Figure PostLab 4 — Query mahasiswa tanpa nilai](images/placeholder.png)
*Gambar PostLab 4: Screenshot hasil query `LEFT JOIN` — menampilkan mahasiswa yang belum memiliki nilai.*

### 5. Jelaskan peran user `app_reader` yang dibuat di init script. Apa bedanya dengan `labuser`?

- `app_reader` merupakan user read-only yang hanya memiliki hak `SELECT` pada tabel schema `app`. User ini digunakan untuk aplikasi atau monitoring yang hanya perlu membaca data tanpa melakukan perubahan.
- `labuser` adalah user utama database yang memiliki hak lebih tinggi untuk melakukan operasi seperti `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, dan administrasi database lainnya.
- Pemisahan role ini meningkatkan keamanan karena prinsip least privilege dapat diterapkan.

![Figure PostLab 5 — Perbandingan privileges app_reader vs labuser](images/placeholder.png)
*Gambar PostLab 5: Screenshot perbandingan privileges: (kiri) koneksi sebagai `app_reader` hanya bisa SELECT, (kanan) koneksi sebagai `labuser` bisa INSERT/UPDATE/DELETE.*

---

## DAFTAR GAMBAR

| Figure | Deskripsi |
|--------|-----------|
| 0.1 | Membuat direktori project PostgreSQL |
| 1.1 | Membuat init script `01-create-schema.sql` |
| 1.2 | Membuat custom PostgreSQL config |
| 1.3 | Membuat `docker-compose.yml` |
| 1.4 | `docker compose up -d` |
| 1.5 | `docker compose ps` |
| 1.6 | `docker compose logs db` |
| 2.1 | Instalasi `postgresql-client` |
| 2.2 | Koneksi `psql` ke database `akademik` |
| 2.3 | `\l` — list database |
| 2.4 | `\dn` — list schema |
| 2.5 | `\dt app.*` — list tabel |
| 2.6 | `\d+ app.mahasiswa` — detail struktur tabel |
| 2.7 | Query `SELECT` dan `JOIN` data mahasiswa |
| 2.8 | `docker exec` psql ke container |
| 2.9 | Query join mahasiswa dengan total SKS |
| 2.10 | pgAdmin login dan koneksi server |
| 3.1 | Hasil operasi CRUD |
| 4.1 | Backup database format custom dan SQL |
| 4.2 | Membuat database untuk restore |
| 4.3 | Restore database dan verifikasi |
| 4.4 | Membuat auto-backup script |
| 4.5 | Menjalankan `auto-backup.sh` |
| 4.6 | `ls -lh backup/` — daftar file backup |
| 5.1 | Query statistik koneksi aktif |
| 5.2 | Query statistik tabel |
| 5.3 | Log PostgreSQL di container |
| PostLab 1 | Verifikasi data setelah `docker compose down` dan `up` |
| PostLab 2 | `docker compose down -v` dan recreate |
| PostLab 3 | Perbandingan ukuran backup custom vs SQL |
| PostLab 4 | Query mahasiswa tanpa nilai |
| PostLab 5 | Perbandingan privileges `app_reader` vs `labuser` |

---

*Laporan ini dibuat sebagai bagian dari praktikum Workshop Administrasi Jaringan, Program Studi D4 Teknik Informatika, Politeknik Elektronika Negeri Surabaya (PENS), 2026.*
