# Full Monitoring Stack Demo

> **Goal:** Production-style local monitoring — Prometheus scrapes metrics and fires alerts via Alertmanager, Blackbox Exporter probes HTTP endpoints synthetically, Nginx acts as the target, and Grafana visualises everything with a pre-provisioned data source.

---

## Architecture

```
                  ┌──────────────────────────────────────┐
                  │           Docker network              │
                  │                                       │
 ┌─────────────┐  │  ┌──────────────┐  probe /probe      │
 │   Grafana   │  │  │  Prometheus  │ ──────────────────► │ Blackbox
 │   :3000     │◄─┼──│    :9090     │                     │ Exporter :9115
 │ (provisioned│  │  │              │ ◄────────────────── │
 │  datasource)│  │  │  alert rules │     metrics         │
 └─────────────┘  │  │              │                     └──────────┐
                  │  │              │ ──── fire alerts ──► Alertmgr  │
                  │  └──────────────┘                     :9093      │
                  │                                                   │
                  │  ┌──────────────┐  ◄── probed via HTTP           │
                  │  │    Nginx     │                                 │
                  │  │    :80       │                                 │
                  │  └──────────────┘                                 │
                  └───────────────────────────────────────────────────┘
```

---

## Stack

| Service | Image | Port | Role |
|---------|-------|------|------|
| Prometheus | `prom/prometheus:latest` | `9090` | Metrics scraper + alert evaluator |
| Alertmanager | `prom/alertmanager:latest` | `9093` | Alert routing & delivery |
| Blackbox Exporter | `prom/blackbox-exporter:latest` | `9115` | Synthetic HTTP/TCP probing |
| Nginx | `nginx:latest` | `80` | Monitored target service |
| Grafana | `grafana/grafana:latest` | `3000` | Visualisation UI (auto-provisioned) |

---

## File Layout

```
.
├── docker-compose.yml
├── prometheus.yml
├── blackbox.yml
├── alertmanager.yml
├── alert.rules.yml
└── grafana-provisioning/
    └── datasources/
        └── datasource.yml
```

---

## Config Files

### `docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: unless-stopped
    networks:
      - monitoring

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox.yml
    command:
      - '--config.file=/etc/blackbox.yml'
    restart: unless-stopped
    networks:
      - monitoring

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana-provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  monitoring:
    driver: bridge
```

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alert.rules.yml"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: blackbox
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - 'http://nginx'          # internal Nginx service
          - 'https://google.com'    # external site (synthetic monitoring)
    relabel_configs:
      # Pass the original target as the probe ?target= query param
      - source_labels: [__address__]
        target_label: __param_target
      # Preserve the target URL as the instance label
      - source_labels: [__param_target]
        target_label: instance
      # Redirect scraping to the Blackbox Exporter
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### `blackbox.yml`

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      headers:
        User-Agent: "Mozilla/5.0 (compatible; Blackbox/1.0)"
```

### `alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: log

receivers:
  - name: log
    webhook_configs:
      # Replace with a real endpoint (Slack, PagerDuty, etc.) in production
      - url: 'http://localhost:9093'
```

### `alert.rules.yml`

```yaml
groups:
  - name: synthetic_monitoring_alerts
    rules:
      - alert: TargetDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"
          description: >-
            {{ $labels.instance }} has failed its HTTP probe
            for more than 1 minute.
```

### `grafana-provisioning/datasources/datasource.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
```

> **How provisioning works:** Grafana reads `./grafana-provisioning/` at startup via the volume mount in `docker-compose.yml`. The Prometheus data source is created automatically — no manual setup in the UI required.

---

## Verify

| Check | URL / Command | Expected |
|-------|---------------|----------|
| Prometheus targets | `http://localhost:9090/targets` | `blackbox` job shows two targets |
| Manual probe test | `http://localhost:9115/probe?target=http://nginx&module=http_2xx` | `probe_success 1` |
| Alertmanager UI | `http://localhost:9093` | No active alerts (Nginx is up) |
| Grafana UI | `http://localhost:3000` | Login: `admin` / `admin` |
| Data source | Grafana → **Connections → Data sources** | Prometheus pre-configured |

### Sample PromQL Queries

Run in **Grafana → Explore** or **Prometheus → Graph**:

```promql
# HTTP probe success (1 = up, 0 = down) per target
probe_success

# Total HTTP roundtrip duration in seconds
probe_duration_seconds

# Detailed phase breakdown (DNS, TCP connect, TLS, processing)
probe_dns_lookup_time_seconds
probe_http_duration_seconds{phase="connect"}
probe_http_duration_seconds{phase="tls"}

# Prometheus own uptime in seconds
time() - process_start_time_seconds{job="prometheus"}
```
