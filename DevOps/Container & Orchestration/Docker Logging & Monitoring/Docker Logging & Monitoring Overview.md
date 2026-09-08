---
title: Docker Logging & Monitoring
tags:
  - docker
  - logging
  - monitoring
  - deep-dive
date: 2026-04-26
---

# Docker Logging & Monitoring

## Logging Architecture

```
Container process
  │ stdout / stderr
  ▼
Docker daemon (log driver)
  │
  ├─ json-file  → /var/lib/docker/containers/<id>/<id>-json.log
  ├─ syslog     → /var/log/syslog hoặc remote syslog
  ├─ journald   → systemd journal
  ├─ fluentd    → Fluentd/Fluentbit daemon
  ├─ loki       → Grafana Loki
  └─ awslogs    → AWS CloudWatch
```

**Best practice**: container chỉ write tới **stdout/stderr** — không write log files bên trong container. Log driver xử lý routing.

---

## Logging Drivers

### json-file (default)

```bash
# Cấu hình mặc định
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \     # rotate sau 10MB
  --log-opt max-file=3 \       # giữ 3 files (30MB total)
  --log-opt compress=true \    # compress rotated files
  nginx:alpine
```

```json
// /etc/docker/daemon.json — set default cho tất cả containers
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "compress": "true",
    "labels": "app,env",
    "env": "HOSTNAME"
  }
}
```

```bash
# Đọc logs
docker logs web
docker logs -f web                        # follow
docker logs --tail 100 web
docker logs --since 1h web
docker logs --since "2026-04-26T08:00:00" web
docker logs --until "2026-04-26T09:00:00" web

# Raw log file (json format)
cat /var/lib/docker/containers/<container-id>/<container-id>-json.log | jq .
```

---

### syslog

```bash
docker run \
  --log-driver syslog \
  --log-opt syslog-address=tcp://log-server.internal:514 \
  --log-opt syslog-facility=daemon \
  --log-opt tag="myapp/{{.Name}}/{{.ID}}" \
  myapp:latest
```

```json
// daemon.json
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "tcp://log-server:514",
    "tag": "docker/{{.Name}}"
  }
}
```

---

### journald

Gửi logs vào systemd journal — tích hợp tốt với systemd-based Linux.

```bash
docker run --log-driver journald myapp

# Đọc logs qua journalctl
journalctl CONTAINER_NAME=myapp -f
journalctl -u docker.service --since "1 hour ago"
```

---

### fluentd / fluentbit

```bash
docker run \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt fluentd-async=true \           # async, không block nếu fluentd down
  --log-opt fluentd-retry-wait=1s \
  --log-opt fluentd-max-retries=10 \
  --log-opt tag="docker.{{.Name}}" \
  myapp:latest
```

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: fluentd
      options:
        fluentd-address: "localhost:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
```

**Fluent Bit config (xử lý log từ Docker):**
```ini
[INPUT]
    Name              forward
    Listen            0.0.0.0
    Port              24224

[FILTER]
    Name              parser
    Match             docker.*
    Key_Name          log
    Parser            json

[OUTPUT]
    Name              loki
    Match             *
    Host              loki
    Port              3100
    Labels            job=docker, container=$container_name
```

---

### Grafana Loki Driver

```bash
# Install plugin
docker plugin install grafana/loki-docker-driver:latest \
  --alias loki \
  --grant-all-permissions

# Dùng
docker run \
  --log-driver loki \
  --log-opt loki-url="http://loki:3100/loki/api/v1/push" \
  --log-opt loki-labels="job=myapp,env=production" \
  --log-opt loki-retries=5 \
  --log-opt loki-batch-size=400 \
  myapp:latest
```

```json
// daemon.json — set default Loki
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://loki:3100/loki/api/v1/push",
    "loki-labels": "job=docker",
    "loki-retries": "5",
    "loki-batch-size": "400",
    "loki-pipeline-stages": "[{\"docker\":{}}]"
  }
}
```

---

## Log Rotation

**json-file rotation** (built-in):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```

**logrotate** (external rotation):
```
# /etc/logrotate.d/docker-containers
/var/lib/docker/containers/*/*.log {
    daily
    rotate 7
    compress
    missingok
    delaycompress
    copytruncate
    sharedscripts
    postrotate
        docker kill --signal HUP $(docker ps -q) 2>/dev/null || true
    endscript
}
```

---

## Monitoring — docker stats

```bash
# Live stats tất cả containers
docker stats

# Stats cụ thể
docker stats web db redis

# Snapshot (không follow)
docker stats --no-stream

# Custom format
docker stats --no-stream \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Output:
# NAME     CPU %   MEM USAGE / LIMIT   NET I/O         BLOCK I/O
# web      0.5%    45.2MiB / 512MiB    1.5kB / 2.1kB  10MB / 5MB
```

**Metrics giải thích:**
- `CPU %`: % CPU của host core (200% = 2 full cores)
- `MEM USAGE / LIMIT`: RAM đang dùng / limit đặt ra
- `MEM %`: % của limit
- `NET I/O`: tổng bytes network in/out
- `BLOCK I/O`: tổng bytes disk read/write
- `PIDS`: số processes trong container

---

## cAdvisor — Container Advisor

cAdvisor (Google) collect container metrics và expose Prometheus metrics endpoint.

```yaml
# docker-compose.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    restart: unless-stopped
```

**Metrics quan trọng:**

| Metric | Description |
|---|---|
| `container_cpu_usage_seconds_total` | CPU time consumed |
| `container_memory_usage_bytes` | Current memory usage |
| `container_memory_working_set_bytes` | Active memory (OOM killer dùng metric này) |
| `container_network_receive_bytes_total` | Network ingress |
| `container_network_transmit_bytes_total` | Network egress |
| `container_fs_reads_bytes_total` | Disk read bytes |
| `container_fs_writes_bytes_total` | Disk write bytes |
| `container_last_seen` | Container still running |

---

## Full Observability Stack

### Prometheus + cAdvisor + Grafana

```yaml
# compose.yml — full monitoring stack
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    ports:
      - "9090:9090"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "8080:8080"

  node-exporter:
    image: prom/node-exporter:v1.8.0
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.rootfs=/rootfs
      - --path.sysfs=/host/sys
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)
    ports:
      - "9100:9100"

  grafana:
    image: grafana/grafana:10.4.0
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: "false"
    ports:
      - "3000:3000"

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: cadvisor
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: node
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

### Loki + Promtail + Grafana (Logs)

```yaml
services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki_data:/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./promtail-config.yml:/etc/promtail/config.yml:ro

  grafana:
    image: grafana/grafana:10.4.0
    # ... (cấu hình như trên)
    # Thêm Loki datasource trong provisioning
```

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets: [localhost]
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*-json.log

    pipeline_stages:
      - json:
          expressions:
            stream: stream
            attrs: attrs
            tag: attrs.tag
      - json:
          source: attrs
          expressions:
            container: container_name
      - labels:
          stream:
          container:
      - timestamp:
          source: time
          format: RFC3339Nano
```

---

## Alerting với Alertmanager

```yaml
# prometheus.yml — thêm alerting rules
rule_files:
  - /etc/prometheus/alerts/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

```yaml
# alerts/docker.yml
groups:
  - name: docker
    rules:
      - alert: ContainerDown
        expr: absent(container_last_seen{name!=""}) or container_last_seen{name!=""} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} is down"

      - alert: ContainerHighCPU
        expr: rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} CPU > 80%"

      - alert: ContainerHighMemory
        expr: |
          container_memory_working_set_bytes{name!=""}
          / container_spec_memory_limit_bytes{name!=""} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} memory > 85%"

      - alert: ContainerOOMKilled
        expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.container }} OOMKilled"
```

---

## Ops Runbook

```bash
# Xem disk space dùng bởi logs
du -sh /var/lib/docker/containers/*/
ls -lh /var/lib/docker/containers/<id>/<id>-json.log

# Xem log driver của container
docker inspect --format='{{.HostConfig.LogConfig}}' mycontainer

# Xem events
docker events                                    # realtime events
docker events --since 1h                         # last 1 hour
docker events --filter type=container            # chỉ container events
docker events --filter event=oom                 # OOM events

# Debug container health
docker inspect --format='{{json .State.Health}}' mycontainer | jq .

# Top processes trong container
docker exec mycontainer top
docker exec mycontainer ps aux

# Resource usage từng layer
docker system df -v                              # verbose disk usage

# Flush logs (khi dùng json-file)
docker kill --signal SIGUSR1 mycontainer         # một số apps reload log handle
# Hoặc restart container (logs xoá nếu không có retention config)
```

---

## Gotchas

- **`docker logs` không hoạt động với non json-file driver**: khi dùng syslog/fluentd/loki driver, `docker logs` trả về error. Phải đọc logs từ destination (syslog server, Loki, ...).
- **json-file mặc định không có rotation**: nếu không set `max-size`, log file grow unlimited → disk full → Docker daemon crash. Luôn set rotation trong daemon.json.
- **Log mất khi container bị xoá**: `docker rm` xoá log files. Dùng external log driver (fluentd, loki) hoặc bind mount log directory để giữ logs sau khi container rm.
- **cAdvisor privileged**: cAdvisor cần `privileged: true` để đọc cgroup stats. Đây là trade-off bảo mật cần biết — chỉ chạy trên nodes tin cậy.
- **Loki driver async**: nếu Loki down và driver không async (`loki-async: false`), container sẽ block khi write log. Luôn dùng `loki-async: true` trong production.
- **container_memory_working_set_bytes vs container_memory_usage_bytes**: OOM killer dùng working_set (không tính reclaimable cache). `usage_bytes` bao gồm cache → thường cao hơn → misleading. Alert trên `working_set`.
- **Stdout/stderr buffering**: một số apps buffer stdout khi không có TTY → logs xuất hiện trễ. Fix: chạy với unbuffered mode (Python: `PYTHONUNBUFFERED=1`, Node: `--no-stdin`, Java: `-Djava.io.stdout.flush=true`).
