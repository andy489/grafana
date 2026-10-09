# Grafana Concepts and Basic Configuration

**Instructor:** Craig Golightly, Senior Software Consultant

Slide notes and structured reference material extracted from the course PDFs, organized as a Jupyter notebook.

## Modules

| # | Title | Topics |
|---|-------|--------|
| 01 | What Is Grafana? | Grafana overview, deployment options, monitoring vs. observability, SRE principles, alerting, related OSS projects |
| 02 | Installing and Configuring Grafana | Grafana Cloud, Linux install, Windows/Mac/container options, UI navigation, organizations, teams, roles |
| 03 | Adding a Data Source to Grafana | Data source concepts, aggregating multiple sources, Prometheus setup, pre-built dashboards |

## Files

```
grafana-concepts-basic-config/
└── grafana_concepts_notes.ipynb   # All course notes in one notebook
```

## Usage

```bash
jupyter notebook grafana_concepts_notes.ipynb
```

## Key Concepts

**Monitoring vs. Observability**

| Monitoring | Observability |
|-----------|---------------|
| Reactive — detect known issues | Proactive — ask open-ended questions |
| Is the service up? | Why did the CPU spike? |
| Pre-established metrics | Holistic system view |

**Deployment Options:** Grafana Cloud · Self-host · Container · Managed · Enterprise

**Roles:** Viewer · Editor · Admin

## Reference Links

| Resource | URL |
|----------|-----|
| Install docs (Debian) | https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/ |
| Start/restart Grafana | https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/ |
| Data sources | https://grafana.com/docs/grafana/latest/datasources |
| Prometheus | https://prometheus.io |
| Node Exporter | https://prometheus.io/docs/guides/node-exporter/ |
| Open Telemetry | https://opentelemetry.io |
| Graphite | https://graphiteapp.org |
