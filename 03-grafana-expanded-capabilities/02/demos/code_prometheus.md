# docker-compose.yml

services:
  prometheus:
    image: prom/prometheus:v2.50.0
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
    container_name: "prometheus"
  alloy:
    image: grafana/alloy:latest
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
    container_name: "alloy"
  loki:
    image: grafana/loki:latest
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
---

### Config Files
1. **prometheus.yml**:
   ```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'test_noise'
    static_configs:
      - targets: ['localhost:9999']
    scrape_interval: 5s
   ```

2. **alloy-config.alloy**:
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

3. **loki-config.yaml**:
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


4. **Verify**:
   - Prometheus: `http://localhost:9090/targets` (see `test_noise` DOWN).
   - Loki: `curl http://localhost:3100/ready`
   	Ingester not ready: waiting for 15s after being ready
   - Grafana: `http://localhost:3000` (login: admin/admin if not anonymous).
   - Add Loki data source: `http://loki:3100`, query `{job="prometheus"}`.

# Add more noise (if necessary):
while true; do curl http://localhost:9090/api/v1/query?query=up; sleep 1; done
nohup bash -c 'while true; do docker logs prometheus > ./prometheus.log 2>&1; sleep 10; done' &
---


