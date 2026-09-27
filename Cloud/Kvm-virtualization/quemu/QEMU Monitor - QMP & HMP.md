---
tags:
  - qemu
  - operations
---

# QEMU Monitor — QMP & HMP

**QEMU Monitor** là giao diện điều khiển **runtime** của một QEMU process đang chạy — dùng để query trạng thái hoặc gửi lệnh (thêm device, chụp snapshot, đổi ISO CD-ROM...) mà **không cần** tắt VM. Có 2 dạng: **HMP** (Human Monitor Protocol — cú pháp text người đọc được, kiểu console) và **QMP** (QEMU Machine Protocol — JSON, dùng cho automation). libvirt giao tiếp với mỗi QEMU process nó quản lý **hoàn toàn qua QMP** — đây là cách `virsh` "nói chuyện" được với VM đang chạy.

> [!tip] libvirt không tự chạy VM — nó điều khiển QEMU qua QMP
> Hiểu lại kiến trúc: `virsh start <domain>` khiến libvirtd fork/exec ra process QEMU, mở 1 UNIX socket QMP riêng cho process đó, rồi từ giờ **mọi** lệnh `virsh` runtime (suspend, resume, thêm disk, migrate...) đều là libvirt dịch lệnh cấp cao thành QMP command gửi qua socket này. Đây là lý do bạn hiếm khi cần đụng QMP tay — nhưng khi `virsh` báo lỗi mơ hồ, đọc trực tiếp QMP response (qua `virsh qemu-monitor-command`) thường ra thông tin chi tiết hơn.

## How — truy cập Monitor qua libvirt

```bash
# Gửi lệnh HMP (cú pháp cũ, dễ đọc) qua libvirt — không cần biết socket path
virsh qemu-monitor-command <domain> --hmp "info cpus"
virsh qemu-monitor-command <domain> --hmp "info block"
virsh qemu-monitor-command <domain> --hmp "info status"

# Gửi lệnh QMP (JSON) trực tiếp — dùng khi cần automation/script chính xác
virsh qemu-monitor-command <domain> --pretty \
  '{"execute":"query-status"}'
```

| Lệnh HMP hay dùng | Ý nghĩa |
|---|---|
| `info cpus` | Map vCPU logic → thread id host — dùng khi debug CPU pinning |
| `info block` | Trạng thái từng disk device, backing file, đang dirty bitmap nào |
| `info status` | VM đang running/paused/postmigrate... |
| `info network` | Trạng thái NIC ảo, backend nào (tap/vhost) |
| `info balloon` | Trạng thái virtio-balloon — RAM thực guest đang giữ |

## Key Config — QMP socket trong Domain XML

```xml
<domain>
  <!-- libvirt tự tạo QMP socket, thường không cần khai báo tay -->
  <!-- nhưng có thể thấy trong /var/run/libvirt/qemu/<domain>.monitor -->
</domain>
```

```bash
# Xem trực tiếp qua socket (bypass libvirt, ít dùng, chỉ khi debug sâu)
sudo socat - UNIX-CONNECT:/var/run/libvirt/qemu/<domain>.monitor
```

## Ops Runbook

- **Đổi ISO gắn vào CD-ROM VM đang chạy** (không cần tắt máy):
```bash
virsh change-media <domain> sda /path/to/new.iso --update
```
- **Chụp screenshot màn hình VM đang chạy** (hữu ích debug khi VNC/SPICE không kết nối được):
```bash
virsh screenshot <domain> /tmp/screenshot.ppm
```
- **Kiểm tra VM có đang bị "stuck" ở paused vì thiếu disk space không** (thường gặp khi thin-provisioned storage hết dung lượng):
```bash
virsh qemu-monitor-command <domain> --hmp "info status"
# "VM status: paused (io-error)" → hết dung lượng backing storage
```

## Gotchas & Lessons Learned

> [!warning] Lesson learned: gửi lệnh QMP tay trực tiếp qua socket có thể làm libvirt "mất đồng bộ" trạng thái
> Vì libvirt giữ 1 kết nối QMP riêng và theo dõi state qua đó, nếu bạn tự `socat` vào monitor socket và gửi lệnh thay đổi trạng thái VM (VD `stop`/`cont` trực tiếp) **không qua** `virsh`, libvirt có thể không nhận biết ngay thay đổi này — dẫn tới `virsh list` báo trạng thái sai lệch tạm thời. Luôn ưu tiên `virsh qemu-monitor-command` (đi qua libvirt) hơn là kết nối socket trực tiếp, trừ khi đang debug và biết rõ hệ quả.

> [!tip] `info block` là lệnh cứu mạng khi disk I/O bị treo
> Khi VM "đứng hình" và nghi ngờ do storage backend (NFS timeout, Ceph RBD chậm), `info block` cho thấy ngay device nào đang ở trạng thái bất thường, kèm thông tin backing file/dirty bitmap — nhanh hơn nhiều so với đoán từ log guest OS bên trong.

## Resources

- QEMU QMP reference: `qemu-qmp-ref` (man page) hoặc `docs/interop/qmp-spec.txt` trong source QEMU
- `virsh help qemu-monitor-command`

---
*Xem thêm: [[QEMU Process Model & Machine Types]] | [[Monitoring & Troubleshooting]] | [[Kvm-virtualization|KVM Virtualization]]*
