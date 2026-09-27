---
tags:
  - storage
  - backup
---

# Snapshots & Backup

Có 2 loại snapshot hoàn toàn khác nhau trong thế giới libvirt/QEMU — nhầm lẫn 2 loại này là nguồn lỗi phổ biến nhất khi mới làm backup KVM: **internal snapshot** (chỉ hoạt động với qcow2, VM phải dừng hoặc chấp nhận hiệu năng giảm) và **external snapshot** (tạo file overlay mới, VM tiếp tục chạy bình thường — đây là cơ chế nền cho backup online).

> [!warning] Internal snapshot đã bị coi là legacy — external snapshot là chuẩn hiện nay
> `virsh snapshot-create-as vm01 snap1` (không thêm cờ `--disk-only`) tạo **internal snapshot** — lưu cả disk state + memory state gộp chung vào 1 file qcow2 duy nhất. Vấn đề: file qcow2 phình to dần vô hạn theo số snapshot, và **không hỗ trợ tốt với QEMU hiện đại** (nhiều tính năng deprecated). Gần như mọi workflow backup production hiện nay dùng **external snapshot** (`--disk-only`) thay thế hoàn toàn.

## How — external snapshot (chuẩn cho backup online)

```bash
# 1. Tạo external snapshot — VM tiếp tục chạy, QEMU tự chuyển ghi sang file overlay mới
virsh snapshot-create-as vm01 backup-$(date +%Y%m%d) \
  --disk-only --atomic --no-metadata

# Sau lệnh này: disk chính (vm01.qcow2) trở thành READ-ONLY base,
# mọi write mới của VM đi vào file overlay mới (vm01.backup-20260926)
```

```
Trước snapshot:
  vm01.qcow2 (đang được QEMU đọc/viết trực tiếp)

Sau snapshot --disk-only:
  vm01.qcow2 (base, frozen — an toàn để backup/copy)
       ↑ backing file
  vm01.backup-20260926.qcow2 (overlay mới, QEMU đang viết vào đây)
```

```bash
# 2. Backup file base (đã frozen, an toàn copy) sang nơi lưu trữ khác
cp vm01.qcow2 /backup/vm01-$(date +%Y%m%d).qcow2
# hoặc rsync/restic tới remote storage

# 3. Merge overlay trở lại vào base (blockcommit) — VM vẫn chạy suốt quá trình
virsh blockcommit vm01 vda --active --pivot
# --pivot: sau khi commit xong, QEMU tự động quay lại viết vào file base gốc
```

> [!info] Vì sao `--atomic` quan trọng
> `--atomic` đảm bảo bước tạo snapshot **không để lại trạng thái nửa vời** nếu có lỗi giữa chừng (VD hết dung lượng disk lúc tạo overlay file) — không có cờ này, một lỗi giữa quá trình có thể để lại domain ở trạng thái khó dự đoán (đã chuyển ghi sang overlay nhưng chưa có metadata hoàn chỉnh).

## Key Config — quiesce (filesystem-consistent snapshot)

```bash
# Yêu cầu qemu-guest-agent CHẠY BÊN TRONG GUEST để đảm bảo
# filesystem flush trước khi cắt snapshot (tránh crash-consistent thay vì clean)
virsh snapshot-create-as vm01 backup1 --disk-only --quiesce
```

| Cờ | Ý nghĩa | Yêu cầu |
|---|---|---|
| `--disk-only` | External snapshot, không chụp memory state | Bắt buộc cho hầu hết workflow backup hiện đại |
| `--quiesce` | Guest tự flush filesystem trước khi cắt snapshot (application-consistent hơn) | Cần `qemu-guest-agent` cài & chạy trong guest |
| `--atomic` | Đảm bảo all-or-nothing | Luôn nên thêm |
| `--no-metadata` | Không lưu snapshot vào metadata domain XML (chỉ tạo file, tự quản lý ngoài virsh) | Dùng khi có tool backup riêng (Bareos, Veeam for KVM...) tự track snapshot |

## Ops Runbook — vòng lặp backup điển hình

```bash
#!/bin/bash
DOMAIN=vm01
SNAP=backup-$(date +%Y%m%d-%H%M)

virsh snapshot-create-as "$DOMAIN" "$SNAP" --disk-only --atomic --quiesce --no-metadata
BASE_DISK=$(virsh domblklist "$DOMAIN" --details | awk '/disk/{print $4}')
cp "$BASE_DISK" "/backup/${DOMAIN}-${SNAP}.qcow2"
virsh blockcommit "$DOMAIN" vda --active --pivot --wait
```

## Gotchas & Lessons Learned

> [!warning] Lesson learned: quên `blockcommit` sau khi backup xong → snapshot chain dài dần, hiệu năng giảm dần
> Nếu chạy backup hàng ngày mà quên (hoặc script lỗi giữa đường) bước `blockcommit --pivot`, mỗi lần backup tạo thêm 1 lớp overlay mới — sau vài tuần, VM đang đọc/viết qua **chuỗi 10+ file qcow2 chồng lên nhau**, mỗi lần I/O phải tra ngược qua nhiều lớp backing file, hiệu năng giảm rõ rệt và khó debug (nhìn `virsh domblklist` thấy tên file lạ, không hiểu vì sao). Luôn kiểm tra `virsh domblklist vm01 --details` định kỳ để xác nhận chain chỉ có 1 file active.

> [!warning] Snapshot không phải backup — snapshot chain vẫn nằm trên CÙNG storage với VM gốc
> Nếu physical disk/storage backend chết, mất luôn cả base image và toàn bộ snapshot chain. Snapshot chỉ là bước trung gian để "đóng băng" 1 điểm nhất quán nhằm copy dữ liệu ra nơi khác — bước `cp`/`rsync` file base sang storage/location khác **mới là backup thật**.

## Resources

- `man virsh` — phần `snapshot-*`, `blockcommit`, `blockpull`
- `qemu-guest-agent` — Red Hat/Fedora package `qemu-guest-agent`, cần cài trong guest

---
*Xem thêm: [[Disk Image Formats - qcow2 vs raw]] | [[Virsh Cheatsheet]] | [[Kvm-virtualization|KVM Virtualization]]*
