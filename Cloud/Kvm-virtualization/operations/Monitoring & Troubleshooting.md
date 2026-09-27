---
tags:
  - operations
  - troubleshooting
---

# Monitoring & Troubleshooting

Debug KVM luôn bắt đầu từ việc xác định lỗi nằm ở **tầng nào** trong 3 lớp (libvirt / QEMU / kernel KVM) — mỗi tầng có log riêng, và triệu chứng giống nhau (VD "VM không start") có thể do nguyên nhân hoàn toàn khác nhau tùy tầng.

## Where — log ở đâu

| Tầng | Vị trí log | Khi nào xem |
|---|---|---|
| **libvirtd** | `journalctl -u libvirtd` (hoặc `virtqemud` ở bản modular mới) | Lỗi define/start domain, lỗi API, lỗi storage pool/network |
| **QEMU process** (per-VM) | `/var/log/libvirt/qemu/<domain>.log` | Lỗi runtime cụ thể của 1 VM — device init fail, migration lỗi, crash |
| **Kernel (kvm.ko)** | `dmesg`, `journalctl -k` | CPU không hỗ trợ, module load lỗi, IOMMU/VFIO lỗi, OOM killer |
| **Guest OS bên trong** | Console (`virsh console`), hoặc log app trong guest | Guest boot fail, application-level issue — không liên quan gì tới host |

```bash
# Bộ 4 lệnh đầu tiên khi nghi ngờ có vấn đề
virsh list --all                              # domain nào đang ở trạng thái bất thường
tail -100 /var/log/libvirt/qemu/vm01.log       # log chi tiết nhất của riêng VM đó
journalctl -u libvirtd --since "10 min ago"    # log libvirtd gần đây
dmesg -T | tail -50                            # lỗi kernel gần đây (KVM, IOMMU, OOM...)
```

## How — symptom-based debug flow

```
VM không start được
   │
   ├─ virsh start vm01 báo lỗi ngay → đọc thẳng error message
   │  (thường đủ rõ: "Failed to find secret", "no space left", "operation not permitted"...)
   │
   ├─ virsh start "thành công" nhưng domstate ngay lập tức shutoff → đọc
   │  /var/log/libvirt/qemu/vm01.log (lỗi QEMU init device, thiếu file disk...)
   │
   └─ VM start OK nhưng guest OS không boot lên (đứng ở màn hình đen/GRUB)
      → virsh console vm01 hoặc virt-viewer để xem trực tiếp màn hình guest
      → thường là vấn đề bootloader/disk driver bên trong guest, không phải KVM
```

```
VM chạy chậm bất thường
   │
   ├─ Kiểm tra vCPU steal time trong guest (top, cột %st) → over-commit CPU,
   │  xem [[vCPU Threads & Scheduling]]
   │
   ├─ Kiểm tra NUMA locality: numastat -p $(pgrep -f vm01) → remote memory access,
   │  xem [[CPU Pinning & NUMA]]
   │
   └─ Kiểm tra I/O: virsh domstats vm01 --block, hoặc info block qua
      QEMU monitor (xem [[QEMU Monitor - QMP & HMP]]) → storage backend chậm
      (đặc biệt kiểm tra Ceph nếu dùng RBD, xem [[Ceph RBD with KVM]])
```

## Key Config — công cụ monitoring hữu ích

```bash
virt-top                          # kiểu "top" nhưng cho danh sách VM (CPU%, MEM%, disk I/O per VM)
virsh domstats vm01               # số liệu chi tiết: cpu, balloon, block, interface, per-vcpu
virsh domstats vm01 --interface   # riêng network — bytes/packets rx/tx, drop, error
libvirt-exporter / node_exporter  # Prometheus exporter phổ biến để đưa metrics vào Grafana lâu dài
```

> [!tip] Với hạ tầng lớn, đừng chỉ dựa vào `virsh` interactive — dựng Prometheus + Grafana ngay từ đầu
> `virt-top`/`virsh domstats` tốt cho debug tức thời 1 VM, nhưng không giúp phát hiện xu hướng (VM nào đang tăng dần I/O wait qua nhiều tuần) hoặc alert tự động. Với cluster nhiều host, nên có `libvirt_exporter` hoặc `node_exporter` + Grafana dashboard từ sớm, tương tự cách vault [[Ceph|Ceph]] khuyến nghị Prometheus cho Ceph.

## Gotchas & Lessons Learned

> [!warning] Lesson learned: log domain xóa mất sau khi `virsh undefine` — backup log trước khi xóa VM nếu đang debug incident
> `/var/log/libvirt/qemu/<domain>.log` không tự động bị xóa khi undefine domain, nhưng dễ bị quên và log tiếp theo (nếu domain được tạo lại cùng tên) sẽ **ghi đè/append tiếp** vào cùng file — làm lẫn log của 2 "đời" VM khác nhau. Nếu đang điều tra incident, copy log ra nơi khác **trước khi** làm bất kỳ thao tác dọn dẹp/tái tạo VM.

> [!warning] `dmesg` bị giới hạn buffer — lỗi cũ bị "trôi" mất nếu không dùng `journalctl -k` (persistent)
> Nếu host không cấu hình `journald` persistent storage (`Storage=persistent` trong `/etc/systemd/journald.conf`), log kernel chỉ tồn tại trong RAM buffer, mất hoàn toàn sau reboot — không giúp gì khi điều tra sự cố đã gây crash/reboot. Đảm bảo persistent journal được bật trên mọi host KVM production.

## Resources

- `man virt-top`, `man virsh` (phần domstats)
- Libvirt exporter (Prometheus): https://github.com/kumina/libvirt_exporter (hoặc tương đương đang duy trì)

---
*Xem thêm: [[QEMU Monitor - QMP & HMP]] | [[Day 2 Operations Cheatsheet]] | [[Kvm-virtualization|KVM Virtualization]]*
