---
tags:
  - cloudstack
  - lab
  - control-plane
---

# CloudStack Control Plane - Triển khai Management Server HA và Galera Database

- **Bối cảnh và vấn đề**: Control plane của CloudStack (Management Server + Database) là "bộ não" của toàn bộ cụm — chết Management Server thì VM đang chạy vẫn sống, nhưng chết Database thì mọi API call fail hoàn toàn, không quản trị được gì. Một Management Server đơn + một MySQL đơn là single point of failure không chấp nhận được cho production dù nhỏ.
- **Cách giải quyết**: Triển khai 2 node Management Server (converged với HAProxy + keepalived) đứng trước một cụm MariaDB Galera **3 node độc lập** (không converged với MS — tách riêng để DB không cạnh tranh tài nguyên với MS và có quorum 3 thành viên thật, không cần `garbd`). HAProxy trên 2 node MS proxy vào cụm DB theo mô hình **single-writer**: 1 node Galera nhận toàn bộ write qua port riêng, 2 node còn lại chỉ là `backup` (failover tự động khi node write chết) — tránh certification conflict nếu để CloudStack ghi đồng thời vào nhiều node Galera cùng lúc. Một port HAProxy riêng round-robin read qua cả 3 node cho các công cụ đọc/báo cáo khác ngoài CloudStack. Healthcheck dùng script `clustercheck` (qua `xinetd`) để HAProxy biết chính xác node nào đang `Synced`, không route nhầm vào node đang SST/IST giữa chừng. VIP dùng chung cho cả API/UI (443) lẫn MySQL write/read. Import System VM Template và hardening API/DB/firewall theo checklist bảo mật.
- **Kết quả sau khi hoàn thành**: Truy cập UI/API CloudStack qua 1 VIP HA, tắt 1 trong 2 Management Server không mất khả năng quản trị. Galera Cluster 3 node đồng bộ đa hướng với quorum thật (2/3 vote), DB không còn là single point of failure, và HAProxy đảm bảo write luôn đi đúng 1 node tại một thời điểm. Đây là nền cho các lab tiếp theo trong series (xem [[CloudStack Production Cluster - Lab Series Overview]]).

> [!NOTE]
> Lab này giả định dòng **Apache CloudStack 4.19.x/4.20.x** trên **Ubuntu 24.04 (noble)** theo đúng phạm vi ghi chú [[Cloudstack|CloudStack Overview]] trong vault này. Luôn xác nhận lại version chính xác và tên codename repo tại `download.cloudstack.org` trước khi cài — số version có thể đã tiến thêm kể từ lúc viết lab.

> [!NOTE]
> Thiết kế này khác bản nháp đầu (2 node MS converged Galera + `garbd`): giờ tách hẳn 3 node Galera ra khỏi MS để có đủ 3 node dữ liệu thật (quorum 2/3 đúng nghĩa, không cần thành viên "chỉ vote" như `garbd`), đồng thời DB không bị cạnh tranh CPU/RAM với tiến trình Management Server lúc SST/IST. Đánh đổi là tốn thêm 1 node hạ tầng so với bản converged — chấp nhận được vì đây là control plane, không phải nơi cần tối ưu chi phí nhất trong cụm.

> [!NOTE]
> Galera cho phép ghi vào bất kỳ node nào (multi-master), nhưng CloudStack không idempotent-safe với ghi đồng thời đa hướng — ghi cùng lúc vào 2 node khác nhau trên cùng row dễ gây certification conflict, transaction bị Galera rollback ngẫu nhiên phía client mà CloudStack không retry đúng cách. Vì vậy HAProxy ở Bước 7 luôn ép **toàn bộ write qua đúng 1 node** (`server ... check backup` cho 2 node còn lại), không bao giờ round-robin write qua cả 3 node.

## Prerequisites

- **Hạ tầng**: DNS nội bộ và NTP hoạt động; các node resolve được lẫn nhau. Chưa có Zone/Pod/Cluster nào được tạo trên CloudStack (lab này dừng lại trước bước tạo Zone).
- **Máy chủ / VM**: 5 node — cấu hình ví dụ dùng trong lab, điều chỉnh lại theo capacity thực tế:

| Node            | Vai trò                                  | CPU    | RAM   | Disk       |
| --------------- | ----------------------------------------- | ------ | ----- | ---------- |
| cloudstack-ms01 | Management Server + HAProxy + keepalived | 8 vCPU | 16 GB | 100 GB SSD |
| cloudstack-ms02 | Management Server + HAProxy + keepalived | 8 vCPU | 16 GB | 100 GB SSD |
| cloudstack-db01 | MariaDB + Galera node 1                  | 8 vCPU | 16 GB | 100 GB SSD |
| cloudstack-db02 | MariaDB + Galera node 2                  | 8 vCPU | 16 GB | 100 GB SSD |
| cloudstack-db03 | MariaDB + Galera node 3                  | 8 vCPU | 16 GB | 100 GB SSD |

- **Tài khoản và quyền**: sudo trên cả 5 node.
- **Mạng**: dải Management network dùng chung cho MS/DB/VIP đã xin từ team Network — xem placeholder ở Planning table.
- **Kiến thức nền**: giả định đã đọc [[CloudStack Management Server]], [[Database HA - MySQL Galera]], và [[CloudStack HA Architecture]] trong vault này — lab không giải thích lại khái niệm nền.

> [!WARNING]
> Việc tắt 1 node MS hoặc failover VIP trong bước Kiểm tra kết quả không ảnh hưởng VM đang chạy (MS không nằm trong data path), nhưng nếu đây là control plane đang phục vụ user thật, hãy làm ở cửa sổ bảo trì đã thông báo trước.

## Thông tin Planning liên quan

| Thành phần | Giá trị | Ghi chú |
| --- | --- | --- |
| cloudstack-ms01 hostname/IP | `<TBD>` | Management Server + HAProxy + keepalived |
| cloudstack-ms02 hostname/IP | `<TBD>` | Management Server + HAProxy + keepalived |
| cloudstack-db01 hostname/IP | `172.29.70.210` | MariaDB + Galera node 1 (bootstrap node) |
| cloudstack-db02 hostname/IP | `172.29.70.211` | MariaDB + Galera node 2 |
| cloudstack-db03 hostname/IP | `172.29.70.212` | MariaDB + Galera node 3 |
| Management network CIDR | `<TBD>` | SSH, Galera replication, API/UI backend |
| VIP control plane | `<TBD>` | keepalived VIP, dùng chung cho HTTPS API/UI (443), MySQL write (3306) và read (3307) |
| HAProxy stats port | `8404` | Nội bộ, chỉ mở cho Management network |
| Galera healthcheck port | `9200` | `clustercheck` qua `xinetd`, chạy trên cả 3 node DB, chỉ mở cho IP 2 node MS |
| VRRP `auth_pass` | `<sinh bằng openssl rand -hex 16>` | Xác thực giữa 2 node keepalived, không dùng giá trị mặc định trong ví dụ tài liệu |
| CloudStack release | `<xác nhận bản mới nhất tại download.cloudstack.org, tham khảo 4.19.x/4.20.x>` | Pin version cụ thể trước khi thêm apt repo |
| Galera cluster name | `cloudstack-mgt` | |
| SST user | `sstuser` | Dùng bởi `mariabackup` để đồng bộ dữ liệu lúc node join cluster, password sinh bằng `openssl rand` |
| Clustercheck user | `clustercheck` | Chỉ có quyền `PROCESS`, dùng riêng cho script healthcheck, không dùng chung với `sstuser` |
| DB user CloudStack | `cloud` | Least privilege, chỉ thao tác trên DB `cloud`/`cloud_usage` |
| DB root password | `<secret store>` | Chỉ dùng lúc `cloudstack-setup-databases`, không dùng lại về sau |
| Dashboard/API admin user | `admin` | Đổi password ngay sau lần đăng nhập đầu tiên |

## Diagram

```mermaid
flowchart TD
    UI[Admin UI / CloudMonkey] -- "1. HTTPS 443" --> VIP["Control Plane VIP<br/>&lt;vip&gt;"]
    VIP --> MS1["cloudstack-ms01<br/>MS + HAProxy + keepalived"]
    VIP --> MS2["cloudstack-ms02<br/>MS + HAProxy + keepalived"]
    MS1 -- "2. VRRP heartbeat" --> MS2

    MS1 -- "3. write :3306 (active)" --> DB1
    MS1 -. "backup, failover only" .-> DB2
    MS1 -. "backup, failover only" .-> DB3
    MS1 -- "4. read :3307 (round-robin)" --> DB1
    MS1 -- "4. read :3307 (round-robin)" --> DB2
    MS1 -- "4. read :3307 (round-robin)" --> DB3
    MS1 -- "5. clustercheck :9200" --> DB1
    MS1 -- "5. clustercheck :9200" --> DB2
    MS1 -- "5. clustercheck :9200" --> DB3

    DB1["cloudstack-db01<br/>172.29.70.210"] <-. "wsrep replication" .-> DB2["cloudstack-db02<br/>172.29.70.211"]
    DB2 <-. "wsrep replication" .-> DB3["cloudstack-db03<br/>172.29.70.212"]
    DB1 <-. "wsrep replication" .-> DB3
```

---

## Installation

### Bước 1 - Chuẩn bị hệ điều hành Ubuntu 24.04 trên 5 node

`wsrep` (giao thức replication của Galera) rất nhạy với lệch giờ — clock drift là nguyên nhân phổ biến nhất gây lỗi IST/SST khó hiểu lúc join cluster ở Bước 2, nên NTP ở bước này quan trọng không kém gì ở lab Ceph.

- Đặt hostname, khai báo `/etc/hosts`, cài NTP, tạo firewall baseline — thực hiện trên cả 5 node:

```bash
sudo hostnamectl set-hostname <hostname-theo-planning-table>
sudo apt update && sudo apt install -y chrony ufw
sudo systemctl enable chrony --now
timedatectl status | grep "System clock synchronized"
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from <management-cidr> to any port 22 proto tcp
sudo ufw enable
```

- Thêm resolve giữa 5 node vào `/etc/hosts` trên từng node:

```text
<ip-ms01>  cloudstack-ms01
<ip-ms02>  cloudstack-ms02
172.29.70.210  cloudstack-db01
172.29.70.211  cloudstack-db02
172.29.70.212  cloudstack-db03
```

- Kiểm tra kết quả bước này:

```bash
chronyc tracking | grep "Leap status"
ping -c1 cloudstack-ms02
```

Kết quả mong đợi: `Leap status: Normal`, ping resolve đúng IP.

### Bước 2 - Cài đặt cụm MariaDB Galera 3 node (cloudstack-db01/02/03)

Milestone này dựng xong tầng DB trước — Management Server ở Bước 4 cần Galera đã chạy để `cloudstack-setup-databases` kết nối vào. 3 node dữ liệu đầy đủ cho quorum thật (2/3 vote), không cần `garbd`.

- Cài package trên cả 3 node DB (tuyệt đối cùng version MariaDB giữa 3 node — mismatch version giữa các node là nguyên nhân khó debug nhất):

```bash
sudo apt install -y mariadb-server mariadb-client mariadb-backup galera-4
sudo systemctl stop mariadb
```

> [!NOTE]
> `mariadb-backup` bắt buộc cài — đây là engine SST (State Snapshot Transfer) mặc định dùng để đồng bộ full data khi 1 node join cluster, ổn định và không blocking như `rsync`.

- Chỉnh sửa `/etc/mysql/mariadb.conf.d/60-galera.cnf` trên **cả 3 node**, chỉ khác `wsrep_node_address`/`wsrep_node_name`:

```ini
[mysqld]
binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0

wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "cloudstack-mgt"
wsrep_cluster_address = "gcomm://172.29.70.210,172.29.70.211,172.29.70.212"

# --- Đổi 2 dòng này theo từng node ---
wsrep_node_address = "<ip-của-chính-node-này>"
wsrep_node_name = "<hostname-của-chính-node-này>"
# ---------------------------------------

wsrep_sst_method = mariabackup
wsrep_sst_auth = "sstuser:<sst-password-theo-planning-table>"
```

> [!WARNING]
> `innodb_autoinc_lock_mode = 2` bắt buộc phải set — CloudStack DB dùng nhiều bảng `AUTO_INCREMENT`, thiếu dòng này gây certification conflict/deadlock ngẫu nhiên trên Galera (xem lesson learned trong [[Database HA - MySQL Galera]]).

> [!WARNING]
> `wsrep_sst_auth` chứa password dạng plaintext trong file config — `chmod 640` file này, owner `mysql:mysql`, không commit vào git. `bind-address = 0.0.0.0` khiến MariaDB nghe toàn bộ interface, nên bắt buộc phải mở firewall đúng scope ở bước dưới, không được để port lộ ra ngoài Management network.

- Mở firewall cho Galera replication (áp dụng trên cả 3 node DB, chỉ trong Management network) — 4567 (gcomm/replication, cả tcp lẫn udp), 4568 (IST), 4444 (SST), 3306 (client + SST auth):

```bash
sudo ufw allow from <management-cidr> to any port 3306,4567,4568,4444 proto tcp comment 'galera'
sudo ufw allow from <management-cidr> to any port 4567 proto udp comment 'galera gcomm'
```

- Bật mã hoá cho replication traffic giữa các node Galera bằng TLS (nội bộ, dùng self-signed hoặc internal CA vì traffic không rời khỏi Management network):

```ini
# Thêm vào cùng file 60-galera.cnf, dùng cert internal CA đã chuẩn bị sẵn
wsrep_provider_options = "socket.ssl_key=/etc/mysql/ssl/galera-key.pem;socket.ssl_cert=/etc/mysql/ssl/galera-cert.pem;socket.ssl_ca=/etc/mysql/ssl/ca.pem"
```

- Bootstrap cluster **chỉ trên node đầu tiên (cloudstack-db01)**:

```bash
sudo galera_new_cluster
```

> [!WARNING]
> `galera_new_cluster` chỉ chạy đúng 1 lần trên node đầu tiên khi cụm chưa có dữ liệu. Chạy nhầm trên node khác cùng lúc sẽ tạo ra 2 cluster riêng biệt (split-brain), dữ liệu diverge, rollback cực khổ. Nếu sau này toàn bộ cụm cùng down và cần bootstrap lại, phải xác định đúng node có `seqno` cao nhất trong `/var/lib/mysql/grastate.dat` — xem chi tiết cảnh báo trong [[Database HA - MySQL Galera]], bootstrap sai node gây mất dữ liệu mới nhất.

- Verify cluster size = 1, sau đó tạo user `sstuser` (dùng cho SST khi 2 node còn lại join) và `clustercheck` (dùng cho healthcheck ở Bước 3) — chỉ cần tạo 1 lần trên cloudstack-db01, Galera tự đồng bộ sang các node join sau:

```bash
sudo mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"   # expect: 1

sudo mysql <<'EOF'
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY '<sst-password-theo-planning-table>';
GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';

CREATE USER 'clustercheck'@'127.0.0.1' IDENTIFIED BY '<clustercheck-password-theo-planning-table>';
GRANT PROCESS ON *.* TO 'clustercheck'@'127.0.0.1';
FLUSH PRIVILEGES;
EOF
```

- Khởi động node thứ hai và thứ ba để join cluster (**không** dùng `galera_new_cluster` — start bình thường, node tự SST/IST đồng bộ full data từ cluster hiện có):

```bash
# Trên cloudstack-db02 và cloudstack-db03
sudo systemctl start mariadb
```

> [!NOTE]
> Nếu data lớn, SST sẽ block write trên donor node tạm thời — nên join lần đầu vào cluster đang có traffic thật ở cửa sổ bảo trì off-peak.

- Kiểm tra kết quả bước này (chạy trên cả 3 node):

```bash
sudo mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
sudo mysql -e "SHOW STATUS LIKE 'wsrep_ready';"
sudo mysql -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
```

Kết quả mong đợi: `wsrep_cluster_size` = `3`, `wsrep_ready` = `ON`, `wsrep_local_state_comment` = `Synced` trên cả 3 node.

### Bước 3 - Cấu hình `clustercheck` healthcheck trên cloudstack-db01/02/03

HAProxy ở Bước 7 cần biết chính xác node nào đang `Synced` để không route write/read vào node đang SST/IST giữa chừng (dữ liệu chưa đầy đủ) — chỉ check port 3306 sống/chết là không đủ, đây là bước hay bị bỏ qua nhất và gây outage âm thầm nhất trong vận hành Galera.

- Cài `xinetd` để expose 1 endpoint HTTP nhỏ cho HAProxy healthcheck (thực hiện trên cả 3 node DB):

```bash
sudo apt install -y xinetd
```

- Tạo script `/usr/local/bin/clustercheck.sh` (giống nhau trên cả 3 node), dùng user `clustercheck` đã tạo ở Bước 2:

```bash
#!/bin/bash
MYSQL_USER="clustercheck"
MYSQL_PASSWORD="<clustercheck-password-theo-planning-table>"

STATUS=$(mysql -u$MYSQL_USER -p$MYSQL_PASSWORD -h127.0.0.1 -e "SHOW STATUS LIKE 'wsrep_local_state_comment';" 2>/dev/null | awk 'NR==2{print $2}')

if [ "$STATUS" = "Synced" ]; then
    echo -e "HTTP/1.1 200 OK\r\n\r\nMariaDB Cluster Node is Synced."
else
    echo -e "HTTP/1.1 503 Service Unavailable\r\n\r\nMariaDB Cluster Node is $STATUS."
fi
```

```bash
sudo chmod 700 /usr/local/bin/clustercheck.sh
sudo chown mysql:mysql /usr/local/bin/clustercheck.sh
```

> [!WARNING]
> Password của `clustercheck` đang nằm plaintext trong script — quyền file phải là `700`, owner `mysql`. Không dùng chung password với `sstuser` hay `client.admin`-tương-đương nào khác, để nếu file này lộ thì blast radius chỉ giới hạn ở quyền `PROCESS` (không đọc/ghi được dữ liệu).

- Khai báo service cho `xinetd` tại `/etc/xinetd.d/mysqlchk`, giới hạn `only_from` chỉ 2 IP của HAProxy (chính là 2 node MS) — không để mở public vì đây là port lộ trạng thái cluster ra ngoài:

```text
service mysqlchk
{
    disable         = no
    flags           = REUSE
    socket_type     = stream
    port            = 9200
    wait            = no
    user            = nobody
    server          = /usr/local/bin/clustercheck.sh
    log_on_failure  += USERID
    only_from       = <ip-ms01> <ip-ms02>
    per_source      = UNLIMITED
}
```

```bash
sudo systemctl enable --now xinetd
```

- Mở firewall cho port healthcheck, chỉ từ 2 node MS:

```bash
sudo ufw allow from <ip-ms01> to any port 9200 proto tcp comment 'clustercheck from haproxy'
sudo ufw allow from <ip-ms02> to any port 9200 proto tcp comment 'clustercheck from haproxy'
```

- Kiểm tra kết quả bước này (chạy từ cloudstack-ms01/ms02, hoặc `curl` cục bộ trên từng node DB):

```bash
curl -s http://<ip-db01>:9200/
```

Kết quả mong đợi: `HTTP/1.1 200 OK` kèm nội dung `MariaDB Cluster Node is Synced.` trên cả 3 node.

### Bước 4 - Cài đặt CloudStack Management Server trên cloudstack-ms01 và cloudstack-ms02

- Thêm apt repo chính thức của CloudStack (thực hiện trên cả 2 node MS):

```bash
CS_CODENAME=<xác-nhận-tại-download.cloudstack.org>   # ví dụ: noble
CS_VERSION=<xác-nhận-bản-cụ-thể>                      # ví dụ: 4.19
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL http://download.cloudstack.org/release.asc | sudo gpg --dearmor -o /etc/apt/keyrings/cloudstack.gpg
echo "deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://download.cloudstack.org/deb $CS_CODENAME $CS_VERSION main" \
  | sudo tee /etc/apt/sources.list.d/cloudstack.list
sudo apt update
```

- Cài package Management Server:

```bash
sudo apt install -y cloudstack-management
```

- Setup database — chỉ chạy trên **cloudstack-ms01** (khởi tạo schema `cloud`/`cloud_usage` một lần), trỏ thẳng vào node bootstrap của Galera (cloudstack-db01):

```bash
sudo cloudstack-setup-databases cloud:<db-cloud-password>@172.29.70.210:3306 \
  --deploy-as=root:<db-root-password>
```

> [!NOTE]
> Trỏ thẳng vào `cloudstack-db01` chỉ dùng cho bước khởi tạo schema một lần — Galera tự đồng bộ schema sang db02/db03 ngay sau khi tạo. Sau khi setup xong, `db.properties` ở Bước 5 sẽ trỏ qua VIP HAProxy (port write 3306) để có failover — không trỏ cứng vào 1 node như cảnh báo trong [[Database HA - MySQL Galera]].

- Kiểm tra kết quả bước này (chạy `mysql` client từ ms01, kết nối remote vào db01 vì DB không còn chạy local trên node MS):

```bash
sudo mysql -h 172.29.70.210 -ucloud -p -e "SHOW DATABASES;" | grep -E "cloud|cloud_usage"
```

Kết quả mong đợi: thấy cả 2 database `cloud` và `cloud_usage`.

### Bước 5 - Cấu hình `server.properties`/`db.properties` đúng cho từng node

- Trên **cloudstack-ms01**, chạy setup management (sinh keystore, cấu hình mặc định):

```bash
sudo cloudstack-setup-management
```

- Trên **cloudstack-ms02**, cài package rồi copy `db.properties` từ ms01 sang (schema đã tồn tại, không chạy lại `cloudstack-setup-databases`):

```bash
sudo scp cloudstack-ms01:/etc/cloudstack/management/db.properties /etc/cloudstack/management/db.properties
sudo cloudstack-setup-management
```

- Sửa `db.cloud.host` trong `/etc/cloudstack/management/db.properties` trên **cả 2 node**, trỏ qua VIP thay vì IP cố định (VIP sẽ có ở Bước 7). Port `3306` ở đây là **write pool** của HAProxy (single-writer, xem Bước 7) — CloudStack MS chỉ dùng 1 connection pool nên luôn đi qua port write, không dùng port read `3307`:

```properties
db.cloud.host=<vip-control-plane>
db.cloud.port=3306
db.cloud.autoReconnect=true
```

- Sửa `cluster.node.IP` trong `/etc/cloudstack/management/server.properties` — **giá trị này phải khác nhau giữa 2 node**, đúng bằng IP của chính node đó:

```properties
# Trên cloudstack-ms01
cluster.node.IP=<ip-ms01>
```

```properties
# Trên cloudstack-ms02
cluster.node.IP=<ip-ms02>
```

> [!WARNING]
> Copy nguyên `server.properties` từ node này sang node kia mà quên sửa `cluster.node.IP` khiến cả 2 MS đăng ký nhầm địa chỉ trong bảng `mshost` — job bị "kẹt" ở trạng thái pending một cách khó hiểu. Đây là lesson learned đã ghi trong [[CloudStack Management Server]]. Luôn kiểm tra `SELECT * FROM cloud.mshost;` sau bước này.

- Kiểm tra kết quả bước này:

```bash
sudo mysql -h 172.29.70.210 -ucloud -p -e "SELECT id, name, service_ip, state FROM cloud.mshost;"
```

Kết quả mong đợi: 2 dòng, mỗi dòng `service_ip` đúng IP tương ứng của từng node.

### Bước 6 - Import System VM Template

Bắt buộc trước khi tạo Zone ở lab sau — thiếu bước này, SSVM/CPVM/VR không bao giờ khởi tạo được dù Zone đã cấu hình xong.

- Chạy trên **cloudstack-ms01**, trỏ vào NFS Secondary Storage đã dựng ở [[Ceph Cluster - Triển khai Primary và Secondary Storage cho CloudStack]] (mount tạm để import, không cần giữ mount sau khi xong):

```bash
sudo mkdir -p /mnt/secondary
sudo mount -t nfs4 <nfs-vip>:/cloudstack-secondary /mnt/secondary

/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
  -m /mnt/secondary \
  -u <url-systemvm-template-kvm-đúng-version-CloudStack> \
  -h kvm -F

sudo umount /mnt/secondary
```

> [!NOTE]
> URL system VM template phải khớp đúng dòng version CloudStack đang cài — lấy từ trang release notes chính thức tương ứng với `$CS_VERSION` ở Bước 4, không dùng URL của version khác.

- Kiểm tra kết quả bước này:

```bash
sudo mysql -h 172.29.70.210 -u cloud -p -e "SELECT name, state FROM cloud.vm_template WHERE type='SYSTEM';"
```

Kết quả mong đợi: có ít nhất 1 dòng template `state = Ready`.

### Bước 7 - Triển khai HAProxy + keepalived cho VIP control plane

- Cài đặt trên cả 2 node MS:

```bash
sudo apt install -y haproxy keepalived
```

- Cấu hình `/etc/haproxy/haproxy.cfg` (giống nhau trên cả 2 node). Gồm 4 khối: frontend API (HTTPS terminate tại HAProxy rồi re-encrypt xuống backend 8443 để giữ mã hoá end-to-end), write pool Galera (single-writer, 2 node còn lại chỉ là `backup` — failover tự động, không round-robin write để tránh certification conflict như đã cảnh báo ở đầu bài), read pool Galera (round-robin cả 3 node, dùng cho công cụ đọc/báo cáo khác ngoài CloudStack), và trang stats để theo dõi trạng thái từng node real-time:

```ini
global
    stats socket /var/run/haproxy/admin.sock mode 660 level admin

#---------------------------------------------------------------------
# CloudStack API/UI
#---------------------------------------------------------------------
frontend cloudstack_api
    bind *:443 ssl crt /etc/haproxy/certs/control-plane.pem
    default_backend cloudstack_ms

backend cloudstack_ms
    balance roundrobin
    option httpchk GET /client/api?command=listCapabilities
    server ms01 <ip-ms01>:8443 check ssl verify none
    server ms02 <ip-ms02>:8443 check ssl verify none

#---------------------------------------------------------------------
# Galera Cluster - Write pool (single active writer, 2 node backup)
#---------------------------------------------------------------------
listen galera_write
    bind *:3306
    mode tcp
    option tcpka
    option httpchk
    option tcplog
    http-check expect status 200
    default-server port 9200 inter 2s fall 3 rise 2 on-marked-down shutdown-sessions
    server db01 172.29.70.210:3306 check
    server db02 172.29.70.211:3306 check backup
    server db03 172.29.70.212:3306 check backup

#---------------------------------------------------------------------
# Galera Cluster - Read pool (round-robin toàn bộ node Synced)
#---------------------------------------------------------------------
listen galera_read
    bind *:3307
    mode tcp
    balance roundrobin
    option tcpka
    option httpchk
    option tcplog
    http-check expect status 200
    default-server port 9200 inter 2s fall 3 rise 2
    server db01 172.29.70.210:3306 check
    server db02 172.29.70.211:3306 check
    server db03 172.29.70.212:3306 check

#---------------------------------------------------------------------
# Stats page - theo dõi node nào UP/DOWN real-time
#---------------------------------------------------------------------
listen galera_stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 5s
    stats auth admin:<haproxy-stats-password-theo-planning-table>
```

> [!NOTE]
> Healthcheck port `9200` chính là script `clustercheck` qua `xinetd` đã cấu hình trên cả 3 node DB ở Bước 3 — trả HTTP 200 khi node Galera ở trạng thái `Synced`, HTTP 503 khi không, nhờ đó HAProxy tự loại node đang desync (kể cả đang SST/IST giữa chừng) khỏi backend mà không cần can thiệp tay.

> [!NOTE]
> `backup` trên db02/db03 ở write pool nghĩa là chúng chỉ nhận traffic **khi db01 down** (failover tự động, không round-robin write). `on-marked-down shutdown-sessions` cắt luôn session cũ đang treo trên node vừa bị mark down, tránh app retry vô ích vào connection chết. `fall 3 rise 2` — phải fail 3 lần liên tiếp mới mark down (tránh false positive do network glitch tạm thời) nhưng chỉ cần pass 2 lần là mark up lại.

> [!WARNING]
> Failback: khi db01 sống lại sau sự cố, HAProxy sẽ tự đưa nó về làm writer chính ngay khi healthcheck pass — nhưng lúc đó db01 vừa join lại có thể đang SST/IST, chưa Synced xong. Vì healthcheck đã check đúng trạng thái `Synced` (không chỉ port sống/chết) nên case này về cơ bản đã được che chắn; vẫn nên theo dõi `wsrep_local_state_comment` thủ công sau mỗi lần failback để chắc chắn.

- Validate syntax trước khi reload, đừng reload mù:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

- Cấu hình `/etc/keepalived/keepalived.conf` trên **cloudstack-ms01** (MASTER):

```ini
vrrp_instance CS_VIP {
    state MASTER
    interface <management-nic>
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <vrrp-auth-pass-theo-planning-table>
    }
    virtual_ipaddress {
        <vip-control-plane>/<prefix>
    }
}
```

- Trên **cloudstack-ms02** (BACKUP), giống hệt nhưng `state BACKUP` và `priority 100`.

> [!WARNING]
> `auth_pass` VRRP chỉ dài tối đa 8 ký tự theo giới hạn giao thức VRRPv2 — không phải nơi lưu secret mạnh, chỉ chống thiết bị lạ vô tình tham gia nhóm VRRP cùng VLAN. Không dùng lại giá trị này cho mục đích bảo mật khác.

- Mở firewall trên cả 2 node MS cho các port HAProxy vừa cấu hình, chỉ trong Management network — port `8404` (stats) nên siết chặt hơn, chỉ cho IP admin/jumphost thay vì cả dải mạng:

```bash
sudo ufw allow from <management-cidr> to any port 443,3306,3307 proto tcp comment 'cloudstack api + galera lb'
sudo ufw allow from <admin-jumphost-cidr> to any port 8404 proto tcp comment 'haproxy stats'
```

- Khởi động và kiểm tra kết quả bước này:

```bash
sudo systemctl enable haproxy keepalived --now
ip addr show <management-nic> | grep <vip-control-plane>
curl -k https://<vip-control-plane>/client/api?command=listCapabilities
mysql -h <vip-control-plane> -P3306 -ucloud -p -e "SELECT @@hostname;"   # phải luôn trả về hostname của node write đang active
mysql -h <vip-control-plane> -P3307 -ucloud -p -e "SELECT @@hostname;"   # chạy vài lần, thấy hostname đổi round-robin qua cả 3 node
```

Kết quả mong đợi: VIP xuất hiện trên node MASTER, `curl` trả về JSON capabilities thay vì connection refused, port `3306` luôn trả về cùng 1 hostname (writer), port `3307` xoay vòng qua cả 3 node DB.

### Bước 8 - Hardening API, UI và firewall

- Chặn truy cập trực tiếp vào từng MS (8080/8443), chỉ cho phép từ chính cặp HAProxy và Management network — traffic thật phải luôn đi qua VIP:

```bash
sudo ufw allow from <ip-ms01>,<ip-ms02> to any port 8080,8443 proto tcp comment 'internal LB healthcheck only'
sudo ufw deny 8080,8443 proto tcp comment 'block direct access, force via VIP'
```

- Vô hiệu hoá Integration API port — đây là cổng API không cần chữ ký HMAC, chỉ nên bật tạm thời khi cần debug/script nội bộ đáng tin cậy, không để mặc định mở:

```bash
cmk -u https://<vip-control-plane>/client/api list configurations name=integration.api.port
cmk -u https://<vip-control-plane>/client/api updateConfiguration name=integration.api.port value=
sudo systemctl restart cloudstack-management
```

> [!WARNING]
> `integration.api.port` khi bật cho phép gọi API **không cần ký HMAC** — bất kỳ ai reach được port này trên Management network đều có thể gọi API với quyền admin. Chỉ bật tạm thời khi thật sự cần, và luôn tắt lại (để trống) ngay sau khi dùng xong.

- Đổi password tài khoản `admin` mặc định ngay lần đăng nhập đầu, và rà soát các Global Setting liên quan tới giới hạn số lần đăng nhập sai/session timeout trước khi mở UI cho người dùng khác:

```bash
cmk -u https://<vip-control-plane>/client/api list configurations keyword=login
cmk -u https://<vip-control-plane>/client/api list configurations keyword=session
```

> [!NOTE]
> Tên chính xác của từng Global Setting có thể khác nhau giữa các minor version — dùng 2 lệnh `list configurations keyword=...` ở trên để xác nhận key hiện có trên đúng version đang chạy thay vì đoán tên, rồi set giá trị phù hợp với chính sách bảo mật của tổ chức (số lần đăng nhập sai tối đa, thời gian khoá, session timeout).

- Kiểm tra kết quả bước này:

```bash
curl -k https://<ip-ms01>:8443/client/api?command=listCapabilities
```

Kết quả mong đợi: connection bị từ chối/timeout khi gọi trực tiếp vào IP node (không qua VIP), xác nhận firewall đã chặn đúng.

### Khai báo thông tin nhạy cảm

- Các giá trị nhạy cảm trong lab này: `<db-cloud-password>`, `<db-root-password>`, `<sst-password>`, `<clustercheck-password>`, `<haproxy-stats-password>`, `<vrrp-auth-pass>`, password tài khoản `admin` trên UI. Toàn bộ được sinh bằng `openssl rand -base64 20` (riêng `auth_pass` giới hạn 8 ký tự theo giao thức VRRP), lưu vào secret store của tổ chức, không hardcode trong file cấu hình public hay script tự động hoá:

```bash
openssl rand -base64 20 > /root/db-cloud.pass
openssl rand -base64 20 > /root/db-root.pass
openssl rand -base64 20 > /root/galera-sst.pass
openssl rand -base64 20 > /root/galera-clustercheck.pass
openssl rand -base64 20 > /root/haproxy-stats.pass
chmod 600 /root/db-cloud.pass /root/db-root.pass /root/galera-sst.pass /root/galera-clustercheck.pass /root/haproxy-stats.pass
```

> [!WARNING]
> `sstuser` và `clustercheck` phải dùng 2 password khác nhau, và khác với `db-root-password`/`db-cloud-password` — mỗi user chỉ nên có blast radius đúng bằng quyền của nó (`clustercheck` chỉ có `PROCESS`, `sstuser` chỉ có quyền phục vụ SST). Dùng chung 1 password cho nhiều tài khoản khiến việc rotate về sau thành ác mộng và tăng blast radius nếu 1 file config bị lộ.

- `db.properties` chứa password DB dạng plaintext trên cả 2 node MS — quyền file phải là `600`, chỉ user chạy `cloudstack-management` đọc được. `60-galera.cnf` (chứa `wsrep_sst_auth`) và `clustercheck.sh` (chứa password `clustercheck`) trên 3 node DB cũng phải giới hạn quyền tương tự — đã nêu cụ thể ở Bước 2 và Bước 3:

```bash
sudo chmod 600 /etc/cloudstack/management/db.properties
```

## Kiểm tra kết quả

- Login UI qua VIP, xác nhận version đúng dòng đã pin:

```bash
cmk -u https://<vip-control-plane>/client/api list infos filter=cloudstackversion
```

- Test failover: dừng `cloudstack-management` trên node đang giữ VIP, xác nhận UI vẫn truy cập được qua VIP (đã chuyển sang node còn lại):

```bash
sudo systemctl stop cloudstack-management   # chạy trên node đang là keepalived MASTER
curl -k https://<vip-control-plane>/client/api?command=listCapabilities
sudo systemctl start cloudstack-management  # khôi phục lại sau khi xác nhận
```

  | Hạng mục cần kiểm tra | Cách kiểm tra | Kết quả đúng |
  | --- | --- | --- |
  | Galera cluster quorum | `SHOW STATUS LIKE 'wsrep_cluster_size';` trên cả 3 node DB | `3` |
  | Cả 2 MS đăng ký đúng IP | `SELECT * FROM cloud.mshost;` | 2 dòng, `service_ip` đúng từng node |
  | System VM template sẵn sàng | `SELECT name,state FROM cloud.vm_template WHERE type='SYSTEM';` | `state = Ready` |
  | VIP hoạt động khi 1 MS down | `curl -k https://<vip>/client/api?command=listCapabilities` sau khi stop 1 MS | Vẫn trả JSON, không lỗi |
  | Write pool luôn đi đúng 1 node | `mysql -h <vip> -P3306 -e "SELECT @@hostname;"` lặp lại nhiều lần | Luôn cùng 1 hostname (node writer đang active) |
  | Write pool failover khi node writer chết | Stop mariadb trên node writer, lặp lại query write pool | Chuyển sang 1 trong 2 node backup, không downtime kéo dài |
  | Read pool round-robin | `mysql -h <vip> -P3307 -e "SELECT @@hostname;"` lặp lại nhiều lần | Hostname xoay vòng qua cả 3 node |
  | HAProxy stats page | `curl -s -u admin:<haproxy-stats-password> http://<ip-ms01>:8404/` | HTTP 200, thấy đủ 3 backend db01/02/03 |

## Troubleshooting

Không áp dụng - lab dựng mới theo hướng dẫn triển khai chuẩn, chưa có log lỗi thực tế phát sinh trong quá trình build để ghi nhận.

## Rollback

- Gỡ HAProxy/keepalived (VIP biến mất, mất khả năng truy cập control plane qua 1 địa chỉ duy nhất):

```bash
sudo systemctl disable --now haproxy keepalived
```

- Gỡ Management Server (không ảnh hưởng dữ liệu DB):

```bash
sudo systemctl stop cloudstack-management
sudo apt remove --purge -y cloudstack-management
```

- Gỡ `clustercheck`/`xinetd` trên 3 node DB (HAProxy sẽ mark toàn bộ backend down ngay sau bước này, chỉ làm khi đã dừng hẳn):

```bash
sudo systemctl disable --now xinetd   # trên db01, db02, db03
```

- Gỡ Galera — chỉ thực hiện nếu chắc chắn không còn cần dữ liệu:

```bash
sudo systemctl stop mariadb   # trên db01, db02, db03
```

> [!CAUTION]
> Xoá dữ liệu MySQL (`/var/lib/mysql`) trên cả 3 node DB cùng lúc làm mất toàn bộ state của CloudStack (Zone, VM, network, account...) không thể khôi phục nếu chưa backup. Chỉ xoá sau khi đã `mariabackup` đầy đủ, theo hướng dẫn backup trong [[Database HA - MySQL Galera]].

## Reference

- [Apache CloudStack - Installation Guide](https://docs.cloudstack.apache.org/en/latest/installguide/index.html)
- [Apache CloudStack - Management Server package repository](https://docs.cloudstack.apache.org/en/latest/installguide/management-server/index.html)
- [MariaDB - Galera Cluster - Getting Started](https://mariadb.com/kb/en/getting-started-with-mariadb-galera-cluster/)
- [MariaDB - mariabackup SST method](https://mariadb.com/kb/en/mariabackup-sst-method/)
- [Percona/MariaDB - clustercheck script reference](https://github.com/olafz/percona-clustercheck)
- [HAProxy - Configuration Manual](https://docs.haproxy.org/)
- [Keepalived - VRRP configuration](https://www.keepalived.org/manpage.html)
- Ghi chú liên quan trong vault: [[CloudStack Management Server]] | [[Database HA - MySQL Galera]] | [[CloudStack HA Architecture]] | [[Key Configuration Reference]]
