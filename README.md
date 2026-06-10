# Workshop Administrasi Jaringan — Docker Lab Modules

Praktikum Workshop Administrasi Jaringan — Program Studi D4 Teknik Informatika, PENS.

Modul praktikum Docker dari instalasi hingga monitoring stack, meliputi 6 modul dengan laporan langkah praktikum, dokumentasi, dan screenshot.

## Modul

| # | Judul | Topik |
|---|-------|-------|
| 1 | [Docker dan Instalasi](Modul%201) | Instalasi Docker Engine di Ubuntu & Docker Desktop di Windows, operasi dasar image/container, custom Dockerfile |
| 2 | [Docker Service Mount](Modul%202) | Docker network (bridge, user-defined), volume, bind mount, tmpfs, multi-container dengan Docker Compose |
| 3 | [Web Service Docker](Modul%203) | Apache virtual host, Nginx reverse proxy + SSL termination, self-signed certificate, Flask backend |
| 4 | [Database PostgreSQL](Modul%204) | Deploy PostgreSQL dengan Docker Compose, pgAdmin, operasi CRUD, backup/restore, monitoring |
| 5 | [Logging Service Docker](Modul%205) | Centralized logging dengan Fluent Bit + PostgreSQL, structured logging, log generator, log query API |
| 6 | [Grafana Service Docker](Modul%206) | Prometheus + Node Exporter + cAdvisor, Grafana provisioning dashboard, stress test monitoring |

## Struktur Repository

```
📁 Modul 1/
├── README.md       — Laporan praktikum
└── images/         — Screenshot langkah praktikum
📁 Modul 2/ ...
📁 Modul 3/ ...
📁 Modul 4/ ...
📁 Modul 5/ ...
📁 Modul 6/ ...
📄 .gitignore
📄 README.md
```

## Tools & Teknologi

- **Docker Engine** & **Docker Desktop**
- **Docker Compose** (multi-container orchestration)
- **PostgreSQL 16** + **pgAdmin 4**
- **Apache httpd** + **Nginx** (reverse proxy)
- **Flask** (Python web API)
- **Fluent Bit** (log collector)
- **Prometheus** + **Node Exporter** + **cAdvisor** (metrics)
- **Grafana** (visualization & dashboard)

---
*Politeknik Elektronika Negeri Surabaya (PENS) — 2026*
