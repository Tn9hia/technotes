---
tags:
  - openstack
  - ha
  - database
  - mariadb
  - galera
---

# Database HA — Galera Cluster

MariaDB Galera Cluster cung cấp **synchronous multi-master replication** cho OpenStack database.

## Galera Replication Model

```
Write vào Ctrl1               Write vào Ctrl2
     │                               │
  Certify                         Certify
     │ ←── Broadcast writeset ─────► │
     │ ←── Broadcast writeset ─────► │
     │                               │
  Commit (nếu không conflict)    Commit
     │                               │
  Ctrl3 nhận và apply
```

- **Synchronous**: transaction commit chỉ khi TẤT CẢ nodes xác nhận
- **Multi-master**: có thể write vào bất kỳ node nào
- **No split-brain**: quorum mechanism đảm bảo consistency

## Cài đặt

```bash
# Ubuntu
apt install mariadb-server galera-4

# /etc/mysql/mariadb.conf.d/galera.cnf
[mysqld]
binlog_format = ROW
default-storage-engine = innodb
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0
innodb_buffer_pool_size = 4G  # Tune theo RAM

[galera]
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "openstack_galera"
wsrep_cluster_address = "gcomm://ctrl1,ctrl2,ctrl3"
wsrep_sst_method = mariabackup  # hoặc rsync

# Node-specific
wsrep_node_address = "10.0.0.11"
wsrep_node_name = "ctrl1"
wsrep_sst_auth = "galera:galera_password"
```

## Bootstrap Cluster (lần đầu)

```bash
# CHỈ trên node đầu tiên
galera_new_cluster

# Kiểm tra
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_size';"
# Phải là 1

# Trên ctrl2, ctrl3
systemctl start mariadb

# Kiểm tra lại
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_size';"
# Phải là 3
```

## Kiểm tra Health

```bash
# Status variables quan trọng
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep%';" | grep -E \
  'wsrep_cluster_size|wsrep_local_state_comment|wsrep_ready|wsrep_connected|wsrep_flow_control'

# Giải thích:
# wsrep_cluster_size = 3        ← tất cả nodes trong cluster
# wsrep_local_state_comment = Synced  ← node này synced
# wsrep_ready = ON              ← ready nhận transactions
# wsrep_connected = ON          ← kết nối với cluster
# wsrep_flow_control_paused = 0 ← không bị throttle
```

## HAProxy cho MySQL

HAProxy cần biết node nào healthy để route queries:

```bash
# haproxy.cfg
listen mysql_cluster
    bind *:3306
    option tcp-check
    # CHỈ gửi traffic vào node đang synced
    tcp-check connect port 9200
    tcp-check expect string Galera\ node\ is\ synced
    server ctrl1 10.0.0.11:3306 check port 9200 inter 5s rise 2 fall 3
    server ctrl2 10.0.0.12:3306 check inter 5s rise 2 fall 3 backup
    server ctrl3 10.0.0.13:3306 check inter 5s rise 2 fall 3 backup
```

```bash
# Clustercheck script (port 9200)
# /usr/bin/clustercheck
# Returns 200 OK nếu Galera node synced
apt install -y xinetd
# Cài mysqlchk vào xinetd
```

## SST (State Snapshot Transfer)

Khi node join cluster hoặc bị lag quá nhiều:

```bash
# mariabackup SST (hot backup, không lock tables)
# Yêu cầu tạo user galera
mysql -e "CREATE USER 'galera'@'localhost' IDENTIFIED BY 'galera_password';"
mysql -e "GRANT RELOAD,PROCESS,LOCK TABLES,REPLICATION CLIENT ON *.* TO 'galera'@'localhost';"
```

## Recover Cluster sau sự cố

### Scenario: tất cả nodes shutdown an toàn

```bash
# Xem seqno trên từng node (node có seqno cao nhất phải bootstrap)
cat /var/lib/mysql/grastate.dat
# safe_to_bootstrap: 1  ← node này có thể bootstrap

# Bootstrap từ node có safe_to_bootstrap=1 hoặc seqno cao nhất
galera_new_cluster  # trên node đó
```

### Scenario: cluster split hoặc crash

```bash
# Tìm node có seqno cao nhất
grep 'seqno' /var/lib/mysql/grastate.dat
# Hoặc
mysqld_safe --wsrep-recover 2>&1 | grep "Recovered position"

# Set safe_to_bootstrap=1 trên node đó
sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat

galera_new_cluster
```

> [!danger] Galera split-brain
> Nếu cluster bị split và cả 2 sides tiếp tục write, data sẽ diverge. Không thể merge tự động. Phải chọn 1 side và SST lại bên kia.

## Performance Tuning

```ini
[mysqld]
# Buffer pool: 70-80% RAM nếu MySQL dùng riêng
innodb_buffer_pool_size = 16G
innodb_buffer_pool_instances = 8  # 1 per GB pool

# Galera write set cache
wsrep_provider_options = "gcache.size=2G"  # Cho phép IST thay SST

# Connections
max_connections = 1000
wait_timeout = 3600
interactive_timeout = 3600
```

## Backup

```bash
# Backup với mariabackup (không lock)
mariabackup --backup \
  --target-dir=/backup/$(date +%Y%m%d) \
  --user=root \
  --password=secret

# Prepare backup
mariabackup --prepare \
  --target-dir=/backup/20240101

# Hoặc dùng mysqldump (lock tables)
mysqldump --all-databases --single-transaction > backup.sql
```

---
*Xem thêm: [[HA Architecture]] | [[Pacemaker & Corosync]] | [[Operations/Troubleshooting]]*
