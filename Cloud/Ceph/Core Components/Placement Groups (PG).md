---
tags:
  - ceph
  - pg
  - architecture
---

# Placement Groups (PG)

**Placement Group (PG)** là lớp trung gian giữa "object" và "OSD" — mỗi object không được map trực tiếp vào 1 OSD, mà được băm vào 1 PG trước, rồi CRUSH mới map PG đó vào một tập OSD cụ thể. Đây là khái niệm **hoàn toàn mới** với người quen VMware/vSAN (không có khái niệm tương đương), và cũng là điểm gây bối rối nhất khi mới học Ceph — nhưng lại là thứ khiến Ceph vận hành hiệu quả ở quy mô lớn.

> [!tip] So với VMware vSAN
> vSAN không có khái niệm tương đương PG. Trong vSAN, mỗi object (VMDK) được chia thành các **component** và đặt trực tiếp lên disk theo Storage Policy — quản lý per-object. Ceph thì gom **hàng triệu object** vào một số lượng PG cố định và tương đối nhỏ (vài trăm PG mỗi pool) — PG mới là đơn vị mà CRUSH thực sự tính toán vị trí, replicate, và peer. Lợi ích: khi cần rebalance hay recovery, Ceph thao tác theo PG (số lượng nhỏ, dễ quản lý) thay vì phải theo dõi từng object riêng lẻ (số lượng khổng lồ) — giống như quản lý theo "lô hàng" thay vì từng món hàng lẻ.

## Vì sao cần PG — bài toán nó giải quyết

Nếu CRUSH phải tính toán vị trí cho **từng object** riêng lẻ (có thể hàng trăm triệu object trong 1 cluster lớn), thì mỗi khi topology đổi (thêm/bớt OSD), toàn bộ object phải được duyệt lại để tính remap — không khả thi ở quy mô lớn. PG giải quyết việc này bằng cách:

1. Object được băm (hash) vào 1 trong N PG cố định của pool (N = `pg_num`, thường vài trăm).
2. CRUSH chỉ cần tính toán vị trí cho N PG này (con số nhỏ, ổn định) thay vì hàng triệu object.
3. Khi OSD thêm/bớt, chỉ một số PG bị remap (không phải toàn bộ), và việc replicate/recover diễn ra theo đơn vị PG — gom nhiều object lại xử lý cùng lúc, hiệu quả hơn nhiều so với xử lý từng object.

```
Object "vm-101-disk-0/rb.0.abc123" 
   │ hash(object_name) mod pg_num
   ▼
PG 3.7a  (pool_id=3, pg_id=7a)
   │ CRUSH(PG 3.7a, crushmap, rule)
   ▼
[osd.4 (primary), osd.11, osd.19]   ← acting set, size=3
```

Mỗi PG có 1 **primary OSD** (nhận I/O từ client, điều phối ghi tới các replica) và các **replica OSD** — tập hợp này gọi là **acting set**.

## Bảng trạng thái PG

Trạng thái PG hiển thị ở `ceph -s` / `ceph pg dump` / `ceph pg stat`, thường là tổ hợp nhiều trạng thái cùng lúc (vd `active+clean`, `active+undersized+degraded+backfilling`).

| Trạng thái | Ý nghĩa | Bình thường-thoáng qua hay Cần hành động |
|---|---|---|
| **active** | PG sẵn sàng phục vụ I/O | Bình thường (trạng thái mong muốn) |
| **clean** | Đủ số bản sao theo policy, không có object nào cần sync | Bình thường (trạng thái mong muốn) |
| **peering** | Các OSD trong acting set đang thống nhất trạng thái PG (sau khi OSD lên/xuống) | Bình thường-thoáng qua, thường vài giây |
| **degraded** | Thiếu ít nhất 1 bản sao so với policy (vd 1 OSD down) | Cần theo dõi — nếu kéo dài, kiểm tra OSD down |
| **undersized** | Acting set có ít OSD hơn `size` yêu cầu (không đủ OSD hợp lệ để đặt đủ bản sao) | Cần hành động — thường do thiếu OSD/host thỏa failure domain |
| **backfilling** | Đang copy toàn bộ object sang OSD mới (do remap CRUSH) | Bình thường-thoáng qua, nhưng tốn I/O, có thể throttle |
| **recovering** | Đang đồng bộ lại các object bị thiếu/lệch (không phải toàn bộ, chỉ phần chênh lệch) | Bình thường-thoáng qua sau sự cố OSD |
| **remapped** | Acting set đã đổi (do CRUSH tính lại) nhưng chưa backfill xong | Bình thường-thoáng qua |
| **incomplete** | Thiếu thông tin để biết trạng thái PG đúng là gì (có thể do mất nhiều OSD cùng lúc) | **Cần hành động ngay** — nguy cơ mất dữ liệu/không truy cập được |
| **inconsistent** | Scrub phát hiện dữ liệu giữa các bản sao không khớp nhau | **Cần hành động** — chạy `ceph pg repair <pgid>` |
| **stale** | PG không report trạng thái trong thời gian dài (thường do toàn bộ OSD giữ nó đều down) | **Cần hành động ngay** |
| **down** | Không đủ OSD hoạt động để phục vụ I/O cho PG này | **Cần hành động ngay** — I/O tới PG này bị treo |

```bash
# Xem tổng quan trạng thái toàn bộ PG
ceph pg stat

# Liệt kê PG không ở trạng thái clean
ceph pg dump_stuck

# Chi tiết 1 PG cụ thể
ceph pg 3.7a query

# Sửa PG inconsistent sau khi scrub phát hiện lỗi
ceph pg repair 3.7a
```

> [!warning] Lesson learned: đừng hoảng khi thấy `degraded`/`backfilling` — nhưng đừng phớt lờ `incomplete`/`stale`
> Nhiều người mới thấy health chuyển `HEALTH_WARN` với PG `degraded` hoặc `backfilling` liền hoảng loạn — thực ra đây là hành vi **tự phục hồi bình thường** sau khi thêm/bớt OSD hoặc 1 OSD restart. Ngược lại, `incomplete`, `stale`, hoặc `down` kéo dài là dấu hiệu nghiêm trọng (thường do mất nhiều OSD cùng lúc vượt quá khả năng chịu lỗi của failure domain) — cần điều tra ngay, không đợi "tự khỏi". Luôn phân biệt bằng cách đọc kỹ tổ hợp trạng thái, không chỉ nhìn màu health tổng quát.

## PG Autoscaler — mặc định hiện đại, khỏi phải tính tay

Trước đây (Ceph cũ), operator phải tự tính `pg_num`/`pgp_num` bằng công thức thủ công (số OSD × 100 / replication size, làm tròn lũy thừa 2) — dễ sai, và sai thì rất khó sửa (thay đổi pg_num gây data movement lớn). Từ Nautilus trở đi, **PG autoscaler bật mặc định** và là khuyến nghị chuẩn hiện nay:

```bash
# Kiểm tra trạng thái autoscale của từng pool
ceph osd pool autoscale-status

# Bật/tắt cho 1 pool cụ thể (mặc định thường đã "on" hoặc "warn")
ceph osd pool set my-pool pg_autoscale_mode on

# Gợi ý autoscaler dựa trên dữ liệu thực tế + target_size nếu biết trước dung lượng dự kiến
ceph osd pool set my-pool target_size_ratio 0.4
```

`ceph osd pool autoscale-status` cho thấy `PG_NUM` hiện tại, `NEW PG_NUM` khuyến nghị, và lý do — autoscaler tự điều chỉnh dần dần (không nhảy đột ngột) để tránh gây shock cho cluster.

> [!warning] Lesson learned: quá ít hoặc quá nhiều PG/OSD đều gây vấn đề
> Rule of thumb kinh điển: **~100-200 PG mỗi OSD** (tính theo tổng PG × replication size / tổng số OSD) là vùng an toàn.
> - **Quá ít PG/OSD** (dưới ~50): phân bố dữ liệu không đều giữa các OSD, một số OSD gánh nhiều hơn hẳn dẫn tới đầy sớm trong khi OSD khác vẫn trống, recovery cũng kém song song hóa (ít PG để phân tán công việc).
> - **Quá nhiều PG/OSD** (trên ~300-400): mỗi OSD phải giữ peering state cho quá nhiều PG cùng lúc → tốn RAM/CPU đáng kể, peering chậm sau khi OSD restart, thời gian `HEALTH_WARN` kéo dài hơn mỗi lần có sự cố nhỏ.
> Với autoscaler bật sẵn, tình huống này ít xảy ra hơn nhiều so với thời kỳ tính tay — nhưng vẫn cần kiểm tra định kỳ bằng `ceph osd pool autoscale-status`, đặc biệt sau khi thêm/bớt nhiều OSD hoặc tạo pool mới mà quên set `target_size_ratio`.

---
*Xem thêm: [[CRUSH Algorithm & CRUSH Map]] | [[RADOS & Cluster Architecture]] | [[Pools, Replication & Erasure Coding|Pools]] | [[Ceph|Ceph]]*
