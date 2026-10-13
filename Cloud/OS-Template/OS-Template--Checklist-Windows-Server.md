---
tags:
  - os-template
  - checklist
  - windows
  - windows-server
---

# OS Template - Checklist Windows Server

Parent: [[OS-Template--Checklist-Chung]]
Related: [[OS-Template--Index]]

> [!WARNING]
> Windows không dùng cloud-init — dùng **cloudbase-init** (tương đương port sang Windows). Quy trình generalize dùng **sysprep**, khác hoàn toàn với Linux ở [[OS-Template--Checklist-Linux]].

File này chỉ liệt kê phần **đặc thù Windows**, bổ sung thêm vào [[OS-Template--Checklist-Chung]] (vẫn phải tick đủ checklist chung, đặc biệt mục hardening và versioning).

## 1. VirtIO drivers

- [ ] Mount virtio-win ISO (từ dự án Fedora virtio-win, chọn bản **stable**, không luôn lấy `latest`) trong lúc cài Windows hoặc trước khi sysprep
- [ ] Cài driver: NetKVM (network), viostor/vioscsi (disk), Balloon (memory ballooning), qxl/virtio-gpu (display — cải thiện console nếu platform hỗ trợ)
- [ ] Cài QEMU Guest Agent (có sẵn trong virtio-win ISO) — enable service

## 2. cloudbase-init

- [ ] Cài cloudbase-init, cấu hình `metadata_services` đúng thứ tự ưu tiên theo platform đích trong `cloudbase-init.conf`:
  - CloudStack: ConfigDrive hoặc password server riêng
  - OpenStack: Nova metadata service
  - OpenNebula: context ISO (`one-context` tương đương)
- [ ] Set username mặc định để cloudbase-init tạo/đặt random password lúc boot đầu — không để password cố định baked trong image

## 3. Cấu hình trước sysprep

- [ ] Tắt Windows Update auto-restart trong lúc build (tránh restart giữa chừng làm hỏng quá trình chuẩn bị)
- [ ] Enable RDP + firewall rule RDP nếu đây là access method chính
- [ ] Enable WinRM nếu cần automation (Ansible/Terraform) — dùng HTTPS listener, không để HTTP không mã hoá
- [ ] Cài đầy đủ bản vá Windows Update tại thời điểm build, sau đó **tắt auto-update** — patch management chuyển qua day-2 ops để tránh version lệch giữa các VM clone từ cùng template
- [ ] Disable/rename tài khoản Administrator mặc định theo chuẩn bảo mật tổ chức
- [ ] Set time source: domain join thì theo NTP của domain, standalone thì set NTP server nội bộ qua `w32tm`

## 4. Licensing

> [!NOTE]
> Model license đã chốt: **KMS**.

- [ ] Không activate Windows trong image — để trống, không nhập product key KMS client trong lúc build
- [ ] Để client tự activate qua KMS server nội bộ ở lần boot đầu (`slmgr /ato` chạy tự động nếu KMS client key đã preset sẵn trong image theo edition, hoặc set qua `unattend.xml`/cloudbase-init script)
- [ ] Xác nhận VM sau khi boot resolve được KMS server nội bộ qua DNS/SRV record (`_vlmcs._tcp`) hoặc IP cấu hình cứng, tuỳ hạ tầng KMS hiện có
- [ ] Kiểm tra trạng thái activation sau boot đầu: `slmgr /dlv` — xác nhận `Licensed`
- [ ] Ghi rõ "KMS" vào metadata/changelog của template, không cần theo dõi activation count như MAK

## 5. Sysprep (generalize)

- [ ] Chuẩn bị `unattend.xml` đúng cho các OOBE pass cần thiết (đặt lại computer name, join domain nếu cần, trigger cloudbase-init chạy lần đầu)
- [ ] Chạy sysprep:
  ```powershell
  C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown /unattend:unattend.xml
  ```
- [ ] Xác nhận VM **shutdown** (không restart) ngay sau sysprep — nếu máy boot lại trước khi chụp image, phải chạy lại sysprep từ đầu
- [ ] Kiểm tra sysprep log (`C:\Windows\System32\Sysprep\Panther\setupact.log`) không có lỗi trước khi tắt máy

## 6. Test boot

- [ ] Boot thử VM từ image đã sysprep, xác nhận OOBE chạy xong, cloudbase-init set đúng hostname/password/network
- [ ] Kiểm tra Device Manager không có "Unknown device" nào (VirtIO driver load đủ)
- [ ] Kiểm tra RDP/WinRM access từ ngoài vào thành công

## Reference

- [virtio-win ISO (Fedora Project)](https://fedorapeople.org/groups/virt/virtio-win/)
- [Cloudbase-Init documentation](https://cloudbase-init.readthedocs.io/)
- [Microsoft Sysprep documentation](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview)
