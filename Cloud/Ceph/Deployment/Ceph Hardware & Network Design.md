---
tags:
  - ceph
  - deployment
  - hardware
  - network
---

# Ceph Hardware & Network Design

Ceph là software-defined storage — nó chạy trên phần cứng x86 thông thường, nhưng **thiết kế phần cứng và mạng sai ngay từ đầu là nguyên nhân số 1** gây ra sự cố hiệu năng production sau này (nhiều hơn cả bug phần mềm). Node role, tỉ lệ NVMe/HDD, và đặc biệt là tách mạng public/cluster là 3 quyết định khó sửa nhất sau khi cluster đã chạy thật.

> [!tip] So với VMware vSAN
> vSAN chạy converged (compute + storage chung host) gần như mặc định. Ceph cho bạn lựa chọn: **converged** (OSD + MON/MGR chung host, giống mô hình vSAN) hoặc **dedicated** (tách hẳn node OSD ra khỏi node MON/MGR, gần giống kiến trúc "storage array" truyền thống). Ở quy mô nhỏ converged tiết kiệm phần cứng hơn; ở quy mô lớn dedicated cho phép scale từng thành phần độc lập và giảm blast radius khi 1 node gặp sự cố.

## Vai trò node: converged vs dedicated

| Mô hình | Đặc điểm | Khi nào phù hợp |
|---|---|---|
| **Converged** (hyperconverged) | MON/MGR chạy chung host với OSD, thường 3-5 node đầu tiên kiêm cả 2 vai trò | Cluster nhỏ (< 10-15 node), muốn tiết kiệm phần cứng, giống tinh thần vSAN ROBO |
| **Dedicated MON/MGR** | 3-5 node riêng chỉ chạy MON/MGR (CPU/RAM khiêm tốn, không cần disk lớn), toàn bộ node còn lại là OSD thuần | Cluster lớn (hàng chục node OSD trở lên), muốn MON/MGR không bị ảnh hưởng bởi tải I/O nặng trên OSD host |

Với một cluster backend cho CloudStack quy mô vừa (ví dụ 6-12 OSD host), mô hình phổ biến nhất trong thực tế là: 3 node đầu vừa làm MON+MGR vừa làm OSD (converged), các node còn lại thuần OSD. Đây là điểm cân bằng hợp lý giữa chi phí phần cứng và cô lập rủi ro.

## Disk: HDD, NVMe cho DB/WAL, hoặc all-NVMe

BlueStore (backend OSD duy nhất trong Ceph hiện đại — không còn Filestore) tách dữ liệu thành 3 phần có thể đặt trên thiết bị khác nhau: **data** (object thật), **DB** (metadata, RocksDB), **WAL** (write-ahead log). Đặt DB/WAL trên NVMe nhanh trong khi data ở trên HDD chậm giúp tăng đáng kể IOPS ghi nhỏ mà không cần toàn bộ cluster là NVMe.

| Kiểu OSD | Thiết bị | Tỉ lệ khuyến nghị (NVMe : HDD) | Use case |
|---|---|---|---|
| HDD + NVMe DB/WAL (hybrid) | HDD 8-18TB cho data, NVMe cho DB/WAL | 1 NVMe : 4-12 HDD (tùy dung lượng ổ NVMe và kích thước DB cần) | Pool dung lượng lớn, cost-sensitive, chấp nhận latency vừa phải |
| All-NVMe | Toàn bộ OSD trên NVMe/SSD | Không cần tách DB/WAL riêng (đã đủ nhanh) | Pool hiệu năng cao — RBD cho VM I/O-sensitive, database workload |
| All-HDD (không NVMe) | HDD thuần, DB/WAL cùng disk | — | Chỉ nên dùng cho archive/cold data, không khuyến nghị cho primary storage của CloudStack |

> [!tip] Ước lượng kích thước DB partition
> Ceph khuyến nghị DB partition tối thiểu khoảng 4% dung lượng OSD data tương ứng (ví dụ OSD 10TB → DB tối thiểu ~400GB) để tránh RocksDB "spillback" xuống HDD chậm khi DB đầy — hiện tượng này âm thầm làm giảm hiệu năng ghi mà không có cảnh báo rõ ràng nếu không theo dõi `ceph osd df` và log OSD.

## CPU/RAM cho OSD

| Loại OSD | vCPU core / OSD (rule of thumb) | RAM / OSD |
|---|---|---|
| HDD-backed OSD | ~0.5-1 core | 3-5 GB |
| NVMe/SSD-backed OSD | ~2-4 core (I/O nhiều hơn, cần xử lý nhanh hơn) | 4-8 GB |
| MON/MGR (dedicated node) | 4-8 core tổng cho cả node | 16-32 GB (dashboard + Prometheus module ăn RAM đáng kể) |

Đừng tính theo "tổng CPU node / số OSD" một cách máy móc mà quên dự trù thêm cho việc recovery/backfill — các thao tác này ăn CPU đáng kể trên toàn bộ OSD liên quan cùng lúc, không chỉ OSD hỏng.

## Thiết kế mạng: public network vs cluster network

Đây là quyết định thiết kế quan trọng nhất trong chương này. Ceph khuyến nghị tách 2 mạng vật lý/logic riêng:

- **Public network**: client (RBD/CephFS/RGW, bao gồm CloudStack qua `librbd`) nói chuyện với MON/OSD.
- **Cluster network**: OSD nói chuyện với nhau — replication, recovery, backfill, heartbeat giữa các OSD.

```
        ┌─────────────── Public Network (vd. 10.10.10.0/24) ───────────────┐
        │                                                                   │
   [CloudStack KVM Host] ──librbd──> [MON] [MON] [MON]  <──client I/O── [OSD-1..N]
                                                                             │
                                                    ┌────────────────────────┘
                                                    │  Cluster Network
                                                    │  (vd. 10.10.20.0/24)
                                                    │  replication/recovery/heartbeat
                                                    ▼
                                          [OSD-1] <──> [OSD-2] <──> [OSD-3] ...
```

> [!tip] So với VMware vSAN
> Đây gần như tương đương nguyên văn với việc vSAN yêu cầu 1 VMkernel port/VLAN riêng cho vSAN traffic, tách biệt khỏi management và VM traffic. Lý do y hệt: traffic replication/rebuild rất nặng, nếu chung đường với traffic khách hàng (VM I/O) thì một sự kiện rebuild lớn sẽ "nuốt" băng thông và làm nghẽn I/O của toàn bộ VM đang chạy.

Không tách 2 mạng này (chạy chung 1 subnet/switch) vẫn **hoạt động được về mặt kỹ thuật** — Ceph không bắt buộc phải tách — nhưng trong production có tải thật, đây gần như luôn là nguồn gốc của sự cố latency khi có sự kiện recovery lớn.

### Khai báo trong cấu hình cluster

```ini
# /etc/ceph/ceph.conf (hoặc set qua `ceph config set` với cephadm — khuyến nghị hơn
# vì áp dụng động, không cần restart daemon và không lệch giữa các host)
[global]
public_network  = 10.10.10.0/24
cluster_network = 10.10.20.0/24
```

```bash
# Cách chuẩn với cephadm — set qua config database tập trung, không sửa file trên từng host
ceph config set global public_network 10.10.10.0/24
ceph config set global cluster_network 10.10.20.0/24
ceph config get global public_network
```

> [!warning] Cluster network gần như không thể đổi "nóng" dễ dàng trên cluster đã có dữ liệu
> Khai báo `public_network`/`cluster_network` áp dụng cho daemon **mới khởi tạo** — đổi giá trị này trên cluster đang chạy production đòi hỏi restart lần lượt từng OSD/MON để chúng bind lại đúng NIC/subnet mới, và dễ gây gián đoạn nếu làm không cẩn thận. Đây là lý do thiết kế network (đặc biệt là quyết định tách hay không tách 2 mạng) nên chốt **trước** khi đưa cluster vào production, không phải việc "để sau tính".

## Băng thông, MTU

| Thành phần | Khuyến nghị tối thiểu | Ghi chú |
|---|---|---|
| Public network | 10GbE | Dưới 10GbE (1GbE) chỉ chấp nhận được cho lab/POC, không cho production |
| Cluster network | 10GbE, ưu tiên bằng hoặc lớn hơn public network | Recovery traffic có thể bão hòa hoàn toàn 1 link 10GbE khi rebuild nhiều OSD cùng lúc |
| Cluster all-NVMe | 25GbE trở lên | NVMe OSD có thể tạo throughput vượt xa khả năng của 10GbE, network trở thành bottleneck trước cả disk |
| MTU | Jumbo frame (9000) khuyến nghị nếu toàn bộ switch/NIC trên đường đi hỗ trợ đồng nhất | Chỉ bật khi **chắc chắn** mọi thiết bị trung gian (switch, NIC, cả 2 đầu) đều support — MTU mismatch gây packet loss âm thầm, khó debug |

> [!warning] Lesson learned: mạng public và cluster dùng chung 1 uplink, latency VM tăng vọt khi thay ổ/host lỗi
> Một cluster triển khai ban đầu cho rằng "10GbE là đủ, không cần tách VLAN riêng cho cluster traffic" nên để cả public và cluster network đi chung 1 bond 10GbE. Vận hành bình thường không vấn đề gì — cho tới khi một host OSD gặp sự cố phần cứng phải remove & rebuild lại từ đầu (nhiều OSD cùng lúc, dữ liệu re-replicate hàng chục TB). Traffic backfill/recovery chiếm gần hết băng thông uplink, khiến CloudStack VM chạy trên RBD bị **latency I/O tăng vọt trong nhiều giờ**, một số VM timeout ứng dụng. Bài học: (1) tách VLAN/subnet vật lý riêng cho cluster network ngay từ thiết kế ban đầu, không đợi tới khi có sự cố mới tách, (2) nếu bắt buộc dùng chung do giới hạn phần cứng, phải nắm rõ và chủ động dùng các knob throttle recovery (`osd_max_backfills`, `osd_recovery_max_active`, xem [[Recovery, Backfill & Self-healing]]) để giới hạn tải trong giờ cao điểm.

## Checklist thiết kế trước khi đưa cluster vào production

- [ ] Đã xác định rõ node nào converged (OSD + MON/MGR), node nào dedicated — và ghi lại lý do chọn.
- [ ] Tỉ lệ NVMe:HDD cho DB/WAL đã tính theo dung lượng OSD thực tế, không chỉ theo số lượng ổ.
- [ ] Public network và cluster network nằm trên **NIC/switch vật lý tách biệt** (không chỉ tách VLAN logic trên cùng uplink vật lý, nếu ngân sách cho phép).
- [ ] Băng thông cluster network đủ chịu được kịch bản rebuild toàn bộ 1 host cùng lúc, không chỉ 1 OSD.
- [ ] MTU đồng nhất đã được xác nhận trên toàn bộ đường đi (switch + NIC 2 đầu) trước khi bật jumbo frame, có test `ping -M do -s 8972`.
- [ ] Đã dự trù RAM/CPU đủ cho kịch bản recovery đồng thời nhiều OSD, không chỉ tính theo tải bình thường.

---
*Xem thêm: [[Ceph Sizing & Capacity Planning]] | [[Recovery, Backfill & Self-healing]] | [[Ceph|Ceph]]*
