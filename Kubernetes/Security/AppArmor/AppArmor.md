# AppArmor
Tags: #security #linux #lsm #kubernetes #cks
Last updated: 2026-09-05

---

### **1. What — Nó là cái gì?**

AppArmor là Linux Security Module (LSM) cung cấp Mandatory Access Control (MAC) — giới hạn 1 chương trình được làm gì (đọc/ghi file nào, mở network loại nào, có capability gì), bất kể user chạy nó là ai, kể cả root. Khác DAC (owner/group/permission) truyền thống: MAC là policy do admin định nghĩa (gọi là **profile**), kernel enforce, app không tự override được dù chạy với quyền cao.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có nó: 1 service bị exploit (RCE) thì attacker có full quyền của user chạy service đó — service chạy root hoặc có setuid thì attacker leo thang toàn hệ thống ngay lập tức. AppArmor giới hạn "blast radius": dù bị RCE, process cũng chỉ đọc/ghi/exec/connect được đúng những gì profile cho phép → thêm 1 lớp containment, KHÔNG thay thế patch/hardening khác.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi**: cần defense-in-depth cho service/container chạy code không tin tưởng hoàn toàn (web app public-facing, container image pull từ ngoài, service có lịch sử CVE). Trong K8s: pod chạy trên node Debian/Ubuntu, đặc biệt cluster multi-tenant.

**KHÔNG dùng / cân nhắc kỹ**:
- Node chạy RHEL/CentOS/Fedora → mặc định dùng SELinux; chạy song song 2 LSM khác kiểu (path-based + label-based) cho cùng resource thường không khả thi/không được distro hỗ trợ — chọn 1 trong 2, đừng cố nhét cả hai.
- App tự sinh path ngẫu nhiên / self-update liên tục → rule path-based dễ vỡ, phải maintain profile không dừng.
- Không có ai chịu trách nhiệm maintain profile lâu dài → to ở `complain` mode mãi mãi = có cũng như không (xem mục Gotchas).

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
 User space                          Kernel space
┌─────────────────┐    load        ┌──────────────────────────┐
│ /etc/apparmor.d/ │ ─────────────▶ │  LSM hook (apparmor)      │
│  profile files   │ apparmor_parser│  attached vào mọi syscall │
└─────────────────┘                │  đụng file/network/        │
        ▲                          │  capability/ptrace/signal │
        │ aa-genprof/aa-logprof    │                            │
        │ (đọc log, gợi ý rule)    │  quyết định: ALLOW / DENY  │
        │                          └──────────────┬─────────────┘
        │                                         │ log
   /var/log/audit/audit.log  ◀── auditd/journald ─┘  (apparmor="DENIED"/"ALLOWED")
```

- Nằm giữa syscall và VFS/network stack — chặn TRƯỚC khi request chạm resource thật.
- Component chính: kernel module `apparmor`, userspace parser `apparmor_parser`, profile files trong `/etc/apparmor.d/`, `securityfs` mount tại `/sys/kernel/security/apparmor/` (nơi profile thực sự nằm trong kernel, đọc trạng thái runtime).
- Trong K8s: kubelet chỉ đọc field/annotation rồi truyền **tên** profile cho container runtime (containerd/CRI-O) qua CRI → runtime attach profile khi tạo container process. **Kubelet KHÔNG tự load profile lên node** — gotcha quan trọng nhất cho vận hành & thi CKS, xem [[apparmor--kubernetes-cks]].
- Dependency: kernel phải compile với `CONFIG_SECURITY_APPARMOR=y` và bật ở boot param — kiểm tra `cat /sys/module/apparmor/parameters/enabled`.

### **5. How — Cơ chế hoạt động**

- **Path-based, không phải label-based**: profile áp cho 1 binary theo đường dẫn thực thi, không gắn label vào inode như SELinux. Hệ quả: bind mount, symlink, hardlink, chroot có thể "né" rule nếu path thực tế khác path profile match — đây là khác biệt cốt lõi khi so AppArmor với SELinux.
- **Whitelist + default deny bên trong profile**: cái gì không liệt kê trong profile = deny ngầm định, trừ khi mode = `complain`.
- **4 mode chính**: `enforce` (chặn + log), `complain`/learning (chỉ log, không chặn), `audit` (log cả action được allow), `unconfined` (có "profile" gán rõ ràng nhưng cho phép tất cả — khác với KHÔNG có profile nào). Chi tiết lệnh quản lý mode → [[apparmor--tooling-workflow]].
- **Execute transition**: khi process A exec ra B, hậu tố `ix/px/cx/ux` quyết định B chạy dưới profile nào (kế thừa / profile riêng / child profile / unconfined). Nguồn lỗi phổ biến khi confine service có exec chain dài (vd: nginx exec ra script CGI). → [[apparmor--profile-syntax]].
- **Workflow chuẩn tạo profile mới**: chạy app ở `complain` → generate traffic thật (mọi tính năng) → `aa-logprof` đọc log, gợi ý rule → duyệt/confirm → chuyển `aa-enforce`. Không viết profile tay từ đầu cho app phức tạp.
- Backlinks: [[apparmor--profile-syntax]] · [[apparmor--tooling-workflow]] · [[apparmor--kubernetes-cks]] · [[apparmor--debugging-runbook]]

### **6. Key Config — Cấu hình cần nhớ**

- `/etc/apparmor.d/` — nơi chứa profile, tên **file** theo convention = path binary thay `/` bằng `.` (vd `usr.sbin.nginx`), nhưng tên **profile** khai báo bên trong file (`profile <name> {`) mới là thứ được kernel/K8s match — 2 cái không bắt buộc giống nhau, dễ nhầm.
- Default khi 1 binary **không có profile nào** = unconfined hoàn toàn, KHÔNG phải deny-all — default nguy hiểm hay bị hiểu nhầm, càng dễ nhầm hơn khi so với hành vi container runtime trong K8s (xem [[apparmor--kubernetes-cks]]).
- `complain` mode không có cảnh báo nổi bật trong output ngắn của `aa-status` — dễ quên chuyển `enforce` sau khi test xong → false sense of security.
- `deny` rule mặc định **âm thầm, không log** trừ khi thêm `audit deny <rule>` — chặn vẫn hoạt động nhưng không có dấu vết, khó debug/forensics.
- Sửa file profile **không tự** áp dụng — bắt buộc `apparmor_parser -r <file>` (hoặc reload service) sau mỗi lần edit tay.

### **7. Security Considerations**

- **Attack surface**: chính là chất lượng profile — profile quá lỏng (nhiều `ux`, wildcard rộng `/** rw`) khiến MAC gần như vô nghĩa.
- **Misconfig dễ gây breach**:
  - Để `complain` mode ở production dài hạn thay vì `enforce`.
  - Path-based bypass qua bind mount / symlink / hardlink trỏ ra ngoài phạm vi rule.
  - `ux`/`Ux` (execute unconfined) cho 1 binary con → phá vỡ toàn bộ chain confinement từ điểm đó trở đi.
  - K8s: field/annotation trỏ `localhost/<profile>` nhưng profile chưa load trên node → tối thiểu là pod lỗi `CreateContainerError` (an toàn nhưng down); nguy hiểm hơn nếu hiểu sai fallback và tưởng "không set = an toàn mặc định".
- **Hardening checklist tối thiểu**:
  - [ ] `aa-status` xác nhận mọi profile quan trọng đang `enforce`, không phải `complain`.
  - [ ] Không dùng wildcard file rule quá rộng (`/ rw`, `/** rw`) ở top-level profile.
  - [ ] Có `audit deny` cho path nhạy cảm (`/etc/shadow`, ssh keys, `docker.sock`) để log rõ khi có exploit attempt.
  - [ ] Config management / CI đảm bảo profile được `apparmor_parser -r` sau mỗi lần node/image cập nhật — đừng bake 1 lần vào image rồi quên.
  - [ ] K8s: dùng node label + nodeSelector/taint để pod cần AppArmor luôn schedule đúng node đã có profile, tránh crashloop rải rác.

### **8. Ops Runbook — Production Notes**

- **Health check**: `aa-status` (tổng quan số profile enforce/complain/unconfined, process đang bị confine) hoặc `systemctl status apparmor`.
- **Log quan trọng cần monitor**: `journalctl -k | grep -i apparmor`, hoặc `/var/log/audit/audit.log` (có auditd), hoặc `/var/log/kern.log`/`/var/log/syslog` (Debian/Ubuntu không auditd). Pattern grep: `apparmor="DENIED"`.
- **Metric cần alert**: tăng đột biến `DENIED` cho 1 profile cụ thể (nghi bị tấn công hoặc code mới thiếu quyền), hoặc số profile ở `complain` > 0 trên node production.
- **Reload sau khi sửa**: `sudo apparmor_parser -r /etc/apparmor.d/<file>` — không cần restart cả `apparmor.service`, thường không cần restart app trừ khi app tự cache hành vi.
- **Rollback nhanh** khi 1 profile mới làm app die: `sudo aa-complain /etc/apparmor.d/<file>` — chuyển ngay về complain, app sống lại, vẫn giữ profile để sửa tiếp (nhanh và an toàn hơn `aa-disable`).
- Chi tiết lệnh & flow chẩn đoán sự cố → [[apparmor--debugging-runbook]].

### **9. Gotchas & Lessons Learned**

_(điền dần khi thao tác thật trên hệ thống sắp bàn giao)_

- [ ] Ghi lại distro + kernel version của hệ thống sắp nhận bàn giao — AppArmor availability/behavior khác nhau (Ubuntu enable default; một số cloud image tối giản có thể disable).
- [ ] Ghi lại container runtime đang dùng (Docker/containerd/CRI-O) và version Kubernetes — cách khai báo profile (annotation cũ vs field `securityContext.appArmorProfile` mới từ 1.30) khác nhau hoàn toàn.

### **10. Resources**

- Official: https://apparmor.net/ — man pages: `man apparmor.d`, `man aa-status`, `man aa-logprof`
- Kubernetes docs: https://kubernetes.io/docs/tutorials/security/apparmor/
- Ubuntu Server Guide — chương AppArmor (thao tác thực tế, không phải hello-world)
- Ghi chú con: [[apparmor--profile-syntax]] · [[apparmor--tooling-workflow]] · [[apparmor--kubernetes-cks]] · [[apparmor--debugging-runbook]]
