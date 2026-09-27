---
tags:
  - security
  - selinux
  - apparmor
---

# sVirt — SELinux & AppArmor Isolation

**sVirt** là lớp tích hợp giữa libvirt và MAC (Mandatory Access Control — SELinux trên RHEL/Fedora/CentOS, AppArmor trên Ubuntu/Debian) — mục tiêu: nếu 1 QEMU process bị compromise (VD qua lỗ hổng emulate device), kẻ tấn công **không thể** dùng quyền QEMU đó để đọc/viết file của VM khác hoặc tài nguyên host ngoài phạm vi được cấp, dù đang chạy với quyền user Linux thông thường (thường là user `qemu`/`libvirt-qemu`, không phải root).

> [!tip] Vì sao cần thêm lớp MAC dù QEMU đã chạy unprivileged
> Chạy QEMU bằng user không phải root (`qemu`) đã giảm rủi ro đáng kể, nhưng **không đủ** — theo DAC (Discretionary Access Control) thông thường, user `qemu` vẫn có quyền đọc file của **bất kỳ VM nào khác** cũng chạy bằng chính user `qemu` đó (vì file permission chỉ phân biệt theo user/group, không phân biệt "VM nào"). sVirt giải quyết đúng lỗ hổng này: gán 1 **category SELinux/AppArmor riêng biệt** cho từng QEMU process, đảm bảo process VM A dù có bị chiếm quyền cũng không đọc được file disk của VM B, kể cả khi cả 2 cùng chạy chung user Linux.

## How — cơ chế hoạt động (SELinux)

```
Mỗi lần VM start:
   │
   ├─ libvirt tự sinh 1 category SELinux ngẫu nhiên, riêng cho lần chạy này
   │  (VD: system_u:system_r:svirt_t:s0:c123,c456)
   │
   ├─ libvirt tự relabel (chcon) file disk của VM đó với CÙNG category
   │  (VD: system_u:object_r:svirt_image_t:s0:c123,c456)
   │
   └─ QEMU process chạy với category đó — SELinux policy chỉ cho phép
      process truy cập file/resource có CÙNG category, chặn truy cập
      chéo sang file của VM khác dù cùng type svirt_image_t
```

```bash
# Kiểm tra label SELinux hiện tại của 1 file disk VM
ls -Z /var/lib/libvirt/images/vm01.qcow2
# system_u:object_r:svirt_image_t:s0:c123,c456

# Kiểm tra process QEMU đang chạy với context nào
ps -eZ | grep qemu-system

# Trạng thái SELinux tổng quan
sestatus
getenforce   # Enforcing / Permissive / Disabled
```

## Key Config

| Chế độ | Ý nghĩa | Khuyến nghị |
|---|---|---|
| `Enforcing` | Vi phạm policy bị **chặn** thật, ghi log | **Bắt buộc cho production** |
| `Permissive` | Vi phạm chỉ ghi log, không chặn | Chỉ dùng khi debug lỗi SELinux, không dùng lâu dài |
| `Disabled` | Tắt hoàn toàn SELinux | **Không nên** — mất toàn bộ lớp bảo vệ sVirt |

```xml
<domain>
  <!-- libvirt tự quản lý, nhưng có thể ghi đè nếu cần custom label -->
  <seclabel type='dynamic' model='selinux' relabel='yes'/>
</domain>
```

> [!warning] Lesson learned: `getenforce` trả về `Disabled` sau khi troubleshoot rồi quên bật lại
> Rất phổ biến: gặp lỗi VM không start được, nghi ngờ SELinux, set `Permissive` (hoặc tắt hẳn) để "test xem có phải do SELinux không" — rồi quên bật lại `Enforcing` sau khi xác định nguyên nhân thật. Luôn dùng `setenforce 0` (Permissive **tạm thời**, mất khi reboot) thay vì sửa `/etc/selinux/config` thành `Disabled` (permanent) khi debug — và luôn có việc theo dõi (checklist/monitoring) để phát hiện host nào đang chạy Permissive lâu ngày không rõ lý do.

## Ops Runbook — debug lỗi SELinux chặn nhầm

```bash
# Tìm log SELinux denial gần nhất liên quan tới qemu/libvirt
ausearch -m avc -ts recent | grep -i qemu
journalctl -t audit | grep denied

# Dùng audit2allow để hiểu policy nào đang chặn (KHÔNG tự động apply mù quáng)
ausearch -m avc -ts recent | audit2allow -a
```

> [!warning] Đừng tự `audit2allow -a -M mymodule && semodule -i mymodule.pp` mà không hiểu rule đang thêm
> `audit2allow` chỉ generate policy cho phép đúng hành vi **đã bị chặn gần đây** — nếu áp dụng mù quáng, dễ vô tình mở quyền rộng hơn cần thiết (VD cho phép QEMU truy cập 1 loại file mà lẽ ra không nên, chỉ vì 1 lần denial do config sai chỗ khác). Luôn đọc rule được sinh ra trước khi `semodule -i`.

## Resources

- Red Hat SELinux sVirt documentation: https://access.redhat.com/documentation (chương Virtualization Security)
- `man sVirt`, `man selinux`

---
*Xem thêm: [[Guest Isolation & Attack Surface]] | [[KVM Kernel Module & Hardware Virtualization Extensions]] | [[Kvm-virtualization|KVM Virtualization]]*
