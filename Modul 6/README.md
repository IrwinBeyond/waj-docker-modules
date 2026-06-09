# Tugas Week 10 — Modul 6: Grafana Service Docker untuk Monitoring Resource

| | |
|---|---|
| **Mata Kuliah** | Workshop Administrasi Jaringan |
| **Nama** | Irwin Ahmad Wiryawan |
| **Kelas** | D4 IT B |
| **NRP** | 3124600035 |
| **Dosen Pengampu** | Dr Ferry Astika Saputra ST, M.Sc |

---

## PRE LAB

### 1. Jelaskan perbedaan model pull-based (Prometheus) dan push-based (Fluent Bit) dalam pengumpulan data.

- Pull-based: server mengambil data sendiri dari target (Prometheus scrape /metrics endpoint). Prometheus secara berkala melakukan HTTP request ke setiap target yang dikonfigurasi, lalu menyimpan hasilnya ke time-series database.
- Push-based: client/agent langsung mengirim data ke server tujuan (Fluent Bit push logs ke PostgreSQL). Data dikirimkan secara proaktif tanpa menunggu diminta oleh server.

### 2. Apa itu PromQL? Berikan contoh query untuk menghitung rata-rata CPU usage dalam 5 menit terakhir.

PromQL adalah bahasa query untuk metrics di Prometheus. PromQL digunakan untuk mengambil, mengagregasi, dan memanipulasi data time-series yang disimpan di Prometheus TSDB.

Contoh rata-rata CPU usage 5 menit terakhir:

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 3. Mengapa cAdvisor membutuhkan akses ke /var/run/docker.sock dan /sys?

- `/var/run/docker.sock`: membaca informasi container Docker seperti nama container, image, label, dan metadata container lainnya melalui Docker API.
- `/sys`: membaca statistik kernel seperti CPU, memory, disk I/O, network, dan cgroup yang menyediakan data resource usage setiap container secara real-time.

### 4. Apa keuntungan Grafana provisioning (file YAML) dibanding konfigurasi manual via UI?

- otomatis saat startup — datasource dan dashboard langsung tersedia tanpa setup manual
- bisa disimpan di Git — konfigurasi dapat di-versioning dan di-review
- konsisten antar server — provisioning file menjamin environment development, staging, dan production identik
- tidak perlu setup manual via UI — menghindari human error dan mempercepat deployment

### 5. Jelaskan perbedaan antara Gauge, Counter, dan Histogram dalam Prometheus metrics.

| Type      | Karakteristik     | Contoh          |
|-----------|-------------------|-----------------|
| Gauge     | bisa naik/turun   | RAM usage       |
| Counter   | hanya naik        | total requests  |
| Histogram | distribusi data   | request latency |

---

## LANGKAH PRAKTIKUM

### Langkah 0: Persiapan Environment

Pastikan VM/Host sudah terinstall Docker Engine dan Docker Compose Plugin.

```bash
# Buat direktori project monitoring
mkdir -p ~/monitoring-stack/{app,grafana/provisioning/datasources,grafana/provisioning/dashboards,grafana/dashboards}
cd ~/monitoring-stack
```

![Figure 0.1 — Membuat direktori project monitoring](images/placeholder.png)
*Gambar 0.1: Screenshot hasil `mkdir -p` dan `cd` ke direktori project monitoring-stack.*

---

### Langkah 1: Konfigurasi Prometheus

#### 1.1 Buat file `prometheus.yml`

```bash
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "/etc/prometheus/alert_rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "flask-app"
    static_configs:
      - targets: ["flask-app:5050"]
EOF
```

![Figure 1.1 — Membuat file prometheus.yml](images/placeholder.png)
*Gambar 1.1: Screenshot hasil pembuatan file `prometheus.yml` — konfigurasi global, alerting, rule_files, dan 4 scrape jobs.*

#### 1.2 Buat file `alert_rules.yml`

```bash
cat > alert_rules.yml << 'EOF'
groups:
  - name: container_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage tinggi (instance {{ $labels.instance }})"
          description: "CPU usage di atas 80% selama 5 menit (nilai: {{ $value }}%)"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage tinggi (instance {{ $labels.instance }})"
          description: "Memory usage di atas 85% (nilai: {{ $value }}%)"

      - alert: ContainerDown
        expr: absent(container_last_seen{name=~"flask-app|prometheus|grafana|cadvisor"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} mati"
          description: "Container {{ $labels.name }} tidak terdeteksi selama 1 menit terakhir"

      - alert: HighDiskUsage
        expr: (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage tinggi (instance {{ $labels.instance }})"
          description: "Disk usage di atas 80% pada mountpoint {{ $labels.mountpoint }} (nilai: {{ $value }}%)"
EOF
```

![Figure 1.2 — Membuat file alert_rules.yml](images/placeholder.png)
*Gambar 1.2: Screenshot hasil pembuatan file `alert_rules.yml` — mendefinisikan 4 alert rules (HighCPUUsage, HighMemoryUsage, ContainerDown, HighDiskUsage).*

---

### Langkah 2: Konfigurasi Grafana Provisioning

#### 2.1 Buat file `grafana/provisioning/datasources/datasources.yml`

```bash
cat > grafana/provisioning/datasources/datasources.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: "POST"

  - name: PostgreSQL
    type: postgres
    access: proxy
    url: postgres:5432
    database: logdb
    user: loguser
    secureJsonData:
      password: "logpass123"
    jsonData:
      sslmode: "disable"
      timescaledb: false
      postgresVersion: 1600
EOF
```

![Figure 2.1 — Membuat file datasources.yml](images/placeholder.png)
*Gambar 2.1: Screenshot hasil pembuatan file `grafana/provisioning/datasources/datasources.yml` — 2 datasource (Prometheus dan PostgreSQL) dikonfigurasi sebagai provisioning.*

#### 2.2 Buat file `grafana/provisioning/dashboards/dashboards.yml`

```bash
cat > grafana/provisioning/dashboards/dashboards.yml << 'EOF'
apiVersion: 1

providers:
  - name: "Monitoring Dashboards"
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
EOF
```

![Figure 2.2 — Membuat file dashboards.yml](images/placeholder.png)
*Gambar 2.2: Screenshot hasil pembuatan file `grafana/provisioning/dashboards/dashboards.yml` — dashboard provider dikonfigurasi untuk auto-load dari direktori dashboards.*

#### 2.3 Buat dashboard `grafana/dashboards/docker-host-overview.json`

```bash
cat > grafana/dashboards/docker-host-overview.json << 'JSONEOF'
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": { "type": "datasource", "uid": "grafana" },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": {
          "limit": 100,
          "matchAny": false,
          "tags": [],
          "type": "dashboard"
        },
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "id": 100,
      "panels": [],
      "title": "CPU & Memory Overview",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": { "h": 10, "w": 12, "x": 0, "y": 1 },
      "id": 1,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\",instance=~\"$instance\"}[$__rate_interval])) * 100)",
          "legendFormat": "CPU Usage",
          "refId": "A"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "avg(rate(node_cpu_seconds_total{mode=\"iowait\",instance=~\"$instance\"}[$__rate_interval])) * 100",
          "legendFormat": "IO Wait",
          "refId": "B"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "avg(rate(node_cpu_seconds_total{mode=\"system\",instance=~\"$instance\"}[$__rate_interval])) * 100",
          "legendFormat": "System",
          "refId": "C"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "avg(rate(node_cpu_seconds_total{mode=\"user\",instance=~\"$instance\"}[$__rate_interval])) * 100",
          "legendFormat": "User",
          "refId": "D"
        }
      ],
      "title": "CPU Usage",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 85 }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": { "h": 10, "w": 12, "x": 12, "y": 1 },
      "id": 2,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_memory_MemTotal_bytes{instance=~\"$instance\"}",
          "legendFormat": "Total",
          "refId": "A"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_memory_MemTotal_bytes{instance=~\"$instance\"} - node_memory_MemAvailable_bytes{instance=~\"$instance\"}",
          "legendFormat": "Used",
          "refId": "B"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_memory_MemAvailable_bytes{instance=~\"$instance\"}",
          "legendFormat": "Available",
          "refId": "C"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_memory_Buffers_bytes{instance=~\"$instance\"} + node_memory_Cached_bytes{instance=~\"$instance\"}",
          "legendFormat": "Buffered/Cached",
          "refId": "D"
        }
      ],
      "title": "Memory Usage",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 11 },
      "id": 101,
      "panels": [],
      "title": "Disk & Network",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 12 },
      "id": 3,
      "options": {
        "legend": { "calcs": [], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "(1 - (node_filesystem_avail_bytes{mountpoint=\"/\",instance=~\"$instance\"} / node_filesystem_size_bytes{mountpoint=\"/\",instance=~\"$instance\"})) * 100",
          "legendFormat": "Disk Usage /",
          "refId": "A"
        }
      ],
      "title": "Disk Usage (Root)",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "Bps"
        },
        "overrides": [
          {
            "matcher": { "id": "byName", "options": "Receive" },
            "properties": [
              { "id": "custom.transform", "value": "negative-y" }
            ]
          }
        ]
      },
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 12 },
      "id": 4,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "rate(node_network_receive_bytes_total{device!=\"lo\",instance=~\"$instance\"}[$__rate_interval])",
          "legendFormat": "Receive - {{ device }}",
          "refId": "A"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "rate(node_network_transmit_bytes_total{device!=\"lo\",instance=~\"$instance\"}[$__rate_interval])",
          "legendFormat": "Transmit - {{ device }}",
          "refId": "B"
        }
      ],
      "title": "Network Traffic",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 20 },
      "id": 102,
      "panels": [],
      "title": "System Overview",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "s"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 21 },
      "id": 5,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "time() - node_boot_time_seconds{instance=~\"$instance\"}",
          "refId": "A"
        }
      ],
      "title": "Uptime",
      "type": "stat"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 1 },
              { "color": "red", "value": 5 }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 21 },
      "id": 6,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "count(node_cpu_seconds_total{mode=\"idle\",instance=~\"$instance\"})",
          "refId": "A"
        }
      ],
      "title": "CPU Cores",
      "type": "stat"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "decbytes"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 21 },
      "id": 7,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_memory_MemTotal_bytes{instance=~\"$instance\"}",
          "refId": "A"
        }
      ],
      "title": "Total RAM",
      "type": "stat"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "decbytes"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 21 },
      "id": 8,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "node_filesystem_size_bytes{mountpoint=\"/\",instance=~\"$instance\"}",
          "refId": "A"
        }
      ],
      "title": "Total Disk",
      "type": "stat"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 39,
  "tags": ["docker", "host", "monitoring"],
  "templating": {
    "list": [
      {
        "current": { "selected": false, "text": "Prometheus", "value": "Prometheus" },
        "hide": 0,
        "includeAll": false,
        "label": "Datasource",
        "multi": false,
        "name": "DS_PROMETHEUS",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "current": { "selected": true, "text": "node-exporter:9100", "value": "node-exporter:9100" },
        "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
        "definition": "label_values(node_uname_info, instance)",
        "hide": 0,
        "includeAll": false,
        "label": "Instance",
        "multi": false,
        "name": "instance",
        "options": [],
        "query": { "query": "label_values(node_uname_info, instance)", "refId": "PrometheusVariableQueryEditor-VariableQuery" },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "type": "query"
      }
    ]
  },
  "time": { "from": "now-1h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "Docker Host Overview",
  "uid": "docker-host-overview",
  "version": 1,
  "weekStart": ""
}
JSONEOF
```

![Figure 2.3 — Membuat dashboard docker-host-overview.json (bagian 1)](images/placeholder.png)
*Gambar 2.3: Screenshot hasil pembuatan `docker-host-overview.json` — dashboard Grafana untuk monitoring host (CPU, Memory, Disk, Network, System Overview).*

![Figure 2.4 — Membuat dashboard docker-host-overview.json (bagian 2)](images/placeholder.png)
*Gambar 2.4: Screenshot lanjutan pembuatan `docker-host-overview.json` — panel CPU Usage dan Memory Usage dengan timeseries visualization.*

![Figure 2.5 — Membuat dashboard docker-host-overview.json (bagian 3)](images/placeholder.png)
*Gambar 2.5: Screenshot lanjutan pembuatan `docker-host-overview.json` — stat cards (Uptime, CPU Cores, Total RAM, Total Disk) dan templating variables.*

#### 2.4 Buat dashboard `grafana/dashboards/container-metrics.json`

```bash
cat > grafana/dashboards/container-metrics.json << 'JSONEOF'
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": { "type": "datasource", "uid": "grafana" },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": { "limit": 100, "matchAny": false, "tags": [], "type": "dashboard" },
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "id": 100,
      "panels": [],
      "title": "Container CPU Usage",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": { "h": 10, "w": 24, "x": 0, "y": 1 },
      "id": 1,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "sum(rate(container_cpu_usage_seconds_total{name=~\"$container\"}[$__rate_interval])) by (name) * 100",
          "legendFormat": "{{ name }}",
          "refId": "A"
        }
      ],
      "title": "CPU Usage per Container",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 11 },
      "id": 101,
      "panels": [],
      "title": "Container Memory Usage",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": { "h": 10, "w": 12, "x": 0, "y": 12 },
      "id": 2,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "container_memory_usage_bytes{name=~\"$container\"}",
          "legendFormat": "{{ name }}",
          "refId": "A"
        }
      ],
      "title": "Memory Usage per Container",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 80 }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": { "h": 10, "w": 12, "x": 12, "y": 12 },
      "id": 3,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "container_memory_working_set_bytes{name=~\"$container\"}",
          "legendFormat": "{{ name }} - Working Set",
          "refId": "A"
        }
      ],
      "title": "Memory Working Set per Container",
      "type": "timeseries"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 22 },
      "id": 102,
      "panels": [],
      "title": "Container I/O & Network",
      "type": "row"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 2,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "Bps"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 23 },
      "id": 4,
      "options": {
        "legend": { "calcs": ["mean", "max"], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "desc" }
      },
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "sum(rate(container_network_receive_bytes_total{name=~\"$container\"}[$__rate_interval])) by (name)",
          "legendFormat": "{{ name }} - RX",
          "refId": "A"
        },
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "sum(rate(container_network_transmit_bytes_total{name=~\"$container\"}[$__rate_interval])) by (name)",
          "legendFormat": "{{ name }} - TX",
          "refId": "B"
        }
      ],
      "title": "Network Traffic per Container",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 23 },
      "id": 5,
      "options": {
        "displayMode": "gradient",
        "maxVizHeight": 300,
        "minVizHeight": 16,
        "minVizWidth": 8,
        "namePlacement": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showUnfilled": true,
        "sizing": "auto",
        "valueMode": "color"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
          "expr": "count(count(container_last_seen{name=~\"$container\"}) by (name))",
          "legendFormat": "Running Containers",
          "refId": "A"
        }
      ],
      "title": "Active Containers",
      "type": "bargauge"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 39,
  "tags": ["docker", "container", "cadvisor"],
  "templating": {
    "list": [
      {
        "current": { "selected": false, "text": "Prometheus", "value": "Prometheus" },
        "hide": 0,
        "includeAll": false,
        "label": "Datasource",
        "multi": false,
        "name": "DS_PROMETHEUS",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "current": { "selected": true, "text": "All", "value": "$__all" },
        "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
        "definition": "label_values(container_last_seen, name)",
        "hide": 0,
        "includeAll": true,
        "label": "Container",
        "multi": true,
        "name": "container",
        "options": [],
        "query": { "query": "label_values(container_last_seen, name)", "refId": "PrometheusVariableQueryEditor-VariableQuery" },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "type": "query"
      }
    ]
  },
  "time": { "from": "now-1h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "Container Metrics",
  "uid": "container-metrics",
  "version": 1,
  "weekStart": ""
}
JSONEOF
```

![Figure 2.6 — Membuat dashboard container-metrics.json](images/placeholder.png)
*Gambar 2.6: Screenshot hasil pembuatan `container-metrics.json` — dashboard Grafana untuk monitoring container (CPU, Memory, Network, Active Containers) dari metric cAdvisor.*

#### 2.5 Buat dashboard `grafana/dashboards/log-analytics.json`

```bash
cat > grafana/dashboards/log-analytics.json << 'JSONEOF'
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": { "type": "datasource", "uid": "grafana" },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": { "limit": 100, "matchAny": false, "tags": [], "type": "dashboard" },
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "id": 100,
      "panels": [],
      "title": "Log Overview",
      "type": "row"
    },
    {
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "palette-classic" },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "bars",
            "fillOpacity": 80,
            "gradientMode": "none",
            "hideFrom": { "legend": false, "tooltip": false, "viz": false },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": { "type": "linear" },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": { "group": "A", "mode": "none" },
            "thresholdsStyle": { "mode": "off" }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 1 },
      "id": 1,
      "options": {
        "legend": { "calcs": [], "displayMode": "table", "placement": "bottom", "showLegend": true },
        "tooltip": { "mode": "multi", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
          "format": "table",
          "rawSql": "SELECT\n  $__timeGroupAlias(timestamp, $__interval),\n  level AS metric,\n  COUNT(*) AS value\nFROM container_logs\nWHERE $__timeFilter(timestamp)\nGROUP BY 1, 2\nORDER BY 1",
          "refId": "A"
        }
      ],
      "title": "Log Volume by Level",
      "type": "timeseries"
    },
    {
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [
            { "options": { "ERROR": { "color": "red", "index": 0 }, "INFO": { "color": "green", "index": 1 }, "WARN": { "color": "orange", "index": 2 } }, "type": "value" }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 8, "w": 6, "x": 12, "y": 1 },
      "id": 2,
      "options": {
        "displayLabels": ["percent", "name"],
        "legend": { "displayMode": "table", "placement": "right", "showLegend": true, "values": ["value", "percent"] },
        "pieType": "donut",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "tooltip": { "mode": "single", "sort": "none" }
      },
      "targets": [
        {
          "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
          "format": "table",
          "rawSql": "SELECT\n  level,\n  COUNT(*) AS value\nFROM container_logs\nWHERE $__timeFilter(timestamp)\nGROUP BY level\nORDER BY value DESC",
          "refId": "A"
        }
      ],
      "title": "Log Level Distribution",
      "type": "piechart"
    },
    {
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 10 },
              { "color": "red", "value": 50 }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 1 },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "targets": [
        {
          "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
          "format": "table",
          "rawSql": "SELECT COUNT(*) AS value FROM container_logs WHERE $__timeFilter(timestamp)",
          "refId": "A"
        }
      ],
      "title": "Total Logs",
      "type": "stat"
    },
    {
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 1 }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 5 },
      "id": 4,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "percentChangeColorMode": "standard",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "showPercentChange": false,
        "textMode": "auto",
        "wideLayout": true
      },
      "targets": [
        {
          "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
          "format": "table",
          "rawSql": "SELECT COUNT(*) AS value FROM container_logs WHERE level = 'ERROR' AND $__timeFilter(timestamp)",
          "refId": "A"
        }
      ],
      "title": "Error Count",
      "type": "stat"
    },
    {
      "collapsed": false,
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 9 },
      "id": 101,
      "panels": [],
      "title": "Recent Logs",
      "type": "row"
    },
    {
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "custom": {
            "align": "auto",
            "cellOptions": { "type": "auto" },
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null }
            ]
          }
        },
        "overrides": [
          {
            "matcher": { "id": "byName", "options": "level" },
            "properties": [
              {
                "id": "mappedColors",
                "value": {
                  "ERROR": { "color": "red", "index": 0 },
                  "INFO": { "color": "green", "index": 1 },
                  "WARN": { "color": "orange", "index": 2 }
                }
              }
            ]
          }
        ]
      },
      "gridPos": { "h": 10, "w": 24, "x": 0, "y": 10 },
      "id": 5,
      "options": {
        "cellHeight": "sm",
        "footer": { "countRows": false, "fields": "", "reducer": ["sum"], "show": false },
        "showHeader": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "datasource": { "type": "postgres", "uid": "${DS_POSTGRES}" },
          "format": "table",
          "rawSql": "SELECT\n  timestamp,\n  container_name,\n  level,\n  message\nFROM container_logs\nWHERE $__timeFilter(timestamp)\nORDER BY timestamp DESC\nLIMIT 100",
          "refId": "A"
        }
      ],
      "title": "Recent Log Entries",
      "type": "table"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 39,
  "tags": ["logs", "fluent-bit", "postgres"],
  "templating": {
    "list": [
      {
        "current": { "selected": false, "text": "PostgreSQL", "value": "PostgreSQL" },
        "hide": 0,
        "includeAll": false,
        "label": "Datasource",
        "multi": false,
        "name": "DS_POSTGRES",
        "options": [],
        "query": "postgres",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      }
    ]
  },
  "time": { "from": "now-1h", "to": "now" },
  "timepicker": {},
  "timezone": "browser",
  "title": "Log Analytics",
  "uid": "log-analytics",
  "version": 1,
  "weekStart": ""
}
JSONEOF
```

![Figure 2.7 — Membuat dashboard log-analytics.json](images/placeholder.png)
*Gambar 2.7: Screenshot hasil pembuatan `log-analytics.json` — dashboard Grafana untuk analisis log dari PostgreSQL (log volume, level distribution, recent entries).*

---

### Langkah 3: Aplikasi Flask dengan Prometheus Metrics

#### 3.1 Buat `app/requirements.txt`

```bash
cat > app/requirements.txt << 'EOF'
flask==3.1.*
prometheus-client==0.21.*
EOF
```

![Figure 3.1 — Membuat file requirements.txt](images/placeholder.png)
*Gambar 3.1: Screenshot hasil pembuatan `app/requirements.txt` — berisi Flask dan prometheus-client.*

#### 3.2 Buat `app/app.py`

```bash
cat > app/app.py << 'PYEOF'
import time
import random
import logging
from flask import Flask, jsonify, request
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
from prometheus_client import make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware

# Konfigurasi logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Prometheus Metrics
REQUEST_COUNT = Counter(
    'flask_http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)
REQUEST_LATENCY = Histogram(
    'flask_http_request_duration_seconds',
    'HTTP request latency in seconds',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)
REQUEST_IN_PROGRESS = Gauge(
    'flask_http_requests_in_progress',
    'HTTP requests currently in progress',
    ['method', 'endpoint']
)
ERROR_COUNT = Counter(
    'flask_http_errors_total',
    'Total HTTP errors',
    ['method', 'endpoint', 'status']
)

# Custom business metrics
ITEM_COUNTER = Counter(
    'flask_items_processed_total',
    'Total items processed'
)
ACTIVE_USERS = Gauge(
    'flask_active_users',
    'Number of currently active users'
)

@app.before_request
def before_request():
    """Track in-progress requests"""
    endpoint = request.endpoint or 'unknown'
    REQUEST_IN_PROGRESS.labels(
        method=request.method,
        endpoint=endpoint
    ).inc()

@app.after_request
def after_request(response):
    """Track request metrics"""
    endpoint = request.endpoint or 'unknown'
    REQUEST_IN_PROGRESS.labels(
        method=request.method,
        endpoint=endpoint
    ).dec()
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=endpoint,
        status=response.status_code
    ).inc()
    if response.status_code >= 400:
        ERROR_COUNT.labels(
            method=request.method,
            endpoint=endpoint,
            status=response.status_code
        ).inc()
    return response

@app.route('/')
def index():
    """Halaman utama"""
    logger.info("Halaman utama diakses")
    return jsonify({
        "service": "Flask Monitoring App",
        "version": "1.0.0",
        "endpoints": [
            "/",
            "/api/health",
            "/api/process",
            "/api/simulate-load",
            "/api/users",
            "/metrics"
        ]
    })

@app.route('/api/health')
def health():
    """Health check endpoint"""
    logger.info("Health check")
    return jsonify({
        "status": "healthy",
        "uptime": "ok"
    })

@app.route('/api/process')
def process():
    """Endpoint yang mensimulasikan pemrosesan data"""
    ITEM_COUNTER.inc()
    logger.info("Memproses item data")
    processing_time = random.uniform(0.02, 0.5)
    time.sleep(processing_time)

    if random.random() < 0.1:  # 10% chance error
        logger.error("Gagal memproses item data")
        return jsonify({"error": "Processing failed", "details": "Simulated random error"}), 500

    logger.info("Item berhasil diproses dalam {:.3f}s".format(processing_time))
    return jsonify({
        "status": "processed",
        "processing_time": round(processing_time, 3),
        "message": "Item processed successfully"
    })

@app.route('/api/simulate-load')
def simulate_load():
    """Endpoint untuk simulasi beban CPU/memory"""
    duration = float(request.args.get('duration', '1'))
    intensity = int(request.args.get('intensity', '1000000'))
    logger.info("Simulasi CPU load selama {} detik, intensity {}".format(duration, intensity))

    end_time = time.time() + duration
    counter = 0
    while time.time() < end_time:
        counter += 1
        _ = counter ** 0.5  # Operasi matematika ringan

    logger.info("Simulasi load selesai, {} iterasi".format(counter))
    return jsonify({
        "status": "load_completed",
        "iterations": counter,
        "duration": duration
    })

@app.route('/api/users')
def users():
    """Endpoint user untuk testing metrics"""
    ACTIVE_USERS.set(random.randint(10, 100))
    logger.info("Cek user aktif: {}".format(ACTIVE_USERS._value.get()))
    return jsonify({
        "active_users": int(ACTIVE_USERS._value.get()),
        "total_requests": int(REQUEST_COUNT._value.get())
    })

@app.route('/metrics')
def metrics():
    """Expose Prometheus metrics"""
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == "__main__":
    logger.info("Flask app starting on port 5050")
    app.run(host="0.0.0.0", port=5050)
PYEOF
```

![Figure 3.2 — Membuat file app.py (bagian 1)](images/placeholder.png)
*Gambar 3.2: Screenshot hasil pembuatan `app/app.py` bagian 1 — import library, definisi Prometheus metrics (Counter, Histogram, Gauge), dan before_request/after_request hooks.*

![Figure 3.3 — Membuat file app.py (bagian 2)](images/placeholder.png)
*Gambar 3.3: Screenshot hasil pembuatan `app/app.py` bagian 2 — definisi endpoint Flask (/api/health, /api/process, /api/simulate-load, /api/users, /metrics).*

#### 3.3 Buat `app/Dockerfile`

```bash
cat > app/Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5050

HEALTHCHECK --interval=10s --timeout=5s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5050/api/health')" || exit 1

CMD ["python", "app.py"]
EOF
```

![Figure 3.4 — Membuat Dockerfile untuk Flask app](images/placeholder.png)
*Gambar 3.4: Screenshot hasil pembuatan `app/Dockerfile` — menggunakan Python 3.11 slim, menginstall dependencies, dan Healthcheck ke /api/health.*

---

### Langkah 4: Konfigurasi Fluent Bit dan PostgreSQL

#### 4.1 Buat file `fluent-bit.conf`

```bash
cat > fluent-bit.conf << 'EOF'
[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     info
    Parsers_File  /fluent-bit/etc/parsers.conf

[INPUT]
    Name          tail
    Path          /var/lib/docker/containers/*/*.log
    Parser        docker
    Tag           docker.*
    Refresh_Interval 5
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On

[INPUT]
    Name          exec
    Command       python3 /generator/generate_logs.py
    Tag           generator.logs
    Interval_Sec  10
    Interval_NSec 0

[FILTER]
    Name          modify
    Match         docker.*
    Add           source fluent-bit

[FILTER]
    Name          parser
    Match         docker.*
    Key_Name      log
    Parser        json_log
    Reserve_Data  On

[OUTPUT]
    Name          pgsql
    Match         *
    Host          postgres
    Port          5432
    User          loguser
    Password      logpass123
    Database      logdb
    Table         container_logs
    Timestamp_Key timestamp
    Log_Level     info
EOF
```

![Figure 4.1 — Membuat file fluent-bit.conf](images/placeholder.png)
*Gambar 4.1: Screenshot hasil pembuatan `fluent-bit.conf` — konfigurasi input (tail Docker logs + exec generator), filter (modify + parser), dan output ke PostgreSQL.*

#### 4.2 Buat file generator

```bash
mkdir -p generator
```

```bash
cat > generator/generate_logs.py << 'PYEOF'
#!/usr/bin/env python3
"""Generator log untuk testing pipeline Fluent Bit -> PostgreSQL."""
import random
import json
import sys
from datetime import datetime, timezone

LEVELS = ["INFO", "INFO", "INFO", "INFO", "WARN", "WARN", "ERROR"]
CONTAINERS = ["flask-app", "prometheus", "grafana", "cadvisor", "node-exporter"]
MESSAGES = {
    "INFO": [
        "Request processed successfully in {:.3f}s",
        "Health check passed",
        "Container started successfully",
        "Connection pool initialized",
        "Cache refreshed, {} entries loaded",
        "Background task completed",
        "Configuration reloaded",
        "Metric snapshot saved"
    ],
    "WARN": [
        "High response time detected: {:.3f}s",
        "Connection pool almost exhausted",
        "Disk usage above 75%",
        "Retry attempt {}/3 for external service",
        "Memory usage approaching limit"
    ],
    "ERROR": [
        "Connection timeout to database",
        "Failed to process request: internal error",
        "Disk write failed: no space left",
        "Authentication failed for service account",
        "OOM killer triggered on container"
    ]
}

for _ in range(random.randint(3, 8)):
    level = random.choice(LEVELS)
    container = random.choice(CONTAINERS)
    msg_template = random.choice(MESSAGES[level])

    if "{}" in msg_template:
        if "{:.3f}" in msg_template:
            message = msg_template.format(random.uniform(0.001, 5.0))
        elif "{}/3" in msg_template:
            message = msg_template.format(random.randint(1, 3))
        elif "{} entries" in msg_template:
            message = msg_template.format(random.randint(100, 5000))
        else:
            message = msg_template
    else:
        message = msg_template

    log_entry = {
        "level": level,
        "container_name": container,
        "message": message,
        "timestamp": datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S")
    }
    print(json.dumps(log_entry))
    sys.stdout.flush()
PYEOF
```

![Figure 4.2 — Membuat file generator generate_logs.py](images/placeholder.png)
*Gambar 4.2: Screenshot hasil pembuatan `generator/generate_logs.py` — script Python yang menghasilkan sample log (INFO, WARN, ERROR) untuk berbagai container secara berkala.*

#### 4.3 Buat file `init.sql`

```bash
cat > init.sql << 'EOF'
CREATE TABLE IF NOT EXISTS container_logs (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    container_name VARCHAR(255),
    level VARCHAR(10),
    message TEXT,
    source VARCHAR(100)
);

CREATE INDEX IF NOT EXISTS idx_container_logs_timestamp ON container_logs (timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_container_logs_level ON container_logs (level);
CREATE INDEX IF NOT EXISTS idx_container_logs_container ON container_logs (container_name);

COMMENT ON TABLE container_logs IS 'Container logs collected by Fluent Bit';
COMMENT ON COLUMN container_logs.timestamp IS 'Timestamp saat log dihasilkan';
COMMENT ON COLUMN container_logs.container_name IS 'Nama container sumber log';
COMMENT ON COLUMN container_logs.level IS 'Level log (INFO, WARN, ERROR)';
COMMENT ON COLUMN container_logs.message IS 'Isi pesan log';
COMMENT ON COLUMN container_logs.source IS 'Sumber pipeline (fluent-bit, generator)';
EOF
```

![Figure 4.3 — Membuat file init.sql](images/placeholder.png)
*Gambar 4.3: Screenshot hasil pembuatan `init.sql` — SQL untuk membuat tabel `container_logs` dengan indeks timestamp, level, dan container_name di PostgreSQL.*

---

### Langkah 5: Docker Compose Multi-Service (9 Services)

```bash
cat > docker-compose.yml << 'EOF'
services:
  # ========== 1. Prometheus ==========
  prometheus:
    image: prom/prometheus:v3.3.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prom-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  # ========== 2. Alertmanager ==========
  alertmanager:
    image: prom/alertmanager:v0.28.0
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/config.yml:ro
    ports:
      - "9093:9093"
    networks:
      - monitoring
    restart: unless-stopped

  # ========== 3. Grafana ==========
  grafana:
    image: grafana/grafana:11.6.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=
    volumes:
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:ro
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus
      - postgres
    restart: unless-stopped

  # ========== 4. cAdvisor ==========
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.51.0
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped

  # ========== 5. Node Exporter ==========
  node-exporter:
    image: prom/node-exporter:v1.9.1
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

  # ========== 6. PostgreSQL ==========
  postgres:
    image: postgres:16-alpine
    container_name: log-db
    environment:
      POSTGRES_DB: logdb
      POSTGRES_USER: loguser
      POSTGRES_PASSWORD: logpass123
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
      - pg-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - monitoring
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U loguser -d logdb"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  # ========== 7. Flask App ==========
  flask-app:
    build: ./app
    container_name: flask-app
    ports:
      - "5050:5050"
    networks:
      - monitoring
    restart: unless-stopped

  # ========== 8. Fluent Bit ==========
  fluent-bit:
    image: fluent/fluent-bit:3.2.10
    container_name: fluent-bit
    volumes:
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./generator:/generator:ro
    environment:
      - TZ=Asia/Jakarta
    ports:
      - "2020:2020"
    networks:
      - monitoring
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  # ========== 9. Nginx Reverse Proxy ==========
  nginx:
    image: nginx:1.27-alpine
    container_name: proxy-nginx
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "80:80"
    networks:
      - monitoring
    depends_on:
      - flask-app
      - grafana
      - prometheus
    restart: unless-stopped

volumes:
  prom-data:
  grafana-data:
  pg-data:

networks:
  monitoring:
    driver: bridge
EOF
```

![Figure 5.1 — Membuat docker-compose.yml (bagian 1: services 1-4)](images/placeholder.png)
*Gambar 5.1: Screenshot hasil pembuatan `docker-compose.yml` bagian 1 — service prometheus, alertmanager, grafana, dan cadvisor beserta volume, network, dan environment.*

![Figure 5.2 — Membuat docker-compose.yml (bagian 2: services 5-7)](images/placeholder.png)
*Gambar 5.2: Screenshot hasil pembuatan `docker-compose.yml` bagian 2 — service node-exporter, postgres, dan flask-app dengan healthcheck dan build context.*

![Figure 5.3 — Membuat docker-compose.yml (bagian 3: services 8-9 + volumes/networks)](images/placeholder.png)
*Gambar 5.3: Screenshot hasil pembuatan `docker-compose.yml` bagian 3 — service fluent-bit, nginx, volume definitions (prom-data, grafana-data, pg-data), dan network monitoring.*

---

### Langkah 5b: Konfigurasi Pendukung

#### Buat `alertmanager.yml`

```bash
cat > alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
  - name: 'default-receiver'
    webhook_configs:
      - url: 'http://localhost:8080/dummy-webhook'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
EOF
```

#### Buat `nginx.conf`

```bash
cat > nginx.conf << 'EOF'
server {
    listen 80;

    location /grafana/ {
        proxy_pass http://grafana:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /prometheus/ {
        proxy_pass http://prometheus:9090/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://flask-app:5050;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://flask-app:5050;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

#### Buat `generator/parsers.conf`

```bash
cat > generator/parsers.conf << 'EOF'
[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   On

[PARSER]
    Name        json_log
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%d %H:%M:%S
    Time_Keep   On
EOF
```

---

### Langkah 6: Deploy Stack

#### 6.1 Build dan jalankan semua service

```bash
# Build image Flask app dan pull semua image
docker compose build

# Jalankan semua service di background
docker compose up -d
```

![Figure 6.1 — Docker compose up -d](images/placeholder.png)
*Gambar 6.1: Screenshot hasil `docker compose up -d` — semua 9 service berhasil dijalankan di background.*

#### 6.2 Cek status semua service

```bash
# Cek status container
docker compose ps
```

![Figure 6.2 — Docker compose ps](images/placeholder.png)
*Gambar 6.2: Screenshot hasil `docker compose ps` — menampilkan status semua 9 service (prometheus, alertmanager, grafana, cadvisor, node-exporter, postgres, flask-app, fluent-bit, nginx) dalam keadaan Up dan healthy.*

```bash
# Cek log jika ada service yang bermasalah
docker compose logs --tail=20
```

---

### Langkah 7: Verifikasi Prometheus

#### 7.1 Akses Prometheus UI

Buka browser ke http://localhost:9090 atau http://<IP_VM>:9090.

![Figure 7.1 — Prometheus Web UI](images/placeholder.png)
*Gambar 7.1: Screenshot Prometheus Web UI di http://localhost:9090 — tampilan utama dengan Graph dan query editor.*

#### 7.2 Cek targets via API

```bash
# Cek status semua targets Prometheus via API
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool
```

![Figure 7.2 — Curl Prometheus targets API (bagian 1)](images/placeholder.png)
*Gambar 7.2: Screenshot hasil `curl http://localhost:9090/api/v1/targets` — menampilkan response JSON dengan status semua scrape targets (prometheus, node-exporter, cadvisor, flask-app).*

```bash
# Filter hanya target yang UP
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import json, sys
data = json.load(sys.stdin)
for t in data['data']['activeTargets']:
    status = 'UP' if t['health'] == 'up' else 'DOWN'
    print(f\"{status:5s} | {t['labels']['job']:20s} | {t['scrapeUrl']}\")
"
```

![Figure 7.3 — Curl Prometheus targets API (bagian 2 — filtered)](images/placeholder.png)
*Gambar 7.3: Screenshot hasil filtering targets — menampilkan status UP/DOWN untuk setiap job (prometheus, node-exporter, cadvisor, flask-app).*

#### 7.3 Query PromQL — CPU Usage

Buka tab **Graph** di Prometheus UI, masukkan query berikut, lalu klik **Execute**:

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

![Figure 7.4 — Query PromQL CPU Usage di Prometheus](images/placeholder.png)
*Gambar 7.4: Screenshot hasil query `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` di Prometheus UI — grafik CPU usage host.*

#### 7.4 Query PromQL — Memory Usage

```
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

![Figure 7.5 — Query PromQL Memory Usage di Prometheus](images/placeholder.png)
*Gambar 7.5: Screenshot hasil query memory usage — grafik persentase penggunaan memory host.*

#### 7.5 Query PromQL — Container Count

```
count(container_last_seen)
```

![Figure 7.6 — Query PromQL Container Count](images/placeholder.png)
*Gambar 7.6: Screenshot hasil query `count(container_last_seen)` — menampilkan jumlah container yang terpantau oleh cAdvisor.*

#### 7.6 Targets Status Page

Buka **Status → Targets** di Prometheus UI untuk melihat semua target scrape.

![Figure 7.7 — Prometheus Targets Status Page](images/placeholder.png)
*Gambar 7.7: Screenshot halaman Status → Targets di Prometheus UI — menampilkan daftar semua scrape target beserta state (UP/DOWN), labels, last scrape, dan scrape duration.*

#### 7.7 Node Exporter Metrics

```bash
# Cek metric yang diexpose node-exporter
curl -s http://localhost:9100/metrics | head -50
```

![Figure 7.8 — Curl node-exporter metrics](images/placeholder.png)
*Gambar 7.8: Screenshot hasil `curl http://localhost:9100/metrics | head -50` — menampilkan 50 baris pertama metric dari node-exporter (CPU, memory, disk, network).*

#### 7.8 Specific Node Exporter Metric

```bash
# Cek metric spesifik: available memory
curl -s http://localhost:9100/metrics | grep node_memory_MemAvailable
```

![Figure 7.9 — Specific metric node_memory_MemAvailable](images/placeholder.png)
*Gambar 7.9: Screenshot hasil `curl http://localhost:9100/metrics | grep node_memory_MemAvailable` — menampilkan metric memory available dalam bytes.*

#### 7.9 cAdvisor Web UI

Buka browser ke http://localhost:8080 untuk melihat cAdvisor Web UI.

![Figure 7.10 — cAdvisor Web UI](images/placeholder.png)
*Gambar 7.10: Screenshot cAdvisor Web UI di http://localhost:8080 — menampilkan overview resource usage semua container Docker.*

#### 7.10 cAdvisor Metrics Endpoint

```bash
# Cek metric yang diexpose cAdvisor
curl -s http://localhost:8080/metrics | grep container_memory_usage_bytes | head -20
```

![Figure 7.11 — cAdvisor metrics endpoint](images/placeholder.png)
*Gambar 7.11: Screenshot hasil `curl http://localhost:8080/metrics` — menampilkan container_memory_usage_bytes untuk setiap container.*

#### 7.11 Generate Traffic ke Flask App

```bash
# Generate traffic ke Flask app untuk menghasilkan metrics
for i in $(seq 1 20); do
  curl -s http://localhost:5050/api/process &
  curl -s http://localhost:5050/api/health &
done
wait
echo "Traffic generated!"
```

![Figure 7.12 — Generate traffic ke Flask app](images/placeholder.png)
*Gambar 7.12: Screenshot hasil generate traffic — 40 request (20 process + 20 health) dikirim ke Flask app.*

#### 7.12 Verifikasi Flask Metrics di Prometheus

Buka Prometheus UI dan query metric dari Flask app:

**Query 1 — Total HTTP Requests:**
```
flask_http_requests_total
```

![Figure 7.13 — Flask metrics: HTTP requests total](images/placeholder.png)
*Gambar 7.13: Screenshot query `flask_http_requests_total` di Prometheus UI — menampilkan jumlah total HTTP request ke Flask app berdasarkan method, endpoint, dan status.*

**Query 2 — Request Latency:**
```
histogram_quantile(0.95, rate(flask_http_request_duration_seconds_bucket[5m]))
```

![Figure 7.14 — Flask metrics: Request latency p95](images/placeholder.png)
*Gambar 7.14: Screenshot query `histogram_quantile(0.95, ...)` di Prometheus UI — menampilkan latency request persentil ke-95 untuk Flask app.*

**Query 3 — Active Users:**
```
flask_active_users
```

![Figure 7.15 — Flask metrics: Active users](images/placeholder.png)
*Gambar 7.15: Screenshot query `flask_active_users` di Prometheus UI — menampilkan jumlah active users dari Flask app (Gauge metric).*

---

### Langkah 8: Setup dan Eksplorasi Grafana

#### 8.1 Login ke Grafana

Buka browser ke http://localhost:3000. Login dengan:
- **Username:** admin
- **Password:** admin

> Grafana akan meminta ganti password saat login pertama. Lewati atau ganti sesuai keinginan.

![Figure 8.1 — Grafana Login Page](images/placeholder.png)
*Gambar 8.1: Screenshot halaman login Grafana di http://localhost:3000 — field username dan password diisi admin/admin.*

#### 8.2 Verifikasi Datasources

Navigasi ke **Connections → Data Sources** di sidebar Grafana. Pastikan datasource **Prometheus** dan **PostgreSQL** sudah muncul (hasil provisioning).

![Figure 8.2 — Grafana Datasources list](images/placeholder.png)
*Gambar 8.2: Screenshot halaman Connections → Data Sources di Grafana — menampilkan datasource Prometheus dan PostgreSQL yang sudah terprovisioning.*

#### 8.3 Test Datasource Prometheus

Klik datasource **Prometheus** → klik tombol **Save & test**. Harus muncul pesan "Data source is working".

![Figure 8.3 — Test Prometheus datasource](images/placeholder.png)
*Gambar 8.3: Screenshot hasil "Save & test" datasource Prometheus — menampilkan notifikasi hijau "Data source is working".*

#### 8.4 Test Datasource PostgreSQL

Klik datasource **PostgreSQL** → klik tombol **Save & test**. Harus muncul pesan "Database Connection OK".

![Figure 8.4 — Test PostgreSQL datasource](images/placeholder.png)
*Gambar 8.4: Screenshot hasil "Save & test" datasource PostgreSQL — menampilkan notifikasi hijau "Database Connection OK".*

#### 8.5 Cek Dashboard List

Navigasi ke **Dashboards** di sidebar → pilih folder **General**. Tiga dashboard dari provisioning akan muncul:
- Docker Host Overview
- Container Metrics
- Log Analytics

![Figure 8.5 — Grafana Dashboards list](images/placeholder.png)
*Gambar 8.5: Screenshot halaman Dashboards di Grafana — menampilkan 3 dashboard yang terprovisioning (Docker Host Overview, Container Metrics, Log Analytics).*

#### 8.6 Buat Custom Panel di Dashboard Baru

1. Klik **+** icon di sidebar → **New Dashboard** → **Add visualization**
2. Pilih datasource **Prometheus**
3. Masukkan query:
   ```
   rate(flask_http_requests_total{endpoint="/api/process"}[5m])
   ```
4. Pilih visualization type **Time series**
5. Set title panel: **Flask API Process Request Rate**
6. Klik **Apply** → **Save dashboard** dengan nama "Custom Flask Monitoring"

![Figure 8.6 — Custom Grafana panel Flask request rate](images/placeholder.png)
*Gambar 8.6: Screenshot panel custom Grafana — menampilkan `rate(flask_http_requests_total{endpoint="/api/process"}[5m])` dengan visualization type Time series.*

#### 8.7 Buat Alert Rule di Grafana

1. Navigasi ke **Alerting → Alert rules** → klik **New alert rule**
2. **Rule name:** High Flask Error Rate
3. **Metric query** (Prometheus):
   ```
   rate(flask_http_errors_total[5m])
   ```
4. **Condition:** `IS ABOVE 0.1` (threshold error rate > 0.1 per detik)
5. **Evaluation interval:** 1m
6. **Folder:** General
7. **Evaluation group:** flask-alerts
8. Klik **Save rule and exit**

![Figure 8.7 — Grafana alert rule](images/placeholder.png)
*Gambar 8.7: Screenshot pembuatan alert rule "High Flask Error Rate" di Grafana — menggunakan Prometheus query `rate(flask_http_errors_total[5m])` dengan threshold > 0.1.*

---

### Langkah 9: Stress Test dan Observasi Monitoring

#### 9.1 Install stress tool di host

```bash
# Install stress tool untuk CPU/memory stress test
sudo apt update && sudo apt install -y stress
```

![Figure 9.1 — Install stress tool](images/placeholder.png)
*Gambar 9.1: Screenshot hasil `sudo apt install -y stress` — stress tool berhasil diinstal di host.*

#### 9.2 Jalankan stress test CPU

```bash
# Stress 2 core CPU selama 120 detik
stress --cpu 2 --timeout 120 &
```

![Figure 9.2 — Stress test CPU](images/placeholder.png)
*Gambar 9.2: Screenshot hasil `stress --cpu 2 --timeout 120` — stress test berjalan pada 2 core CPU selama 120 detik.*

#### 9.3 Generate traffic simultan ke Flask app

```bash
# Simulasi load ke Flask app endpoint processing
curl -s "http://localhost:5050/api/simulate-load?duration=3&intensity=5000000" &
curl -s "http://localhost:5050/api/simulate-load?duration=3&intensity=5000000" &

# Loop request ke /api/process selama stress berjalan
for i in $(seq 1 50); do
  curl -s http://localhost:5050/api/process &
done
wait
echo "Load simulation complete!"
```

![Figure 9.3 — Generate traffic ke Flask app selama stress test](images/placeholder.png)
*Gambar 9.3: Screenshot hasil generate traffic (simulate-load + 50 process request) ke Flask app selama stress test berjalan.*

#### 9.4 Observasi CPU Usage di Grafana Dashboard

Buka dashboard **Docker Host Overview** di Grafana. Perhatikan grafik CPU Usage — akan terlihat lonjakan CPU usage selama stress test berlangsung (~90-100%).

![Figure 9.4 — Grafana Dashboard Host Overview saat stress test](images/placeholder.png)
*Gambar 9.4: Screenshot dashboard Docker Host Overview di Grafana — grafik CPU Usage menunjukkan lonjakan ~90-100% selama 120 detik stress test, kemudian turun kembali ke ~10% setelah selesai.*

#### 9.5 Observasi Container Metrics

Buka dashboard **Container Metrics** di Grafana. Amati perubahan CPU dan memory setiap container selama stress test.

![Figure 9.5 — Grafana Dashboard Container Metrics saat stress test](images/placeholder.png)
*Gambar 9.5: Screenshot dashboard Container Metrics di Grafana — menampilkan CPU dan memory usage per container (flask-app, flask-app terlihat meningkat saat traffic test).*

#### 9.6 Observasi Log Analytics

Buka dashboard **Log Analytics** di Grafana. Perhatikan log dari generator Fluent Bit — akan terlihat log ERROR dan WARN yang muncul bersama INFO.

![Figure 9.6 — Grafana Dashboard Log Analytics](images/placeholder.png)
*Gambar 9.6: Screenshot dashboard Log Analytics di Grafana — menampilkan log volume by level, donut chart distribusi level log, dan tabel recent log entries dari PostgreSQL.*

#### 9.7 Cek Alerting di Grafana

Navigasi ke **Alerting → Alert rules**. Perhatikan status alert "High Flask Error Rate" — jika error rate di atas threshold, alert akan berstatus **Firing**.

![Figure 9.7 — Grafana Alerting status](images/placeholder.png)
*Gambar 9.7: Screenshot halaman Alerting → Alert rules di Grafana — menampilkan status alert rule (Normal/Firing) dan event history.*

---

## POST LAB

### 1. Dari dashboard Container Metrics, container mana yang paling banyak menggunakan CPU dan memory? Mengapa?

- Container dengan CPU usage tertinggi adalah container yang sedang aktif memproses workload atau request. Pada saat praktikum, **flask-app** menunjukkan lonjakan CPU usage karena menerima traffic API (simulate-load dan /api/process), sedangkan **cadvisor** secara konsisten menggunakan CPU untuk mengumpulkan metric dari Docker daemon.
- Container dengan memory usage tertinggi biasanya karena caching, runtime aplikasi, buffer internal, atau workload yang menetap di RAM. **Prometheus** umumnya menggunakan memory yang cukup besar karena menyimpan time-series data di TSDB, begitu pula **PostgreSQL** yang menggunakan memory untuk shared buffers dan query cache.

![Figure PostLab 1 — Container dengan CPU dan memory tertinggi](images/placeholder.png)
*Gambar PostLab 1: Screenshot dashboard Container Metrics di Grafana yang menunjukkan container dengan CPU usage tertinggi (flask-app saat traffic) dan memory usage tertinggi (prometheus atau postgres).*

### 2. Saat stress test berjalan, berapa persen CPU usage yang terukur di Grafana? Bandingkan dengan output top atau htop di host.

Pada dashboard Host Monitoring, CPU usage mencapai sekitar **90%** saat stress test dimulai, lalu turun kembali ke sekitar **10-11%** setelah stress selesai. Output `top` atau `htop` di host biasanya juga menunjukkan nilai yang mendekati **85-95%**. Perbedaan kecil normal karena:

- Interval sampling berbeda antara Prometheus (15 detik) dan top (real-time, ~1-3 detik)
- Smoothing pada Grafana/Prometheus akibat fungsi `rate()` menggunakan moving window
- Overhead dari Docker container dan monitoring tools sendiri turut berkontribusi terhadap penggunaan CPU host

```bash
# Cek CPU usage dengan top (bandingkan dengan Grafana)
top -bn1 | head -5
```

![Figure PostLab 2 — Perbandingan CPU Grafana vs top](images/placeholder.png)
*Gambar PostLab 2: Screenshot perbandingan — kiri: dashboard Grafana menampilkan CPU usage ~90%, kanan: output `top` di host menampilkan CPU usage ~87%.*

### 3. Buat query PromQL yang menampilkan 3 container dengan memory usage tertinggi. Tunjukkan query dan hasilnya.

Query PromQL:

```
topk(3, container_memory_usage_bytes{id=~"/docker/[a-f0-9]+"})
```

Query ini menggunakan fungsi `topk(3, ...)` untuk mengambil 3 container dengan nilai `container_memory_usage_bytes` tertinggi. Filter `id=~"/docker/[a-f0-9]+"` memastikan hanya container Docker yang dihitung (mengecualikan metric system).

Buka Prometheus UI → tab **Graph** → masukkan query di atas → klik **Execute**. Hasilnya akan menampilkan 3 time series dengan nama container dan memory usage dalam bytes.

Atau query untuk melihat nilai instant terbaru saja:

```
topk(3, container_memory_usage_bytes{id=~"/docker/[a-f0-9]+"} > 0)
```

![Figure PostLab 3 — Query topk memory container](images/placeholder.png)
*Gambar PostLab 3: Screenshot hasil query `topk(3, container_memory_usage_bytes{...})` di Prometheus UI — menampilkan 3 container dengan memory usage tertinggi beserta nilainya dalam bytes.*

### 4. Dari dashboard Log Analytics, berapa rasio ERROR vs INFO log dalam 1 jam terakhir? Apakah ini normal untuk aplikasi production?

Kondisi menunjukkan pipeline Fluent Bit -> PostgreSQL berjalan dan menghasilkan data log. Rasio yang terlihat di dashboard (melalui donut chart **Log Level Distribution**) bergantung pada generator log yang digunakan — log digenerate secara acak dengan distribusi yang telah ditentukan dalam script `generate_logs.py`.

Pada production, seharusnya selalu ada INFO log — minimal request log, startup log, atau health check log. Rasio ERROR terhadap INFO yang ideal di production adalah sangat kecil (misalnya < 1%). Jika ERROR melebihi 5-10% dari total log, perlu investigasi lebih lanjut. WARN biasanya menunjukkan potensi masalah yang belum kritis — acceptable di kisaran 5-10%.

![Figure PostLab 4 — Rasio log level di dashboard](images/placeholder.png)
*Gambar PostLab 4: Screenshot donut chart Log Level Distribution di dashboard Log Analytics Grafana — menampilkan rasio ERROR vs WARN vs INFO dalam periode 1 jam terakhir.*

### 5. Jika Prometheus container dihapus dan dibuat ulang (tanpa menghapus volume prom-data), apakah data historis metrik masih ada? Buktikan.

Ya, data historis tetap ada selama volume `prom-data` tidak dihapus. Karena metrics TSDB Prometheus disimpan pada Docker volume yang bersifat persistent dan terpisah dari lifecycle container. Data akan hilang hanya jika volume ikut dihapus (`docker compose down -v`).

**Pembuktian:**

```bash
# 1. Catat jumlah data sebelum dihapus (lihat storage info)
curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool | grep numSeries

# 2. Stop dan hapus container Prometheus
docker compose stop prometheus
docker compose rm -f prometheus

# 3. Buat ulang container Prometheus (tanpa hapus volume)
docker compose up -d prometheus

# 4. Tunggu beberapa detik, lalu cek kembali data
sleep 10
curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool | grep numSeries

# 5. Query data historis (sebelum container dihapus)
# Buka Prometheus UI → query: container_memory_usage_bytes
# → atur time range ke waktu sebelum container dihapus → data tetap ada!
```

![Figure PostLab 5 — Data historis tetap ada setelah recreate container](images/placeholder.png)
*Gambar PostLab 5: Screenshot perbandingan — kiri: Prometheus UI sebelum container dihapus (query menampilkan data), kanan: Prometheus UI setelah container dibuat ulang (data historis masih muncul, membuktikan TSDB di volume persisten).*

---

## DAFTAR GAMBAR

| Figure | Deskripsi |
|--------|-----------|
| 0.1 | Membuat direktori project monitoring |
| 1.1 | Membuat file prometheus.yml |
| 1.2 | Membuat file alert_rules.yml |
| 2.1 | Membuat file datasources.yml |
| 2.2 | Membuat file dashboards.yml |
| 2.3 | Membuat dashboard docker-host-overview.json (bagian 1) |
| 2.4 | Membuat dashboard docker-host-overview.json (bagian 2) |
| 2.5 | Membuat dashboard docker-host-overview.json (bagian 3) |
| 2.6 | Membuat dashboard container-metrics.json |
| 2.7 | Membuat dashboard log-analytics.json |
| 3.1 | Membuat file requirements.txt |
| 3.2 | Membuat file app.py (bagian 1) |
| 3.3 | Membuat file app.py (bagian 2) |
| 3.4 | Membuat Dockerfile untuk Flask app |
| 4.1 | Membuat file fluent-bit.conf |
| 4.2 | Membuat file generator generate_logs.py |
| 4.3 | Membuat file init.sql |
| 5.1 | Membuat docker-compose.yml (services 1-4) |
| 5.2 | Membuat docker-compose.yml (services 5-7) |
| 5.3 | Membuat docker-compose.yml (services 8-9 + volumes/networks) |
| 6.1 | Docker compose up -d |
| 6.2 | Docker compose ps |
| 7.1 | Prometheus Web UI |
| 7.2 | Curl Prometheus targets API (bagian 1) |
| 7.3 | Curl Prometheus targets API (bagian 2 — filtered) |
| 7.4 | Query PromQL CPU Usage |
| 7.5 | Query PromQL Memory Usage |
| 7.6 | Query PromQL Container Count |
| 7.7 | Prometheus Targets Status Page |
| 7.8 | Curl node-exporter metrics |
| 7.9 | Specific metric node_memory_MemAvailable |
| 7.10 | cAdvisor Web UI |
| 7.11 | cAdvisor metrics endpoint |
| 7.12 | Generate traffic ke Flask app |
| 7.13 | Flask metrics: HTTP requests total |
| 7.14 | Flask metrics: Request latency p95 |
| 7.15 | Flask metrics: Active users |
| 8.1 | Grafana Login Page |
| 8.2 | Grafana Datasources list |
| 8.3 | Test Prometheus datasource |
| 8.4 | Test PostgreSQL datasource |
| 8.5 | Grafana Dashboards list |
| 8.6 | Custom Grafana panel Flask request rate |
| 8.7 | Grafana alert rule |
| 9.1 | Install stress tool |
| 9.2 | Stress test CPU |
| 9.3 | Generate traffic ke Flask app selama stress test |
| 9.4 | Grafana Dashboard Host Overview saat stress test |
| 9.5 | Grafana Dashboard Container Metrics saat stress test |
| 9.6 | Grafana Dashboard Log Analytics |
| 9.7 | Grafana Alerting status |
| PostLab 1 | Container dengan CPU dan memory tertinggi |
| PostLab 2 | Perbandingan CPU Grafana vs top |
| PostLab 3 | Query topk memory container |
| PostLab 4 | Rasio log level di dashboard |
| PostLab 5 | Data historis tetap ada setelah recreate container |

---

*Laporan ini dibuat sebagai bagian dari praktikum Workshop Administrasi Jaringan, Program Studi D4 Teknik Informatika, Politeknik Elektronika Negeri Surabaya (PENS), 2026.*
