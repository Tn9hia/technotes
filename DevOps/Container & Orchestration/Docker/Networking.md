---
title: Docker Networking
tags:
  - docker
  - networking
  - deep-dive
date: 2026-04-26
---

# Docker Networking

## Cơ chế hoạt động

Docker networking xây trên các Linux primitives:

```
Container A          Container B
    │                    │
  veth0               veth1         ← virtual ethernet pairs
    │                    │
┌───▼────────────────────▼───┐
│         docker0 (bridge)    │     ← Linux bridge (172.17.0.0/16)
└───────────────┬─────────────┘
                │
           iptables NAT            ← MASQUERADE cho outbound traffic
                │
          eth0 (host NIC)
```

Mỗi container có:
- `veth` pair: 1 đầu trong container namespace (eth0), 1 đầu gắn vào bridge host
- Network namespace riêng: routing table, iptables, interfaces độc lập
- IP từ subnet của bridge network

---

## Network Drivers

### bridge (default)

Linux software bridge. Containers cùng bridge → L2 communication. Traffic ra ngoài → NAT qua iptables MASQUERADE.

```bash
# Default bridge network (docker0) — không có DNS resolution theo tên
docker run -d --name c1 nginx
docker run -d --name c2 nginx
docker exec c2 ping c1   # FAIL — không resolve tên

# Custom bridge network — có embedded DNS
docker network create --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  mynet

docker run -d --name db --network mynet postgres:16
docker run -d --name app --network mynet myapp
docker exec app ping db         # OK — DNS resolution hoạt động
docker exec app nslookup db     # OK
```

**Kết nối container vào nhiều networks:**
```bash
docker network connect mynet container_name
docker network disconnect mynet container_name
```

**Inspect bridge:**
```bash
docker network inspect mynet
# → Containers list với IP của từng container
# → Subnet, Gateway info
ip link show docker0    # xem bridge trên host
brctl show docker0      # xem ports gắn vào bridge (brctl-utils)
```

---

### host

Container dùng network namespace của host — không có NAT, không có network isolation.

```bash
docker run -d --network host nginx
# nginx listen trực tiếp trên host port 80 — không cần -p 80:80
```

**Khi nào dùng:**
- Cần max network performance (tránh overhead NAT + veth)
- Container cần bind vào host IP cụ thể
- Monitoring tools cần thấy host network traffic (tcpdump, prometheus node-exporter)

**Gotcha:** Container thấy host network → không cô lập, dễ conflict port với host services.

---

### none

Container không có network interface nào ngoài loopback (lo).

```bash
docker run -d --network none myapp
```

Dùng cho batch jobs không cần network, hoặc custom networking (tự inject network interface).

---

### overlay

Multi-host networking cho Docker Swarm. Dùng VXLAN (UDP 4789) để tunnel L2 traffic qua L3 network giữa các Docker hosts.

```
Host A                          Host B
┌──────────────────┐            ┌──────────────────┐
│ container1       │            │ container2       │
│  10.0.0.3        │            │  10.0.0.4        │
│     │            │            │     │            │
│  veth            │            │  veth            │
│     │            │            │     │            │
│  br0 (overlay)   │            │  br0 (overlay)   │
│     │            │            │     │            │
│  VTEP (vxlan0)   │            │  VTEP (vxlan0)   │
│     │            │            │     │            │
└─────┼────────────┘            └─────┼────────────┘
      │                               │
      └──── UDP 4789 (VXLAN) ─────────┘
              eth0 ↔ eth0
```

```bash
# Tạo overlay network (phải có Swarm initialized)
docker network create \
  --driver overlay \
  --attachable \          # cho phép standalone containers attach
  --subnet 10.0.9.0/24 \
  myoverlay

# Service sử dụng overlay
docker service create \
  --network myoverlay \
  --replicas 3 \
  nginx:alpine
```

**Ports cần mở giữa Swarm nodes:**
- TCP 2377 — Swarm cluster management
- TCP/UDP 7946 — node discovery (gossip)
- UDP 4789 — overlay network (VXLAN data plane)

---

### macvlan

Container nhận MAC address riêng, kết nối trực tiếp vào physical network → container có IP trong cùng subnet với LAN.

```bash
# Xác định physical interface và subnet của host
ip addr show eth0
# inet 192.168.1.100/24

docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \         # interface vật lý của host
  macnet

docker run -d --network macnet \
  --ip 192.168.1.200 \        # IP cụ thể trong LAN
  nginx

# Container 192.168.1.200 accessible trực tiếp từ LAN
# Host KHÔNG thể communicate với container (macvlan limitation)
# Workaround: tạo macvlan sub-interface trên host:
ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.201/32 dev macvlan0
ip link set macvlan0 up
```

**Khi nào dùng:** Legacy apps cần IP trong LAN, L2 multicast/broadcast, VLAN routing.

---

## Port Mapping & iptables

Docker tự động thêm iptables rules khi map port:

```bash
docker run -p 8080:80 nginx
# Docker thêm iptables rule:
# DOCKER chain: -j DNAT --to-destination 172.17.0.2:80

# Xem rules:
iptables -t nat -L DOCKER -n --line-numbers
```

**Lưu ý:** Docker bypass UFW/firewalld — port mapping là iptables rules trực tiếp, UFW không chặn được nếu không config đúng. Để hạn chế:
```bash
# Chỉ bind localhost, không expose ra ngoài
docker run -p 127.0.0.1:8080:80 nginx
```

---

## DNS trong Docker

Custom bridge networks có **embedded DNS server** (127.0.0.11):
- Resolve container names → IP
- Resolve service names (Swarm) → VIP
- Forward external queries ra host DNS

```bash
# Verify DNS trong container
docker exec myapp cat /etc/resolv.conf
# nameserver 127.0.0.11
# options ndots:0

docker exec myapp nslookup db
# Server: 127.0.0.11
# Address: 127.0.0.11#53
# Name: db
# Address: 172.20.0.3
```

**Aliases:**
```bash
docker run --network mynet --network-alias cache redis:7
# "cache" cũng resolve về container này
```

---

## Inter-container communication patterns

```bash
# 1. Same network — dùng tên container
docker run -d --name redis --network mynet redis:7
docker run -d --name app --network mynet -e REDIS_URL=redis://redis:6379 myapp

# 2. Container với nhiều networks (nối 2 segments)
docker network create frontend
docker network create backend
docker run -d --name proxy --network frontend nginx
docker network connect backend proxy      # proxy có cả 2 networks

# 3. Link (legacy — không dùng trong production mới)
docker run -d --name db mysql
docker run -d --link db:mysql myapp      # inject env MYSQL_* vào myapp
# Dùng custom network thay thế
```

---

## Troubleshooting Network

```bash
# Xem network list
docker network ls

# Inspect network — xem containers và IPs
docker network inspect mynet

# Test connectivity giữa containers
docker exec app ping db -c 3
docker exec app nc -zv db 5432     # test TCP port
docker exec app curl http://api:8080/health

# Xem iptables rules Docker tạo
iptables -t nat -L -n -v
iptables -L DOCKER-USER -n          # custom rules để không bị overwrite

# Capture traffic vào container
tcpdump -i docker0 -w /tmp/cap.pcap

# Network namespace debug
pid=$(docker inspect -f '{{.State.Pid}}' mycontainer)
nsenter -t $pid -n ip addr show     # xem network interfaces của container từ host
```

---

## Gotchas

- **Default bridge không có DNS**: container trong `bridge` (docker0) không resolve tên nhau. Luôn tạo custom bridge network.
- **Container restart → IP thay đổi**: IP của container trong bridge network không fixed (trừ `--ip`). Dùng DNS (tên container) thay vì hardcode IP.
- **Docker bypass UFW**: UFW/firewalld không chặn port mapping. Bind vào `127.0.0.1` hoặc dùng `DOCKER-USER` iptables chain cho custom firewall rules.
- **Overlay cần Swarm**: overlay network chỉ tạo được khi `docker swarm init`. Với standalone multi-host, cần `--attachable`.
- **macvlan và host isolation**: host không thể communicate với containers trên macvlan network qua `eth0`. Cần sub-interface trên host.
- **publish port = 0.0.0.0**: `-p 8080:80` bind vào tất cả interfaces (kể cả public IP). Dùng `-p 127.0.0.1:8080:80` cho services chỉ dùng nội bộ.
