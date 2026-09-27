---
tags:
  - operations
  - day2
---

# Day 2 Operations Cheatsheet

Tier: 2
Parent: [[Kvm-virtualization|KVM Virtualization]]

Các thao tác vận hành thường ngày sau khi VM đã chạy production — resize, thêm tài nguyên, patch host — không trùng với [[Virsh Cheatsheet]] (đó là lệnh cơ bản); note này tập trung vào **quy trình** cho việc thay đổi trên VM đang sống.

## Resize disk (tăng dung lượng)

```bash
# 1. Tăng dung lượng ở tầng backend TRƯỚC (tùy backend)
qemu-img resize vm01.qcow2 +20G                     # local qcow2/raw
rbd resize cloudstack-primary/vm01-disk --size 70G  # Ceph RBD, xem Ceph RBD with KVM

# 2. Báo cho QEMU biết disk đã lớn hơn (block device size, không cần restart VM)
virsh blockresize vm01 vda 70G

# 3. Bên TRONG guest — resize partition + filesystem (guest OS tự làm, KVM/libvirt không can thiệp)
# Linux: growpart /dev/vda 1 && resize2fs /dev/vda1  (hoặc xfs_growfs cho XFS)
# Windows: Disk Management > Extend Volume
```

> [!warning] Chỉ tăng được, không giảm — và luôn resize backend TRƯỚC khi resize trong guest
> `qemu-img resize`/`rbd resize` giảm dung lượng (dùng `--size` nhỏ hơn hiện tại) có rủi ro **mất dữ liệu ngay** nếu phần dữ liệu nằm trong vùng bị cắt — hầu như không có use case an toàn để giảm disk online. Chỉ tăng, không giảm.

## Thêm vCPU / RAM (hotplug — nếu guest OS hỗ trợ)

```bash
# vCPU hotplug — cần domain XML khai báo maxvcpu từ trước lúc VM còn tắt
virsh setvcpus vm01 4 --live         # tăng lên 4 vCPU đang chạy (không vượt maxvcpu đã định nghĩa)
virsh setvcpus vm01 6 --config --maximum   # đổi maxvcpu — CẦN RESTART VM mới áp dụng

# Memory hotplug qua virtio-balloon (xem Virtio Devices) — chỉ giảm/tăng trong giới hạn đã cấp lúc đầu
virsh setmem vm01 8G --live
virsh setmaxmem vm01 16G --config    # cần restart để đổi giới hạn tối đa
```

> [!warning] Lesson learned: hotplug vCPU/RAM cần guest OS hỗ trợ — không phải mọi guest tự nhận ngay
> Linux hiện đại tự nhận vCPU/RAM mới gần như ngay lập tức (qua ACPI hotplug event). Windows một số bản cần thêm bước trong Device Manager hoặc thậm chí không hỗ trợ hotplug RAM tùy edition. Luôn xác nhận guest OS đã "thấy" tài nguyên mới (`nproc`, `free -h` trong guest) trước khi coi thao tác là hoàn tất — đừng chỉ tin `virsh` báo thành công ở phía host.

## Patch/reboot host mà không downtime VM (maintenance mode)

```bash
# Quy trình chuẩn cho host cần patch/reboot trong cluster có shared storage
for vm in $(virsh list --name); do
  virsh migrate --live "$vm" qemu+ssh://host-khac/system
done
# Xác nhận host hiện tại không còn VM nào
virsh list --all
# Giờ mới patch/reboot an toàn
```

> [!tip] Với cluster orchestrator (CloudStack/OpenStack), dùng tính năng maintenance mode của orchestrator thay vì tự script migrate
> CloudStack có "Host Maintenance Mode" tự động drain VM ra host khác theo policy; tự viết script `virsh migrate` tay trên host do orchestrator quản lý dễ gây lệch state DB, xem lưu ý tương tự ở [[Libvirt Storage Pools & Volumes]].

## Đổi cấu hình cần restart vs không cần restart — bảng tra nhanh

| Thay đổi | Cần restart VM? |
|---|---|
| Tăng vCPU (trong giới hạn `maxvcpu`) | Không (`--live`) |
| Tăng `maxvcpu` | **Có** |
| Tăng RAM (trong giới hạn `maxmem`, qua balloon) | Không (`--live`) |
| Tăng `maxmem` | **Có** |
| Đổi machine type (`pc` ↔ `q35`) | **Có**, và rủi ro cao (xem [[QEMU Process Model & Machine Types]]) |
| Thêm/xóa disk, NIC | Không, nếu dùng `--live` với device hỗ trợ hotplug |
| Đổi CPU model | **Có** |
| Resize disk đã gắn | Không ở tầng block device, nhưng filesystem trong guest cần tự resize thêm |

## Resources

- `man virsh` — phần `setvcpus`, `setmem`, `blockresize`
- Xem [[Virsh Cheatsheet]] cho lệnh cơ bản, [[Monitoring & Troubleshooting]] khi có sự cố phát sinh sau thay đổi

---
*Xem thêm: [[Virsh Cheatsheet]] | [[Live Migration]] | [[Kvm-virtualization|KVM Virtualization]]*
