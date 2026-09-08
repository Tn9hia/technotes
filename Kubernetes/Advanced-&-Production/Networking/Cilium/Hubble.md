---
title: Cilium Hubble — Observability
tags:
  - cilium
  - hubble
  - observability
  - networking
date: 2026-04-30
---

# Cilium Hubble — Observability

Hubble là observability platform built-in của Cilium — cung cấp network flow visibility, service map, và metrics mà không cần thêm agent hay sidecar. Mọi data đều đến từ eBPF programs trong kernel.

```
eBPF programs (kernel)
  │  emit flow events vào perf ring buffer
  ▼
Hubble Agent (trong cilium-agent pod, mỗi node)
  │  đọc ring buffer, expose gRPC server (port 4244)
  ▼
Hubble Relay (Deployment)
  │  aggregate flows từ tất cả nodes
  ▼
Hubble CLI / Hubble UI / Prometheus metrics
```

---

## Install & Enable

```yaml
# Helm values
hubble:
  enabled: true
  
  relay:
    enabled: true
    replicas: 2           # HA cho relay
    
  ui:
    enabled: true
    replicas: 1
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
      hosts:
        - hubble.internal
        
  metrics:
    enabled:
      - drop                    # packets bị drop + reason
      - tcp                     # TCP connections (SYN, FIN, RST)
      - flow                    # flow count by source/destination
      - port-distribution       # port usage distribution
      - icmp                    # ICMP flows
      - http                    # HTTP metrics (status codes, latency)
      - "dns:query;ignoreAAAA"  # DNS queries (ignoreAAAA = skip AAAA records)
      - "httpV2:exemplars=true;labelsContext=source_ip,source_namespace,source_workload,destination_ip,destination_namespace,destination_workload,traffic_direction"
      
  tls:
    enabled: true               # TLS cho gRPC connection tới relay
    
  # Ring buffer size (tăng nếu miss events trên high-traffic clusters)
  eventQueueSize: 50000
  eventBufferCapacity: 100000
```

```bash
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz"
tar xzvf hubble-linux-amd64.tar.gz
sudo mv hubble /usr/local/bin/

# Port-forward để dùng Hubble CLI locally
cilium hubble port-forward &
hubble status
```

---

## Hubble CLI — Flow Observation

### Basic Commands

```bash
# Xem flows realtime
hubble observe

# Flows trong namespace cụ thể
hubble observe --namespace production

# Chỉ flows bị drop
hubble observe --verdict DROPPED

# Chỉ flows được allow
hubble observe --verdict FORWARDED

# HTTP L7 flows
hubble observe --protocol http

# DNS flows
hubble observe --protocol dns

# Flows từ/đến pod cụ thể
hubble observe --pod production/frontend-abc
hubble observe --from-pod production/frontend-abc
hubble observe --to-pod production/backend-xyz

# Flows theo service
hubble observe --from-service production/frontend
hubble observe --to-service production/backend

# Flows theo namespace
hubble observe --from-namespace production --to-namespace databases
```

### Output Formats

```bash
# Default: human-readable
hubble observe --namespace production

# JSON (for parsing/alerting)
hubble observe --namespace production -o json | jq '
  select(.verdict == "DROPPED") |
  {
    time: .time,
    src: .source.pod_name,
    dst: .destination.pod_name,
    port: .destination_port,
    reason: .drop_reason_desc
  }'

# Compact JSON
hubble observe -o jsonpb

# Table
hubble observe -o table
```

### Common Investigation Queries

```bash
# Tìm tất cả drops trong 10 phút qua
hubble observe --verdict DROPPED --since 10m

# Service-to-service communication map
hubble observe --namespace production -o json | \
  jq -r '[.source.workload_name, .destination.workload_name] | @csv' | \
  sort | uniq -c | sort -rn

# HTTP 5xx errors
hubble observe --protocol http -o json | \
  jq 'select(.l7.http.code >= 500) | {src:.source.pod_name, url:.l7.http.url, code:.l7.http.code}'

# DNS failures (NXDOMAIN)
hubble observe --protocol dns -o json | \
  jq 'select(.l7.dns.rcode == 3) | {pod:.source.pod_name, query:.l7.dns.query}'

# Top talkers (most flows)
hubble observe --last 10000 -o json | \
  jq -r '.source.pod_name' | sort | uniq -c | sort -rn | head 20

# Policy drops theo pod
hubble observe --verdict DROPPED --last 1000 -o json | \
  jq -r '.destination.pod_name' | sort | uniq -c | sort -rn
```

---

## Hubble UI — Service Map

Hubble UI hiển thị:
- **Service dependency graph**: arrow từ source → destination service
- **Flow rate** per connection (requests/sec)
- **Drop rate** highlighted màu đỏ
- **Namespace filter**: xem từng namespace riêng
- **HTTP metrics**: status code distribution, latency

```bash
# Access UI locally
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# → http://localhost:12000

# Hoặc via Ingress (nếu đã config)
```

---

## Hubble Metrics — Prometheus

### Key Metrics

```bash
# Drop metrics
hubble_drop_total{reason, direction, source, destination}
# reason: POLICY_DENIED, CT_TRUNCATED_OR_INVALID_HEADER, ...

# Flow metrics
hubble_flows_processed_total{subtype, verdict, direction}

# HTTP metrics (cần enable httpV2)
hubble_http_requests_total{source, destination, status_code, method}
hubble_http_request_duration_seconds{source, destination, method}   # histogram

# TCP metrics
hubble_tcp_flags_total{family, flag}    # SYN, FIN, RST counts

# DNS metrics
hubble_dns_queries_total{qtypes, rcode, source}
hubble_dns_responses_total{qtypes, rcode, destination}
```

### Prometheus Alert Rules

```yaml
groups:
  - name: hubble
    rules:
      # High drop rate
      - alert: HubbleHighDropRate
        expr: |
          rate(hubble_drop_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High packet drop rate: {{ $value }}/s"
          description: "Reason: {{ $labels.reason }}, Direction: {{ $labels.direction }}"

      # Policy denied spikes
      - alert: HubblePolicyDenied
        expr: |
          rate(hubble_drop_total{reason="POLICY_DENIED"}[5m]) > 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Policy denying traffic — possible misconfiguration"

      # HTTP 5xx error rate
      - alert: HubbleHTTP5xxHigh
        expr: |
          rate(hubble_http_requests_total{status_code=~"5.."}[5m]) /
          rate(hubble_http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP 5xx error rate > 5% for {{ $labels.destination }}"

      # HTTP latency P99
      - alert: HubbleHTTPLatencyHigh
        expr: |
          histogram_quantile(0.99, rate(hubble_http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP P99 latency > 1s from {{ $labels.source }} to {{ $labels.destination }}"
```

### Grafana Dashboard

```bash
# Import Cilium official Grafana dashboards
# Dashboard IDs từ grafana.com:
# 16611 — Cilium Overview
# 16612 — Hubble L4 Forwarding
# 16613 — Hubble DNS
# 16614 — Hubble HTTP

# Hoặc install qua Cilium Helm với Grafana integration:
helm upgrade cilium cilium/cilium \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true
```

---

## Hubble Relay API

```bash
# Hubble Relay expose gRPC API (port 4245)
# Useful cho custom tooling và CI/CD pipeline checks

# Check relay status
hubble status --server localhost:4245

# Stream flows programmatically (Go/Python)
# API: observer.ObserverClient.GetFlows()

# Health check trong K8s
kubectl exec -n kube-system deploy/hubble-relay -- \
  hubble status --server localhost:4245
```

---

## Troubleshooting với Hubble

```bash
# Scenario 1: Service B không nhận được traffic từ Service A
hubble observe \
  --from-service production/service-a \
  --to-service production/service-b \
  --verdict DROPPED \
  --last 100

# → Nếu thấy POLICY_DENIED: check CiliumNetworkPolicy
# → Nếu thấy CT_TRUNCATED: MTU issue
# → Nếu không thấy gì: packet không đến node (routing issue)

# Scenario 2: DNS resolution thất bại
hubble observe \
  --protocol dns \
  --namespace production \
  --last 200 \
  -o json | jq 'select(.l7.dns.rcode != 0)'

# → rcode 3 (NXDOMAIN): tên không tồn tại
# → Không thấy DNS flow: DNS egress bị block bởi policy

# Scenario 3: HTTP requests chậm
hubble observe \
  --protocol http \
  --from-service production/frontend \
  --to-service production/backend \
  -o json | jq '{
    url: .l7.http.url,
    latency_ms: (.l7.http.latency_ns / 1000000),
    code: .l7.http.code
  }' | jq 'select(.latency_ms > 500)'
```

---

## Gotchas

- **Ring buffer và event loss**: Hubble eBPF ring buffer có fixed size. Trên high-throughput nodes (>100k flows/s), events có thể bị drop trước khi Hubble Agent đọc. Monitor `hubble_lost_events_total`. Tăng `eventBufferCapacity` nếu cần, nhưng tăng memory usage của cilium-agent.
- **Hubble Relay và partial data**: Hubble Relay chỉ aggregate flows đang stream — nếu Relay restart, flows trong khoảng đó bị mất. Không phải long-term storage.
- **L7 metrics và Envoy requirement**: HTTP/gRPC metrics trong Hubble chỉ available khi có L7 CiliumNetworkPolicy. Nếu không có L7 policy, Cilium không inspect L7 traffic → không có HTTP metrics. Để collect HTTP metrics không cần policy enforcement: dùng annotation `policy.cilium.io/proxy-visibility: "<Ingress/8080/TCP/HTTP>"`.
- **Proxy visibility annotation**: Có thể enable L7 visibility (cho metrics/observability) mà không cần enforce L7 policy:
  ```yaml
  annotations:
    policy.cilium.io/proxy-visibility: "<Ingress/8080/TCP/HTTP>,<Egress/9090/TCP/HTTP>"
  ```
- **Hubble UI và sensitive data**: Flow logs chứa HTTP URLs, headers — có thể chứa tokens, PII. Restrict access tới Hubble UI bằng OAuth2 proxy hoặc internal-only Ingress.
