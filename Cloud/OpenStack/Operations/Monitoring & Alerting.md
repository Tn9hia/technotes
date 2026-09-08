---
tags:
  - openstack
  - monitoring
  - prometheus
  - grafana
  - alerting
  - operations
---

# Monitoring & Alerting

## Monitoring Stack

```
OpenStack Services
      │
  Exporters (Prometheus format)
  ├── openstack-exporter (API metrics)
  ├── node_exporter (host metrics)
  ├── ceph-exporter (ceph metrics)
  └── rabbitmq-exporter
      │
  Prometheus (scrape + store)
      │
  Grafana (visualize)
  Alertmanager (alert routing)
```

## OpenStack Exporter

```bash
# Cài openstack-exporter
# https://github.com/openstack-exporter/openstack-exporter

# clouds.yaml
cat > /etc/openstack/clouds.yaml <<EOF
clouds:
  mycloud:
    auth:
      auth_url: http://controller:5000/v3
      project_name: admin
      username: admin
      password: secret
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
    identity_api_version: 3
EOF

# Chạy exporter
openstack-exporter --cloud mycloud &
# Port mặc định: 9180
```

## Prometheus cấu hình

```yaml
# prometheus.yml
global:
  scrape_interval: 60s

scrape_configs:
  - job_name: 'openstack'
    static_configs:
      - targets: ['controller:9180']

  - job_name: 'node'
    static_configs:
      - targets:
          - 'ctrl1:9100'
          - 'ctrl2:9100'
          - 'ctrl3:9100'
          - 'compute01:9100'
          - 'storage01:9100'

  - job_name: 'ceph'
    static_configs:
      - targets: ['ceph-mgr:9283']

  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['ctrl1:15692']  # RabbitMQ Prometheus plugin
```

## Metrics quan trọng cần monitor

### Compute

```
openstack_nova_running_vms                    # Tổng VMs đang chạy
openstack_nova_vcpus_available               # vCPUs available
openstack_nova_memory_available_bytes        # RAM available
openstack_nova_agent_state{service="nova-compute"} # Compute agent health
openstack_nova_server_status{status="ERROR"} # VMs bị error
```

### Network

```
openstack_neutron_agent_state               # Agent health (L2, L3, DHCP)
openstack_neutron_floating_ips              # Số FIPs đang dùng
openstack_neutron_routers                   # Số routers
```

### Storage

```
openstack_cinder_volume_status{status="error"}  # Volume error
openstack_cinder_pool_capacity_bytes           # Pool capacity
# Ceph metrics:
ceph_cluster_total_bytes                    # Total raw capacity
ceph_cluster_total_used_bytes              # Used
ceph_osd_stat_bytes                        # Per-OSD usage
ceph_pg_state{state!="active+clean"}       # Unhealthy PGs
ceph_pool_rd_bytes                         # Read IOPS
ceph_pool_wr_bytes                         # Write IOPS
```

### Infrastructure

```
# MariaDB Galera
mysql_global_status_wsrep_cluster_size    # Galera cluster size (phải là 3)
mysql_global_status_wsrep_local_state{state!="4"} # 4 = Synced
mysql_global_status_threads_connected    # DB connections

# RabbitMQ
rabbitmq_queue_messages_total            # Queue depth
rabbitmq_node_mem_used_bytes            # Memory usage
rabbitmq_connections_total              # Connections

# HAProxy
haproxy_backend_status{status!="UP"}   # Backend down
haproxy_requests_total                 # Request rate
```

## Grafana Dashboards

### Import dashboards từ Grafana.com

| Dashboard | ID | Mô tả |
|-----------|-----|-------|
| OpenStack Exporter | Search "openstack" | VM, network, storage |
| Ceph Overview | 2842 | Ceph health, performance |
| Node Exporter | 1860 | Host CPU, RAM, disk, network |
| RabbitMQ | 4279 | Queue, connections |
| MariaDB | 7362 | MySQL/MariaDB metrics |
| HAProxy | 2428 | HAProxy stats |

## Alerting Rules

```yaml
# alert_rules.yml
groups:
  - name: openstack
    rules:
      # Nova compute service down
      - alert: NovaComputeDown
        expr: openstack_nova_agent_state{service="nova-compute"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Nova compute agent down on {{ $labels.hostname }}"

      # Nhiều VMs bị error
      - alert: TooManyVMsInError
        expr: openstack_nova_server_status{status="ERROR"} > 5
        for: 5m
        labels:
          severity: warning

      # Ceph cluster unhealthy
      - alert: CephHealthCritical
        expr: ceph_health_status == 2  # 2 = HEALTH_ERR
        for: 1m
        labels:
          severity: critical

      # Galera cluster size < 3
      - alert: GaleraClusterDegraded
        expr: mysql_global_status_wsrep_cluster_size < 3
        for: 1m
        labels:
          severity: critical

      # Disk usage cao
      - alert: HighDiskUsage
        expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} > 85%"

      # RabbitMQ queue depth cao
      - alert: RabbitMQQueueDepthHigh
        expr: rabbitmq_queue_messages_total > 1000
        for: 5m
        labels:
          severity: warning
```

## Log Aggregation

```yaml
# Loki + Promtail (hoặc ELK stack)

# Promtail config để scrape OpenStack logs
scrape_configs:
  - job_name: nova
    static_configs:
      - targets: [localhost]
        labels:
          job: nova
          __path__: /var/log/nova/*.log

  - job_name: neutron
    static_configs:
      - targets: [localhost]
        labels:
          job: neutron
          __path__: /var/log/neutron/*.log
```

## Ceilometer (OpenStack native telemetry)

Ceilometer là OpenStack service để collect metrics:

```bash
# Enable trong Kolla
# globals.yml
enable_ceilometer: "yes"
enable_aodh: "yes"  # Alerting
enable_gnocchi: "yes"  # Time series metrics store

# List meters
openstack metric list
openstack alarm list
```

## Health Check Checklist (Daily)

```bash
#!/bin/bash
# daily_check.sh

echo "=== Nova Services ==="
openstack compute service list

echo "=== Neutron Agents ==="
openstack network agent list

echo "=== Volume Services ==="
openstack volume service list

echo "=== Ceph Health ==="
ceph status

echo "=== Galera ==="
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_size';"

echo "=== RabbitMQ ==="
rabbitmqctl cluster_status | head -20

echo "=== VMs in ERROR ==="
openstack server list --all-projects --status ERROR

echo "=== Floating IPs without server ==="
openstack floating ip list | grep "None"
```

---
*Xem thêm: [[Troubleshooting]] | [[Day 2 Operations]] | [[CLI Commands]] | [[OpenStack]]*
