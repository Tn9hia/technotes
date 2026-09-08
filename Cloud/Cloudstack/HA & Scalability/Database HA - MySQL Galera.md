---
tags:
  - cloudstack
  - ha
  - database
---

# Database HA — MySQL/MariaDB Galera

CloudStack lưu **toàn bộ state** (Zone, VM, network, account, job...) trong MySQL/MariaDB — đây là **thành phần quan trọng nhất cần bảo vệ** trong toàn bộ hệ thống. Mất DB gần như tương đương mất "não" của cả cloud.

> [!warning] Mức độ nghiêm trọng cần khắc sâu
> Khác vCenter (DB chủ yếu phục vụ inventory/lịch sử, ESXi vẫn tự chủ nếu vCenter chết), CloudStack MS **không hoạt động được gì cả** nếu mất kết nối DB — mọi API call fail. VM đang chạy vẫn sống (do hypervisor tự quản), nhưng bạn **mù hoàn toàn** về khả năng quản trị cho tới khi DB được khôi phục.

## Kiến trúc Galera Cluster

```
   MS #1 ──┐
   MS #2 ──┼──► Galera Node 1 ◄──sync multi-master──► Galera Node 2 ◄──► Galera Node 3
   MS #3 ──┘         (thường qua HAProxy/ProxySQL đặt trước, hoặc MS trỏ trực tiếp danh sách node)
```

- Galera là **multi-master, synchronous replication** — khác Master-Slave truyền thống, mọi node đều ghi được (nhưng thực tế nên route write qua 1 node tại 1 thời điểm để tránh certification conflict).
- Cần **số node lẻ ≥ 3** để tránh split-brain (quorum).

```bash
# Cài đặt cơ bản (Ubuntu/Debian, MariaDB Galera)
apt install mariadb-server galera-4 mariadb-backup

# my.cnf quan trọng
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://node1,node2,node3"
wsrep_cluster_name="cloudstack_galera"
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```

> [!warning] Lesson learned: `innodb_autoinc_lock_mode` sai gây lỗi ID trùng khó hiểu
> CloudStack DB dùng nhiều bảng với `AUTO_INCREMENT`. Trên Galera, nếu không set `innodb_autoinc_lock_mode=2` (interleaved), có thể xảy ra **certification conflict** hoặc deadlock khi nhiều MS ghi đồng thời — biểu hiện là job CloudStack fail ngẫu nhiên với lỗi DB mơ hồ, rất khó liên tưởng tới nguyên nhân gốc là cấu hình Galera.

## Vận hành thường ngày

```bash
# Kiểm tra trạng thái cluster (chạy trên từng node)
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mysql -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"   # phải là "Synced"
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_status';"        # phải là "Primary"

# Bootstrap cluster sau khi TOÀN BỘ node cùng down (cold start)
galera_new_cluster   # CHỈ chạy trên node có dữ liệu mới nhất!
```

> [!warning] Lesson learned nghiêm trọng nhất: bootstrap sai node = mất dữ liệu mới nhất
> Khi toàn bộ cụm Galera cùng bị tắt (mất điện toàn bộ datacenter chẳng hạn), việc "bootstrap" lại cluster **phải chọn đúng node có `seqno` (sequence number) cao nhất** trong file `grastate.dat`. Bootstrap nhầm node cũ hơn sẽ khiến các node khác **đồng bộ đè theo node cũ đó khi join lại**, gây mất toàn bộ thay đổi xảy ra sau thời điểm đó dù dữ liệu thật sự mới hơn vẫn còn nguyên trên node khác. **Luôn kiểm tra `cat /var/lib/mysql/grastate.dat` trên tất cả node trước khi quyết định bootstrap từ node nào**, tuyệt đối không đoán hay chọn node đầu tiên khởi động lại được.

## Backup

```bash
# mysqldump (đơn giản, cần downtime nhẹ hoặc dùng --single-transaction)
mysqldump -u root -p --single-transaction --databases cloud cloud_usage > cloudstack_backup.sql

# Hoặc mariabackup (hot backup, không khóa bảng, khuyến nghị cho production)
mariabackup --backup --target-dir=/backup/full
```

> [!tip] Tần suất & retention khuyến nghị
> DB CloudStack thường nhỏ (vài trăm MB đến vài GB tùy quy mô), nên backup **nhiều lần trong ngày** là hoàn toàn khả thi và nên làm — chi phí thấp, giá trị phục hồi rất cao so với kích thước. Đừng chỉ backup 1 lần/ngày cho DB điều khiển toàn bộ hạ tầng cloud.

## Kết nối MS tới Galera

```properties
# db.properties trên mỗi Management Server
db.cloud.host=<VIP hoặc danh sách qua HAProxy>
db.cloud.autoReconnect=true
db.cloud.name=cloud
```

> [!tip] Nên đặt HAProxy/ProxySQL phía trước Galera thay vì trỏ thẳng MS vào 1 node cố định
> Nếu MS trỏ cứng vào IP của 1 Galera node cụ thể, node đó chết sẽ làm **toàn bộ MS mất kết nối DB** dù 2 node còn lại vẫn "Synced" và sẵn sàng phục vụ. Đặt 1 lớp LB (HAProxy với healthcheck `wsrep_local_state`, hoặc ProxySQL) phía trước giúp MS tự động failover sang node còn sống.

---
*Xem thêm: [[CloudStack HA Architecture]] | [[CloudStack Management Server]] | [[CloudStack Day 2 Operations]] | [[Cloudstack|CloudStack]]*
