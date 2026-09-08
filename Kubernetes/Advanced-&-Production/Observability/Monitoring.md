---
title: Kubernetes Monitoring — Prometheus Stack
tags:
  - kubernetes
  - observability
  - prometheus
  - grafana
  - alertmanager
date: 2026-04-26
---

# Kubernetes Monitoring

## Prometheus Operator Stack

Prometheus Operator quản lý Prometheus, Alertmanager, và exporters như K8s custom resources — không cần edit config files thủ công.

### Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=5Gi
```

Bao gồm: Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, prometheus-adapter.

### Components

```
┌─────────────────────────────────────────────────────────────┐
│  kube-prometheus-stack                                       │
│                                                             │
│  Prometheus Operator                                        │
│    ├─ Prometheus (scrape metrics)                           │
│    ├─ Alertmanager (route/silence alerts)                   │
│    ├─ PrometheusRule CRDs (alert rules)                     │
│    ├─ ServiceMonitor CRDs (scrape targets)                  │
│    └─ PodMonitor CRDs (pod-level scrape)                   │
│                                                             │
│  Exporters                                                  │
│    ├─ node-exporter (node CPU/mem/disk metrics)             │
│    ├─ kube-state-metrics (deployment/pod state)             │
│    └─ cAdvisor (container metrics, built-in kubelet)        │
│                                                             │
│  Grafana (visualization)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## ServiceMonitor & PodMonitor

Thay vì edit Prometheus config, dùng ServiceMonitor/PodMonitor CRDs để auto-discover scrape targets.

### ServiceMonitor

```yaml
# Cho app expose /metrics qua Service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-metrics
  namespace: production
  labels:
    release: kube-prometheus-stack    # phải match Prometheus selector
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: myapp
      metrics: enabled               # label trên Service
  endpoints:
    - port: metrics                  # port name trong Service
      interval: 30s
      path: /metrics
      # Basic auth nếu cần
      # basicAuth:
      #   username:
      #     name: metrics-auth
      #     key: username
      #   password:
      #     name: metrics-auth
      #     key: password
```

```yaml
# Service phải có label matching ServiceMonitor selector
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    metrics: enabled   # ← label này
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics    # ← port name này phải match ServiceMonitor
      port: 9090
      targetPort: 9090
  selector:
    app: myapp
```

### PodMonitor

```yaml
# Scrape trực tiếp từ pods (không qua Service)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pods
  namespace: production
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames: [production]
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
      # Scrape annotations nếu app expose via annotations
      # relabelings:
      #   - sourceLabels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      #     action: replace
      #     targetLabel: __address__
```

### Verify ServiceMonitor

```bash
# Check Prometheus đã discover targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Truy cập http://localhost:9090/targets
# → Tìm job "serviceMonitor/production/myapp-metrics"

# Nếu không thấy: check Prometheus selector config
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
# → phải match labels trên ServiceMonitor
```

---

## PrometheusRule — Alert Rules

```yaml
# PrometheusRule — define alerting và recording rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: production
  labels:
    release: kube-prometheus-stack   # phải match Prometheus ruleSelector
spec:
  groups:
    - name: myapp.rules
      interval: 30s     # evaluate interval
      rules:

        # Recording rule — pre-compute expensive queries
        - record: job:myapp_request_duration:p99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))
              by (le, job))

        # Alert rules
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{job="myapp",status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="myapp"}[5m]))
            > 0.05
          for: 5m           # phải true trong 5 phút liên tục
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate for MyApp"
            description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
            runbook_url: "https://wiki.internal/runbooks/myapp-errors"

        - alert: MyAppHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))
              by (le))
            > 1.0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High p99 latency for MyApp"
            description: "p99 latency is {{ $value }}s (threshold: 1s)"

        - alert: MyAppPodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: MyAppHighMemory
          expr: |
            container_memory_working_set_bytes{namespace="production", container="myapp"}
            / container_spec_memory_limit_bytes{namespace="production", container="myapp"}
            > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container memory at {{ $value | humanizePercentage }} of limit"
```

---

## Alertmanager Configuration

```yaml
# AlertmanagerConfig CRD (namespace-scoped)
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: production-alerts
  namespace: production
  labels:
    alertmanagerConfig: production   # must match Alertmanager selector
spec:
  route:
    receiver: slack-critical
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    routes:
      - matchers:
          - name: severity
            value: critical
        receiver: pagerduty
        repeatInterval: 1h
      - matchers:
          - name: severity
            value: warning
        receiver: slack-warning

  receivers:
    - name: slack-critical
      slackConfigs:
        - apiURL:
            name: alertmanager-slack
            key: webhook-url
          channel: '#alerts-critical'
          title: '{{ .CommonAnnotations.summary }}'
          text: |
            *Severity:* {{ .CommonLabels.severity }}
            *Namespace:* {{ .CommonLabels.namespace }}
            {{ range .Alerts }}
            *Description:* {{ .Annotations.description }}
            *Runbook:* {{ .Annotations.runbook_url }}
            {{ end }}

    - name: pagerduty
      pagerdutyConfigs:
        - routingKey:
            name: alertmanager-pagerduty
            key: routing-key
          description: '{{ .CommonAnnotations.summary }}'

    - name: slack-warning
      slackConfigs:
        - apiURL:
            name: alertmanager-slack
            key: webhook-url
          channel: '#alerts-warning'

  inhibitRules:
    # Suppress warning nếu critical đang fire cho cùng alertname+namespace
    - sourceMatchers:
        - name: severity
          value: critical
      targetMatchers:
        - name: severity
          value: warning
      equal: ['alertname', 'namespace']
```

---

## SLI / SLO

### Concepts

```
SLI (Service Level Indicator): metric đo quality của service
  Ví dụ: request success rate, p99 latency, availability

SLO (Service Level Objective): target cho SLI
  Ví dụ: 99.9% requests thành công trong 30 ngày

SLA (Service Level Agreement): business contract về SLO
  Thường = SLO - safety margin (SLO 99.9% → SLA 99.5%)

Error Budget = 1 - SLO
  SLO 99.9% → Error Budget = 0.1% = 43.8 phút/tháng
  Nếu exceed error budget → freeze deployments, focus reliability
```

### SLO Recording Rules

```yaml
# PrometheusRule cho SLO tracking
spec:
  groups:
    - name: slo.rules
      rules:
        # SLI: request success rate
        - record: slo:myapp_requests:success_rate5m
          expr: |
            sum(rate(http_requests_total{job="myapp",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="myapp"}[5m]))

        # SLO: 99.9% success rate
        - alert: SLOViolation
          expr: |
            (
              1 - (
                sum_over_time(slo:myapp_requests:success_rate5m[30d])
                / count_over_time(slo:myapp_requests:success_rate5m[30d])
              )
            ) > 0.001   # > 0.1% error = SLO violated
          labels:
            severity: critical
          annotations:
            summary: "SLO violation: 30-day error rate exceeds 0.1%"

        # Error budget burn rate alert (burn through budget too fast)
        - alert: ErrorBudgetBurnHigh
          expr: |
            (
              sum(rate(http_requests_total{job="myapp",status=~"5.."}[1h]))
              /
              sum(rate(http_requests_total{job="myapp"}[1h]))
            ) > (14.4 * 0.001)   # 14.4x burn rate for 1h window
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Error budget burn rate too high (1h window)"
```

### Pyrra / Sloth (SLO management tools)

```yaml
# Sloth — generate SLO rules từ high-level spec
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: myapp-slo
  namespace: production
spec:
  service: myapp
  labels:
    team: backend
  slos:
    - name: requests-availability
      objective: 99.9
      description: "99.9% of requests must succeed"
      sli:
        events:
          errorQuery: |
            sum(rate(http_requests_total{job="myapp",status=~"5.."}[{{.window}}]))
          totalQuery: |
            sum(rate(http_requests_total{job="myapp"}[{{.window}}]))
      alerting:
        name: MyAppAvailability
        pageAlert:
          labels:
            severity: critical
        ticketAlert:
          labels:
            severity: warning
```

---

## Key Metrics cho K8s

### USE Method (Resources)
- **Utilization** — CPU usage (%), memory usage (%)
- **Saturation** — CPU throttle %, memory pressure, disk I/O wait
- **Errors** — OOMKill count, restart count, failed requests

### RED Method (Services)
- **Rate** — requests/second
- **Errors** — error rate (%)
- **Duration** — p50/p95/p99 latency

```bash
# Useful PromQL queries

# CPU throttling (container bị CPU limit)
sum(rate(container_cpu_cfs_throttled_seconds_total{namespace="production"}[5m]))
by (pod, container)
/
sum(rate(container_cpu_cfs_periods_total{namespace="production"}[5m]))
by (pod, container)

# Memory pressure
container_memory_working_set_bytes{namespace="production"}
/ container_spec_memory_limit_bytes{namespace="production"} * 100

# Pod restart rate
increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])

# OOMKilled containers (last 24h)
kube_pod_container_status_last_terminated_reason{reason="OOMKilled",namespace="production"}

# Node disk pressure
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes
```

---

## Gotchas

- **ServiceMonitor label matching**: Prometheus Operator có `serviceMonitorSelector` — nếu ServiceMonitor không có matching labels, Prometheus sẽ ignore nó silently. Luôn check `prometheus.prometheusSpec.serviceMonitorSelector` vs labels trên ServiceMonitor.
- **High cardinality metrics**: Labels với high cardinality (user ID, session ID, request ID) → rất nhiều unique time series → Prometheus OOM. Chỉ dùng labels có bounded cardinality.
- **`for` duration trong alerts**: Alert với `for: 0m` fire ngay lập tức → noise. Dùng ít nhất `for: 5m` để tránh flapping alerts. `for` theo use case: infra issues = 5-15m, perf degradation = 10-30m.
- **Recording rules cho heavy queries**: PromQL queries phức tạp (range queries, many time series) tốn CPU mỗi lần evaluate. Dùng recording rules để pre-compute, store, và query kết quả pre-computed.
- **Alertmanager inhibition**: Nếu configure inhibit rules sai → alerts bị silence không mong muốn. Test inhibit rules với silence API trước khi deploy.
- **Grafana dashboard as code**: Không edit dashboards trực tiếp trong UI — chúng bị overwrite khi helm upgrade. Dùng ConfigMap-based dashboards hoặc grafonnet.
