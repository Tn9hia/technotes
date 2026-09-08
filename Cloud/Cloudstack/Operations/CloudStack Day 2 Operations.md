---
tags:
  - cloudstack
  - operations
  - day2
---

# CloudStack Day 2 Operations

## Vận hành hằng ngày — checklist nhanh

- [ ] Kiểm tra `cmk list alerts` — có alert mới nào chưa xử lý không
- [ ] Kiểm tra capacity Zone/Pod/Cluster (CPU, RAM, Storage) — `cmk list capacity`
- [ ] Kiểm tra trạng thái Host: có host nào `Disconnected`/`Alert` không
- [ ] Kiểm tra System VM: SSVM/CPVM/VR có VM nào `Stopped`/`Error` bất thường không
- [ ] Kiểm tra Galera cluster health (nếu có DB HA)
- [ ] Kiểm tra dung lượng Secondary Storage còn trống
- [ ] Xem log Management Server có exception lặp lại bất thường không

## Maintenance Mode cho Host

```bash
cmk prepareHostForMaintenance id=<host-id>
# CloudStack cố live-migrate VM sang host khác cùng cluster
cmk listVirtualMachines hostid=<host-id>   # kiểm tra còn VM nào chưa di chuyển xong không

# Sau khi bảo trì xong
cmk cancelHostMaintenance id=<host-id>
```

> [!warning] Xem thêm cảnh báo chi tiết ở [[Zones, Pods, Clusters & Hosts]]
> Maintenance mode có thể treo vô thời hạn nếu không đủ capacity hoặc VM dùng local storage — luôn kiểm tra trước khi bắt đầu bảo trì theo lịch.

## Patch / Update Agent trên KVM Host (rolling, không downtime toàn cluster)

```
Với mỗi host trong cluster (làm tuần tự, KHÔNG song song toàn bộ):
1. prepareHostForMaintenance (di dời hết VM)
2. apt update && apt upgrade cloudstack-agent qemu-kvm libvirt-daemon-system
3. reboot (nếu cần, VD: cập nhật kernel)
4. Kiểm tra agent kết nối lại MS thành công (state = Up)
5. cancelHostMaintenance
6. Theo dõi vài phút trước khi chuyển sang host tiếp theo
```

> [!warning] Lesson learned: patch đồng loạt cả cluster = mất live-migration an toàn
> Nếu vá lỗi/khởi động lại nhiều host cùng lúc trong 1 cluster, tại một thời điểm sẽ **không đủ host "khỏe" để chứa VM di dời từ host đang bảo trì** — CloudStack có thể chọn cách dừng VM tạm hoặc treo job maintenance. Luôn patch **tuần tự từng host**, đảm bảo cluster luôn còn dư capacity, giống nguyên tắc rolling update ESXi qua vSphere Lifecycle Manager.

## Xử lý VM "kẹt" trạng thái

```bash
# VM kẹt ở "Expunging"/"Stopping"/"Migrating" quá lâu
cmk listVirtualMachines id=<vm-id>

# Force stop (chỉ dùng khi chắc chắn cần thiết, có thể để lại state không nhất quán)
cmk stopVirtualMachine id=<vm-id> forced=true

# Reset trạng thái VM (dùng thận trọng — chỉ sửa metadata DB, không đảm bảo hypervisor đồng bộ theo)
cmk updateVirtualMachine id=<vm-id> ...
```

> [!warning] `forced=true` không phải nút "sửa mọi thứ"
> Force stop chỉ ép CloudStack coi VM là đã dừng ở tầng DB — nếu hypervisor thực tế **chưa** dừng được VM (do lỗi libvirt/qemu), bạn sẽ có tình trạng **DB nói VM đã Stopped nhưng process qemu vẫn chạy trên host**, dẫn tới nguy cơ start lại VM đó ở nơi khác trong khi bản gốc vẫn sống — tương tự rủi ro split-brain mô tả ở [[CloudStack HA Architecture]]. Luôn xác nhận trực tiếp trên host (`virsh list --all`) trước khi dùng `forced=true` trong tình huống nghi ngờ.

## Dọn dẹp định kỳ

```bash
# Expunge VM đã xóa (mặc định có delay trước khi xóa hẳn — global setting expunge.delay)
cmk updateConfiguration name=expunge.delay value=86400
cmk updateConfiguration name=expunge.interval value=86400

# Dọn snapshot/template cũ theo policy nội bộ (không có lệnh "cleanup all" mặc định — cần script)
cmk listSnapshots listall=true
```

> [!tip] `expunge.delay` là "thùng rác" trước khi xóa vĩnh viễn
> Giống Recycle Bin, VM bị xóa (`destroyVirtualMachine`) không mất ngay — nó chờ `expunge.delay` giây rồi mới bị xóa vĩnh viễn (giải phóng storage thật). Đây là cơ hội cứu vãn nếu ai đó lỡ tay xóa nhầm VM — biết rõ giá trị này đang được set bao lâu trong hệ thống bàn giao rất quan trọng để biết "cửa sổ cứu hộ" còn bao lâu.

## Backup DB định kỳ (nhắc lại, cực kỳ quan trọng)

Xem chi tiết ở [[Database HA - MySQL Galera]]. Đảm bảo có cron job backup DB **và đã test restore thử** ít nhất 1 lần.

## Quy trình xử lý sự cố chuẩn (runbook tối thiểu)

```
1. Xác định phạm vi: 1 VM? 1 network? 1 cluster? toàn hệ thống?
2. Kiểm tra cmk list alerts + log Management Server trước tiên
3. Nếu liên quan network → theo flow ở [[Virtual Router Deep Dive]]
4. Nếu liên quan storage → theo flow ở [[Primary Storage Backends]]
5. Nếu nghi ngờ liên quan DB/MS → theo [[Database HA - MySQL Galera]] và [[CloudStack Management Server]]
6. Ghi lại timeline sự cố + nguyên nhân gốc (root cause) sau khi xử lý xong
7. Cập nhật vào [[Lessons Learned & Common Pitfalls]] nếu là bài học mới
```

---
*Xem thêm: [[CloudStack Monitoring & Alerting]] | [[CloudStack Troubleshooting]] | [[CloudStack HA Architecture]] | [[Cloudstack|CloudStack]]*
