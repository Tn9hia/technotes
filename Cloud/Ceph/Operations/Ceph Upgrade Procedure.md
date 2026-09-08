---
tags:
  - ceph
  - operations
  - upgrade
---

# Ceph Upgrade Procedure

Với **cephadm**, upgrade là **orchestrated rolling upgrade** — cephadm tự đẩy image mới xuống từng daemon theo đúng thứ tự an toàn, không cần bạn tự SSH vào từng node như thời `ceph-deploy` (đã obsolete). Nhưng "tự động" không có nghĩa là "an toàn bất chấp trạng thái cluster" — checklist trước khi bấm nút mới là phần quan trọng nhất.

## Nguyên tắc chung

- Ceph chỉ hỗ trợ upgrade **tuần tự N → N+1 major release**, không được nhảy cóc (Reef 18 → Squid 19 → Tentacle 20, không được Reef → Tentacle trực tiếp).
- cephadm tự quyết định **thứ tự upgrade từng daemon** dựa trên failure domain và vai trò — bạn không tự chọn thứ tự này.
- Không có cơ chế **rollback tự động** một khi OSD đã chuyển sang on-disk format mới — xem phần Rollback bên dưới.

> [!tip] So với VMware vSAN
> vSphere/vSAN cho phép mix version ESXi trong cluster tạm thời khi rolling upgrade qua vLCM, và có snapshot/rollback tương đối dễ ở tầng hypervisor. Ceph cephadm cũng rolling-upgrade từng daemon một, nhưng **không có "undo" ở tầng on-disk format** — một khi OSD ghi dữ liệu ở format mới, quay lại binary cũ có thể không đọc được nữa. Tư duy "cứ thử, không được thì rollback" của vSAN không áp dụng được ở đây.

## Thứ tự upgrade tự động của cephadm — và tại sao

```
1. MGR (Manager)       → upgrade trước tiên, vì chính MGR là daemon điều khiển tiến trình upgrade
2. MON (Monitor)       → upgrade từng MON một, giữ quorum trong suốt quá trình
3. OSD                 → upgrade theo batch nhỏ, tôn trọng CRUSH failure domain
                          (không bao giờ đưa xuống 2 OSD replica cùng object cùng lúc)
4. MDS (nếu dùng CephFS)
5. RGW (nếu dùng Object Gateway)
6. Các daemon phụ khác (crash, node-exporter, prometheus, alertmanager...)
```

| Vì sao thứ tự này quan trọng | Giải thích |
|---|---|
| MGR trước | MGR chứa orchestrator module (`cephadm`) — nếu MGR không hoạt động đúng, tiến trình upgrade các daemon còn lại không tự lái được |
| MON trước OSD | MON giữ cluster map (osdmap, crushmap...) — cần đồng thuận quorum ổn định trước khi đụng vào OSD |
| OSD theo batch nhỏ, theo failure domain | Tránh đưa nhiều OSD cùng PG (cùng replica) offline cùng lúc → tránh mất khả năng phục vụ I/O của PG đó |
| MDS/RGW sau cùng | Đây là service layer phía trên RADOS — chỉ nên cập nhật sau khi core (MON/OSD) đã ổn định trên version mới |

## Checklist trước khi upgrade (bắt buộc)

- [ ] `ceph -s` phải là **HEALTH_OK** — không upgrade khi đang HEALTH_WARN/HEALTH_ERR (xem lesson learned bên dưới)
- [ ] Đọc kỹ Release Notes của version đích — đặc biệt phần "Upgrade" và "Deprecations" (config option bị đổi tên/xóa, behavior thay đổi)
- [ ] Xác nhận `ceph osd require-osd-release <release>` đã match version hiện tại trước khi bắt đầu (đảm bảo cluster không còn "di sản" từ release cũ hơn nữa)
- [ ] Backup MON DB trên ít nhất 1 MON (`ceph-mon` store.db) — dự phòng nếu quá trình upgrade MON gặp sự cố nghiêm trọng
- [ ] Kiểm tra dung lượng trống trên các node (image container mới cần tải về, chiếm thêm disk tạm thời)
- [ ] Test trên môi trường staging giống production nếu có điều kiện — đặc biệt nếu cluster có customize config khác default
- [ ] Thông báo maintenance window, đặc biệt nếu đang throttle recovery/backfill theo giờ hành chính (xem [[Ceph Performance Tuning]])

```bash
# Backup nhanh MON store trước khi upgrade (chạy trên 1 MON node)
cephadm shell -- ceph-mon -i <mon-id> --extract-monmap /tmp/monmap.bak
tar czf mon-store-backup-$(date +%F).tar.gz /var/lib/ceph/<fsid>/mon.<mon-id>/store.db

# Xác nhận osd require-osd-release đã đúng release hiện tại
ceph osd require-osd-release reef
```

## Quy trình upgrade

```bash
# 1. Xác nhận cluster khỏe mạnh tuyệt đối trước khi bắt đầu
ceph -s
ceph health detail

# 2. Xem các version daemon hiện tại đang chạy (phát hiện daemon nào bị bỏ sót từ trước)
ceph orch upgrade check --image quay.io/ceph/ceph:v19

# 3. Bắt đầu upgrade tới image/version đích
ceph orch upgrade start --image quay.io/ceph/ceph:v19

# 4. Theo dõi tiến trình liên tục
ceph orch upgrade status
ceph -s  # health nên duy trì HEALTH_OK hoặc chỉ HEALTH_WARN tạm thời do daemon đang restart

# 5. Nếu cần tạm dừng (VD: phát hiện bất thường giữa chừng) mà chưa muốn hủy
ceph orch upgrade pause

# 6. Resume lại khi đã xác nhận an toàn
ceph orch upgrade resume

# 7. Nếu cần hủy hẳn (chỉ dừng daemon CHƯA upgrade, daemon đã upgrade rồi giữ nguyên)
ceph orch upgrade stop
```

| Lệnh | Ý nghĩa |
|---|---|
| `ceph orch upgrade start --image <target>` | Bắt đầu rolling upgrade tới image chỉ định |
| `ceph orch upgrade status` | Xem tiến trình: daemon nào đã xong, đang làm, còn lại |
| `ceph orch upgrade pause` | Tạm dừng — daemon đang dở sẽ hoàn tất, không đẩy tiếp daemon mới |
| `ceph orch upgrade resume` | Tiếp tục từ điểm dừng |
| `ceph orch upgrade stop` | Hủy tiến trình — cluster ở trạng thái **mixed version** cho tới khi bạn hoàn tất hoặc downgrade lại thủ công |

> [!warning] Lesson learned: bấm upgrade khi cluster đang HEALTH_WARN với degraded PG
> Một cluster có vài PG `degraded` do OSD vừa restart, admin chủ quan nghĩ "chỉ warning nhỏ, không ảnh hưởng gì" và chạy `ceph orch upgrade start` luôn. Kết quả: cephadm upgrade OSD theo batch, nhưng logic chờ PG healthy trước khi tiếp tục batch kế tiếp bị **kẹt vô thời hạn** — vì các PG degraded đó không bao giờ tự lành trong khi code cũ/mới đang trộn lẫn giữa các OSD replica của cùng PG. Upgrade treo giữa chừng, cluster ở trạng thái mixed-version kéo dài nhiều giờ trước khi phát hiện ra nguyên nhân gốc. Bài học: **luôn đưa cluster về HEALTH_OK tuyệt đối trước khi bắt đầu**, không chỉ "gần OK". Xem thêm [[Recovery, Backfill & Self-healing]].

## Version skip policy

| Từ | Đến | Được phép? |
|---|---|---|
| Reef (18.x) | Squid (19.x) | Có — N → N+1 |
| Squid (19.x) | Tentacle (20.x) | Có — N → N+1 |
| Reef (18.x) | Tentacle (20.x) trực tiếp | **Không** — phải qua Squid trước |

Ceph chính thức chỉ hỗ trợ và test upgrade qua từng major release liên tiếp. Nhảy cóc không được hỗ trợ, không có nghĩa là "chắc chắn fail" nhưng **không nằm trong test matrix chính thức** — không nên mạo hiểm trên production.

## Rollback — thực tế phũ phàng

> [!warning] Ceph KHÔNG dễ rollback sau khi đã upgrade
> Khác với việc snapshot VM rồi revert, một khi OSD đã khởi động lại với binary mới và ghi dữ liệu theo on-disk format mới (BlueStore metadata, OMAP format...), **downgrade binary về version cũ có thể khiến OSD không khởi động được hoặc không đọc được dữ liệu đã ghi**. "Rollback" thực chất không phải lưới an toàn đáng tin cậy ở đây. Lưới an toàn thật sự là: **test kỹ ở staging trước + backup dữ liệu quan trọng ở tầng ứng dụng (RBD snapshot, RGW versioning...)** — không phải kỳ vọng downgrade Ceph binary khi có sự cố.

Nếu upgrade thất bại giữa chừng: cách xử lý thực tế là **tiếp tục sửa lỗi và hoàn tất upgrade tiến về phía trước** (fix rồi resume), không phải lùi lại.

## Sau khi upgrade

- [ ] `ceph -s` về HEALTH_OK, `ceph versions` xác nhận mọi daemon đã cùng 1 version
- [ ] `ceph osd require-osd-release <new-release>` để chốt release mới (mở khóa các tính năng/optimizations chỉ dùng được khi toàn cluster đã ở release mới)
- [ ] Test I/O cơ bản: tạo RBD image thử, kiểm tra CloudStack vẫn provision/snapshot volume bình thường (xem [[Ceph with CloudStack]])
- [ ] Theo dõi `ceph -W cephadm` và dashboard sát sao 24-48h đầu
- [ ] Cập nhật baseline config nếu Release Notes có đổi default option nào ảnh hưởng tới cluster (xem [[Key Configuration Reference (Ceph)]])

---
*Xem thêm: [[Ceph Day 2 Operations]] | [[Recovery, Backfill & Self-healing]] | [[Lessons Learned & Common Pitfalls (Ceph)]] | [[Ceph|Ceph]]*
