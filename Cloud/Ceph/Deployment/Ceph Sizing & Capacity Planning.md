---
tags:
  - ceph
  - deployment
  - capacity-planning
---

# Ceph Sizing & Capacity Planning

Dung lượng "raw" hiển thị khi mua ổ đĩa **không phải** dung lượng bạn thực sự dùng được — replication/erasure coding ăn vào một phần lớn, và Ceph có ngưỡng `nearfull`/`full` mang tính **toàn cluster** chứ không phải cục bộ từng OSD. Hiểu sai phần này là nguyên nhân phổ biến khiến một cluster "trông còn dư chỗ" bỗng nhiên từ chối ghi dữ liệu.

> [!tip] So với VMware vSAN
> vSAN cũng có khái niệm tương tự: FTT (Failures To Tolerate) ăn vào capacity giống size=N của Ceph, và vSAN cũng khuyến nghị giữ slack space (thường ~25-30%) để chừa chỗ cho rebuild. Điểm khác biệt lớn: Ceph không tự động "cấm ghi" chỉ vì thiếu slack cho rebuild như cách vSAN cảnh báo — Ceph có ngưỡng cứng (`full_ratio`) áp dụng dựa trên OSD đầy nhất, không phải dung lượng trung bình toàn cluster, nên hành vi "tưởng còn dư mà vẫn bị chặn" dễ xảy ra hơn nếu không hiểu cơ chế balancer.

## Usable vs raw capacity

| Protection scheme | Công thức usable | Ví dụ (100TB raw) | Overhead |
|---|---|---|---|
| Replication size=2 | raw / 2 | 50 TB | 50% (không khuyến nghị cho production — rủi ro mất dữ liệu khi 1 OSD chết giữa lúc recovery) |
| Replication size=3 (mặc định, khuyến nghị) | raw / 3 | ~33 TB | 67% |
| Erasure Coding 4+2 | raw × k/(k+m) = raw × 4/6 | ~66.7 TB | 33% |
| Erasure Coding 2+2 | raw × 2/4 | 50 TB | 50% |
| Erasure Coding 8+3 | raw × 8/11 | ~72.7 TB | ~27% |

> [!tip] Khi nào chọn EC thay vì Replication
> EC tiết kiệm dung lượng hơn nhiều nhưng đắt CPU hơn khi ghi (encode) và chậm hơn khi recovery (phải đọc lại nhiều chunk để tái tạo). Thực tế: EC phù hợp cho RGW (object storage, workload đọc nhiều/ghi tuần tự) hoặc cold data; **replication size=3 vẫn là lựa chọn mặc định cho RBD pool phục vụ CloudStack** vì cần latency thấp và ổn định cho I/O ngẫu nhiên của VM. Xem chi tiết ở [[Pools, Replication & Erasure Coding|Replication & Erasure Coding]].

## Ngưỡng nearfull/full — cơ chế quan trọng nhất cần hiểu

Ceph theo dõi usage của **từng OSD** và so với 2 ngưỡng cấu hình toàn cluster:

| Ngưỡng | Giá trị mặc định | Hành vi khi vượt |
|---|---|---|
| `mon_osd_nearfull_ratio` | 0.85 (85%) | `HEALTH_WARN`, cảnh báo "OSD nearfull" — cluster vẫn hoạt động bình thường |
| `mon_osd_backfillfull_ratio` | 0.90 (90%) | OSD đó bị loại khỏi target nhận thêm dữ liệu backfill/recovery — có thể làm recovery bị kẹt cục bộ |
| `mon_osd_full_ratio` | 0.95 (95%) | **Cluster từ chối MỌI ghi mới** — không chỉ trên OSD đầy, mà toàn bộ pool có liên quan tới OSD đó (thường là ảnh hưởng diện rộng vì CRUSH rải dữ liệu khắp cluster) |

```bash
# Xem ngưỡng hiện tại
ceph osd dump | grep -i ratio

# Chỉnh ngưỡng (chỉ nên chỉnh tạm thời để xử lý sự cố, không nên coi là giải pháp lâu dài)
ceph osd set-nearfull-ratio 0.85
ceph osd set-full-ratio 0.95
```

> [!warning] Điểm dễ hiểu lầm nhất: "full" là sự kiện toàn cluster, không phải cục bộ 1 ổ đĩa
> Khác với một filesystem/datastore thông thường (1 LUN đầy chỉ ảnh hưởng chính LUN đó), khi **một OSD** chạm `full_ratio`, Ceph coi toàn bộ cluster ở trạng thái nguy hiểm và **chặn ghi (write) trên các pool liên quan** cho tới khi giải phóng được dung lượng — ngay cả khi 20 OSD khác trong cùng cluster vẫn còn trống rất nhiều. Đây chính là hệ quả của thiết kế "không có bộ não trung tâm điều phối I/O" (xem [[RADOS & Cluster Architecture]]) — mỗi OSD tự bảo vệ chính nó khỏi bị ghi tràn, không có cơ chế "định tuyến sang chỗ khác còn trống" theo thời gian thực.

## Vì sao phân bố dữ liệu không đều gây "full giả"

CRUSH là thuật toán hash giả-ngẫu-nhiên — về lý thuyết phân bố đều, nhưng thực tế với số lượng PG hữu hạn và pattern object không đồng nhất, một số OSD luôn nhận nhiều dữ liệu hơn mức trung bình đáng kể (lệch 10-20% là bình thường nếu không can thiệp). Hệ quả: OSD "xui" nhất có thể chạm ngưỡng full trong khi utilization trung bình toàn cluster mới chỉ 60-70%.

```bash
# Xem phân bố usage từng OSD — cột %USE là thứ cần soi kỹ, không chỉ nhìn ceph df tổng
ceph osd df tree

# Bật balancer module (mgr) — chuẩn hiện đại dùng mode "upmap"
ceph balancer mode upmap
ceph balancer on
ceph balancer status
```

`balancer` (mgr module) là cơ chế hiện đại để giải quyết vấn đề này: nó liên tục tính toán và áp các `pg-upmap` exception để dịch chuyển PG từ OSD quá tải sang OSD còn trống, mà không cần sửa CRUSH weight thủ công. Mode `upmap` là khuyến nghị mặc định cho cluster hiện đại (yêu cầu toàn bộ client hỗ trợ luxury feature này — không phải vấn đề với client hiện tại).

## Bảng worksheet lập kế hoạch dung lượng

| Raw capacity | Scheme | Usable (lý thuyết) | Headroom khuyến nghị (~70-75%) | Dung lượng nên plan sử dụng thực tế |
|---|---|---|---|---|
| 100 TB | Replication x3 | 33.3 TB | giữ dưới 75% usable | ~25 TB |
| 300 TB | Replication x3 | 100 TB | giữ dưới 75% usable | ~75 TB |
| 300 TB | EC 4+2 | 200 TB | giữ dưới 75% usable | ~150 TB |
| 600 TB | Replication x3 | 200 TB | giữ dưới 75% usable | ~150 TB |

> [!tip] Vì sao headroom 70-75% chứ không phải chờ tới gần 85%
> Ngưỡng `nearfull` (85%) là mức cảnh báo của Ceph, không phải mức lập kế hoạch. Lý do cần margin thêm: (1) khi 1 host/rack (failure domain) chết, dữ liệu của nó phải re-replicate vào các OSD còn lại — cần đủ chỗ trống để hấp thụ lượng dữ liệu đó ngay cả khi bạn đang ở 70%, (2) phân bố không đều (mục trên) khiến OSD lệch nhất chạm ngưỡng sớm hơn nhiều so với con số trung bình toàn cluster gợi ý.

## Đọc nhanh dung lượng cluster bằng `ceph df`

```bash
$ ceph df
--- RAW STORAGE ---
CLASS     SIZE     AVAIL    USED     RAW USED    %RAW USED
hdd     300 TiB   210 TiB   90 TiB      90 TiB        30.00
TOTAL   300 TiB   210 TiB   90 TiB      90 TiB        30.00

--- POOLS ---
POOL           ID   PGS   STORED   OBJECTS   USED     %USED   MAX AVAIL
rbd-pool        1   128   28 TiB    7.3M     84 TiB    28.57     70 TiB
rgw.buckets     2    64    2 TiB    1.1M      6 TiB     2.04     70 TiB
```

Cột quan trọng nhất là **`MAX AVAIL`** — đây là dung lượng usable *thực tế còn dùng được* cho pool đó theo protection scheme đã cấu hình, **đã được Ceph tự tính dựa trên OSD đầy nhất liên quan tới pool**, không phải một phép chia đơn giản dung lượng trống toàn cluster. Đây là lý do `MAX AVAIL` giữa các pool có thể khác xa nhau dù cùng chung 1 cluster, nếu mỗi pool dùng CRUSH rule/device class khác nhau (ví dụ pool NVMe riêng, pool HDD riêng).

## Quy trình khi cluster báo nearfull/full

1. Xác định OSD nào đang gần/đã full: `ceph osd df tree | sort -k17 -n` (sort theo %USE).
2. Nếu balancer chưa bật, bật ngay `ceph balancer on` — thường tự giảm được vài % trong vài giờ.
3. Nếu cần giải pháp tức thời hơn: tăng tạm `mon_osd_full_ratio` là **cách chữa cháy nguy hiểm**, chỉ dùng khi đã xác nhận có đủ physical capacity thật sự (không phải do lệch phân bố) và đang chờ bổ sung phần cứng.
4. Giải pháp bền vững: thêm OSD/host mới để tăng raw capacity, hoặc dọn dữ liệu không cần thiết (snapshot cũ, pool test...).

> [!warning] Lesson learned: cluster "trông còn nhiều chỗ" nhưng bất ngờ từ chối ghi toàn bộ vì 1 OSD vượt 95%
> Một cluster có dashboard tổng thể báo utilization trung bình ~68% — trông rất an toàn. Nhưng balancer chưa từng được bật (mặc định tắt trên một số bản triển khai cũ/migrate từ ceph-ansible), và vài OSD dung lượng nhỏ hơn các OSD khác trong cùng host (bổ sung disk không đồng nhất theo thời gian) liên tục nhận PG nhiều hơn tỉ lệ dung lượng của nó. Một trong các OSD đó âm thầm vượt 95% `full_ratio` trong khi trung bình cluster vẫn thấp — toàn bộ pool RBD phục vụ CloudStack **ngừng nhận write**, VM bắt đầu báo lỗi I/O ở tầng guest OS. Xử lý khẩn cấp lúc đó: `ceph balancer on` ngay lập tức + tạm nới `full_ratio` trong lúc theo dõi balancer dịch chuyển PG, sau đó bổ sung capacity đúng nghĩa. Bài học rút ra: **luôn bật balancer (upmap mode) như một cấu hình mặc định ngay từ ngày đầu**, không đợi tới khi có sự cố mới nhớ tới nó — và giám sát `ceph osd df` theo từng OSD, không chỉ nhìn con số utilization trung bình.

## Các yếu tố khác ăn vào capacity thực tế (dễ bị quên khi lập kế hoạch)

| Yếu tố | Ảnh hưởng |
|---|---|
| Số lượng PG chưa tối ưu (quá ít hoặc quá nhiều) | PG quá ít → phân bố lệch nặng hơn; PG quá nhiều → overhead metadata, RAM OSD tăng. Dùng `pg_autoscaler` (bật mặc định) để giảm thiểu vấn đề này |
| Snapshot RBD giữ lại lâu ngày | Snapshot giữ lại block đã bị overwrite/xóa ở live image — dùng dung lượng thực dù "nhìn" volume gốc không lớn |
| Object đã xóa nhưng cluster chưa GC xong | Có độ trễ giữa "xóa logic" và giải phóng thực tế trên OSD, đặc biệt với RGW (garbage collection theo lịch) |
| Metadata pool CephFS/RGW index | Thường nhỏ nhưng cần đặt trên device nhanh (NVMe) — không tính vào capacity dữ liệu chính nhưng vẫn cần dự trù riêng |
| Compression (nếu bật ở BlueStore) | Có thể tăng usable thực tế cao hơn con số lý thuyết ở bảng trên, nhưng tỉ lệ nén phụ thuộc hoàn toàn vào dạng dữ liệu — không nên tính sẵn vào kế hoạch capacity ban đầu |

---
*Xem thêm: [[Ceph Hardware & Network Design]] | [[Pools, Replication & Erasure Coding|Replication & Erasure Coding]] | [[Placement Groups (PG)]] | [[Ceph|Ceph]]*
