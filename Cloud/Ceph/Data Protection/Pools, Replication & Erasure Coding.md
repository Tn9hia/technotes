---
tags:
  - ceph
  - pools
  - replication
  - erasure-coding
---

# Pools, Replication & Erasure Coding

Pool là **namespace cấp cao nhất** trong Ceph — nơi định nghĩa chính sách bảo vệ dữ liệu (replication hoặc erasure coding), CRUSH rule (dữ liệu được đặt ở đâu), và chứa toàn bộ Placement Group (PG) của nó. Mọi RBD image, mọi CephFS data, mọi RGW bucket data cuối cùng đều nằm trong 1 pool cụ thể. Hiểu đúng pool = hiểu đúng "dữ liệu của tôi được bảo vệ như thế nào và với chi phí gì".

> [!tip] So với VMware vSAN
> Pool không có tương đương 1-1 trong vSAN — gần nhất là hình dung nó như **Storage Policy** (FTT + RAID method) áp cho cả 1 nhóm object cùng lúc, thay vì cấu hình per-VMDK như vSAN Storage Policy Based Management. Khi bạn tạo pool với `size=3`, mọi object trong pool đó tự động có 3 bản sao — không cần chọn policy riêng cho từng RBD image.

## Pool chứa gì

```bash
# Tạo 1 pool replicated
ceph osd pool create rbd-pool 128 128 replicated

# Xem cấu hình pool
ceph osd pool get rbd-pool size
ceph osd pool get rbd-pool min_size
ceph osd pool get rbd-pool crush_rule
```

| Thuộc tính | Ý nghĩa |
|---|---|
| PG count | Số Placement Group — đơn vị phân phối dữ liệu (autoscaler tự điều chỉnh theo mặc định, xem [[Placement Groups (PG)]]) |
| CRUSH rule | Quy tắc chọn OSD nào lưu bản sao, dựa trên topology (host/rack/datacenter) |
| `size` (replicated) hoặc `k+m` (EC) | Số bản sao hoặc số chunk dữ liệu+parity |
| `min_size` | Số bản sao/chunk tối thiểu để pool còn chấp nhận I/O ghi |

## Replicated Pool — mặc định cho production

```bash
ceph osd pool set rbd-pool size 3
ceph osd pool set rbd-pool min_size 2
```

`size=3, min_size=2` là cấu hình **chuẩn production** phổ biến nhất: 3 bản sao đầy đủ của mỗi object, phân bố trên 3 OSD (thường trên 3 host khác nhau nhờ CRUSH rule mặc định `host` failure domain).

**`min_size` bảo vệ điều gì?** Đây là ngưỡng số bản sao *available* tối thiểu để pool còn cho phép **ghi**. Khi số bản sao còn sống rơi xuống dưới `min_size` (VD: 2 trong 3 host cùng lúc down), pool **không "chỉ degraded"** — nó chuyển sang **read-only cho các PG bị ảnh hưởng**, từ chối mọi write mới. Đây là cơ chế bảo vệ có chủ đích: Ceph thà từ chối ghi còn hơn ghi vào tình trạng chỉ còn 1 bản sao (nguy cơ mất dữ liệu nếu bản sao cuối cùng đó cũng hỏng trước khi recovery kịp hoàn tất).

## Erasure Coded (EC) Pool

EC pool không lưu N bản sao đầy đủ, mà chia object thành **k chunk dữ liệu + m chunk parity**, tổng cộng k+m chunk rải trên các OSD khác nhau — chỉ cần bất kỳ k trong số k+m chunk còn sống là phục hồi được toàn bộ dữ liệu.

```bash
# Tạo EC profile 4+2
ceph osd erasure-code-profile set ec-4-2 k=4 m=2 crush-failure-domain=host

# Tạo pool erasure-coded dùng profile trên
ceph osd pool create ec-data-pool 128 128 erasure ec-4-2
```

**Ví dụ cụ thể 4+2:** 4 chunk dữ liệu + 2 chunk parity = 6 chunk/object. Hiệu suất không gian = k/(k+m) = 4/6 ≈ **66.7% dung lượng thô dùng được** (so với 33.3% của replicated `size=3`). Chịu được mất tối đa **2 chunk** (2 OSD/host, tùy failure domain) mà vẫn đọc/phục hồi được đầy đủ dữ liệu.

| Profile | k+m | Dung lượng dùng được | Chịu mất tối đa |
|---|---|---|---|
| 2+1 | 3 chunk | 66.7% | 1 |
| 4+2 | 6 chunk | 66.7% | 2 |
| 8+3 | 11 chunk | 72.7% | 3 |
| Replicated size=3 | — | 33.3% | 2 |

> [!warning] Gotcha: EC pool không lưu được omap/metadata trực tiếp — cần `--data-pool`
> EC pool **không hỗ trợ omap** (key-value metadata mà RBD, RGW bucket index, CephFS metadata cần dùng nội bộ) — chỉ hỗ trợ lưu data thuần túy. Vì vậy không thể tạo RBD image trực tiếp trên EC pool. Cách dùng đúng: tạo image trên 1 pool **replicated** (chứa metadata/omap), nhưng chỉ định pool **data thật** là EC qua `--data-pool`:
> ```bash
> rbd create rbd-pool/big-image --size 1T --data-pool ec-data-pool
> ```
> `rbd-pool` (replicated) chỉ giữ metadata nhỏ gọn, còn toàn bộ object dữ liệu 4MB nằm ở `ec-data-pool`. Quên `--data-pool` và tạo thẳng image trên EC pool sẽ báo lỗi ngay — đây là điểm gây bối rối phổ biến nhất khi mới làm quen EC, vì log lỗi không luôn rõ ràng là "thiếu omap support".

## So sánh Replicated vs Erasure Coding

| Tiêu chí | Replicated | Erasure Coding |
|---|---|---|
| Hiệu suất không gian | Thấp (33% với size=3) | Cao (66-90% tùy k/m) |
| CPU cost | Thấp | Cao hơn (tính toán encode/decode parity) |
| Latency ghi | Thấp | Cao hơn (phải chờ đủ chunk ghi + tính parity) |
| Tốc độ recovery khi mất OSD | Nhanh (copy nguyên bản sao) | Chậm hơn (phải đọc k chunk khác để tái tạo) |
| Hỗ trợ omap/metadata trực tiếp | Có | Không — cần `--data-pool` kết hợp |
| Use case phù hợp | RBD (VM disk — cần latency thấp), CephFS metadata | RGW object cold/bulk, backup, archive — ít nhạy latency |

> [!warning] Lesson learned: `size=2 min_size=1` "để tiết kiệm dung lượng" — cạm bẫy tư duy từ vSAN FTT=1
> Đây là một trong những sai lầm nghiêm trọng nhất một operator mới có thể mắc, đặc biệt nếu quen tư duy "FTT=1 là đủ" từ vSAN (nơi RAID-1 2-way mirror + witness vẫn có cơ chế quorum riêng). Với Ceph, `size=2, min_size=1` nghĩa là: (1) chỉ 2 bản sao — mất 1 OSD là dữ liệu chỉ còn tồn tại ở đúng 1 nơi, không có "witness" trung gian nào bảo vệ, và (2) `min_size=1` cho phép cluster **tiếp tục nhận ghi ngay cả khi chỉ còn 1 bản sao sống** — nếu OSD chứa bản sao duy nhất đó chết trước khi recovery kịp tạo bản sao thứ 2, dữ liệu mất vĩnh viễn. Tệ hơn, trong kịch bản network partition, `min_size=1` mở ra nguy cơ **split-brain thực sự**: 2 nhóm OSD bị cô lập nhau đều nghĩ mình có quyền ghi vì "đủ 1 bản sao". Đây không phải rủi ro lý thuyết — nhiều case mất dữ liệu thực tế trong cộng đồng Ceph bắt nguồn từ đúng cấu hình này. Production luôn giữ `size=3, min_size=2` (hoặc EC tương đương độ bền cao hơn) — không có ngoại lệ "tạm thời" nào đáng đánh đổi.

---
*Xem thêm: [[Placement Groups (PG)]] | [[Recovery, Backfill & Self-healing]] | [[CRUSH Algorithm & CRUSH Map]] | [[Ceph|Ceph]]*
