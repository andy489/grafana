# Mimir Stack Demo

> **Goal:** Minimal local stack for scalable metrics storage — Prometheus and Alloy both remote-write to Mimir; Grafana queries Mimir via the Prometheus-compatible API.
>
> **Reference:** <https://grafana.com/docs/mimir/latest/get-started/>

> [!WARNING]
> The Mimir configuration below uses in-memory state and a local filesystem. It is for **demonstration only** — do not use in production.

---

## Architecture

```
┌─────────────┐  remote_write   ┌─────────────┐
│  Prometheus │ ───────────────► │    Mimir    │
│  :9090      │                  │    :9009    │
└─────────────┘                  │             │
                                 │  blocks on  │
┌─────────────┐  remote_write   │  filesystem │
│    Alloy    │ ───────────────► │             │
│   :12345    │                  └──────┬──────┘
└─────────────┘                         │
                                        │ /prometheus API
                              ┌─────────▼──────┐
                              │    Grafana     │
                              │    :3000       │
                              └────────────────┘
```

---

## Stack

| Service | Image | Port | Role |
|---------|-------|------|------|
| Prometheus | `prom/prometheus:v2.50.0` | `9090` | Scraper + remote-write source |
| Alloy | `grafana/alloy:latest` | `12345` | Self-metrics + remote-write source |
| Mimir | `grafana/mimir:latest` | `9009` | Scalable metrics store |
| Grafana | `grafana/grafana:latest` | `3000` | Visualisation UI |

---

## Config Files

### `docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.50.0
    container_name: prometheus
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--log.level=debug"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    networks:
      - grafanet

  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    volumes:
      - ./alloy.yaml:/etc/alloy/alloy-config.alloy
    command:
      - "run"
      - "/etc/alloy/alloy-config.alloy"
    depends_on:
      - prometheus
    ports:
      - "12345:12345"
    networks:
      - grafanet

  mimir:
    image: grafana/mimir:latest
    container_name: mimir
    volumes:
      - ./mimir.yaml:/etc/mimir/mimir.yaml
    command:
      - "--config.file=/etc/mimir/mimir.yaml"
    ports:
      - "9009:9009"
    networks:
      - grafanet

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - grafanet

networks:
  grafanet:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
```

### `mimir.yaml`

```yaml
# Demo only — not for production use.
multitenancy_enabled: false

blocks_storage:
  backend: filesystem
  bucket_store:
    sync_dir: /tmp/mimir/tsdb-sync
  filesystem:
    dir: /tmp/mimir/data/tsdb
  tsdb:
    dir: /tmp/mimir/tsdb

compactor:
  data_dir: /tmp/mimir/compactor
  sharding_ring:
    kvstore:
      store: memberlist

distributor:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist

ingester:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist
    replication_factor: 1

ruler_storage:
  backend: filesystem
  filesystem:
    dir: /tmp/mimir/rules

server:
  http_listen_port: 9009
  log_level: error

store_gateway:
  sharding_ring:
    replication_factor: 1
```

### `prometheus.yaml`

```yaml
remote_write:
  - url: http://mimir:9009/api/v1/push

scrape_configs:
  - job_name: prometheus
    honor_labels: true
    static_configs:
      - targets: ["localhost:9090"]
```

### `alloy.yaml`

```alloy
prometheus.exporter.self "self_metrics" {}

prometheus.scrape "self_scrape" {
  targets    = [{"__address__" = "localhost:12345"}]
  forward_to = [prometheus.remote_write.mimir.receiver]
  job_name   = "self_scrape"
}

logging {
  level = "debug"
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir:9009/api/v1/push"
  }
}
```

---

## Verify

| Check | Steps | Expected |
|-------|-------|----------|
| Prometheus targets | `http://localhost:9090` → **Status → Targets** | `prometheus` job is UP |
| Prometheus delivering | **Graph** → query `up{job="prometheus"}` | Returns `1` |
| Grafana data source | Add **Prometheus**-compatible source, URL `http://mimir:9009/prometheus` | "Data source connected" |

### Sample PromQL Queries

Run these in **Grafana → Explore** or **Prometheus → Graph**:

```promql
# All series arriving from the Alloy self-scrape job
{job="self_scrape"}

# How long Prometheus has been running (seconds)
time() - process_start_time_seconds{job="prometheus"}

# Last scrape duration for the Alloy self-scrape
prometheus_scrape_duration_seconds{job="self_scrape"}
```
