# Monitoring & Observability Reference

## Table of Contents
1. The Three Pillars
2. Prometheus + Grafana Setup
3. Logging with Loki
4. Alerting Rules
5. File Structure

---

## 1. The Three Pillars of Observability

```
Metrics  → Numbers over time (CPU %, request rate, error rate)
Logs     → Text events (errors, access logs, audit trails)
Traces   → Request journey across services (latency per service)
```

Standard stack:
- **Metrics**: Prometheus (collect) + Grafana (visualize)
- **Logs**: Loki (store) + Grafana (query)
- **Traces**: Jaeger or Tempo + Grafana

---

## 2. Prometheus Configuration

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s       # Collect metrics every 15 seconds
  evaluation_interval: 15s   # Evaluate alert rules every 15 seconds

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'my-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: /metrics     # Your app must expose this endpoint

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

---

## 3. Alerting Rules

```yaml
# monitoring/prometheus/alerts/app.yml
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
```

---

## 4. Docker Compose — Full Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3001:3000"
    depends_on:
      - prometheus

  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki:/etc/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log
      - ./monitoring/promtail:/etc/promtail
    command: -config.file=/etc/promtail/config.yml

volumes:
  prometheus_data:
  grafana_data:
```

---

## 5. File Structure

```
monitoring/
├── prometheus/
│   ├── prometheus.yml          # Scrape config
│   └── alerts/
│       ├── app.yml             # App-specific alerts
│       └── infrastructure.yml  # Node/pod alerts
├── grafana/
│   └── dashboards/
│       ├── app-dashboard.json  # Exported Grafana dashboard
│       └── k8s-dashboard.json
├── loki/
│   └── loki-config.yml
└── promtail/
    └── config.yml              # Log shipper config
```