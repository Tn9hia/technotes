---
title: Kubernetes Logging — Loki Stack & Audit
tags:
  - kubernetes
  - observability
  - logging
  - loki
  - falco
  - audit
date: 2026-04-26
---

# Kubernetes Logging

## Logging Architecture

```
Pod stdout/stderr
      │
      ▼
Node (container runtime writes to /var/log/containers/)
      │
      ▼
Log Agent (Fluent Bit / Fluentd — DaemonSet, runs on every node)
      │
      ▼
Log Aggregator / Storage
      ├─ Loki (efficient, label-based, works với Grafana)
      ├─ Elasticsearch (full-text search, more resource intensive)
      └─ Cloud (CloudWatch, GCP Logging, Azure Monitor)
```

### Sidecar vs DaemonSet

| | DaemonSet agent | Sidecar agent |
|---|---|---|
| **Resource usage** | Shared per node | Per pod |
| **Configuration** | Centralized | Per application |
| **Access** | /var/log/containers/* | Direct process output |
| **Use case** | Standard | Custom parsing, stdout không đủ |

---

## Loki Stack

Loki — "Prometheus for logs" — không index log content (chỉ index labels), compress và store raw log chunks. Rẻ hơn Elasticsearch nhiều.

### Install PLG Stack (Promtail + Loki + Grafana)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Loki (với S3 backend — production)
helm install loki grafana/loki \
  -n monitoring \
  --set loki.storage.type=s3 \
  --set loki.storage.s3.region=ap-southeast-1 \
  --set loki.storage.s3.bucketnames=my-loki-chunks \
  --set loki.storage.s3.endpoint=s3.amazonaws.com \
  --set loki.storage.bucketNames.chunks=loki-chunks \
  --set loki.storage.bucketNames.ruler=loki-ruler \
  --set loki.storage.bucketNames.admin=loki-admin

# Fluent Bit (lightweight, recommended thay Promtail)
helm install fluent-bit grafana/fluent-bit \
  -n monitoring \
  --set loki.serviceName=loki-gateway
```

### Fluent Bit — DaemonSet Config

```yaml
# fluent-bit-values.yaml cho Helm
config:
  service: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri              # containerd log format
        DB                /run/fluent-bit/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  filters: |
    # Enrich với K8s metadata (namespace, pod, container, labels)
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On    # exclude pods với annotation fluentbit.io/exclude: "true"

    # Drop health check logs (noise)
    [FILTER]
        Name    grep
        Match   kube.*
        Exclude log /health

    # Parse JSON logs nếu app output JSON
    [FILTER]
        Name         parser
        Match        kube.*
        Key_Name     log
        Parser       json
        Reserve_Data On

  outputs: |
    [OUTPUT]
        Name             loki
        Match            kube.*
        Host             loki-gateway
        Port             80
        Labels           job=fluent-bit,cluster=prod-cluster
        Label_Keys       $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']
        # Dynamic labels từ log field
        dynamic_labels   level,request_id
        Batch_Wait       1s
        Batch_Size       1024000
        Line_Format      json
        Remove_Keys      kubernetes,stream
```

### LogQL — Query Language

```logql
-- Tất cả logs từ production namespace
{namespace="production"}

-- Logs từ specific pod
{namespace="production", pod=~"myapp-.*"}

-- Filter: chỉ error logs
{namespace="production"} |= "ERROR" or "error"

-- Pattern parser (extract fields từ unstructured logs)
{namespace="production"}
  | pattern `<timestamp> <level> <message>`
  | level = "ERROR"

-- JSON parser (cho structured JSON logs)
{namespace="production"}
  | json
  | level="error"
  | line_format "{{.message}}"

-- Metric query: error rate
sum(rate({namespace="production"} |= "ERROR" [5m])) by (pod)

-- Count logs by level trong 1h
sum by (level) (
  count_over_time(
    {namespace="production"} | json | __error__="" [1h]
  )
)

-- Latency từ JSON logs
{namespace="production"}
  | json
  | duration > 1s
  | line_format "{{.pod}} took {{.duration}}"
```

### Grafana Dashboard cho Logs

```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Add Loki datasource: http://loki-gateway:80
# Explore → Loki → build queries

# Useful panels:
# 1. Log volume by namespace (bar chart)
# 2. Error rate over time (time series)
# 3. Log stream (logs panel)
# 4. Top error messages (table)
```

---

## Kubernetes Audit Logging

Audit log ghi lại **tất cả API server requests** — ai đã làm gì, khi nào.

### Audit Policy Levels

```
None     → không log request này
Metadata → log request metadata (user, resource, verb) — không log body
Request  → log metadata + request body
RequestResponse → log metadata + request body + response body
```

### Production Audit Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:             # không log các stages này
  - RequestReceived     # chỉ log khi completed

rules:
  # Không log read-only operations trên non-sensitive resources (noise)
  - level: None
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "configmaps"]
      - group: "coordination.k8s.io"
        resources: ["leases"]

  # Không log node/kubelet heartbeats
  - level: None
    users:
      - "system:kube-proxy"
      - "system:node"
    verbs: ["get", "list", "watch"]

  # Không log health/readiness check requests
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/readyz*"
      - "/livez*"
      - "/metrics"

  # Log Secrets ở mức Metadata (không log values!)
  - level: Metadata
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

  # Log RBAC changes ở mức RequestResponse (quan trọng cho audit)
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

  # Log tất cả auth failures
  - level: Request
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources: ["*"]
    users:
      - "system:anonymous"

  # Log Pod exec/attach/portforward (potential malicious activity)
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Log workload mutations trong production namespace
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    namespaces: ["production", "staging"]
    resources:
      - group: "apps"
        resources: ["deployments", "statefulsets", "daemonsets"]
      - group: ""
        resources: ["pods"]

  # Default: log metadata cho tất cả operations còn lại
  - level: Metadata
    omitStages:
      - RequestReceived
```

### Audit Log kube-apiserver Flags

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30        # giữ 30 ngày
    - --audit-log-maxbackup=10     # giữ 10 backup files
    - --audit-log-maxsize=100      # rotate khi đạt 100MB
    - --audit-log-format=json      # JSON format

    # Webhook backend (stream tới Falco/SIEM)
    - --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
    - --audit-webhook-batch-max-wait=5s

  volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit-policy
    readOnly: true
  - mountPath: /var/log/kubernetes
    name: audit-log

  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
```

### Ship Audit Logs tới Loki

```yaml
# fluent-bit config cho audit logs
[INPUT]
    Name    tail
    Tag     audit.*
    Path    /var/log/kubernetes/audit.log
    Parser  json
    DB      /run/fluent-bit/audit.db

[FILTER]
    Name    grep
    Match   audit.*
    # Chỉ giữ log level >= RequestResponse hoặc có user quan trọng
    Regex   verb (create|update|patch|delete)

[OUTPUT]
    Name    loki
    Match   audit.*
    Labels  job=k8s-audit,cluster=prod-cluster
    Label_Keys  $verb,$user[username],$objectRef[resource],$objectRef[namespace]
```

```logql
-- Query audit logs trong Loki
{job="k8s-audit"}
  | json
  | verb = "delete"
  | objectRef_resource = "secrets"

-- Ai đã exec vào pods?
{job="k8s-audit"}
  | json
  | objectRef_subresource = "exec"
  | line_format "{{.user.username}} → {{.objectRef.namespace}}/{{.objectRef.name}}"

-- Failed auth attempts
{job="k8s-audit"}
  | json
  | responseStatus_code = "403"
  | user_username = "system:anonymous"
```

---

## Falco — Runtime Security Detection

Falco monitor syscalls và K8s API events — detect anomalous behavior at runtime.

Cross-reference: [[Supply Chain Security]]

### Install Falco

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  -n falco \
  --create-namespace \
  --set driver.kind=ebpf \               # eBPF driver (không cần kernel module)
  --set falcosidekick.enabled=true \     # forward alerts
  --set falcosidekick.config.slack.webhookurl=https://hooks.slack.com/...
```

### Custom Falco Rules

```yaml
# /etc/falco/rules.d/custom-rules.yaml

# Detect shell spawned trong container
- rule: Terminal shell in container
  desc: Detect shell execution in a running container
  condition: >
    spawned_process
    and container
    and not container.image.repository in (debug_images)
    and proc.name in (shell_binaries)
  output: >
    Shell spawned in container (
      user=%user.name user_loginuid=%user.loginuid
      container_id=%container.id
      container_name=%container.name
      image=%container.image.repository:%container.image.tag
      shell=%proc.name
      parent=%proc.pname
      cmdline=%proc.cmdline
    )
  priority: CRITICAL
  tags: [container, shell, T1059]

# Detect sensitive file read
- rule: Read sensitive file
  desc: Detect reads of sensitive files by non-trusted programs
  condition: >
    open_read
    and container
    and not proc.name in (trusted_processes)
    and (
      fd.name startswith /etc/shadow
      or fd.name startswith /etc/kubernetes/pki
      or fd.name startswith /var/lib/kubelet/pki
    )
  output: >
    Sensitive file read (
      user=%user.name
      container=%container.name
      file=%fd.name
      proc=%proc.cmdline
    )
  priority: WARNING

# Detect outbound connection tới unexpected IP
- rule: Unexpected outbound connection
  desc: Outbound connection to non-whitelisted IP
  condition: >
    outbound
    and container
    and not fd.sip in (allowed_ips)
    and not proc.name in (allowed_network_binaries)
    and container.image.repository != "debug-image"
  output: >
    Unexpected outbound connection (
      user=%user.name
      container=%container.name
      image=%container.image.repository
      ip=%fd.rip
      port=%fd.rport
      proc=%proc.cmdline
    )
  priority: WARNING

# Detect privilege escalation
- rule: Container privilege escalation
  desc: Detect privileged container launch
  condition: >
    spawned_process
    and container
    and proc.name = "sudo"
  output: >
    sudo executed in container (
      user=%user.name
      container=%container.name
      proc=%proc.cmdline
    )
  priority: CRITICAL

# Detect crypto mining
- rule: Crypto mining detected
  desc: Detect known crypto miner binaries
  condition: >
    spawned_process
    and container
    and proc.name in (crypto_miners)
  output: >
    Crypto mining binary detected (
      container=%container.name
      image=%container.image.repository
      proc=%proc.name
    )
  priority: CRITICAL
```

```yaml
# Falco macros và lists
- list: shell_binaries
  items: [bash, sh, zsh, fish, dash]

- list: debug_images
  items: [nicolaka/netshoot, busybox, debug]

- list: trusted_processes
  items: [filebeat, fluentbit, datadog-agent]

- list: allowed_ips
  items: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]

- list: crypto_miners
  items: [xmrig, minerd, cpuminer, claymore, ethminer]

- macro: container
  condition: container.id != host

- macro: outbound
  condition: >
    (evt.type = connect or evt.type = sendto)
    and evt.dir = <
    and fd.typechar = 4
    and not fd.sip in (rfc_1918_addresses)
```

### Falco Sidekick — Alert Routing

```yaml
# falcosidekick-config.yaml
config:
  slack:
    webhookurl: "https://hooks.slack.com/..."
    channel: "#security-alerts"
    minimumpriority: "warning"

  pagerduty:
    routingkey: "<routing-key>"
    minimumpriority: "critical"

  loki:
    hostport: "http://loki-gateway:80"
    user: "falco"
    apikey: ""
    minimumpriority: "notice"
    # Logs đến Loki để query cùng với app logs

  elasticsearch:
    hostport: "http://elasticsearch:9200"
    index: "falco"
    minimumpriority: "debug"
```

---

## Structured Logging Best Practices

```go
// Go app — sử dụng structured logging (zap/zerolog)
logger.Info("HTTP request",
    zap.String("method", r.Method),
    zap.String("path", r.URL.Path),
    zap.Int("status", w.Status()),
    zap.Duration("duration", time.Since(start)),
    zap.String("request_id", requestID),
    zap.String("user_id", userID),
)
// → {"level":"info","method":"GET","path":"/api/v1/users","status":200,"duration":"45ms","request_id":"abc123"}
```

```yaml
# Log levels nên dùng:
# ERROR: cần action ngay (unexpected error, data loss risk)
# WARN:  cần attention nhưng không khẩn cấp (retry succeeded, degraded mode)
# INFO:  normal operations (request completed, job started/finished)
# DEBUG: developer info (không dùng trong production — too verbose)

# Không log:
# - Sensitive data (passwords, tokens, PII)
# - Health check endpoints (noise)
# - Verbose debug trong production
```

---

## Log Retention Policy

```yaml
# Loki retention (trong loki-config.yaml)
compactor:
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

limits_config:
  retention_period: 30d    # global default

  # Per-tenant/stream retention (per label)
  per_tenant_override_config: /etc/loki/overrides.yaml

# overrides.yaml
overrides:
  production:
    retention_period: 90d   # production giữ lâu hơn
  staging:
    retention_period: 14d
```

---

## Gotchas

- **Fluent Bit memory buffer và backpressure**: Khi Loki unavailable, Fluent Bit buffer logs trong memory. Mặc định `Mem_Buf_Limit 50MB` — khi đầy → drop logs. Cần filesystem buffer cho production.
- **Loki label cardinality**: Như Prometheus, high-cardinality labels (request_id, user_id) → Loki rất chậm và tốn memory. Chỉ index labels có bounded cardinality (namespace, pod, level).
- **Audit log volume**: Với `RequestResponse` level cho tất cả operations → audit log rất lớn. Tune policy để chỉ log những gì cần. 1 cluster lớn có thể tạo 10GB+ audit logs/day.
- **Falco kernel module vs eBPF**: Kernel module yêu cần kernel headers và specific version. eBPF driver portable hơn (không cần compile against kernel). Dùng eBPF cho managed K8s (EKS, GKE).
- **Fluent Bit CRI parser**: containerd dùng CRI log format (khác Docker json-file format). Phải dùng parser `cri` không phải `docker` nếu dùng containerd.
- **Log aggregation và multi-line**: Stack traces thường trải qua nhiều dòng. Fluent Bit `multiline` parser cần config cho từng language. Java stack traces cần `multiline.parser java`.
