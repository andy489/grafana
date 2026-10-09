# Loki + Prometheus Stack Demo

> **Goal:** Minimal local stack for log aggregation — Prometheus scrapes metrics, Alloy tails a log file and ships entries to Loki, Grafana visualises both.
>
> **Reference:** <https://grafana.com/docs/loki/latest/setup/install/docker/>

---

## Architecture

```
┌─────────────┐      scrape      ┌──────────────┐
│  Prometheus │ ◄──────────────  │  (self-scrape)|
│  :9090      │                  └──────────────┘
└─────────────┘
                                 ┌──────────────┐
┌─────────────┐   tail logs      │ prometheus   │
│    Alloy    │ ◄─────────────── │ .log (file)  │
│             │                  └──────────────┘
│             │   push logs
│             │ ────────────────► ┌─────────────┐
└─────────────┘                   │    Loki     │
                                  │    :3100    │
┌─────────────┐   query (LogQL)   └─────────────┘
│   Grafana   │ ──────────────────────────────►
│   :3000     │   query (PromQL)  ┌─────────────┐
│             │ ──────────────────► Prometheus  │
└─────────────┘                   └─────────────┘
```

---

## Stack

| Service | Image | Port | Role |
|---------|-------|------|------|
| Prometheus | `prom/prometheus:v2.50.0` | `9090` | Metrics scraper |
| Alloy | `grafana/alloy:latest` | — | Log collector / shipper |
| Loki | `grafana/loki:latest` | `3100` | Log storage & query |
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
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--log.level=debug"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    networks:
      - monitoring

  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    volumes:
      - ./alloy-config.alloy:/etc/alloy/alloy-config.alloy
      - ./prometheus.log:/prometheus/prometheus.log:ro
    command:
      - "run"
      - "/etc/alloy/alloy-config.alloy"
    depends_on:
      - loki
      - prometheus
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml
      - loki-data:/loki
    command:
      - "-config.file=/etc/loki/loki-config.yaml"
    ports:
      - "3100:3100"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  loki-data:
  grafana-data:
```

### `prometheus.yml`

```yaml
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: test_noise
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9999']   # intentionally unreachable → DOWN state
```

### `alloy-config.alloy`

```alloy
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

loki.source.file "prometheus_logs" {
  targets = [
    {__path__ = "/prometheus/prometheus.log", job = "prometheus"},
  ]
  forward_to = [loki.write.local.receiver]
}
```

### `loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

---

## Verify

| Check | Command / URL | Expected |
|-------|---------------|----------|
| Prometheus targets | `http://localhost:9090/targets` | `test_noise` → **DOWN**, `prometheus` → **UP** |
| Loki ready | `curl http://localhost:3100/ready` | `ready` |
| Grafana UI | `http://localhost:3000` | Login: `admin` / `admin` |
| Add Loki data source | URL `http://loki:3100` | Connection test passes |
| Sample LogQL query | `{job="prometheus"}` | Log lines appear in Explore |

---

## Generate Extra Log Noise (optional)

> Run these after `docker compose up -d` to produce continuous activity for Alloy to tail and ship to Loki.

```bash
# Continuously poll Prometheus metrics endpoint
while true; do
  curl -s "http://localhost:9090/api/v1/query?query=up"
  sleep 1
done

# Refresh the log file Alloy is tailing every 10 s
nohup bash -c \
  'while true; do docker logs prometheus > ./prometheus.log 2>&1; sleep 10; done' &
```
