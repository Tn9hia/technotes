---
tags:
  - ceph
  - osd
  - architecture
---

# OSD - Object Storage Daemon

**OSD (Object Storage Daemon)** là daemon thực sự lưu trữ dữ liệu — Ceph chạy **một OSD daemon cho mỗi ổ đĩa vật lý** (không phải mỗi host, không phải mỗi disk-group). Đây là đơn vị chi tiết nhất trong toàn bộ kiến trúc Ceph và là nơi mọi replicate/recovery/scrub thực sự diễn ra.

> [!tip] So với VMware vSAN
> vSAN gom nhiều disk thành 1 **disk group** (1 cache device + nhiều capacity device), quản lý theo nhóm. Ceph **không có khái niệm disk group** — mỗi ổ đĩa vật lý (HDD hoặc SSD/NVMe làm data) chạy **1 OSD daemon riêng biệt**, độc lập hoàn toàn. Điều này khiến Ceph mịn hạt hơn nhiều: mất 1 disk chỉ mất 1 OSD (không kéo theo cả nhóm disk như vSAN mất cache device), nhưng cũng nghĩa là 1 host với 12 ổ HDD sẽ chạy 12 process OSD riêng, tốn RAM/CPU tương ứng — cần tính toán khi sizing host.

## BlueStore — storage engine hiện tại (mặc định và duy nhất)

Từ Ceph Luminous trở đi, **BlueStore** là backend duy nhất được khuyến nghị (FileStore đã bị loại bỏ hoàn toàn ở các bản hiện đại). BlueStore ghi dữ liệu **trực tiếp lên block device thô**, tự quản lý một mini-filesystem riêng (không qua XFS/ext4 như FileStore cũ) — giảm hẳn write amplification và overhead của filesystem trung gian.

BlueStore chia dữ liệu OSD ra 3 phần logic:

| Thành phần | Chứa gì | Nên đặt ở đâu |
|---|---|---|
| **block** | Dữ liệu object thật (payload chính) | Disk chính của OSD (HDD hoặc SSD) |
| **block.db** | Metadata OSD (RocksDB — object map, allocation, checksums) | SSD/NVMe nhanh, tách riêng nếu block nằm trên HDD |
| **block.wal** | Write-Ahead Log (ghi trước khi commit vào block.db) | SSD/NVMe nhanh nhất có thể, thường gộp chung với block.db nếu cùng 1 NVMe |

```bash
# Tạo OSD với block chính trên HDD, db+wal trên NVMe riêng (đòn bẩy hiệu năng lớn nhất)
ceph orch daemon add osd host01:/dev/sdb,db_devices=/dev/nvme0n1

# Hoặc khai báo qua spec file (khuyến nghị cho production — lặp lại được, review được)
cat <<EOF > osd-spec.yaml
service_type: osd
service_id: hdd-with-nvme-db
placement:
  hosts:
    - host01
data_devices:
  rotational: 1
db_devices:
  rotational: 0
EOF
ceph orch apply -i osd-spec.yaml
```

> [!tip] Đòn bẩy hiệu năng lớn nhất mà người mới hay bỏ lỡ
> Nếu backend là HDD (7.2k RPM), việc tách **block.db/block.wal ra 1 NVMe riêng** thường mang lại cải thiện IOPS/latency lớn hơn nhiều so với bất kỳ tuning tham số nào khác trong `ceph.conf`. Lý do: metadata operations (lookup, allocation, checksum) vốn là random I/O nhỏ — cực chậm trên HDD nhưng cực nhanh trên NVMe. Tỷ lệ khuyến nghị phổ biến: 1 NVMe (đủ nhanh, đủ bền — enterprise-grade) phục vụ db/wal cho khoảng 4-12 OSD HDD tùy dung lượng NVMe và write endurance. Bỏ qua bước này (dùng toàn HDD kể cả cho db/wal) là nguyên nhân phổ biến nhất khiến cluster HDD-only "chậm không rõ lý do" khi mới go-live.

## Trạng thái OSD: up/down và in/out — ma trận hay bị nhầm

Đây là 2 trục **độc lập nhau**, rất hay bị gộp nhầm thành 1 khái niệm:

- **up / down**: OSD daemon **có đang chạy và phản hồi được không** (trạng thái runtime, do MON theo dõi qua heartbeat).
- **in / out**: OSD **có đang được CRUSH tính vào việc đặt dữ liệu không** (trạng thái tham gia placement).

| Trạng thái | Ý nghĩa | Khi nào xảy ra |
|---|---|---|
| **up + in** | Bình thường — daemon chạy, đang chứa dữ liệu | Trạng thái mong muốn của đa số OSD |
| **up + out** | Daemon vẫn chạy nhưng **không** được CRUSH đặt dữ liệu mới (dữ liệu cũ đã backfill đi nơi khác) | Sau khi chủ động `ceph osd out` để chuẩn bị bảo trì/thay disk mà chưa tắt daemon |
| **down + in** | Daemon không phản hồi nhưng **vẫn được tính trong CRUSH** — PG liên quan trở thành degraded | Ngay sau khi OSD crash, trước khi MON đánh dấu `out` (mặc định sau 10 phút, `mon_osd_down_out_interval`) |
| **down + out** | Daemon chết và đã bị loại khỏi CRUSH — dữ liệu đã backfill/redistribute sang OSD khác | OSD chết lâu, hoặc bị `ceph osd out` chủ động rồi mới tắt |

```bash
# Xem trạng thái toàn bộ OSD, kèm cây CRUSH
ceph osd tree

# Xem usage/capacity từng OSD
ceph osd df

# Xem hiệu năng (latency commit/apply) từng OSD — phát hiện disk chậm bất thường
ceph osd perf
```

> [!warning] Lesson learned: down ≠ out, out ≠ down — nhầm 2 khái niệm này dẫn tới thao tác sai
> Người mới hay nghĩ "OSD down thì chắc cũng out rồi" hoặc ngược lại. Thực tế: **down+in** là trạng thái cực kỳ phổ biến trong vài phút đầu sau khi 1 OSD crash — cluster đã bắt đầu báo `degraded` (vì thiếu 1 bản sao) nhưng **chưa** bắt đầu backfill toàn bộ dữ liệu sang OSD khác (vì OSD đó vẫn được tính là `in`, Ceph "chờ" xem nó có tự hồi phục không, mặc định 10 phút). Nếu bạn vội vàng `ceph osd out` ngay khi thấy `down` (thay vì đợi hoặc kiểm tra xem có phải chỉ là restart service), bạn sẽ kích hoạt backfill lớn không cần thiết cho một OSD sắp tự up lại. Ngược lại, một OSD `up+out` (vd sau bảo trì) vẫn tốn RAM/CPU chạy daemon dù không chứa dữ liệu — nếu quên đưa `in` lại, tài nguyên đó coi như lãng phí.

## OSD failure → CRUSH recompute → recovery

```
OSD.7 crash
   │
   ├─ MON không nhận heartbeat → sau ngưỡng, đánh dấu osd.7 = down
   │  (osdmap epoch tăng, broadcast tới toàn cluster)
   │
   ├─ Các PG có osd.7 trong acting set → chuyển "degraded"
   │  (vẫn active nếu còn đủ bản sao khác phục vụ I/O)
   │
   ├─ Sau mon_osd_down_out_interval (mặc định 600s) → osd.7 = out
   │  CRUSH tính lại acting set cho các PG liên quan (không còn osd.7)
   │
   └─ Các OSD còn lại backfill dữ liệu để bù đắp bản sao thiếu
      → PG dần chuyển "active+clean" trở lại
```

## Quy trình thay ổ đĩa hỏng (outline)

```bash
# 1. Đánh dấu out để bắt đầu drain dữ liệu ra khỏi OSD trước khi động vào phần cứng
ceph osd out osd.7

# 2. Theo dõi quá trình backfill hoàn tất
ceph -s   # chờ tới khi active+clean trở lại (hoặc theo dõi ceph -w)

# 3. Dừng daemon và gỡ khỏi cluster
ceph orch daemon stop osd.7
ceph osd purge osd.7 --yes-i-really-mean-it

# 4. Thay ổ đĩa vật lý

# 5. Thêm OSD mới trên ổ vừa thay (cephadm tự phát hiện disk trống nếu dùng spec "available")
ceph orch daemon add osd host01:/dev/sdb
```

> [!warning] Lesson learned: rút ổ ngay khi thấy lỗi vs. đợi drain xong — tùy vào redundancy hiện tại của cluster
> Quy trình "chuẩn" (`out` → đợi backfill xong → mới rút ổ) là an toàn nhất vì nó đảm bảo dữ liệu đã có đủ bản sao ở nơi khác trước khi bạn động vào phần cứng — tránh rơi vào tình huống mất luôn bản sao cuối cùng nếu chẳng may 1 OSD khác trong cùng PG cũng gặp sự cố giữa lúc đó. Tuy nhiên, **nếu ổ đĩa đang lỗi phần cứng thật sự** (SMART báo bad sector lan rộng, I/O error liên tục) và cluster **đang degraded sẵn** (vd 1 host khác trong failure domain cũng đang down), có tình huống cần cân nhắc rút ổ ngay lập tức để tránh nó tiếp tục ghi dữ liệu lỗi hoặc gây treo I/O toàn PG — đánh đổi lấy rủi ro backfill sau đó phải xử lý PG `degraded`/`undersized` trong thời gian dài hơn. Không có công thức cứng cho mọi tình huống — luôn kiểm tra `ceph osd df`, `ceph pg dump_stuck`, và mức độ redundancy còn lại (bao nhiêu OSD/host còn sống trong cùng failure domain) trước khi quyết định tốc độ hành động.

---
*Xem thêm: [[CRUSH Algorithm & CRUSH Map]] | [[Placement Groups (PG)]] | [[Scaling the Cluster - Add-Remove Node & OSD]] | [[Ceph|Ceph]]*
