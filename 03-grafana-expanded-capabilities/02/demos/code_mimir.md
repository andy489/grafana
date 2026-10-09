# Mimir demo
https://grafana.com/docs/mimir/latest/get-started/

docker-compose.yaml
====
services:
  prometheus:
    image: prom/prometheus:v2.50.0
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
    container_name: "prometheus"

  alloy:
    image: grafana/alloy:latest
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
    container_name: "alloy"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - grafanet
    container_name: "grafana"

  mimir:
    image: grafana/mimir:latest
    volumes:
      - ./mimir.yaml:/etc/mimir/mimir.yaml
    command:
      - "--config.file=/etc/mimir/mimir.yaml"
    ports:
      - "9009:9009"
    networks:
      - grafanet
    container_name: "mimir"

networks:
  grafanet:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
=============
# demo.yaml - Mimir config file:
# Do not use this configuration in production.
# It is for demonstration purposes only.
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
========
# prometheus.yaml
remote_write:
  - url: http://mimir:9009/api/v1/push

scrape_configs:
  - job_name: prometheus
    honor_labels: true
    static_configs:
      - targets: ["localhost:9090"]
=========
# alloy.yaml
prometheus.exporter.self "self_metrics" {
}

prometheus.scrape "self_scrape" {
  targets = [{"__address__" = "localhost:12345"}]
  forward_to = [prometheus.remote_write.mimir.receiver]
  job_name = "self_scrape"
}
  logging {
    level = "debug"
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir:9009/api/v1/push"
  }
}

======
# confirm Prometheus is delivering:
http://localhost:9090
	"Status" > "Targets"
	"Graph" tab. | Enter up{job="prometheus"} and hit "Execute" [look for 1 on the right]
create datasource: http://mimir:9009/prometheus
# Queries:
{job="self_scrape"}
time() - process_start_time_seconds{job="prometheus"}
	confirm prometheus
prometheus_scrape_duration_seconds{job="self_scrape"}
	confirm Alloy
