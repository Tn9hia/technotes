---
tags:
  - openstack
  - octavia
  - load-balancer
  - lbaas
---

# Octavia — Load Balancer as a Service

Octavia cung cấp **LBaaS (Load Balancer as a Service)** cho OpenStack. Thay thế Neutron LBaaS v2 (deprecated).

## Kiến trúc

```
octavia-api ──► octavia-housekeeping
      │
octavia-worker ──► Nova (tạo Amphora VMs)
      │                │
      │            Amphora VM (HAProxy inside)
      │
octavia-health-manager (monitor Amphora)
```

### Amphora
- VM/container chạy **HAProxy** bên trong
- Octavia tạo Amphora VM trên compute node để làm LB
- Có thể active/standby hoặc active/active (Active-Active topology)

## Hierarchy

```
Load Balancer
└── Listener (frontend: port, protocol)
    └── Pool (backend server group)
        ├── Member (backend server + port)
        ├── Health Monitor (health check)
        └── L7 Policy (routing rules - optional)
```

## CLI Usage

```bash
# Tạo Load Balancer
openstack loadbalancer create \
  --name my-lb \
  --vip-subnet-id <subnet-id>

# Đợi LB active
openstack loadbalancer show my-lb

# Tạo Listener (HTTP port 80)
openstack loadbalancer listener create \
  --name my-listener \
  --loadbalancer my-lb \
  --protocol HTTP \
  --protocol-port 80

# Tạo Pool với thuật toán Round Robin
openstack loadbalancer pool create \
  --name my-pool \
  --listener my-listener \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN

# Thêm members (backend servers)
openstack loadbalancer member create \
  --name web-01 \
  --address 10.0.1.10 \
  --protocol-port 80 \
  my-pool

openstack loadbalancer member create \
  --name web-02 \
  --address 10.0.1.11 \
  --protocol-port 80 \
  my-pool

# Tạo Health Monitor
openstack loadbalancer healthmonitor create \
  --delay 5 \
  --timeout 5 \
  --max-retries 3 \
  --type HTTP \
  --url-path /health \
  my-pool

# Gán Floating IP cho LB VIP
LB_VIP_PORT=$(openstack loadbalancer show my-lb -f value -c vip_port_id)
FIP=$(openstack floating ip create provider-net -f value -c floating_ip_address)
openstack floating ip set --port $LB_VIP_PORT $FIP
```

## Load Balancing Algorithms

| Algorithm | Mô tả | Dùng khi |
|-----------|-------|---------|
| `ROUND_ROBIN` | Lần lượt từng server | Servers đồng đều |
| `LEAST_CONNECTIONS` | Server ít kết nối nhất | Requests có duration khác nhau |
| `SOURCE_IP` | Hash source IP | Session persistence |
| `SOURCE_IP_PORT` | Hash source IP + port | Fine-grained |

## L7 Routing (Layer 7 Policy)

```bash
# Listener L7 (HTTP/HTTPS)
openstack loadbalancer listener create \
  --name http-listener \
  --loadbalancer my-lb \
  --protocol HTTP \
  --protocol-port 80

# Pool cho API backend
openstack loadbalancer pool create \
  --name api-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN \
  --listener http-listener

# Pool cho static backend
openstack loadbalancer pool create \
  --name static-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN

# L7 Policy: route /api/* đến api-pool
openstack loadbalancer l7policy create \
  --name api-route \
  --listener http-listener \
  --action REDIRECT_TO_POOL \
  --redirect-pool api-pool \
  --position 1

openstack loadbalancer l7rule create \
  --compare-type STARTS_WITH \
  --type PATH \
  --value /api \
  api-route
```

## HTTPS Termination

```bash
# Upload certificate vào Barbican
openstack secret store \
  --name my-cert \
  --payload-content-type "text/plain" \
  --payload "$(cat server.crt)"

# Tạo HTTPS listener
openstack loadbalancer listener create \
  --name https-listener \
  --loadbalancer my-lb \
  --protocol TERMINATED_HTTPS \
  --protocol-port 443 \
  --default-tls-container-ref <barbican-container-ref>
```

## Topology: Active-Standby vs Active-Active

```ini
# octavia.conf
[controller_worker]
loadbalancer_topology = ACTIVE_STANDBY  # hoặc ACTIVE_ACTIVE (cần ECMP)
```

---
*Xem thêm: [[Neutron - Networking]] | [[Nova - Compute]] | [[OpenStack]]*
