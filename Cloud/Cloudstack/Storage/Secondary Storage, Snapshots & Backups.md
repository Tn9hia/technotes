---
tags:
  - cloudstack
  - storage
  - backup
---

# Secondary Storage, Snapshots & Backups

## Secondary Storage

- Lưu: **Template**, **ISO**, **Volume Snapshot**, và (tùy version) system VM template.
- Backend phổ biến: **NFS**, hoặc **S3-compatible object storage** (AWS S3, MinIO, Ceph RGW) qua **NFS Secondary Staging + Object Store**, hoặc **NFS thuần**.
- Dùng chung cho **toàn bộ Zone** (khác Primary Storage vốn thường gắn theo Cluster).

```bash
cmk addImageStore zoneid=<zone-id> provider=NFS \
  url="nfs://192.168.1.20/export/secondary"

# Hoặc S3-compatible
cmk addImageStore zoneid=<zone-id> provider=S3 \
  url="https://s3.example.com" details[0].key=accesskey details[0].value=<key> ...
```

> [!tip] So với VMware
> Không có tương đương trực tiếp. Gần giống việc gộp chung **Content Library** (cho template/ISO) và **backup repository** (cho snapshot) vào 1 nơi duy nhất — và khác biệt lớn: mọi truy cập Secondary Storage đều đi qua **SSVM** làm trung gian, không có host nào mount trực tiếp Secondary Storage để chạy VM từ đó (khác Content Library có thể serve trực tiếp).

## Template & ISO

```bash
# Đăng ký template mới (từ URL hoặc từ volume/snapshot có sẵn)
cmk registerTemplate name=ubuntu-22.04 displaytext="Ubuntu 22.04" \
  format=QCOW2 hypervisor=KVM ostypeid=<ostype-id> \
  url="http://repo.internal/images/ubuntu-22.04.qcow2" zoneid=<zone-id>

# Tạo template từ 1 VM/volume đang có (giống "Convert to Template" bên vSphere)
cmk createTemplate name=my-golden-image volumeid=<volume-id> \
  ostypeid=<ostype-id> displaytext="Golden image v2"
```

> [!warning] Lesson learned: format sai hypervisor = template "đăng ký được nhưng deploy fail"
> `format` (QCOW2, VHD, OVA, RAW...) phải khớp đúng hypervisor đích. CloudStack **cho đăng ký thành công** dù format không khớp hoàn toàn logic hypervisor hiện có trong zone, lỗi chỉ lộ ra khi **deploy VM** báo lỗi khó hiểu liên quan tới disk. Luôn kiểm tra kỹ `format` + `hypervisor` khớp với cluster đích trước khi đăng ký.

## Snapshot

- **Volume Snapshot**: chụp 1 disk cụ thể, lưu tại Secondary Storage (mặc định) hoặc giữ tại Primary Storage nếu backend hỗ trợ (Ceph, một số plugin).
- **KHÔNG có "VM Snapshot" tích hợp sẵn (đa VM/quiesce toàn bộ VM) theo kiểu vSphere Snapshot** trong core — CloudStack snapshot làm việc ở mức **từng volume**, muốn snapshot nhất quán toàn VM (nhiều disk) phải dùng tính năng **VM Snapshot** (có từ các bản mới, hạn chế theo hypervisor) hoặc snapshot từng disk gần như đồng thời.

```bash
# Tạo snapshot 1 volume
cmk createSnapshot volumeid=<volume-id> name=pre-upgrade-snap

# Snapshot Policy (lịch tự động — giống Scheduled Task snapshot bên vSphere)
cmk createSnapshotPolicy volumeid=<volume-id> intervaltype=DAILY \
  schedule="00:00" timezone="Asia/Ho_Chi_Minh" maxsnaps=7

# Tạo VM Snapshot (nếu hypervisor/version hỗ trợ) — chụp toàn bộ VM tại 1 thời điểm
cmk createVMSnapshot virtualmachineid=<vm-id> name=pre-patch --snapshotmemory=true
```

> [!warning] Lesson learned: đừng nhầm "Volume Snapshot" với "VM Snapshot"
> Dân VMware quen thói quen "chụp snapshot VM trước khi patch, rollback nếu lỗi" — thao tác đó tương đương **VM Snapshot** trong CloudStack (nếu hypervisor/version hỗ trợ đầy đủ), **không phải** Volume Snapshot đơn lẻ (chỉ chụp 1 disk, không đồng bộ trạng thái RAM/multi-disk). Nếu VM có nhiều disk và bạn chỉ snapshot volume root, rollback có thể khiến dữ liệu giữa root disk và data disk **không nhất quán** (giống rollback snapshot lẻ tẻ, sai thời điểm giữa các disk bên VMware).

## Backup Volume vs Backup Framework (mới)

- **Backup Volume** (cách cũ, thủ công): tạo snapshot rồi `createTemplate`/`copySnapshot` để lưu trữ dài hạn.
- **Backup and Recovery Framework** (các bản mới): tích hợp trực tiếp với **backup provider bên thứ 3** (VD: Veeam plugin, NAS-based providers) qua API chuẩn hóa — gần giống cách vSphere tích hợp Veeam/Commvault qua vSphere API/CBT.

```bash
cmk listBackupProviders
cmk createBackupOffering zoneid=<zone-id> ... 
cmk assignVirtualMachineToBackupOffering virtualmachineid=<vm-id> backupofferingid=<offering-id>
cmk createBackup virtualmachineid=<vm-id>
```

> [!tip] Kiểm tra ngay khi nhận bàn giao: hệ thống đang backup theo cách nào
> Vì có nhiều "thế hệ" cách làm backup trong CloudStack (thủ công qua snapshot script, tới Backup Framework chính thức), **phải hỏi rõ đội cũ** hệ thống hiện tại backup VM theo cơ chế nào, tần suất, retention, và **thử restore thật một lần** trước khi tin tưởng quy trình — không giả định "chắc là có backup" chỉ vì thấy snapshot tồn tại trong storage.

## Dung lượng Secondary Storage — điểm dễ bị bỏ quên

Secondary Storage tích lũy dần theo thời gian (template cũ không xóa, snapshot chưa dọn) — khác Primary Storage vốn được giám sát sát sao hơn vì ảnh hưởng trực tiếp VM đang chạy.

```bash
# Liệt kê template không còn dùng để dọn dẹp
cmk listTemplates templatefilter=all

# Xóa snapshot cũ theo policy retention
cmk deleteSnapshot id=<snapshot-id>
```

> [!warning] Lesson learned: Secondary Storage đầy → hàng loạt thao tác âm thầm fail
> Khi Secondary Storage gần đầy, các thao tác **tạo template mới, snapshot mới** sẽ fail nhưng thông báo lỗi trên UI đôi khi mơ hồ ("Failed" không rõ nguyên nhân). Cần đặt alert chủ động theo dung lượng Secondary Storage (xem [[CloudStack Monitoring & Alerting]]), đừng đợi đến khi thao tác fail mới phát hiện.

---
*Xem thêm: [[CloudStack Storage Overview]] | [[Primary Storage Backends]] | [[System VMs - SSVM, CPVM & Virtual Router]] | [[Cloudstack|CloudStack]]*
