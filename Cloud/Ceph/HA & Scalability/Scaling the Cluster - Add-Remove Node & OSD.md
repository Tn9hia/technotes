---
tags:
  - ceph
  - scalability
  - cephadm
---

# Scaling the Cluster - Add/Remove Node & OSD

Scale Ceph ra (thêm node/OSD) hay vào (remove) đều là thao tác **thường xuyên** trong vòng đời vận hành — không phải sự kiện hiếm. Với cephadm, quy trình phần lớn khai báo (declarative) qua `ceph orch`, nhưng **thứ tự thao tác** khi remove vẫn quan trọng để tránh rebalance hai lần không cần thiết.

> [!tip] So với VMware vSAN
> Thêm host/disk-group vào vSAN cluster và để vSAN tự rebalance là thao tác quen thuộc — Ceph có tinh thần tương tự khi add OSD (CRUSH tự phân bố lại PG). Khác biệt lớn: Ceph cho bạn **nhiều knob điều khiển tốc độ** rebalance hơn hẳn (throttle số lượng backfill đồng thời, ưu tiên client I/O...) — vSAN phần lớn để bạn chọn "policy" tổng quát chứ không chi tiết bằng.

## Thêm OSD host mới

```bash
# 1. Copy SSH key cephadm sang host mới, add vào cluster
ssh-copy-id -f -i /etc/ceph/ceph.pub root@osd-node-05
ceph orch host add osd-node-05 <IP> --labels osd

# 2. Kiểm tra disk trống mà cephadm nhìn thấy trên host mới
ceph orch device ls osd-node-05

# 3a. Cách nhanh — tạo OSD trên toàn bộ disk trống chưa dùng
ceph orch apply osd --all-available-devices

# 3b. Cách khuyến nghị cho production — dùng Drive Group spec (khai báo YAML,
# kiểm soát rõ device nào làm data, device nào làm DB/WAL)
```

```yaml
# drive-group-hdd.yaml — ví dụ 1 NVMe chia sẻ DB/WAL cho 8 HDD trên mỗi host label "osd"
service_type: osd
service_id: hdd-with-nvme-db
placement:
  label: "osd"
spec:
  data_devices:
    rotational: 1        # chọn HDD làm data device
  db_devices:
    rotational: 0         # chọn NVMe/SSD làm DB/WAL device
  db_slots: 8              # 1 NVMe chia sẻ DB cho tối đa 8 OSD HDD
```

```bash
ceph orch apply -i drive-group-hdd.yaml
ceph orch ls          # theo dõi service osd đang deploy
ceph osd tree          # xác nhận OSD mới đã lên "up/in" và nằm đúng vị trí CRUSH
```

## Remove 1 OSD an toàn — đúng thứ tự

```bash
# BƯỚC 1: đánh dấu OSD "out" — CRUSH ngừng gán PG mới vào OSD này,
# và bắt đầu backfill dữ liệu hiện có của nó sang OSD khác
ceph osd out osd.23

# BƯỚC 2: theo dõi cho tới khi rebalance hoàn tất (PG rời khỏi osd.23 hết)
ceph -s
ceph pg dump | grep 23   # hoặc ceph osd safe-to-destroy osd.23

# BƯỚC 3: CHỈ SAU KHI rebalance xong, mới stop và remove daemon
ceph orch osd rm 23 --replace   # hoặc không kèm --replace nếu bỏ hẳn slot đó
ceph orch osd rm status          # theo dõi tiến trình rm
```

> [!warning] Sai thứ tự = rebalance 2 lần không cần thiết
> Nếu bạn **stop/remove daemon OSD trước** khi đánh `out` và chờ backfill xong, Ceph sẽ coi đó là một sự kiện mất OSD đột ngột: nó phải (1) đánh dấu các PG liên quan `degraded` ngay lập tức và bắt đầu recovery khẩn cấp từ bản sao còn lại, rồi sau đó khi bạn hoàn tất việc xóa OSD khỏi CRUSH, (2) lại phải rebalance thêm lần nữa để lấp đúng vị trí trống trong CRUSH map. Thứ tự đúng (`out` → chờ backfill xong → mới remove daemon) khiến toàn bộ quá trình di dời dữ liệu diễn ra **một lần, có kiểm soát**, ít gây áp lực đột ngột lên cluster hơn.

## Drain và remove cả một host

```bash
# Đưa toàn bộ OSD trên host vào trạng thái drain — cephadm tự out từng OSD,
# đợi backfill, rồi mới cho phép remove host
ceph orch host drain osd-node-02

# Theo dõi tiến trình
ceph orch osd rm status

# Khi toàn bộ OSD trên host đã drain xong (không còn OSD nào thuộc host này)
ceph orch host rm osd-node-02
```

## Kỳ vọng về thời gian rebalance & cách throttle

Thời gian rebalance phụ thuộc lượng dữ liệu di dời, băng thông cluster network, và các knob throttle (`osd_max_backfills`, `osd_recovery_max_active`...). Không có con số cố định — với vài TB trên cluster 10GbE có thể mất vài giờ, với hàng chục TB có thể mất cả ngày nếu không tăng throttle. Chi tiết đầy đủ về các tham số điều khiển tốc độ recovery/backfill xem ở [[Recovery, Backfill & Self-healing]] — không lặp lại ở đây, nhưng cần nhớ nguyên tắc: **throttle thấp hơn khi cluster đang phục vụ tải production cao điểm, throttle cao hơn khi có cửa sổ bảo trì rảnh rỗi**.

## Scale số lượng MON theo quy mô cluster

| Quy mô cluster | Số MON khuyến nghị | Lý do |
|---|---|---|
| Nhỏ (< 10 node) | 3 | Đủ tolerance cho quy mô nhỏ, tiết kiệm tài nguyên |
| Vừa (10-30 node) | 3 (vẫn đủ), cân nhắc 5 nếu trải nhiều rack/room | Tăng MON không tăng hiệu năng — chỉ tăng tolerance chịu lỗi |
| Lớn hoặc trải nhiều failure domain vật lý (nhiều rack/phòng) | 5 | Chịu được mất 2 MON cùng lúc, hữu ích khi 1 rack mất điện toàn bộ mà rack đó có MON |

```bash
# Tăng số MON — cephadm tự chọn host phù hợp theo placement spec
ceph orch apply mon --placement="5 mon-01,mon-02,mon-03,mon-04,mon-05"
```

Luôn giữ số MON là **số lẻ** — số chẵn không tăng khả năng chịu lỗi so với n-1 gần nhất mà chỉ tốn thêm tài nguyên và tăng latency Paxos nhẹ.

> [!warning] Lesson learned: thêm một loạt OSD host trống cùng lúc, rebalance storm làm chậm toàn bộ VM
> Một đợt mở rộng cluster thêm 4 host OSD trống hoàn toàn cùng lúc (tổng dung lượng mới gần bằng 40% dung lượng cluster hiện có), rồi chạy `ceph orch apply osd --all-available-devices` cho toàn bộ 4 host một lần. CRUSH ngay lập tức tính toán lại phân bố và bắt đầu di chuyển một khối lượng dữ liệu rất lớn để lấp đầy các OSD mới — với throttle mặc định không được điều chỉnh, backfill traffic chiếm phần lớn băng thông cluster network, kéo theo latency I/O của VM đang chạy trên CloudStack tăng rõ rệt trong nhiều giờ liền, dù không có OSD nào thực sự "chết" (đây là do balancer/CRUSH chủ động cân bằng lại tự nhiên khi thêm capacity). Bài học: khi thêm capacity lớn, nên (1) thêm dần từng host một thay vì tất cả cùng lúc, (2) hạ tạm `osd_max_backfills` trước khi thêm loạt OSD mới rồi tăng dần lại theo dõi tải, hoặc (3) thêm ngoài giờ cao điểm và giám sát sát I/O client trong suốt quá trình rebalance.

---
*Xem thêm: [[Recovery, Backfill & Self-healing]] | [[Ceph HA Architecture]] | [[CRUSH Algorithm & CRUSH Map]] | [[Ceph|Ceph]]*
