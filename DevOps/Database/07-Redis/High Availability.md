---
title: Redis High Availability
tags:
  - redis
  - high-availability
  - sentinel
  - cluster
  - replication
date: 2026-04-30
---

# Redis High Availability

## Replication (Master-Replica)

```
Master ──────────────────────────────────────┐
  │  full sync (RDB snapshot)                 │
  ▼                                           │
Replica 1 ─── replication stream (commands) ─┤
  │                                           │
  └── Replica 2 (cascade replication)         │
                                              ▼
                                    Client writes → Master only
                                    Client reads  → Master or Replica
```

```bash
# redis.conf trên Replica
replicaof 192.168.1.10 6379
masterauth <master_password>
replica-read-only yes              # replicas từ chối writes

# Replication auth
requirepass <password>

# Verify replication status
redis-cli -h master INFO replication
# role:master
# connected_slaves:2
# slave0:ip=192.168.1.11,port=6379,state=online,offset=12345,lag=0

redis-cli -h replica INFO replication
# role:slave
# master_host:192.168.1.10
# master_link_status:up
# master_repl_offset:12345
# master_last_io_seconds_ago:1
```

**Replication lag monitoring:**
```bash
# Lag = master_repl_offset - replica_repl_offset
# redis.conf — delay writes nếu replica lag quá cao
min-replicas-to-write 1           # master từ chối writes nếu < 1 replica connected
min-replicas-max-lag 10           # và lag của replica > 10s
```

---

## Redis Sentinel — HA cho Non-Cluster

Sentinel monitor master, tự động failover khi master down.

```
                    ┌─────────────────────────────┐
                    │         Sentinel Cluster      │
                    │  S1    S2    S3               │
                    │  ●─────●─────●  quorum=2      │
                    └──┬─────┬─────┴─────────────  │
                       │     │                      │
              monitor  ▼     ▼  monitor             │
┌─────────────────────────────────────────────────┐ │
│            Redis Topology                        │ │
│    Master ────► Replica 1                        │ │
│       │    ────► Replica 2                       │ │
└─────────────────────────────────────────────────┘ │
                                                     │
Clients ──► Sentinel.getMasterAddrByName() ──────────┘
            (discover current master address)
```

### Sentinel Config

```bash
# sentinel.conf (mỗi Sentinel node)
port 26379
sentinel monitor mymaster 192.168.1.10 6379 2    # quorum=2
sentinel auth-pass mymaster <password>
sentinel down-after-milliseconds mymaster 5000   # 5s không respond → subjective down
sentinel failover-timeout mymaster 60000         # 60s để complete failover
sentinel parallel-syncs mymaster 1               # 1 replica sync at a time

# Chạy Sentinel
redis-sentinel sentinel.conf
# hoặc:
redis-server sentinel.conf --sentinel
```

### Failover Flow

```
1. Master không respond > down-after-milliseconds
   → Sentinel S1: "Master is SDOWN (subjectively down)"

2. S1 hỏi S2, S3: "Bạn có thấy master không?"
   → Đủ votes >= quorum → "Master is ODOWN (objectively down)"

3. Sentinels bầu leader (Raft-like) để handle failover

4. Leader chọn replica tốt nhất:
   - replica với replication offset lớn nhất (ít lag nhất)
   - replica với lowest slave-priority

5. Leader gửi REPLICAOF NO ONE tới replica được chọn → thành master mới

6. Các replicas còn lại được redirect tới master mới

7. Old master (khi online lại) → trở thành replica của master mới
```

### Client Connection via Sentinel

```python
# Python redis-py
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379),
], socket_timeout=0.1)

master = sentinel.master_for('mymaster', socket_timeout=0.1)
replica = sentinel.slave_for('mymaster', socket_timeout=0.1)

master.set('key', 'value')    # writes → current master
replica.get('key')            # reads → replica
```

```yaml
# HAProxy config cho Sentinel
listen redis-master
  bind *:6380
  option tcp-check
  tcp-check connect
  tcp-check send "PING\r\n"
  tcp-check expect string +PONG
  tcp-check send "INFO replication\r\n"
  tcp-check expect string role:master
  server redis1 192.168.1.10:6379 check inter 1s
  server redis2 192.168.1.11:6379 check inter 1s
  server redis3 192.168.1.12:6379 check inter 1s
```

---

## Redis Cluster — Horizontal Scaling

Cluster shards data tự động across multiple nodes (không cần proxy).

```
Hash Slots: 0 ──────────────────────────────── 16383
             │                                   │
          [0-5460]      [5461-10922]      [10923-16383]
          Master A       Master B           Master C
          Replica A      Replica B          Replica C

Key → CRC16(key) % 16384 → slot → node
CLUSTER KEYSLOT user:1        # → 10778 (Master B)
```

### Setup Redis Cluster

```bash
# Cần tối thiểu 6 nodes: 3 masters + 3 replicas
# redis.conf cho mỗi node
port 7001
cluster-enabled yes
cluster-config-file nodes-7001.conf
cluster-node-timeout 5000           # node down nếu không respond 5s
cluster-require-full-coverage yes   # từ chối requests nếu có slot uncovered
appendonly yes

# Tạo cluster (Redis 5+)
redis-cli --cluster create \
  192.168.1.10:7001 \
  192.168.1.10:7002 \
  192.168.1.10:7003 \
  192.168.1.11:7001 \
  192.168.1.11:7002 \
  192.168.1.11:7003 \
  --cluster-replicas 1     # 1 replica per master

# Verify
redis-cli -c -h 192.168.1.10 -p 7001 CLUSTER INFO
redis-cli -c -h 192.168.1.10 -p 7001 CLUSTER NODES
```

### Cluster Operations

```bash
# Client dùng -c flag (auto-redirect MOVED)
redis-cli -c -h 192.168.1.10 -p 7001

# MOVED redirect: key trên node khác
# Client nhận: (error) MOVED 10778 192.168.1.10:7002
# → reconnect tới 192.168.1.10:7002

# ASK redirect: resharding đang diễn ra (slot đang migrate)
# Client nhận: (error) ASK 10778 192.168.1.10:7003
# → gửi ASKING tới node mới, rồi thực hiện command

# Cluster management
redis-cli --cluster check 192.168.1.10:7001
redis-cli --cluster info  192.168.1.10:7001

# Add new master
redis-cli --cluster add-node 192.168.1.12:7001 192.168.1.10:7001

# Rebalance slots
redis-cli --cluster rebalance 192.168.1.10:7001 --cluster-use-empty-masters

# Reshard manually (move slots)
redis-cli --cluster reshard 192.168.1.10:7001 \
  --cluster-from <source-node-id> \
  --cluster-to <target-node-id> \
  --cluster-slots 1000 \
  --cluster-yes

# Remove node (phải empty slots first)
redis-cli --cluster del-node 192.168.1.10:7001 <node-id>

# Failover manual
redis-cli -h 192.168.1.11 -p 7001 CLUSTER FAILOVER    # từ replica → takeover

# Cluster key constraint: multi-key operations cần cùng slot
# Hash tags: {user:1}:profile và {user:1}:sessions → cùng slot
MSET {user:1}:profile data1 {user:1}:sessions data2   # OK
MSET user:1 data1 user:2 data2                         # ERROR — different slots
```

### Cluster Python Client

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host='192.168.1.10', port=7001,
    decode_responses=True,
    skip_full_coverage_check=True,   # allow nếu không full coverage
)

rc.set('user:1', 'nghia')
rc.get('user:1')
```

---

## Comparison: Deployment Modes

| Feature | Standalone | Sentinel | Cluster |
|---------|-----------|----------|---------|
| Availability | No auto-failover | Auto-failover | Auto-failover |
| Scalability | Single node | Single master | Horizontal sharding |
| Max data | RAM của 1 node | RAM của 1 node | N × RAM |
| Multi-key ops | Unrestricted | Unrestricted | Same slot only |
| Complexity | Low | Medium | High |
| Client support | Universal | Needs Sentinel client | Needs Cluster client |
| Use case | Dev/small | Production HA | Large scale |

---

## Gotchas

- **Sentinel quorum = majority**: Với 3 Sentinels, quorum phải >= 2. Dùng 2 Sentinels với quorum=1 là anti-pattern — single Sentinel failure gây no quorum. Luôn dùng odd number (3, 5) Sentinels.
- **Cluster và multi-key commands**: `MGET`, `MSET`, transactions, Lua scripts với keys ở different slots → error. Dùng hash tags `{tag}` để force same slot, hoặc redesign data model.
- **`cluster-require-full-coverage yes`**: Default là `yes` — nếu có slot không có node (vì node crash trước khi replica promote), toàn bộ cluster từ chối reads/writes. Set `no` cho availability over consistency.
- **Failover timeout**: `sentinel failover-timeout` là tổng time budget cho failover. Nếu replica sync chậm và timeout → failover fail và retry. Tune cùng với `min-replicas-max-lag`.
- **Replication lag và READ replica**: Đọc từ replica có thể nhận stale data. Với applications cần read-your-writes consistency, phải đọc từ master hoặc implement client-side logic.
- **Cluster resharding và production traffic**: Resharding di chuyển slots từng key một. Trong quá trình này, clients nhận ASK redirects (chậm hơn MOVED). Plan resharding ngoài giờ cao điểm.
