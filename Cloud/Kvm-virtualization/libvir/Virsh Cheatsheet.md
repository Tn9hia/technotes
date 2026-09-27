---
tags:
  - libvirt
  - operations
  - cheatsheet
---

# Virsh Cheatsheet

Tier: 2
Parent: [[Libvirt Architecture & Domain XML]]

Tập lệnh `virsh` hay dùng nhất khi vận hành hàng ngày — không phải danh sách đầy đủ (`man virsh` có ~200 subcommand), chỉ những lệnh thực tế lặp lại nhiều nhất.

## Domain lifecycle

```bash
virsh list --all                    # tất cả domain (kể cả đã shutoff)
virsh start vm01
virsh shutdown vm01                 # graceful, cần guest hỗ trợ ACPI
virsh destroy vm01                  # tắt cứng (không xóa gì, xem lưu ý ở Libvirt Architecture)
virsh reboot vm01
virsh suspend vm01 / virsh resume vm01
virsh autostart vm01                # tự chạy khi host boot
virsh autostart vm01 --disable
```

## Thông tin & debug

```bash
virsh dominfo vm01
virsh domstate vm01
virsh vcpuinfo vm01                 # vCPU đang map vào CPU vật lý nào
virsh domstats vm01 --cpu-total --balloon --block --interface
virsh dumpxml vm01
virsh console vm01                  # attach serial console (cần guest cấu hình serial)
```

## Snapshot & backup

```bash
virsh snapshot-create-as vm01 snap1 --disk-only --atomic   # external snapshot, xem Snapshots & Backup
virsh snapshot-list vm01
virsh snapshot-revert vm01 snap1
virsh blockcommit vm01 vda --active --pivot                # merge snapshot chain sau khi backup xong
```

## Thiết bị — hotplug

```bash
virsh attach-disk vm01 /path/new-disk.qcow2 vdb --live --config
virsh detach-disk vm01 vdb --live
virsh attach-interface vm01 bridge br0 --model virtio --live
virsh change-media vm01 sda /path/new.iso --update           # đổi ISO CD-ROM đang chạy
```

## Storage pool & volume

```bash
virsh pool-list --all
virsh pool-info default
virsh vol-list default
virsh vol-create-as default vm02-disk.qcow2 20G --format qcow2
```

## Network (libvirt virtual network, khác Linux bridge tay)

```bash
virsh net-list --all
virsh net-start default
virsh net-autostart default
virsh net-dhcp-leases default        # xem IP đã cấp qua NAT network mặc định
```

## Migration

```bash
virsh migrate --live vm01 qemu+ssh://host02/system
virsh migrate --live --persistent --undefinesource vm01 qemu+ssh://host02/system
```

## Config & undefine

```bash
virsh edit vm01                                    # sửa XML persistent, có validate
virsh undefine vm01                                # xóa domain definition (giữ disk)
virsh undefine vm01 --remove-all-storage           # xóa cả disk — CẨN TRỌNG
```

> [!warning] Lesson learned: `undefine --remove-all-storage` xóa file thật, không có "recycle bin"
> Không có bước confirm thứ 2, không có thùng rác — chạy nhầm lệnh này trên domain sai là mất dữ liệu thật ngay lập tức. Luôn `virsh dumpxml vm01 | grep "source file"` để xác nhận đúng file/disk trước khi thêm cờ này.

> [!tip] `virsh` không có "dry-run" built-in — dùng `dumpxml` để kiểm tra trước khi hành động không thể hoàn tác
> Với mọi lệnh có khả năng phá hủy dữ liệu (`undefine --remove-all-storage`, `vol-delete`, `pool-delete`), luôn chạy `dumpxml`/`vol-list`/`pool-info` trước để xác nhận đối tượng đúng — virsh tin tưởng tuyệt đối vào tên bạn gõ, không có safety net.

---
*Xem thêm: [[Libvirt Architecture & Domain XML]] | [[Live Migration]] | [[Snapshots & Backup]] | [[Kvm-virtualization|KVM Virtualization]]*
