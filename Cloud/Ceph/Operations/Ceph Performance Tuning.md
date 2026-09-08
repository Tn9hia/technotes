---
tags:
  - ceph
  - operations
  - performance
---

# Ceph Performance Tuning

**Đừng tối ưu ngẫu nhiên** — hiệu năng Ceph phụ thuộc vào một chuỗi các lớp (hardware → network → PG → recovery → client), và các lớp này có **thứ tự ưu tiên rõ ràng** về mức độ ảnh hưởng. Sửa sai thứ tự (VD: tinh chỉnh client cache trước khi network còn nghẽn) chỉ tốn công mà không thấy kết quả.

## Các đòn bẩy hiệu năng — theo thứ tự ưu tiên thực tế

### 1. NVMe cho BlueStore DB/WAL trên HDD OSD — thắng lợi lớn nhất

Nếu cluster dùng HDD cho OSD data mà **chưa tách DB/WAL ra NVMe riêng**, đây gần như luôn là đòn bẩy hiệu năng lớn nhất có thể làm. Metadata + write-ahead-log của BlueStore vốn là random I/O nhỏ — để chung trên HDD khiến HDD phải seek liên tục giữa data lớn tuần tự và metadata nhỏ ngẫu nhiên.

```bash
# Ví dụ tạo OSD với DB/WAL trên NVMe riêng qua cephadm drive group
ceph orch apply osd -i osd-spec.yaml
```

```yaml
# osd-spec.yaml — 1 NVMe làm DB device dùng chung cho nhiều HDD
service_type: osd
service_id: hdd_with_nvme_db
placement:
  host_pattern: 'osd-*'
data_devices:
  rotational: 1
db_devices:
  rotational: 0
db_slots: 6   # 1 NVMe chia sẻ DB cho tối đa 6 HDD
```

Chi tiết vai trò DB/WAL, tỷ lệ sizing khuyến nghị xem [[OSD - Object Storage Daemon]].

### 2. Network — nền tảng bắt buộc phải đủ trước khi tối ưu gì khác

10/25GbE tối thiểu cho production, **tách biệt public network và cluster network** (cluster network gánh replication/recovery traffic — nếu chung dây với public, backfill lớn có thể làm nghẽn cả traffic client). Xem thiết kế chi tiết ở [[Ceph Hardware & Network Design]].

### 3. PG count đúng — tránh cả thiếu lẫn thừa

PG quá ít → phân bố dữ liệu lệch, một số OSD nóng hơn hẳn. PG quá nhiều → overhead CPU/RAM cho MON và OSD, peering chậm. Dùng `pg_autoscaler` làm mặc định, chỉ can thiệp tay khi hiểu rõ traffic pattern. Chi tiết ở [[Placement Groups (PG)]].

### 4. Throttle recovery/backfill trong giờ hành chính

Recovery/backfill mặc định có thể chiếm băng thông đáng kể, ảnh hưởng trực tiếp latency I/O của VM đang chạy. Cần cân bằng giữa "phục hồi nhanh" và "không làm chậm production" — throttle theo khung giờ là thực hành phổ biến. Chi tiết cơ chế và tham số ở [[Recovery, Backfill & Self-healing]].

### 5. `balancer` module — giữ dữ liệu phân bố đều

```bash
ceph balancer status
ceph balancer mode upmap
ceph balancer on
```

OSD lệch tải (một vài OSD gần full trong khi số khác còn nhiều chỗ trống) làm giảm hiệu năng hiệu dụng của cả cluster vì OSD đầy nhất luôn là nút thắt cổ chai. Liên quan tới capacity planning — xem [[Ceph Sizing & Capacity Planning]].

### 6. RBD client-side tuning — quan trọng với CloudStack/KVM

Vì VM CloudStack chạy qua QEMU/librbd, tuning phía client ảnh hưởng trực tiếp tới trải nghiệm I/O trong guest.

| Tham số | Ở đâu | Ghi chú |
|---|---|---|
| `rbd_cache` | `ceph.conf` client hoặc RBD image config | Bật cache phía client giúp gộp write nhỏ, giảm round-trip |
| `rbd_cache_size`, `rbd_cache_max_dirty` | Client config | Kích thước cache và ngưỡng flush |
| Queue depth (`iodepth` phía QEMU) | libvirt XML / KVM disk config | Queue depth thấp giới hạn IOPS đạt được dù cluster còn dư sức |
| `cache_mode` trong CloudStack Compute Offering | CloudStack | Map sang `disk_cachemodes=writeback` phía libvirt — ảnh hưởng trực tiếp throughput/latency guest thấy được |

```xml
<!-- Ví dụ libvirt domain XML - disk cache mode cho RBD -->
<driver name='qemu' type='raw' cache='writeback' io='threads'/>
```

> [!tip] So với VMware vSAN
> vSAN tuning phía client chủ yếu nằm ở Storage Policy (stripe width, cache reservation) áp cho VMDK. Với Ceph/RBD trong CloudStack, tương đương là cache mode + queue depth cấu hình ở tầng libvirt/QEMU — không có UI trực quan như vSphere Storage Policy, phải chỉnh qua Compute Offering hoặc libvirt XML trực tiếp.

### 7. CPU governor / NUMA pinning — cho cluster all-NVMe nhạy latency

Với cluster toàn NVMe hướng tới latency cực thấp (VD: database-heavy workload), CPU governor để `performance` thay vì `powersave`, và pin OSD process theo NUMA node gần NVMe/NIC vật lý có thể giảm thêm vài trăm micro-giây latency. Đây là tối ưu ở mức "vắt kiệt phần trăm cuối", chỉ đáng làm sau khi 6 mục trên đã ổn.

## Benchmark đúng cách

| Công cụ | Dùng để |
|---|---|
| `rados bench` | Benchmark thô ở tầng RADOS, bỏ qua RBD/CephFS layer — đo hiệu năng nền của cluster |
| `rbd bench` | Benchmark trực tiếp trên 1 RBD image — gần hơn với thực tế CloudStack dùng |
| `fio` trên RBD image đã mount | Sát nhất với trải nghiệm VM thật — kiểm soát được block size, iodepth, read/write mix |

```bash
# rados bench — write 60s, 16 threads song song
rados bench -p rbd_pool 60 write -t 16
rados bench -p rbd_pool 60 seq

# rbd bench — mô phỏng random 4K write, io_type giống VM boot disk
rbd bench --io-type write --io-size 4K --io-pattern rand --io-total 1G rbd_pool/testimage

# fio trên block device đã map — quan trọng nhất: dùng block size thực tế của VM workload
fio --name=randrw_test --filename=/dev/rbd0 --ioengine=libaio --direct=1 \
    --rw=randrw --rwmixread=70 --bs=4k-64k --iodepth=32 --runtime=120 --group_reporting
```

> [!warning] Lesson learned: benchmark bằng sequential I/O lớn rồi ngỡ ngàng khi production chậm
> Một đội vận hành mới chạy `fio --bs=1M --rw=write` để "test nhanh xem cluster mạnh cỡ nào", thấy throughput vài GB/s rất ấn tượng, kết luận cluster "khỏe" và đưa vào production ngay. Vài tuần sau, người dùng phàn nàn VM chạy database/OS boot chậm bất thường dù cluster utilization còn thấp. Nguyên nhân: workload thật của VM trên CloudStack (boot disk, database, file server nhỏ lẻ) chủ yếu là **random I/O 4K-64K**, hoàn toàn khác pattern sequential lớn lúc benchmark. IOPS ở random small block thấp hơn rất nhiều so với throughput sequential — con số benchmark ban đầu **không phản ánh đúng trải nghiệm thực tế**. Bài học: luôn benchmark với block size và pattern **giống workload VM thật** (random 4K-64K, mixed read/write), không chỉ chạy benchmark "cho đẹp số".

---
*Xem thêm: [[OSD - Object Storage Daemon]] | [[Ceph Hardware & Network Design]] | [[Recovery, Backfill & Self-healing]] | [[Ceph|Ceph]]*
