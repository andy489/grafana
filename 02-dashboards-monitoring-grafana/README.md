# Dashboards and Monitoring with Grafana

Slide notes and structured reference material extracted from the course PDFs, organized as a Jupyter notebook.

## Modules

| # | Title | Topics |
|---|-------|--------|
| 01 | Understanding Advanced Dashboard Design | Adding TLS encryption to a Grafana server, identifying monitoring events |
| 02 | Exploring Data Visualization Techniques | CloudWatch provisioner YAML, defining manual queries (PromQL / CloudWatch), designing effective dashboards, panel types |
| 03 | Building a Complete Grafana Team Solution | Users, teams, and org roles; unified alerting, alert rules, contact points, notification policies, silences |

## Files

```
02-dashboards-monitoring-grafana/
└── dashboards_monitoring_notes.ipynb   # All course notes in one notebook
```

## Usage

```bash
jupyter notebook dashboards_monitoring_notes.ipynb
```

## Key Concepts

**Dashboard Design Principles**

| Principle | Guidance |
|-----------|----------|
| Audience first | Design for the viewer — ops vs. exec vs. dev |
| One story per dashboard | Avoid cramming unrelated metrics |
| Right panel type | Time series / Stat / Gauge / Table / Logs |
| Template variables | Use `$instance`, `$job` for reusable dashboards |
| Annotations | Mark deployments and incidents on graphs |

**Org Roles:** Viewer · Editor · Admin · Server Admin

**Alerting Components:** Alert rule → Contact point → Notification policy → Silence

## Reference Links

| Resource | URL |
|----------|-----|
| Grafana TLS/security | https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/ |
| Provisioning data sources | https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources |
| CloudWatch data source | https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/ |
| Unified alerting | https://grafana.com/docs/grafana/latest/alerting/ |
| Contact points | https://grafana.com/docs/grafana/latest/alerting/contact-points/ |
| Dashboard best practices | https://grafana.com/docs/grafana/latest/best-practices/best-practices-for-creating-dashboards/ |
| Alerting API | https://grafana.com/docs/grafana/latest/developers/http_api/alerting_provisioning/ |
