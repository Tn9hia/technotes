---
tags:
  - libvirt
  - migration
  - ha
---

# Live Migration

**Live Migration** di chuyển 1 VM đang chạy từ host A sang host B **không (hoặc gần như không) downtime** — khác với "cold migration" (tắt VM, copy disk, khởi động lại nơi khác). KVM/libvirt tự thực hiện được cơ chế này **nếu** đáp ứng đủ điều kiện tiên quyết — không có "tự động" hoàn toàn như vMotion tích hợp sẵn trong vCenter, phần hạ tầng (shared storage, network) phải tự chuẩn bị trước.

> [!warning] Điều kiện bắt buộc — thiếu 1 trong số này migration sẽ fail hoặc guest crash
> 1. **Shared storage** giữa 2 host (NFS, Ceph RBD, iSCSI...) — QEMU đích cần đọc được **cùng file/volume disk** đang chạy ở nguồn. Không có shared storage, phải dùng thêm `--copy-storage-all` (chậm hơn nhiều, copy cả disk qua network).
> 2. **CPU tương thích** giữa 2 host — nếu dùng `host-passthrough`, 2 host phải cùng model CPU chính xác; xem [[QEMU Process Model & Machine Types]] phần CPU model.
> 3. **Cùng version libvirt/QEMU** (hoặc tương thích ngược) — chênh version lớn có thể fail migrate stream.
> 4. **Network đủ băng thông** giữa 2 host cho quá trình transfer RAM — VM RAM lớn + đổi trang liên tục (workload viết nhiều) cần băng thông cao để theo kịp.

## How — cơ chế pre-copy (mặc định)

```
1. QEMU nguồn bắt đầu gửi TOÀN BỘ RAM của VM sang QEMU đích qua network
   (VM vẫn đang chạy bình thường ở nguồn trong lúc này)
        │
2. QEMU nguồn dùng dirty page tracking (dựa vào EPT/NPT dirty bit,
   xem Memory Virtualization - EPT & NPT) để biết trang nào bị guest
   sửa SAU KHI đã gửi lần đầu
        │
3. Lặp lại: gửi tiếp các dirty page mới phát sinh, cho tới khi
   tốc độ dirty page < tốc độ transfer (converge)
        │
4. Khi đủ "gần đồng bộ" → dừng vCPU ở nguồn (downtime thực sự bắt đầu,
   thường vài chục ms tới vài giây tùy workload)
        │
5. Gửi state cuối cùng (vCPU register, device state) sang đích
        │
6. Khởi động vCPU ở đích, VM tiếp tục chạy — nguồn dọn dẹp/hủy QEMU cũ
```

> [!info] Vì sao có workload "không bao giờ migrate xong" (never converge)
> Nếu tốc độ guest tự sửa đổi RAM (dirty rate) **cao hơn** tốc độ network transfer, vòng lặp bước 2-3 không bao giờ hội tụ — QEMU cứ gửi mãi mà dirty page mới sinh ra nhanh hơn. Đây là lý do workload viết RAM cực mạnh (database in-memory, cache server nóng) trên network chậm dễ bị "migration treo vô thời hạn". Giải pháp: tăng băng thông migration, dùng **post-copy** (VM chạy ở đích ngay, fault-in trang thiếu từ nguồn — đánh đổi lấy rủi ro nếu network đứt giữa lúc post-copy), hoặc đơn giản là throttle vCPU nguồn tạm thời để giảm dirty rate.

## Key Config

```bash
# Migration cơ bản qua SSH (yêu cầu key-based auth đã setup giữa 2 host)
virsh migrate --live vm01 qemu+ssh://host02.example.com/system

# Kèm giữ persistent config ở đích + xóa định nghĩa domain ở nguồn
virsh migrate --live --persistent --undefinesource vm01 qemu+ssh://host02/system

# Giới hạn băng thông migration (MiB/s) — tránh chiếm hết network production
virsh migrate --live --bandwidth 500 vm01 qemu+ssh://host02/system

# Post-copy (khi pre-copy không hội tụ được với workload dirty-heavy)
virsh migrate --live --postcopy vm01 qemu+ssh://host02/system
```

| Tham số | Ảnh hưởng |
|---|---|
| `--live` | Bắt buộc để migration không downtime, thiếu cờ này = cold migration (offline) |
| `--persistent` | Domain XML được lưu lại ở đích, không chỉ tồn tại tạm trong RAM |
| `--undefinesource` | Xóa domain definition ở nguồn sau khi migrate xong (không xóa file disk nếu dùng shared storage) |
| `--copy-storage-all` | Copy luôn disk qua network — dùng khi KHÔNG có shared storage (chậm, tốn network) |
| `tunnelled` (URI `qemu+ssh` implicit) | Migration traffic đi qua kênh SSH — an toàn hơn nhưng chậm hơn direct TCP |

## Security Considerations

- Migration traffic **không mã hóa mặc định** nếu dùng URI trực tiếp `qemu+tcp://` — nên dùng `qemu+ssh://` hoặc bật TLS (`qemu+tls://`) cho traffic chứa toàn bộ RAM VM (có thể chứa secret/credential đang xử lý trong memory guest) đi qua network không tin cậy.
- Cần mở đúng port migration (mặc định QEMU dùng range `49152-49215` cho data stream ngoài port libvirt 16509/22).

## Gotchas & Lessons Learned

> [!warning] Lesson learned: migration "thành công" theo virsh không có nghĩa là guest OS mượt mà bên trong
> `virsh migrate` trả về exit code 0 chỉ xác nhận QEMU state đã chuyển thành công — nó không kiểm tra ứng dụng bên trong guest có chịu được vài giây "đứng hình" lúc downtime cuối hay không (VD kết nối TCP dài của app có thể timeout). Với workload nhạy cảm, luôn test migration trong giờ thấp điểm trước, và giám sát ứng dụng bên trong song song với việc theo dõi `virsh domjobinfo`.

> [!tip] `virsh domjobinfo vm01` để theo dõi tiến trình migration đang chạy
> Cho biết % hoàn thành, tốc độ hiện tại, ước tính thời gian còn lại — chạy lệnh này song song khi nghi ngờ migration bị treo, trước khi vội `virsh domjobabort`.

## Resources

- Libvirt migration docs: https://libvirt.org/migration.html
- `man virsh` — phần `migrate`

---
*Xem thêm: [[Memory Virtualization - EPT & NPT]] | [[Libvirt Architecture & Domain XML]] | [[Storage Backends Overview]] | [[Kvm-virtualization|KVM Virtualization]]*
