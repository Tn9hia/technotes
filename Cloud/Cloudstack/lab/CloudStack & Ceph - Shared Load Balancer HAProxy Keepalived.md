---
tags:
  - cloudstack
  - ceph
  - lab
  - haproxy
  - keepalived
  - load-balancer
---

# CloudStack & Ceph - Shared Load Balancer HAProxy + Keepalived

- **Bối cảnh và vấn đề**: Cả CloudStack (API/UI, Galera DB) lẫn Ceph (Dashboard) đều cần một VIP HA đứng trước để không phụ thuộc vào 1 node duy nhất. Dựng riêng một cặp HAProxy + keepalived converged trên mỗi nhóm node (như 2 bản nháp trước của [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]]) nghĩa là nhân bản cùng một tầng hạ tầng nhiều lần cho cùng một chức năng — lãng phí không cần thiết ở quy mô lab/production nhỏ này.
- **Cách giải quyết**: Dựng đúng **1 cặp node `cs-lb-01/02`** chạy HAProxy + keepalived, phục vụ chung cho 3 nhóm backend hoàn toàn khác nhau trên cùng 1 VIP, phân biệt bằng port: `443` (CloudStack UI/API, backend `cs-mgt-01/02`), `3306`/`3307` (Galera write/read, backend `cs-db-01/02/03`), `8443` (Ceph Dashboard, backend `ceph-01/02/03`). Healthcheck cho Galera dùng lại `clustercheck` đã cấu hình ở lab Control Plane; healthcheck cho Ceph Dashboard dựa vào việc chuyển `standby_behaviour` của mgr module từ `redirect` (mặc định) sang `error`, để HAProxy tự loại các mgr đang standby chỉ bằng `httpchk` thông thường, không cần logic phức tạp.
- **Kết quả sau khi hoàn thành**: 1 VIP HA duy nhất phục vụ 3 dịch vụ, tắt 1 trong 2 node `cs-lb` không mất khả năng truy cập bất kỳ dịch vụ nào trong 3 nhóm trên.

> [!NOTE]
> Đây là lab tách ra từ [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]] để dùng chung được cho cả [[Ceph Cluster - Triển khai Primary và Secondary Storage cho CloudStack]] — không lặp lại nội dung cấu hình Galera/CloudStack MS/Ceph Dashboard tại đây, lab này chỉ tập trung vào tầng LB đứng trước.

> [!NOTE]
> Đánh đổi của thiết kế dùng chung: `cs-lb-01/02` trở thành một điểm hạ tầng có "blast radius" rộng hơn — sự cố ở tầng LB (bug HAProxy, mất cả 2 node cùng lúc) ảnh hưởng đồng thời CloudStack UI/API, DB, và Ceph Dashboard, thay vì chỉ 1 trong 3. Đây là đánh đổi hợp lý cho quy mô lab/production nhỏ (ít node, ưu tiên tiết kiệm hạ tầng); ở quy mô lớn hơn nên tách lại LB riêng theo từng domain (CloudStack riêng, Storage riêng) để giảm blast radius.

## Prerequisites

- **Hạ tầng**: Không bắt buộc các lab backend (Control Plane, Ceph) phải xong trước — lab này chỉ cần biết trước IP dự kiến của `cs-mgt-01/02`, `cs-db-01/02/03`, `ceph-01/02/03` để điền vào `haproxy.cfg`, đúng tinh thần "chạy song song, điền placeholder trước" của cả series. Tuy nhiên **kiểm tra kết quả cuối bài** chỉ chạy được sau khi các lab backend đó đã xong.
- **Máy chủ / VM**: 2 node mới:

| Node     | Vai trò              | CPU    | RAM  | Disk      |
| -------- | --------------------- | ------ | ---- | --------- |
| cs-lb-01 | HAProxy + keepalived  | 4 vCPU | 8 GB | 40 GB SSD |
| cs-lb-02 | HAProxy + keepalived  | 4 vCPU | 8 GB | 40 GB SSD |

- **Tài khoản và quyền**: sudo trên cả 2 node.
- **Mạng**: cả 2 node cần reach được cả Management network (CloudStack MS + Galera) lẫn Ceph Mgt/Public network — nếu 2 dải mạng này tách biệt, `cs-lb-01/02` cần 2 NIC hoặc route hợp lệ giữa 2 dải; đơn giản nhất là đặt `cs-lb-01/02` cùng 1 dải mạng "quản trị chung" (Ops network) có route tới cả hai.
- **Kiến thức nền**: giả định đã đọc [[CloudStack HA Architecture]] và phần Galera trong [[Database HA - MySQL Galera]].

## Thông tin Planning liên quan

| Thành phần | Giá trị | Ghi chú |
| --- | --- | --- |
| cs-lb-01 hostname/IP | `<TBD>` | HAProxy + keepalived MASTER |
| cs-lb-02 hostname/IP | `<TBD>` | HAProxy + keepalived BACKUP |
| VIP dùng chung | `<TBD>` | Đây là **nguồn sự thật** cho `<vip-control-plane>` dùng trong [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]] và cho VIP truy cập Ceph Dashboard |
| Ops/Management network CIDR | `<TBD>` | Dải mạng `cs-lb-01/02` dùng để reach backend + VRRP heartbeat |
| Port CloudStack UI/API | `443` → backend `<ip-cs-mgt-01>,<ip-cs-mgt-02>:8443` | |
| Port Galera write | `3306` → backend `cs-db-01` (active), `cs-db-02/03` (backup) | Single-writer, xem cảnh báo certification conflict ở lab Control Plane |
| Port Galera read | `3307` → backend `cs-db-01/02/03` round-robin | |
| Port Ceph Dashboard | `8443` → backend `<ip-ceph-01>,<ip-ceph-02>,<ip-ceph-03>:8443` | Chỉ mgr active trả `200`, xem Bước 5 |
| Galera healthcheck port | `9200` | `clustercheck`, đã cấu hình `only_from cs-lb-01/02` ở lab Control Plane Bước 3 |
| HAProxy stats port | `8404` | Nội bộ, chỉ mở cho IP admin/jumphost |
| VRRP `auth_pass` | `<sinh bằng openssl rand -hex 16, giới hạn 8 ký tự>` | Xác thực giữa 2 node keepalived |
| Cert CloudStack UI (443) | `<TBD - internal CA>` | |
| Cert Ceph Dashboard (8443 frontend) | `<TBD - internal CA>` | Có thể dùng chung cert với backend Ceph Dashboard nếu cùng CA, hoặc terminate riêng |

## Diagram

```mermaid
flowchart TD
    Admin[Admin UI/CloudMonkey] -- "443" --> VIP["VIP dùng chung<br/>&lt;vip&gt;"]
    Browser[Ops - Ceph Dashboard] -- "8443" --> VIP
    DBClient[Reporting/Read tool] -- "3307" --> VIP
    CSMgmt[CloudStack MS] -- "3306 write" --> VIP

    VIP --> LB1["cs-lb-01<br/>HAProxy + keepalived MASTER"]
    VIP -.-> LB2["cs-lb-02<br/>HAProxy + keepalived BACKUP"]
    LB1 -- "VRRP heartbeat" --> LB2

    LB1 -- "443 → 8443" --> MS1[cs-mgt-01]
    LB1 -- "443 → 8443" --> MS2[cs-mgt-02]

    LB1 -- "3306 active" --> DB1[cs-db-01]
    LB1 -. "3306 backup" .-> DB2[cs-db-02]
    LB1 -. "3306 backup" .-> DB3[cs-db-03]
    LB1 -- "3307 roundrobin" --> DB1
    LB1 -- "3307 roundrobin" --> DB2
    LB1 -- "3307 roundrobin" --> DB3
    LB1 -- "clustercheck :9200" --> DB1
    LB1 -- "clustercheck :9200" --> DB2
    LB1 -- "clustercheck :9200" --> DB3

    LB1 -- "8443 (chỉ mgr active pass httpcheck)" --> C1[ceph-01]
    LB1 -. "8443 (standby → 503 do standby_behaviour=error)" .-> C2[ceph-02]
    LB1 -. .-> C3[ceph-03]
```

---

## Installation

### Bước 1 - Chuẩn bị hệ điều hành trên cs-lb-01/02

```bash
sudo hostnamectl set-hostname <hostname-theo-planning-table>
sudo apt update && sudo apt install -y chrony ufw
sudo systemctl enable chrony --now
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from <ops-network-cidr> to any port 22 proto tcp
sudo ufw enable
```

- Kiểm tra kết quả bước này:

```bash
chronyc tracking | grep "Leap status"
ping -c1 cs-lb-02
```

### Bước 2 - Cài đặt HAProxy và keepalived

```bash
sudo apt install -y haproxy keepalived
```

### Bước 3 - Cấu hình `/etc/haproxy/haproxy.cfg` (giống nhau trên cả 2 node)

Gồm 5 khối: frontend CloudStack API (TLS terminate rồi re-encrypt xuống backend 8443, giữ mã hoá end-to-end), write pool Galera (single-writer), read pool Galera (round-robin), backend Ceph Dashboard (chỉ route tới mgr đang active), và trang stats:

```ini
global
    stats socket /var/run/haproxy/admin.sock mode 660 level admin

#---------------------------------------------------------------------
# CloudStack API/UI
#---------------------------------------------------------------------
frontend cloudstack_api
    bind *:443 ssl crt /etc/haproxy/certs/cloudstack-ui.pem
    default_backend cloudstack_ms

backend cloudstack_ms
    balance roundrobin
    option httpchk GET /client/api?command=listCapabilities
    server cs-mgt-01 <ip-cs-mgt-01>:8443 check ssl verify none
    server cs-mgt-02 <ip-cs-mgt-02>:8443 check ssl verify none

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
    server cs-db-01 172.29.70.210:3306 check
    server cs-db-02 172.29.70.211:3306 check backup
    server cs-db-03 172.29.70.212:3306 check backup

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
    server cs-db-01 172.29.70.210:3306 check
    server cs-db-02 172.29.70.211:3306 check
    server cs-db-03 172.29.70.212:3306 check

#---------------------------------------------------------------------
# Ceph Dashboard - chỉ mgr đang active trả 200, standby trả lỗi
# (yêu cầu mgr/dashboard/standby_behaviour=error, xem Bước 5)
#---------------------------------------------------------------------
frontend ceph_dashboard
    bind *:8443 ssl crt /etc/haproxy/certs/ceph-dashboard.pem
    default_backend ceph_dashboard_mgr

backend ceph_dashboard_mgr
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server ceph-01 <ip-ceph-01>:8443 check ssl verify none
    server ceph-02 <ip-ceph-02>:8443 check ssl verify none
    server ceph-03 <ip-ceph-03>:8443 check ssl verify none

#---------------------------------------------------------------------
# Stats page - theo dõi node nào UP/DOWN real-time
#---------------------------------------------------------------------
listen lb_stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 5s
    stats auth admin:<haproxy-stats-password-theo-planning-table>
```

> [!NOTE]
> Healthcheck port `9200` chính là script `clustercheck` đã cấu hình trên cả 3 node DB ở [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]] Bước 3 — trả HTTP 200 khi node Galera ở trạng thái `Synced`, HTTP 503 khi không. Nhớ đã đổi `only_from` trong `xinetd` của 3 node DB thành IP `cs-lb-01/02` (không phải MS) — nếu quên đổi, healthcheck ở đây sẽ luôn fail.

> [!NOTE]
> `backup` trên `cs-db-02/03` ở write pool nghĩa là chúng chỉ nhận traffic **khi cs-db-01 down** (failover tự động, không round-robin write). `on-marked-down shutdown-sessions` cắt luôn session cũ đang treo trên node vừa bị mark down.

> [!WARNING]
> Backend `ceph_dashboard_mgr` route đều `roundrobin` tới cả 3 node — điều làm HAProxy tự loại 2 node standby không phải là `balance` mà là `httpchk`/`http-check expect status 200` kết hợp với việc mgr standby trả về `503`/lỗi thay vì redirect `303` (mặc định). Nếu quên Bước 5 (`standby_behaviour=error`), HAProxy coi cả 3 node đều "healthy" (vì `303` vẫn được server trả, HAProxy TCP-level connect vẫn OK trừ khi dùng `http-check expect status 200` — mà `303` sẽ fail đúng expect này) — trên thực tế `expect status 200` đã tự loại node trả `303`, nhưng vẫn nên xác nhận lại hành vi thật ở Bước 5/Kiểm tra kết quả để chắc chắn không phụ thuộc ngầm vào chi tiết implementation dễ đổi giữa version.

- Validate syntax trước khi reload, đừng reload mù:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

### Bước 4 - Cấu hình keepalived cho VIP dùng chung

- Trên **cs-lb-01** (MASTER):

```ini
vrrp_instance SHARED_VIP {
    state MASTER
    interface <ops-nic>
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <vrrp-auth-pass-theo-planning-table>
    }
    virtual_ipaddress {
        <vip-dung-chung>/<prefix>
    }
}
```

- Trên **cs-lb-02** (BACKUP), giống hệt nhưng `state BACKUP` và `priority 100`.

> [!WARNING]
> `auth_pass` VRRP chỉ dài tối đa 8 ký tự theo giới hạn giao thức VRRPv2 — không phải nơi lưu secret mạnh, chỉ chống thiết bị lạ vô tình tham gia nhóm VRRP cùng VLAN.

- Khởi động và kiểm tra:

```bash
sudo systemctl enable haproxy keepalived --now
ip addr show <ops-nic> | grep <vip-dung-chung>
```

### Bước 5 - Cấu hình Ceph mgr để healthcheck Dashboard hoạt động đúng

Chạy trên node Ceph có label `_admin` (`ceph-01`, theo [[Ceph Cluster - Triển khai Primary và Secondary Storage cho CloudStack]]) — đổi hành vi standby mgr từ redirect (mặc định) sang trả lỗi thẳng, để HAProxy tự loại đúng node không phải active bằng `http-check expect status 200` thông thường, không cần logic follow-redirect phức tạp:

```bash
sudo ceph config set mgr mgr/dashboard/standby_behaviour error
```

- Kiểm tra kết quả bước này (chạy trực tiếp vào từng node Ceph, không qua VIP):

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://<ip-ceph-01>:8443/
curl -sk -o /dev/null -w "%{http_code}\n" https://<ip-ceph-02>:8443/
curl -sk -o /dev/null -w "%{http_code}\n" https://<ip-ceph-03>:8443/
```

Kết quả mong đợi: đúng 1 node trả `200` (mgr active), 2 node còn lại trả về mã lỗi (không phải `303` redirect).

### Bước 6 - Mở firewall trên các backend, chỉ cho phép từ cs-lb-01/02

- Trên `cs-mgt-01/02` — đã cấu hình ở [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]] Bước 7, xác nhận lại rule cho phép `8080,8443` từ `cs-lb-01/02`.
- Trên `cs-db-01/02/03` — đã cấu hình ở lab Control Plane Bước 3 (`only_from` cho port `9200`), xác nhận thêm port `3306` cho phép từ `cs-lb-01/02`:

```bash
sudo ufw allow from <ip-cs-lb-01>,<ip-cs-lb-02> to any port 3306 proto tcp comment 'galera from shared lb'
```

- Trên `ceph-01/02/03` — mở port `8443` (Dashboard) chỉ cho `cs-lb-01/02`, thu hẹp lại so với việc mở cho toàn bộ Mgt/Public network như bản thiết kế cũ:

```bash
sudo ufw allow from <ip-cs-lb-01>,<ip-cs-lb-02> to any port 8443 proto tcp comment 'ceph dashboard from shared lb'
```

- Trên chính `cs-lb-01/02`, mở firewall cho các port frontend:

```bash
sudo ufw allow from <ops-network-cidr> to any port 443,3306,3307,8443 proto tcp comment 'shared lb frontends'
sudo ufw allow from <admin-jumphost-cidr> to any port 8404 proto tcp comment 'haproxy stats'
```

## Kiểm tra kết quả

  | Hạng mục cần kiểm tra | Cách kiểm tra | Kết quả đúng |
  | --- | --- | --- |
  | VIP xuất hiện trên node MASTER | `ip addr show <ops-nic>` | Có `<vip-dung-chung>` |
  | CloudStack UI/API qua VIP | `curl -k https://<vip>/client/api?command=listCapabilities` | Trả JSON |
  | Galera write pool luôn 1 node | `mysql -h <vip> -P3306 -e "SELECT @@hostname;"` lặp lại | Luôn cùng hostname |
  | Galera read pool round-robin | `mysql -h <vip> -P3307 -e "SELECT @@hostname;"` lặp lại | Hostname xoay vòng |
  | Ceph Dashboard qua VIP | `curl -k https://<vip>:8443/` | HTTP 200, load đúng Dashboard của mgr active |
  | Failover LB | Stop `keepalived` trên node MASTER, lặp lại 3 test trên | VIP chuyển sang node còn lại, không mất dịch vụ nào trong 3 nhóm |
  | HAProxy stats page | `curl -s -u admin:<haproxy-stats-password> http://<ip-cs-lb-01>:8404/` | HTTP 200, thấy đủ backend `cloudstack_ms`, `galera_write`, `galera_read`, `ceph_dashboard_mgr` |

## Troubleshooting

Không áp dụng - lab dựng mới theo hướng dẫn triển khai chuẩn, chưa có log lỗi thực tế phát sinh trong quá trình build để ghi nhận.

## Rollback

```bash
sudo systemctl disable --now haproxy keepalived
```

> [!NOTE]
> Gỡ cặp LB này làm mất VIP cho cả CloudStack UI/API, Galera, và Ceph Dashboard cùng lúc — do đây là hạ tầng dùng chung. Chỉ rollback khi đã có phương án truy cập tạm thời trực tiếp vào từng node backend (IP tĩnh), hoặc khi phá bỏ toàn bộ lab.

## Reference

- [HAProxy - Configuration Manual](https://docs.haproxy.org/)
- [Keepalived - VRRP configuration](https://www.keepalived.org/manpage.html)
- [Ceph Dashboard - mgr/dashboard module options](https://docs.ceph.com/en/latest/mgr/dashboard/)
- Ghi chú liên quan trong vault: [[CloudStack Control Plane - Triển khai Management Server HA và Galera Database]] | [[Ceph Cluster - Triển khai Primary và Secondary Storage cho CloudStack]] | [[CloudStack HA Architecture]]
