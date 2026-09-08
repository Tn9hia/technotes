---
tags:
  - ceph
  - mon
  - architecture
---

# MON - Monitor

**MON (Monitor)** là daemon giữ và phân phối các **cluster map** (monmap, osdmap, crushmap, mgrmap...) — "nguồn sự thật" (source of truth) về trạng thái toàn cụm. Các MON hoạt động theo cơ chế **quorum** dùng thuật toán Paxos, không phải một daemon đơn lẻ như nhiều người mới hình dung.

> [!tip] So với VMware vSAN
> MON **không phải** "Ceph's vCenter" theo nghĩa vCenter là một điểm quản lý duy nhất — MON luôn chạy thành **cụm lẻ (3/5/7)** đạt đồng thuận qua Paxos, không có single point kiểu vCenter. Mất quorum MON giống tinh thần "vCenter down" ở chỗ: bạn **không mất VM/dữ liệu đang chạy** (I/O giữa client và OSD vẫn tiếp tục theo map đã cache), nhưng **mọi thay đổi cấu hình cluster đều bị khóa** — không tạo pool mới, không map RBD mới, không xử lý OSD lên/xuống đúng cách. Khác vCenter, MON không phải một máy chạy hệ điều hành đầy đủ mà là daemon nhẹ, gắn với một RocksDB store cục bộ trên mỗi node.

## Vai trò cụ thể

- Giữ **monmap, osdmap, pgmap (qua MGR), crushmap, mgrmap** — cung cấp cho client/daemon khi được hỏi.
- Xác thực (authenticate) client/daemon qua **cephx**.
- Theo dõi OSD báo cáo lẫn nhau (peer failure reports) để quyết định đánh dấu OSD down.
- Là nơi duy nhất **ghi thay đổi cấu hình cluster** (thêm pool, sửa CRUSH rule, thay đổi cluster-wide config qua `ceph config`).

MON **không** nằm trong đường I/O dữ liệu — client đọc/ghi object trực tiếp với OSD sau khi lấy map ban đầu (xem [[RADOS & Cluster Architecture]]), MON chỉ tham gia khi cần map mới hoặc đổi cấu hình.

## Vì sao luôn số lẻ (3, 5, 7)

MON dùng **Paxos** để đạt đồng thuận — cần **quá bán (majority)** số MON đồng ý mới commit được 1 thay đổi map. Với `n` MON, cluster chịu được tối đa `floor((n-1)/2)` MON chết mà vẫn còn quorum:

| Số MON | Chịu được mất tối đa | Ghi chú |
|---|---|---|
| 1 | 0 | Không có HA — chỉ dùng cho lab/demo, **không dùng cho production** |
| 3 | 1 | Tối thiểu khuyến nghị cho production |
| 5 | 2 | Phổ biến cho cluster lớn hoặc cần chịu lỗi 1 rack/AZ |
| 7 | 3 | Hiếm khi cần, thêm MON không tăng hiệu năng, chỉ tăng độ chịu lỗi và overhead Paxos |

Số **chẵn** không có lợi ích gì thêm so với số lẻ liền trước (4 MON vẫn chỉ chịu được mất 1 như 3 MON, nhưng tốn thêm 1 node và tăng round-trip Paxos) — vì vậy Ceph luôn khuyến nghị số lẻ.

```bash
# Xem trạng thái quorum hiện tại
ceph quorum_status --format json-pretty

# Xem nhanh MON stat
ceph mon stat

# Xem chi tiết từng MON
ceph mon dump
```

## MON store — nơi lưu state, và vì sao nó có thể phình to bất thường

Mỗi MON lưu dữ liệu trong **RocksDB** tại:

```
/var/lib/ceph/<fsid>/mon.<hostname>/store.db
```

(với triển khai `cephadm`, đường dẫn này nằm trong container namespace của MON daemon, có thể xem qua `cephadm shell` hoặc `ceph orch ps` để tìm đúng host).

Store này giữ lịch sử các map (osdmap qua nhiều epoch gần nhất) để phục vụ recovery và catch-up cho các daemon/MON bị lag. Ceph tự động **trim (compact)** các epoch cũ khi cluster ở trạng thái `HEALTH_OK` và toàn bộ PG đều `active+clean`.

> [!warning] Lesson learned: "mon is using a lot of disk space" gần như luôn là triệu chứng của PG không clean kéo dài, không phải lỗi MON
> Cảnh báo `HEALTH_WARN` "mon is using a lot of disk space" khiến nhiều người nghĩ MON store bị lỗi hoặc cần tăng disk. Thực tế nguyên nhân phổ biến nhất: **MON không thể trim map cũ vì có PG bị stuck không clean trong thời gian dài** (degraded/undersized/incomplete kéo dài do OSD down lâu, hoặc cluster liên tục backfill do topology thay đổi liên tục) — MON buộc phải giữ lại lịch sử map để các OSD "chưa bắt kịp" vẫn có thể catch-up. Cách xử lý đúng là **tìm và giải quyết PG không clean** (`ceph pg dump_stuck`, `ceph health detail`), không phải tăng dung lượng đĩa cho MON hoặc restart MON — restart MON không giải quyết gốc rễ và có thể làm mất quorum tạm thời nếu làm không cẩn thận.

```bash
# Kiểm tra nguyên nhân gốc khi thấy cảnh báo mon disk space
ceph health detail
ceph pg dump_stuck
ceph report | grep -A5 "num_pg_by_state"
```

## Thêm/bớt MON an toàn qua cephadm

Với các cụm dùng `cephadm` (chuẩn hiện nay), không thao tác tay bằng `ceph-mon` như thời `ceph-deploy` cũ — dùng orchestrator:

```bash
# Xem MON đang chạy ở đâu
ceph orch ps --daemon-type mon

# Khai báo rõ danh sách host muốn chạy MON (khuyến nghị số lẻ, trải rack/AZ khác nhau)
ceph orch apply mon --placement="mon-a,mon-b,mon-c"

# Hoặc để cephadm tự chọn N host (không khuyến nghị cho production — nên chỉ định rõ host)
ceph orch apply mon 3

# Gỡ 1 MON cụ thể (cephadm tự rm sau khi cập nhật monmap)
ceph orch daemon rm mon.mon-old-host --force
```

> [!warning] Lesson learned: thêm/bớt MON đồng loạt nhiều host cùng lúc = mất quorum tạm thời không cần thiết
> Khi migrate MON sang host mới (vd thay hardware), luôn làm **tuần tự từng cái một** — thêm MON mới, đợi nó join quorum ổn định (`ceph quorum_status` xác nhận), rồi mới gỡ MON cũ. Gỡ nhiều MON cùng lúc hoặc thêm/gỡ đồng thời dễ khiến cluster tạm thời rơi xuống dưới ngưỡng majority, dù chỉ trong vài giây — đủ để block mọi thao tác cluster đang chạy dở (tạo pool, orchestrator action...).

---
*Xem thêm: [[MGR - Manager]] | [[RADOS & Cluster Architecture]] | [[Ceph HA Architecture]] | [[Ceph|Ceph]]*
