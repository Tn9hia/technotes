---
tags:
  - ceph
  - crush
  - architecture
---

# CRUSH Algorithm & CRUSH Map

**CRUSH (Controlled Replication Under Scalable Hashing)** là thuật toán cho phép mọi client và daemon trong Ceph **tự tính toán** object nên nằm ở OSD nào, thay vì phải tra cứu một bảng metadata trung tâm. Đây là phát minh cốt lõi khiến Ceph scale được tới hàng nghìn OSD mà vẫn không có bottleneck lookup — và cũng là khái niệm khác biệt lớn nhất so với các hệ storage truyền thống (kể cả vSAN).

> [!tip] So với VMware vSAN
> vSAN's CLOM (Cluster Level Object Manager) và DOM giữ **bảng theo dõi tập trung** cho từng object/component — biết chính xác component nào nằm trên host/disk nào tại một thời điểm, kiểu database. Ceph **không giữ bảng đó**. Mỗi lần cần biết object nằm ở đâu, client chạy một hàm băm xác định (deterministic) trên topology hiện tại (crushmap) — ra kết quả ngay, không cần hỏi ai. Ưu điểm: không có single lookup bottleneck, scale cực tốt. Nhược điểm: vì không ai "canh" từng object riêng lẻ, một thay đổi crushmap (dù nhỏ) có thể khiến hàng loạt object phải tính lại vị trí và di chuyển (data movement) — cần hiểu rõ trước khi sửa tay.

## Vì sao cần CRUSH thay vì bảng tra cứu

| Cách tiếp cận | Ưu điểm | Nhược điểm |
|---|---|---|
| Bảng tra cứu tập trung (kiểu NameNode/DOM) | Đơn giản, dễ audit từng object | Bottleneck khi scale lớn, single point of failure, bảng phình theo số object |
| **CRUSH (deterministic hash + topology map)** | Không bottleneck, client tự tính, scale hàng nghìn OSD | Sửa topology sai ảnh hưởng ngay lập tức toàn cụm; khó "trace" thủ công từng object |

CRUSH chỉ cần biết **topology** (cluster có bao nhiêu host/rack, OSD nào thuộc host nào, weight bao nhiêu) — không cần biết object nào đang ở đâu. Với cùng input (object ID + crushmap + rule), mọi client trong cluster luôn tính ra **cùng một kết quả** — đó là tính "deterministic".

## Cấu trúc CRUSH Map: buckets & hierarchy

CRUSH map là một cây phân cấp (hierarchy) các **bucket**, đáy cây là OSD (leaf), các tầng trên là nhóm logic phản ánh topology vật lý thật:

```
root default
 └─ datacenter dc1
     └─ rack rack1
         └─ host osd-node01
             ├─ osd.0 (weight 1.818, ~1.8TB)
             ├─ osd.1 (weight 1.818)
             └─ osd.2 (weight 3.637, ~3.6TB)
         └─ host osd-node02
             ├─ osd.3
             ├─ osd.4
```

Xem cây hiện tại:

```bash
ceph osd crush tree
ceph osd tree          # tương tự, kèm trạng thái up/down
```

### Weight — hai loại rất dễ nhầm

Đây là một trong những nhầm lẫn phổ biến nhất khi vận hành Ceph.

| Lệnh | Tên gọi | Ý nghĩa | Phạm vi giá trị | Tính chất |
|---|---|---|---|---|
| `ceph osd crush reweight osd.N <weight>` | **CRUSH weight** | Trọng số phản ánh **dung lượng vật lý** của disk (thường = dung lượng TB, vd 3.6 cho ổ 3.6TB) | 0 → bất kỳ số dương | **Vĩnh viễn** — thay đổi thật sự bao nhiêu % dữ liệu OSD đó nên chứa |
| `ceph osd reweight osd.N <weight>` | **Override weight (reweight)** | Hệ số **tạm thời** nhân thêm vào CRUSH weight khi tính placement, dùng để giảm tải nhanh 1 OSD | 0 → 1 | **Tạm thời** — dùng để throttle backfill/recovery, KHÔNG phản ánh dung lượng thật |
| `ceph osd crush reweight-subtree` | CRUSH weight cho cả subtree | Đổi weight toàn bộ host/rack cùng lúc | | Vĩnh viễn |

```bash
# Đúng: khi thêm ổ cứng 3.6TB mới, weight phản ánh dung lượng thật
ceph osd crush reweight osd.12 3.637

# Đúng: khi 1 OSD đang gần đầy (near-full), muốn giảm tạm data trên nó
# mà KHÔNG đổi ý nghĩa "OSD này có bao nhiêu dung lượng"
ceph osd reweight osd.12 0.8

# Ceph còn tự động hoá việc reweight tạm thời này qua module balancer
ceph balancer status
ceph balancer mode upmap   # khuyến nghị mặc định ở bản hiện đại, hiệu quả hơn reweight thủ công
```

> [!warning] Lesson learned: nhầm `crush reweight` với `reweight` gây hậu quả ngược nhau
> Rất nhiều operator mới thấy 1 OSD gần đầy, muốn "giảm tải" và gõ nhầm `ceph osd crush reweight osd.X 0.5` — lệnh này **thay đổi vĩnh viễn** ý nghĩa "OSD này có bao nhiêu TB", làm sai lệch capacity toàn cụm và kích hoạt data movement lớn không cần thiết trên toàn bộ pool liên quan, chứ không chỉ throttle riêng OSD đó. Muốn giảm tải tạm thời (throttle), luôn dùng `ceph osd reweight` (không có chữ `crush`) hoặc để `ceph balancer` tự làm — chỉ dùng `crush reweight` khi thật sự cần khai báo lại dung lượng vật lý (thêm/đổi ổ cứng).

## Failure domain & CRUSH rule

**Failure domain** xác định mức độ "tách biệt vật lý" tối thiểu giữa các bản sao (replica) hoặc mảnh EC của cùng 1 object — tương đương khái niệm **Fault Domain** trong vSAN, nhưng linh hoạt hơn (host, rack, datacenter, hoặc tự định nghĩa bucket type).

```bash
# Xem rule hiện tại
ceph osd crush rule dump replicated_rule

# Tạo rule mới: failure domain = host (khuyến nghị tối thiểu cho production)
ceph osd crush rule create-replicated replicated_host default host

# Gán rule cho 1 pool
ceph osd pool set my-pool crush_rule replicated_host
```

> [!warning] Lesson learned: replication size=3 nhưng failure domain=osd → "giả redundancy"
> Nếu CRUSH rule dùng failure domain `osd` thay vì `host`, Ceph chỉ đảm bảo 3 bản sao nằm trên **3 OSD khác nhau** — hoàn toàn có thể là 3 OSD **cùng nằm trên 1 host vật lý**. Cluster báo `active+clean` bình thường, nhìn health hoàn toàn khỏe mạnh, nhưng khi host đó chết (mất điện, hỏng mainboard), bạn mất **toàn bộ 3 bản sao cùng lúc** — mất dữ liệu thật sự dù "replication size 3" nghe có vẻ an toàn. Luôn kiểm tra failure domain thực tế của rule đang dùng bằng `ceph osd crush rule dump <rule-name>` trước khi tin vào con số replication size. Mặc định `ceph-deploy`/`cephadm` khi bootstrap thường tạo failure domain=host sẵn, nhưng rule custom (đặc biệt do người vận hành cũ tự tạo) có thể sai.

## Xem, decompile, edit, recompile CRUSH map

Với các thay đổi phức tạp (thêm bucket type tùy chỉnh, sửa rule nâng cao) không tiện làm qua CLI từng lệnh, có thể thao tác trực tiếp trên file map nhị phân:

```bash
# 1. Lấy crushmap hiện tại (dạng binary)
ceph osd getcrushmap -o crushmap.bin

# 2. Decompile ra text để đọc/sửa được
crushtool -d crushmap.bin -o crushmap.txt

# 3. Sửa file crushmap.txt (thêm bucket, sửa rule...) bằng editor

# 4. Recompile lại thành binary
crushtool -c crushmap.txt -o crushmap-new.bin

# 5. (khuyến nghị) test thử trước khi áp dụng — mô phỏng phân bố PG
crushtool -i crushmap-new.bin --test --show-mappings --rule 0 --num-rep 3

# 6. Apply vào cluster
ceph osd setcrushmap -i crushmap-new.bin
```

> [!warning] Lesson learned: apply crushmap sai = data movement toàn cụm ngay lập tức
> `ceph osd setcrushmap` áp dụng **ngay lập tức** cho toàn bộ cluster — không có staged rollout, không có "dry-run" ở bước apply (chỉ có `crushtool --test` ở bước trước đó để mô phỏng). Một lỗi nhỏ (gõ nhầm weight, xóa nhầm 1 bucket) có thể kích hoạt hàng loạt PG remap và backfill đồng thời, làm cluster load spike đột ngột, ảnh hưởng latency I/O production. Luôn `crushtool --test` kỹ, backup crushmap cũ (`crushmap.bin` ở bước 1) trước khi setcrushmap, và cân nhắc thực hiện ngoài giờ cao điểm.

## Bảng lệnh tham khảo nhanh

| Việc cần làm | Lệnh |
|---|---|
| Xem cây CRUSH | `ceph osd crush tree` |
| Xem 1 rule cụ thể | `ceph osd crush rule dump <rule>` |
| Liệt kê tất cả rule | `ceph osd crush rule ls` |
| Đổi CRUSH weight (dung lượng) | `ceph osd crush reweight osd.N <weight>` |
| Đổi override weight (tạm thời) | `ceph osd reweight osd.N <0..1>` |
| Di chuyển OSD sang host/bucket khác | `ceph osd crush move osd.N host=<new-host>` |
| Thêm bucket mới (vd rack) | `ceph osd crush add-bucket rack1 rack` |
| Xuất crushmap | `ceph osd getcrushmap -o file.bin` |
| Áp crushmap mới | `ceph osd setcrushmap -i file.bin` |

---
*Xem thêm: [[RADOS & Cluster Architecture]] | [[Placement Groups (PG)]] | [[OSD - Object Storage Daemon]] | [[Ceph|Ceph]]*
