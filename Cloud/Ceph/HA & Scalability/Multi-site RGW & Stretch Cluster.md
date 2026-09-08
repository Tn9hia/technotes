---
tags:
  - ceph
  - rgw
  - stretch-cluster
  - disaster-recovery
---

# Multi-site RGW & Stretch Cluster

Hai kỹ thuật này đều nhằm mục tiêu chịu lỗi ở cấp **site/datacenter**, nhưng hoạt động ở 2 tầng hoàn toàn khác nhau: **RGW multi-site** replicate ở tầng object storage (async, giữa 2 cluster độc lập), còn **stretch cluster** kéo giãn **một** cluster RADOS duy nhất trên nhiều site (đồng bộ ở tầng thấp hơn nhiều). Nhầm lẫn 2 khái niệm này khi thiết kế DR là sai lầm tốn kém — chúng có yêu cầu hạ tầng (đặc biệt là latency mạng) khác nhau một trời một vực.

> [!tip] Đây là chủ đề nâng cao
> Phần lớn cluster Ceph nhỏ-vừa phục vụ CloudStack **không cần** đến 2 kỹ thuật này ngay từ ngày đầu — chúng chỉ trở nên cần thiết khi có yêu cầu DR thực sự giữa nhiều site vật lý. Đọc để biết chúng tồn tại và khác nhau ra sao, không cần triển khai nếu tổ chức chỉ vận hành 1 site.

## RGW Multi-site — replication ở tầng object

RGW multi-site tổ chức theo cây phân cấp: **realm** (namespace cấu hình lớn nhất) → **zonegroup** (nhóm zone, thường ứng với 1 vùng địa lý) → **zone** (thường ứng với 1 cluster Ceph vật lý). Dữ liệu được replicate **async** giữa các zone trong cùng zonegroup.

```
Realm: my-org
  └── Zonegroup: asia
        ├── Zone: dc-hcm (master)   ── cluster Ceph #1, site Hồ Chí Minh
        └── Zone: dc-hn  (secondary) ── cluster Ceph #2, site Hà Nội (async replica)
```

```bash
# Tạo realm/zonegroup/zone (rút gọn, thực hiện trên zone master)
radosgw-admin realm create --rgw-realm=my-org --default
radosgw-admin zonegroup create --rgw-zonegroup=asia --master --default
radosgw-admin zone create --rgw-zonegroup=asia --rgw-zone=dc-hcm --master --default \
  --endpoints=http://rgw-hcm:80

# Trên cluster thứ 2 (secondary zone), pull cấu hình realm từ master rồi tạo zone
radosgw-admin realm pull --url=http://rgw-hcm:80 --access-key=<key> --secret=<secret>
radosgw-admin zone create --rgw-zonegroup=asia --rgw-zone=dc-hn \
  --endpoints=http://rgw-hn:80 --access-key=<key> --secret=<secret>

# Áp dụng cấu hình + restart RGW daemon liên quan (qua cephadm)
radosgw-admin period update --commit

# Theo dõi trạng thái đồng bộ — lệnh quan trọng nhất khi vận hành multi-site
radosgw-admin sync status
```

Mô hình phổ biến: **active-active** — cả 2 zone đều nhận write, tự động sync 2 chiều. Dùng cho geo-distribution (client gần zone nào dùng zone đó) hoặc DR (zone phụ sẵn sàng phục vụ nếu zone chính mất).

## Stretch Cluster — kéo giãn 1 cluster RADOS qua nhiều site

Stretch mode là tính năng cho phép **một cluster RADOS duy nhất** trải trên 2 site chính (data site) cộng thêm 1 site thứ 3 chỉ đặt 1 MON làm **tiebreaker** (để giữ quorum lẻ khi 1 trong 2 site chính mất kết nối).

```bash
# Gán location cho từng MON — Ceph dùng thông tin này để biết MON nào ở site nào
ceph mon set_location mon-a datacenter=site-a
ceph mon set_location mon-b datacenter=site-b
ceph mon set_location mon-c datacenter=site-c   # site thứ 3, chỉ có MON, không có OSD

# Kích hoạt stretch mode, chỉ định tiebreaker MON và CRUSH rule 2-site
ceph mon enable_stretch_mode mon-c stretch_rule datacenter
```

CRUSH rule cho stretch mode phải đảm bảo **mỗi bản sao dữ liệu nằm ở một site khác nhau** (không chỉ khác host) — ví dụ với size=4, 2 bản sao ở site-a và 2 bản sao ở site-b, để 1 site mất hoàn toàn vẫn còn đủ bản sao đọc/ghi ở site còn lại.

## Giám sát trạng thái sync (RGW multi-site)

```bash
# Trạng thái tổng quan sync giữa các zone
radosgw-admin sync status

# Kết quả điển hình khi mọi thứ đồng bộ tốt:
#   realm ... (my-org)
#   zonegroup ... (asia)
#   zone ... (dc-hn)
#     metadata sync: no sync (zone is master) hoặc "caught up with source"
#     data sync source: dc-hcm
#                        syncing
#                        full sync: 0/128 shards
#                        incremental sync: 128/128 shards
#                        data is caught up with source

# Nếu thấy "behind" kéo dài hoặc shard lỗi, kiểm tra chi tiết
radosgw-admin bucket sync status --bucket=<ten-bucket>
radosgw-admin sync error list
```

`behind` kéo dài thường do băng thông WAN không đủ theo kịp tốc độ ghi ở zone master, hoặc RGW ở zone secondary quá tải — đây là chỉ số cần đưa vào alerting nếu multi-site được dùng làm DR thật sự (một zone "trông như đang chạy" nhưng đã lệch dữ liệu hàng giờ so với master là một rủi ro DR âm thầm).

## So sánh — 2 chiến lược DR khác tầng nhau

| Tiêu chí | RGW Multi-site | Stretch Cluster |
|---|---|---|
| Tầng hoạt động | Object storage (RGW) | RADOS (toàn bộ cluster — ảnh hưởng cả RBD, CephFS nếu dùng) |
| Số cluster vật lý | 2+ cluster **độc lập** | **1** cluster duy nhất trải nhiều site |
| Kiểu đồng bộ | Async (eventual consistency, có độ trễ replication) | Đồng bộ chặt — ghi phải xác nhận đủ ở nhiều site trước khi ACK |
| Yêu cầu latency mạng giữa site | Chịu được latency cao, kể cả WAN liên vùng/quốc gia | **Rất nhạy cảm** — cần latency thấp, ổn định (thường khuyến nghị < vài ms, tương đương khoảng cách trong cùng thành phố/metro) |
| Phạm vi bảo vệ | Chỉ dữ liệu qua RGW (S3/Swift) | Toàn bộ cluster (RBD/CephFS/RGW nếu dùng chung cluster) |
| Độ phức tạp vận hành | Trung bình | Cao — cần CRUSH rule đúng, giám sát quorum 3 site chặt chẽ |
| Phù hợp cho | DR địa lý xa, backup site khác vùng/quốc gia | 2 site gần nhau (cùng metro), muốn cluster sống sót nguyên vẹn khi mất 1 site mà không cần failover thủ công |

> [!warning] Lesson learned: thử "stretch" một cluster giữa 2 site cách xa nhau qua WAN chậm — hiệu năng sụp đổ
> Stretch cluster đòi hỏi ghi dữ liệu được xác nhận **đồng bộ** trên nhiều site trước khi client nhận ACK — bản chất khác hẳn RGW multi-site (async, chấp nhận độ trễ đồng bộ). Một số đội mới tiếp cận Ceph nhầm lẫn 2 khái niệm này, dựng thử stretch cluster giữa 2 site cách nhau hàng trăm km qua đường truyền WAN có RTT hàng chục ms — kết quả là **mọi write trên toàn cluster đều phải chờ round-trip WAN đó**, hiệu năng I/O giảm sụp so với kỳ vọng, VM chạy trên CloudStack ì ạch rõ rệt dù phần cứng dư thừa. Bài học: stretch mode chỉ phù hợp khi 2 site **thực sự gần nhau về mặt mạng** (latency thấp, ổn định — tinh thần giống khoảng cách trong cùng metro/campus). Với 2 site cách xa về địa lý (liên tỉnh/liên quốc gia), **RGW multi-site (async)** là mô hình DR thực tế và an toàn hơn nhiều — chấp nhận độ trễ replication để đổi lấy việc không phụ thuộc vào latency mạng tức thời.

---
*Xem thêm: [[Ceph HA Architecture]] | [[RGW - Object Storage Gateway]] | [[CRUSH Algorithm & CRUSH Map]] | [[Ceph|Ceph]]*
