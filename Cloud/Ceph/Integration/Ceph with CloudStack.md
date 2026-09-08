---
tags:
  - ceph
  - cloudstack
  - integration
  - rbd
---

# Ceph with CloudStack

Bài này đào sâu vào cách **Apache CloudStack** (hypervisor **KVM**) tiêu thụ Ceph làm **Primary Storage** qua **RBD**. Nếu bạn đã đọc [[Primary Storage Backends]] ở vault CloudStack (góc nhìn từ phía CloudStack), thì đây là góc nhìn ngược lại — từ phía Ceph: điều gì thực sự xảy ra trên cluster khi CloudStack gọi `createStoragePool`, tạo volume, chụp snapshot, hay xóa VM.

> [!tip] So với VMware vSAN
> Với vSAN, ESXi **là** cả compute lẫn storage node — không có khái niệm "client kết nối vào cluster storage riêng". Với Ceph + CloudStack/KVM, Ceph là một **hệ phân tán độc lập hoàn toàn**, CloudStack (và host KVM) chỉ là **client** gọi vào qua `librbd`. Ceph cluster có thể (và thường nên) chạy trên phần cứng tách biệt khỏi compute node — giống mô hình storage-array truyền thống hơn là mô hình HCI của vSAN.

## Kiến trúc: ai thực sự nói chuyện với Ceph?

Điểm hay bị hiểu nhầm nhất: **Management Server (MS) không nằm trong đường I/O**.

| Thành phần | Nói chuyện với Ceph để làm gì | Qua đường nào |
|---|---|---|
| **Management Server** | Provisioning actions: tạo/xóa volume, tạo/xóa snapshot, resize, lấy thông tin pool/capacity | Gọi lệnh `rbd`/`ceph` tương đương qua plugin `KVMStorageProcessor`/`LibvirtStorageAdaptor` khi cần metadata, nhưng bản thân **I/O của VM không đi qua MS** |
| **KVM Host (libvirt/QEMU)** | Đọc/ghi dữ liệu thật của VM đang chạy | `librbd` — QEMU built-in RBD driver, kết nối thẳng tới MON + OSD, **không qua MS** |
| **Ceph MON** | Cung cấp cluster map (osdmap, crushmap...) cho client (QEMU/librbd) tính toán vị trí object | Giao thức Ceph messenger (msgr2, port 3300/6789) |
| **Ceph OSD** | Lưu trữ object thật, phục vụ I/O trực tiếp từ QEMU | librbd → OSD trực tiếp, không qua MON sau khi đã có cluster map |

```mermaid
graph LR
    MS[CloudStack<br/>Management Server] -->|provisioning: create/snap/resize<br/>rbd/ceph CLI-equivalent calls| MON[Ceph MON]
    subgraph KVMHOST["KVM Host"]
        LIBVIRT[libvirt] --> QEMU[QEMU]
        QEMU -->|librbd - đường I/O thật| OSD[Ceph OSD]
    end
    QEMU -.lấy cluster map lúc đầu.-> MON
    MS -.điều phối, không I/O.-> LIBVIRT
```

> [!warning] Lesson learned: đừng debug I/O chậm bằng cách nhìn vào Management Server log
> Một kỹ sư mới từng dành nửa ngày soi `management-server.log` vì VM báo I/O timeout, trong khi nguyên nhân thật là 2 OSD trên 1 node Ceph bị full và đang throttle write. MS log chỉ ghi lại việc **API call chậm/timeout** ở tầng provisioning — nó **không thấy** những gì xảy ra bên trong luồng I/O thật (đó là việc giữa QEMU và OSD, MS không tham gia). Quy tắc: **nếu volume "bị chậm" hay "bị treo" trong lúc VM đang chạy bình thường (không phải lúc tạo/resize)**, đây gần như chắc chắn là vấn đề I/O path (QEMU↔OSD), hãy nhảy thẳng sang `ceph -s`, `ceph health detail`, và [[Ceph Troubleshooting]] thay vì tiếp tục đọc log CloudStack.

## Đăng ký Primary Storage: `createStoragePool` URL

```bash
cmk createStoragePool name=ceph-primary-01 \
  zoneid=<zone-id> podid=<pod-id> clusterid=<cluster-id> \
  url="rbd://cloudstack:AQDx...==@10.10.20.11;10.10.20.12;10.10.20.13/cloudstack-primary"
```

| Phần URL | Ý nghĩa |
|---|---|
| `rbd://` | Scheme báo cho CloudStack biết đây là RBD storage pool |
| `cloudstack` | **cephx user** (không phải `client.admin` — xem phần cephx bên dưới) |
| `AQDx...==` | **cephx secret key** (base64), lấy bằng `ceph auth get-key client.cloudstack` |
| `10.10.20.11;10.10.20.12;10.10.20.13` | Danh sách **MON IP**, phân cách bởi `;` — nên liệt kê **toàn bộ MON** để chịu được 1-2 MON chết |
| `/cloudstack-primary` | Tên **pool** RBD sẽ dùng làm primary storage |

> [!tip] Không cần điền `.keyring` file
> Khác với việc cấu hình client Ceph thủ công (`/etc/ceph/ceph.client.xxx.keyring`), CloudStack tự lưu secret vào **libvirt secret** trên từng host khi bạn `createStoragePool` — nó không phụ thuộc file keyring nằm sẵn trên host. Tuy vậy `ceph-common` (cung cấp `rbd`, `ceph` CLI để bạn tự debug tay) vẫn nên cài — xem [[Ceph CLI Cheatsheet]].

## cephx user riêng cho CloudStack — đừng dùng `client.admin`

```bash
# Tạo pool riêng cho CloudStack primary storage trước
ceph osd pool create cloudstack-primary 128 128
rbd pool init cloudstack-primary

# Tạo cephx user scoped đúng least-privilege cho pool này
ceph auth get-or-create client.cloudstack \
  mon 'profile rbd' \
  osd 'profile rbd pool=cloudstack-primary' \
  -o /etc/ceph/ceph.client.cloudstack.keyring

# Lấy key để nhét vào URL createStoragePool
ceph auth get-key client.cloudstack
```

| Scope | Vì sao |
|---|---|
| `mon 'profile rbd'` | Cho phép client đọc cluster map, mở/khoá RBD image — không cấp quyền quản trị MON |
| `osd 'profile rbd pool=cloudstack-primary'` | Chỉ cho phép thao tác RBD (read/write object, watch/notify cho exclusive-lock) **trong đúng pool** CloudStack dùng — không đụng được pool khác (VD pool RGW, CephFS metadata...) |
| Không cấp `mgr` | CloudStack không cần gọi MGR module, không nên cấp thừa |

> [!warning] Lesson learned: dùng `client.admin` cho "nhanh" lúc setup rồi quên đổi
> Một team dựng PoC dùng luôn `client.admin` trong `createStoragePool` cho tiện, kế hoạch "sẽ đổi khi lên production" — rồi quên. Hậu quả: nếu URL này (chứa cephx secret) từng bị log ra đâu đó (audit log, screen-share, backup config file không mã hoá), kẻ tấn công có được key **có toàn quyền trên toàn bộ Ceph cluster**, không chỉ pool của CloudStack — bao gồm pool của các hệ thống khác dùng chung cluster (RGW, CephFS...). Luôn tạo user scoped ngay từ đầu, kể cả môi trường PoC — thói quen đúng phải hình thành từ ngày 1. Xem thêm [[Ceph Security Considerations]].

## libvirt secret — bắt buộc đúng trên **mọi** KVM host

Khi bạn `createStoragePool` qua UI/API, CloudStack **tự động** đẩy libvirt secret xuống các host trong cluster đó. Nhưng bạn nên biết cơ chế thật để tự verify/khắc phục khi có vấn đề:

```bash
# Trên từng KVM host — verify secret đã tồn tại
virsh secret-list
# UUID                                  Usage
# ----------------------------------------------------------------
# f2a1b3c4-....                         ceph client.cloudstack secret

# Xem giá trị đang lưu (không hiện plaintext, chỉ confirm có set)
virsh secret-get-value f2a1b3c4-....

# Nếu cần set/refresh tay (VD sau khi rotate key ở Ceph)
virsh secret-set-value --secret f2a1b3c4-.... \
  --base64 "$(ceph auth get-key client.cloudstack)"
```

Vì sao **mọi host** trong cluster đều phải có secret đúng, không chỉ 1 host:

- Live migration di chuyển VM (và quyền truy cập RBD image của nó) sang host khác **bất kỳ lúc nào** — host đích phải tự attach được vào cùng RBD image bằng cùng cephx key.
- HA (host chết, VM được recreate trên host khác) cũng cần host thay thế có secret hợp lệ **sẵn từ trước** — không có thời gian chờ đồng bộ lúc failover.
- Một host thiếu/lệch secret sẽ **không tự báo lỗi cho tới khi thực sự cần dùng** (VM mới khởi tạo trên host đó, hoặc VM migrate tới) — đây chính là lý do gotcha cephx rotation bên dưới nguy hiểm.

## Vòng đời volume CloudStack ↔ thao tác RBD

| Hành động CloudStack | Thao tác RBD tương ứng | Ghi chú |
|---|---|---|
| Tạo Data Disk / Root Disk mới | `rbd create --size <N> --image-feature layering cloudstack-primary/<vol-uuid>` | Feature `layering` bắt buộc để hỗ trợ clone sau này |
| Deploy VM từ Template | `rbd clone` (COW) từ base image template → `rbd flatten` sau đó tuỳ cấu hình | Xem phần "flatten" bên dưới |
| Tạo Volume Snapshot | `rbd snap create cloudstack-primary/<vol-uuid>@<snap-uuid>` | Native Ceph snapshot — nhanh, gần như tức thời (copy-on-write ở tầng object) |
| Tạo Template từ Snapshot / Create Volume from Snapshot | `rbd clone` từ snapshot | Volume mới là **clone**, phụ thuộc (tham chiếu) vào snapshot gốc cho tới khi flatten |
| Detach Volume (tuỳ chọn "full clone") | `rbd flatten` | Copy toàn bộ data từ parent xuống, cắt đứt phụ thuộc COW — tốn I/O + thời gian nếu volume lớn |
| Xoá Volume | `rbd rm cloudstack-primary/<vol-uuid>` | **Bị chặn (fail)** nếu còn snapshot hoặc clone con tham chiếu tới nó — CloudStack phải xoá hết snapshot con trước, hoặc gọi `rbd flatten` lên các clone con trước |
| Resize Volume | `rbd resize` | Online resize được hỗ trợ, nhưng VM guest OS vẫn cần tự nhận dung lượng mới (rescan/growpart) |

> [!tip] Vì sao xoá volume đôi khi báo lỗi "vẫn còn tham chiếu"
> Đây không phải bug CloudStack — nó phản ánh đúng ràng buộc COW của RBD: một snapshot/clone con vẫn đang "mượn" object từ parent, xoá parent trong khi con còn sống sẽ phá vỡ chain dữ liệu. Đây cũng chính là hành vi đã ghi ở [[Primary Storage Backends]] phía CloudStack — flow xử lý (xoá snapshot con trước, hoặc flatten) là logic CloudStack tự làm giúp bạn trong đa số trường hợp, nhưng khi thao tác `rbd` tay để dọn dẹp thủ công, bạn phải tự nhớ thứ tự này.

## Image features — layering là tối thiểu, cẩn thận feature khác

```bash
# Feature set an toàn, tương thích rộng cho CloudStack + QEMU
rbd create --size 20G --image-feature layering cloudstack-primary/vol-example

# Kiểm tra feature đang bật trên 1 image
rbd info cloudstack-primary/vol-example
```

| Feature | Cần cho CloudStack? | Rủi ro nếu bật ẩu |
|---|---|---|
| `layering` | **Bắt buộc** — nền tảng cho clone/snapshot | Không có rủi ro, luôn nên bật |
| `exclusive-lock` | Thường bật mặc định (Ceph mặc định từ Luminous+) | Cần QEMU/librbd version đủ mới hỗ trợ đúng; version cũ có thể lock conflict khi live-migrate |
| `object-map`, `fast-diff` | Tăng tốc `rbd du`, incremental snapshot | Phụ thuộc `exclusive-lock`; nếu QEMU version không đồng bộ hỗ trợ, có thể gây lỗi mount hoặc object-map bị "invalid" cần rebuild (`rbd object-map rebuild`) |
| `deep-flatten` | Hữu ích khi flatten clone nhiều tầng | An toàn để bật, ít rủi ro |
| `journaling` | Dùng cho RBD mirroring (DR), CloudStack không cần | Tốn overhead I/O nếu bật thừa mà không dùng mirroring |

> [!warning] Lesson learned: bật hết feature "cho hiện đại" rồi QEMU cũ không mount được
> Một lần nâng cấp Ceph cluster, admin đổi default image features trong `ceph.conf` (`rbd_default_features`) sang bộ đầy đủ mới nhất mà không kiểm tra version `qemu-kvm`/`librbd` trên các KVM host cũ hơn trong cluster. Volume mới tạo dùng feature set mới, nhưng vài host KVM cũ chưa upgrade **không attach được** volume đó — VM tạo mới trên các host đó fail thẳng ở bước boot. Bài học: **`rbd_default_features` áp dụng toàn cluster, nhưng khả năng hỗ trợ nằm ở từng KVM host** — luôn đồng bộ version `qemu-kvm`/`librbd` trên toàn bộ host trước khi đổi default feature set, hoặc set feature tường minh mỗi lần tạo thay vì tin vào default.

## Storage Tags — điều hướng offering vào đúng pool Ceph

Nếu cluster có nhiều Primary Storage (VD: 1 Ceph pool NVMe nhanh cho tier "premium", 1 pool HDD cho tier "archive"), dùng **storage tag** để ép Disk Offering/Service Offering chọn đúng pool:

```bash
# Gắn tag khi thêm storage pool
cmk createStoragePool name=ceph-nvme-fast tags=ceph-nvme \
  url="rbd://cloudstack:...@mon1;mon2;mon3/cloudstack-nvme" ...

# Disk offering ép dùng đúng tag này
cmk createDiskOffering name="Premium-NVMe" storagetype=shared \
  tags=ceph-nvme customized=true
```

Cơ chế storage tag hoàn toàn nằm ở phía CloudStack (không phải khái niệm Ceph) — nhưng việc **chọn đúng pool Ceph nào ứng với tier hiệu năng nào** là quyết định kiến trúc cần làm ở tầng Ceph trước (CRUSH rule tách theo device class NVMe/HDD — xem [[CRUSH Algorithm & CRUSH Map]]) rồi mới map ngược qua tag.

## EC pool KHÔNG dùng trực tiếp làm CloudStack primary storage

> [!warning] Lesson learned: trỏ CloudStack thẳng vào Erasure Coded pool sẽ fail
> Một đội hạ tầng muốn tiết kiệm chi phí, tạo pool **erasure-coded** (VD 4+2) rồi thử `createStoragePool` trỏ thẳng vào đó để tận dụng overhead thấp hơn 3x-replica — và gặp lỗi ngay khi CloudStack cố tạo volume/snapshot. Nguyên nhân: RBD image cần ghi **metadata và omap** (object map, exclusive-lock, journal...) mà **EC pool không hỗ trợ omap** (EC pool chỉ lưu được data object, không có transaction/xattr đầy đủ như replicated pool). Giải pháp đúng: tạo **2 pool** — 1 EC pool chứa **data thật** (tiết kiệm dung lượng), 1 pool **replicated nhỏ** chứa metadata, rồi tạo image với `--data-pool`:
> ```bash
> ceph osd pool create cloudstack-meta 32 32                  # replicated, nhỏ
> ceph osd pool create cloudstack-ecdata 128 128 erasure ec-42-profile
> rbd pool init cloudstack-meta
> # Tạo image: metadata nằm ở pool replicated, data thật nằm ở EC pool
> rbd create --size 100G --data-pool cloudstack-ecdata \
>   --image-feature layering cloudstack-meta/vol-example
> ```
> CloudStack `createStoragePool` vẫn trỏ vào pool **replicated** (`cloudstack-meta`) như bình thường — cấu hình `--data-pool` phải set sẵn ở cấp default config (`rbd default data pool`) trên cluster nếu muốn mọi volume CloudStack tạo tự động dùng EC data pool, vì CloudStack tự gọi `rbd create` không có tham số `--data-pool` tường minh. Việc này cần test kỹ trước khi áp dụng production — xem thêm [[Pools, Replication & Erasure Coding|Replication & Erasure Coding]].

## Xoay vòng cephx key — quy trình phối hợp bắt buộc

Đây là gotcha đã được nhắc ngắn gọn ở [[Primary Storage Backends]] phía CloudStack; ở đây là **vì sao nó xảy ra** và **quy trình đúng**.

**Vì sao nó xảy ra:** libvirt secret trên mỗi KVM host là một **bản sao cephx key tại thời điểm `createStoragePool`**, được lưu độc lập trong `/etc/libvirt/secrets/`. Ceph không có cơ chế "push" key mới xuống client — khi bạn `ceph auth caps`/tạo lại key cho `client.cloudstack`, **Ceph cluster đổi ngay lập tức**, nhưng libvirt secret trên từng host **vẫn giữ giá trị cũ** cho tới khi có ai chủ động cập nhật.

**Vì sao khó phát hiện sớm:** VM đang chạy **không bị ảnh hưởng ngay** — QEMU process đã authenticate xong lúc VM start, connection tới OSD vẫn giữ session hiện tại. Chỉ **thao tác I/O mới cần re-authenticate** (VM mới, snapshot mới, host reboot khiến QEMU phải reconnect) mới lộ ra lỗi `-EACCES`/permission denied — nghĩa là triệu chứng xuất hiện **trễ, ngẫu nhiên theo host**, rất dễ bị hiểu nhầm thành "storage cluster đang có vấn đề" thay vì "key bị lệch".

**Quy trình rotate đúng (coordinated):**

```bash
# Bước 1 — Ceph: tạo/rotate key cho client.cloudstack
ceph auth get-or-create client.cloudstack \
  mon 'profile rbd' osd 'profile rbd pool=cloudstack-primary' \
  -o /tmp/new-cloudstack.keyring
NEWKEY=$(ceph auth get-key client.cloudstack)

# Bước 2 — NGAY LẬP TỨC cập nhật libvirt secret trên TOÀN BỘ KVM host
# (script/Ansible loop qua danh sách host trong cluster, không làm tay từng host)
for HOST in $(cat kvm-hosts.txt); do
  ssh "$HOST" "virsh secret-set-value --secret <uuid> --base64 $NEWKEY"
done

# Bước 3 — Verify trên từng host trước khi coi là xong
for HOST in $(cat kvm-hosts.txt); do
  ssh "$HOST" "virsh secret-get-value <uuid> >/dev/null && echo OK $HOST || echo FAIL $HOST"
done

# Bước 4 — Test thật: tạo 1 volume/snapshot nhỏ qua CloudStack để xác nhận
# end-to-end trước khi thông báo hoàn tất
```

Nguyên tắc: **không bao giờ coi rotation là "xong" chỉ vì lệnh `ceph auth` chạy thành công** — phải xác nhận **mọi** host đã cập nhật và có ít nhất 1 test I/O thật thành công.

## Giám sát ranh giới CloudStack ↔ Ceph

CloudStack chỉ nhìn thấy 3 trạng thái cho mỗi storage operation: **success / fail / timeout**. Nó **không** biết (và không cần biết) *tại sao* — OSD chậm do rebalance, network saturation, disk gần full, PG stuck... tất cả đều trông giống nhau từ phía CloudStack: "volume operation took too long" hoặc "operation failed".

| Triệu chứng ở CloudStack | Khả năng cao nguyên nhân ở Ceph | Lệnh kiểm tra |
|---|---|---|
| Tạo volume/VM bị treo lâu, cuối cùng timeout | Cluster đang `HEALTH_WARN`/`ERR`, OSD down, hoặc pool gần `nearfull` | `ceph -s`, `ceph health detail` |
| I/O của VM đang chạy chậm bất thường (không liên quan action nào ở CloudStack) | Recovery/backfill đang chạy sau khi thêm/mất OSD, chiếm băng thông I/O | `ceph -s` (mục recovery), `ceph osd pool stats` |
| Xoá volume fail | Snapshot/clone con còn tồn tại (RBD constraint, không phải lỗi CloudStack) | `rbd children`, `rbd snap ls` |
| Toàn bộ storage pool "Alert" trong CloudStack | MON quorum mất, hoặc network giữa host và MON bị đứt | `ceph quorum_status`, kiểm tra network public/cluster |

> [!tip] Quy tắc vàng khi vận hành
> **Nếu CloudStack "storage chậm" — đi kiểm tra Ceph trước, không chỉ đọc log CloudStack.** Chạy `ceph -s` là bước đầu tiên luôn luôn đúng trong 90% trường hợp storage bất thường ở CloudStack/KVM. Xem quy trình đầy đủ ở [[Ceph Troubleshooting]].

---
*Xem thêm: [[RBD - Block Storage]] | [[Ceph Security Considerations]] | [[Ceph CLI Cheatsheet]] | [[Primary Storage Backends]] | [[Lessons Learned & Common Pitfalls]] | [[Ceph|Ceph]]*
