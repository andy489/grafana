# Grafana Expanded Capabilities

**Instructor:** David Clinton — Data Analyst | Linux System Administrator  
bootstrap-it.com | linkedin.com/in/dbclinton

Slide notes and structured reference material extracted from the course PDFs, organized as a Jupyter notebook.

## Modules

| # | Folder | Title | Topics |
|---|--------|-------|--------|
| 01 | [`01-advanced-storage-tracing-rules/`](./01-advanced-storage-tracing-rules/) | Introduction to the Grafana Product Ecosystem | Loki, Mimir, Tempo, Alloy — the LGTM stack and external tool integration |
| 02 | [`02-integrations-and-automation/`](./02-integrations-and-automation/) | Advanced Grafana Integrations and Automation | Custom dashboards, RBAC roles, RBAC YAML provisioning, HTTP API, service accounts |

## Files

```
03-grafana-expanded-capabilities/
├── 01-advanced-storage-tracing-rules/
│   └── demos/
│       ├── mimir-stack-demo.md           # Mimir + Alloy + Prometheus + Grafana compose
│       └── loki-prometheus-stack-demo.md # Loki + Alloy + Prometheus + Grafana compose
├── 02-integrations-and-automation/
│   └── full-monitoring-stack-demo.md     # Prometheus + Blackbox + Alertmanager + Nginx + Grafana compose
└── expanded_capabilities_notes.ipynb     # All course notes in one notebook
```

## Usage

```bash
jupyter notebook expanded_capabilities_notes.ipynb
```

## Key Concepts

**The LGTM Stack**

| Product | Signal | Key Capability |
|---------|--------|----------------|
| Loki | Logs | Label-based indexing, LogQL queries |
| Grafana Mimir | Metrics | Scalable Prometheus, multi-tenant, S3 backend |
| Tempo | Traces | TraceQL, exemplars → trace links, service graph |
| Alloy | All signals | Replaces Promtail; ships metrics/logs/traces in one agent |

**RBAC Actions (common)**

```
dashboards:read / edit / create / delete
datasources:query
explore:read
folders:read
users:read
```

**Mimir S3 storage config key:** `blocks_storage.backend: s3` with `bucket_name`, `region`, `access_key_id`, `secret_access_key`

## Reference Links

| Resource | URL |
|----------|-----|
| Grafana Loki | https://grafana.com/docs/loki/latest/ |
| Grafana Mimir | https://grafana.com/docs/mimir/latest/ |
| Grafana Tempo | https://grafana.com/docs/tempo/latest/ |
| Grafana Alloy | https://grafana.com/docs/alloy/latest/ |
| RBAC provisioning | https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/access-control/rbac-provisioning/ |
| Grafana HTTP API | https://grafana.com/docs/grafana/latest/developers/http_api/ |
| Service accounts | https://grafana.com/docs/grafana/latest/administration/service-accounts/ |
