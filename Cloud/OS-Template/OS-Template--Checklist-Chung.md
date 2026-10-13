---
tags:
  - os-template
  - checklist
  - kvm
  - cloudstack
  - openstack
  - opennebula
---

# OS Template - Checklist Chung (CloudStack / OpenStack / OpenNebula)

Parent: [[OS-Template--Index]]
Related: [[OS-Template--Checklist-Linux]], [[OS-Template--Checklist-Windows-Server]], [[OS-Template--Checklist-VNF]]

- **Bối cảnh và vấn đề**: build template trên nhiều platform KVM khác nhau (CloudStack, OpenStack, OpenNebula) dễ thiếu nhất quán — mỗi lần build lại quên một bước bảo mật, quên generalize đúng cách, hoặc đăng ký sai field khiến VM clone ra bị lỗi ngầm (driver sai, network lệch).
- **Cách giải quyết**: checklist chuẩn hoá áp dụng cho mọi OS/platform, tick thủ công mỗi lần build hoặc rebuild template.
- **Kết quả sau khi hoàn thành**: template publish ra đảm bảo đã qua đủ bước chọn nguồn, hardening, seal, test boot, đăng ký đúng field, versioning rõ ràng.

> [!NOTE]
> Copy checklist này (hoặc tick trực tiếp trên bản gốc rồi lưu snapshot) mỗi lần build template mới. Lưu lại bản đã tick kèm version template để audit sau này.

## 1. Chọn & verify source image

- [ ] Image lấy từ nguồn chính thức (vendor/distro official mirror), không dùng image re-pack từ nguồn không rõ gốc
- [ ] Verify checksum/GPG signature trước khi dùng:
  ```bash
  sha256sum <image>.qcow2
  gpg --verify <image>.qcow2.sig <image>.qcow2
  ```
- [ ] Format đúng yêu cầu platform đích: qcow2 (CloudStack/OpenStack phổ biến) hay raw (OpenNebula/Ceph RBD thường convert raw)
- [ ] Kiểm tra virtual size và actual size hợp lý, image chưa chứa data dư thừa:
  ```bash
  qemu-img info <image>.qcow2
  ```

## 2. Guest tooling bắt buộc

- [ ] `qemu-guest-agent` cài sẵn và enable (để platform lấy IP thật, freeze filesystem khi snapshot)
- [ ] VirtIO drivers cho disk (virtio-blk/virtio-scsi) và network (virtio-net) — Linux kernel hiện đại có sẵn, Windows cần inject riêng ([[OS-Template--Checklist-Windows-Server]])
- [ ] cloud-init (Linux) hoặc cloudbase-init (Windows) cài đúng version, datasource list đúng thứ tự ưu tiên theo platform đích (xem mục 8)

## 3. Cấu hình baseline

- [ ] Serial console bật (`ttyS0` cho Linux, EMS/COM1 cho Windows) — xem được console qua platform mà không cần SSH/RDP
- [ ] Timezone set UTC (khuyến nghị chuẩn), NTP/chrony trỏ đúng NTP server nội bộ
- [ ] Locale/keyboard layout đúng chuẩn tổ chức
- [ ] DHCP là datasource mặc định lúc build, không hardcode static IP trong image

## 4. Hardening bảo mật baseline

- [ ] Không có user/password mặc định baked sẵn — chỉ cho phép SSH key / WinRM cert, tắt password login
- [ ] Xoá toàn bộ `authorized_keys` của người build trước khi seal
- [ ] Firewall mặc định deny inbound trừ port cần thiết — không rely hoàn toàn vào firewall trong image, vẫn phải có security group/firewall rule ở tầng platform
- [ ] Xoá tài khoản demo/test/sample data nếu image có sẵn
- [ ] Áp **CIS Benchmark Level 1** làm chuẩn hardening baseline — chọn Level 1 vì đây là template dùng chung nhiều mục đích, Level 2 quá chặt dễ gãy ứng dụng chạy phía trên

## 5. Patch trước khi seal

- [ ] Update toàn bộ package lên bản mới nhất tại thời điểm build
- [ ] Không update lại sau khi đã generalize — version patch cố định tại thời điểm chụp template, ghi rõ ngày patch vào tên/metadata template

## 6. Generalize / Seal image

- [ ] Xoá `/etc/machine-id` (để trống hoặc xoá, systemd tự sinh lại lần boot đầu)
- [ ] Xoá SSH host keys (`/etc/ssh/ssh_host_*`), sshd tự sinh lại lần boot đầu
- [ ] Xoá log cũ (`/var/log/*`), bash history, cache package manager
- [ ] Xoá udev persistent-net rules (`/etc/udev/rules.d/70-persistent-net.rules` nếu còn) — tránh lệch tên interface khi clone
- [ ] `cloud-init clean --logs --seed` — xoá cache instance cũ, đảm bảo lần boot đầu chạy full cloud-init stages
- [ ] Windows: `sysprep /generalize /oobe /shutdown` — chi tiết ở [[OS-Template--Checklist-Windows-Server]]

## 7. Test boot trước khi đăng ký chính thức

- [ ] Boot thử image (local hoặc 1 compute node test) trước khi đăng ký làm template chính thức
- [ ] Kiểm tra cloud-init không lỗi: `cloud-init status --long`, review `/var/log/cloud-init.log`
- [ ] Kiểm tra network lên đúng qua DHCP/metadata, không treo ở boot
- [ ] Kiểm tra console access (serial) xem được output mà không cần SSH/RDP
- [ ] Kiểm tra disk resize đúng khi offering/flavor có disk lớn hơn disk gốc image (growpart/parted tự động chạy)
- [ ] Xoá VM test sau khi xác nhận, không để lẫn với template chính thức

## 8. Đăng ký template lên platform

Mapping field khác nhau giữa 3 platform:

| Hạng mục | CloudStack | OpenStack (Glance/Nova) | OpenNebula |
| --- | --- | --- | --- |
| Lệnh/API đăng ký | `registerTemplate` | `glance image-create` | `oneimage create` |
| OS type mapping | `ostypeid` (lấy đúng ID qua `list ostypes`) | `os_distro` + `os_version` (image property) | `OS_ID` trong template attribute |
| Disk bus/driver | implicit theo `hypervisor=KVM` | `hw_disk_bus=virtio` hoặc `scsi` | `DEV_PREFIX=vd` (virtio) hoặc `sd` |
| NIC model | implicit | `hw_vif_model=virtio` | `MODEL=virtio` trong NIC context |
| Guest agent flag | không có flag riêng, dựa vào agent cài sẵn trong image | `hw_qemu_guest_agent=yes` | không có flag riêng |
| Password reset qua console | `passwordenabled=true/false` | N/A — dùng metadata injection | dùng `one-context` driver |
| Datasource/metadata | password server riêng của Virtual Router | Nova metadata `169.254.169.254` + config-drive (tuỳ chọn) | context ISO (`one-context`) hoặc cloud-init compat mode |
| Format | QCOW2 phổ biến | qcow2/raw, `disk_format` + `container_format=bare` | qcow2/raw, theo `TM_MAD` của datastore |
| Public/private scope | `ispublic=true/false` | `visibility=public/private/shared` | `PERMISSIONS` + `GROUP_ID` |

- [ ] Xác nhận đúng `ostypeid`/`os_distro`/`OS_ID` tương ứng — sai mapping gây lỗi driver/optimization ngầm (ví dụ CloudStack chọn sai disk controller mặc định)
- [ ] Set đúng flag guest agent nếu platform yêu cầu khai báo tường minh (OpenStack)
- [ ] Set đúng scope public/private theo chính sách chia sẻ template nội bộ

## 9. Versioning & lifecycle

- [ ] Naming convention nhất quán: `<os>-<version>-<build-date>`, ví dụ `ubuntu-24.04-20261009`
- [ ] Changelog ghi lại: patch level, package đặc biệt cài thêm, lý do rebuild
- [ ] Template cũ đánh dấu deprecated thay vì xoá ngay — cho VM đang chạy từ template cũ thời gian migrate
- [ ] Rebuild **định kỳ theo lịch cố định** (ví dụ hàng tháng/hàng quý tuỳ chính sách patch nội bộ) — không rebuild ad-hoc mỗi khi có CVE critical phát sinh, CVE khẩn cấp xử lý qua patch tại chỗ ở VM đang chạy, không qua rebuild template

## 10. Security scan trước khi publish

- [ ] Quét vulnerability trên image (ví dụ `trivy image`, `clamav`, hoặc tool nội bộ) — đặc biệt với image không tự build từ ISO/ISO cloud gốc
- [ ] Review lại mục 4 (hardening) đã tick đủ trước khi chuyển scope sang public

## 11. Image distribution (nếu nhiều zone/region)

- [ ] Sync template sang secondary storage/datastore ở zone/region khác sau khi build xong (CloudStack: copy template giữa zone; OpenStack: Glance multi-region hoặc image sync; OpenNebula: datastore sync giữa cluster)
- [ ] Xác nhận checksum giống nhau sau khi sync, tránh image bị corrupt khi truyền

## 12. Performance tuning (tham khảo nhanh)

Phần sâu dành cho VNF, xem [[OS-Template--Checklist-VNF]]. Với OS template thông thường chỉ cần lưu ý:

- [ ] Virtio-net multiqueue bật cho workload network-intensive (ghi chú trong metadata template, cấu hình thật nằm ở compute offering/flavor)
- [ ] CPU model `host-passthrough` chỉ cần cho workload cần performance gần bare-metal — không bắt buộc cho template OS thông thường
- [ ] Hugepages/CPU pinning khai báo ở compute offering/flavor, không nằm trong image — ghi chú template nào nên đi kèm offering nào

## Reference

- [Apache CloudStack - Working with Templates](https://docs.cloudstack.apache.org/en/latest/adminguide/templates.html)
- [OpenStack Glance - Image Properties](https://docs.openstack.org/glance/latest/admin/useful-image-properties.html)
- [OpenNebula - Image and Datastore Management](https://docs.opennebula.io/)
- [cloud-init documentation](https://cloudinit.readthedocs.io/)
