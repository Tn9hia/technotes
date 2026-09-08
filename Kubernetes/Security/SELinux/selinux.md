# SELinux — RHEL 9.x (selinux-policy targeted)
Tags: #selinux #linux #security #hardening #cks #mac
Last updated: 2026-09-06

> Verify base: Red Hat Enterprise Linux 9 "Using SELinux" guide (docs.redhat.com, truy cập 2026-09-06). Hệ điều hành mục tiêu: RHEL 9 / Rocky 9 / Alma 9 — bản mới nhất tại thời điểm viết là dòng RHEL 9.8. **Trước khi áp dụng bất kỳ lệnh nào, chạy `cat /etc/os-release` và `rpm -q selinux-policy` trên máy thật để đối chiếu version, vì hành vi default boolean/type có thể khác nhau giữa các minor release.**

---

### 1. What — Nó là cái gì?

SELinux (Security-Enhanced Linux) là một **Linux Security Module (LSM)** cài Mandatory Access Control (MAC) vào kernel, chạy song song với DAC (owner/group/permission) truyền thống. Thay vì chỉ hỏi "user này có quyền rwx trên file không", SELinux hỏi thêm: "process với **type** này có được policy cho phép thao tác lên object với **type** kia không". Trên RHEL 9, nó bật mặc định (`selinux-policy-targeted`) ngay từ lúc cài.

### 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có SELinux: một service chạy bằng root (hoặc bị lỗi/bị khai thác — VD: httpd dính RCE) có DAC quyền gì thì kẻ tấn công có quyền đó trên toàn hệ thống — đọc `/etc/shadow`, ghi vào `/home` người khác, mở port bất kỳ, escape ra ngoài phạm vi "đáng lẽ chỉ được phục vụ web". SELinux giới hạn **blast radius**: dù process bị chiếm quyền root, nó vẫn bị khoá trong domain (`httpd_t`) chỉ được đụng vào những gì policy khai — file `httpd_sys_content_t`, port `http_port_t`, v.v. Đây là lớp phòng thủ "defense in depth", không thay thế patching/hardening khác.

### 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Nên giữ Enforcing khi:**
- Server production, đặc biệt server public-facing (web, DB, mail).
- Môi trường cần compliance (PCI-DSS, STIG, CIS Benchmark RHEL đều yêu cầu SELinux Enforcing).
- Container host (Podman/CRI-O dùng SELinux làm lớp cô lập container bổ sung ngoài namespace).

**KHÔNG nên tắt SELinux chỉ vì:**
- "Denied rồi set permissive/disabled cho nhanh" → đây là root cause của gần hết các sự cố SELinux thực tế (xem mục 9). Đúng ra: sửa label/policy/boolean, không tắt MAC.
- Debug tạm thời: dùng `setenforce 0` (permissive) hoặc `semanage permissive -a <domain>` để cô lập **một domain cụ thể**, không disable toàn hệ thống.

**Có thể cân nhắc Disabled** (hiếm, có lý do rõ ràng): benchmark hiệu năng thuần, một số appliance/legacy software tuyên bố không tương thích (hiếm ở 2026) — nhưng luôn ưu tiên Permissive để vẫn có audit log thay vì Disabled (Disabled không label object mới → quay lại Enforcing sau này cực khổ, phải relabel toàn bộ).

### 4. Where - Architecture — Nó nằm ở đâu trong hệ thống?

```
                     ┌─────────────────────────────────────────┐
                     │              Userspace                  │
                     │  policycoreutils, semanage, setsebool,   │
                     │  restorecon, audit2allow, sealert ...    │
                     └───────────────┬───────────────────────--┘
                                     │ đọc/ghi policy, semanage store
                                     ▼
                     ┌─────────────────────────────────────────┐
                     │   Policy store (/etc/selinux/targeted)   │
                     │   compiled binary policy (policy.31...)  │
                     └───────────────┬───────────────────────--┘
                                     │ load vào kernel lúc boot / semodule
                                     ▼
┌──────────┐   syscall    ┌──────────────────────┐   AVC decision   ┌────────────┐
│  Process │ ───────────▶ │  Kernel LSM hook      │ ───────────────▶│  Allow /   │
│ (subject,│              │  (SELinux + AVC cache)│                  │  Deny +    │
│  type_t) │ ◀─────────── │  so sánh source type  │ ◀─────────────── │  log AVC   │
└──────────┘   syscall    │  vs target type theo  │   audit record   └────────────┘
   kết quả                │  policy rule          │        │
                          └──────────┬────────────┘        ▼
                                     │              /var/log/audit/audit.log
                                     ▼                (qua kernel audit subsystem)
                     ┌─────────────────────────────────────────┐
                     │   Object: file/dir/port/process khác     │
                     │   mang label object_r:type_t:level       │
                     └─────────────────────────────────────────┘
```

- **Vị trí:** hook ngay trong kernel, chạy **sau** DAC check — nghĩa là DAC pass rồi mới tới lượt SELinux; DAC deny thì SELinux không cần chạy.
- **Component chính:** kernel LSM hook, AVC (Access Vector Cache — cache quyết định allow/deny để khỏi tính lại mỗi syscall), policy store, audit subsystem (`auditd`), userspace tooling (`policycoreutils`, `libselinux`, `checkpolicy`, `setools`).
- **Data flow:** mọi truy cập (open file, bind port, ptrace process khác...) đều qua AVC trước; AVC miss thì hỏi kernel policy engine, kết quả cache lại; deny → ghi bản ghi `AVC` vào audit log.
- **Dependency:** phụ thuộc `kernel` (LSM framework), `auditd` (để log denial ra được — audit không chạy thì denial vẫn bị deny nhưng khó thấy log), filesystem phải support xattr `security.selinux` (ext4, xfs — hầu hết local fs; NFS thì cần tuỳ chỉnh context ở mount option vì NFS không lưu xattr theo cách thông thường).

### 5. How — Cơ chế hoạt động

**5 core concepts quan trọng nhất:**

1. **Security Context** — nhãn `user:role:type:level` gắn lên MỌI process và object (file, port, socket...). Ví dụ: `system_u:system_r:httpd_t:s0`. Trong policy targeted, **type** (`_t`) là thứ quyết định access control gần như 100% thời gian; user/role quan trọng hơn ở RBAC/MLS. Chi tiết → [[selinux--contexts-and-labeling]]

2. **Type Enforcement (TE)** — cơ chế lõi: policy định nghĩa hàng chục ngàn rule dạng `allow httpd_t httpd_sys_content_t:file { read getattr };`. Domain của process (vd `httpd_t`) chỉ được làm đúng những gì rule cho phép lên type của object.

3. **Modes: Enforcing / Permissive / Disabled** — quyết định policy có được **thi hành** hay chỉ **log**. Permissive vẫn label object và ghi AVC log, chỉ không chặn — cực hữu ích để debug. Chi tiết relabel/boot flow → [[selinux--modes-and-relabeling]]

4. **Booleans** — công tắc runtime bật/tắt cụm rule đã biên dịch sẵn, không cần compile lại policy. VD `httpd_can_network_connect`. Chi tiết → [[selinux--booleans]]

5. **MCS/MLS categories** — lớp thứ hai độc lập với TE, dùng `s0:c1,c2` để cô lập các instance cùng type (VD: 2 container cùng chạy `container_t` nhưng khác category → vẫn không đụng được file của nhau dù cùng type). Đây là cơ chế containers/VM dùng để cô lập lẫn nhau. Chi tiết → [[selinux--mls-mcs-sandbox]]

**Request lifecycle (khi 1 process cố truy cập 1 file):**
`syscall → DAC check (pass) → kernel tra AVC cache → cache miss → tra policy binary (TE rule + MCS category match) → allow/deny → cache kết quả → nếu deny: sinh audit record type=AVC → auditd ghi vào /var/log/audit/audit.log`

### 6. Key Config — Cấu hình cần nhớ

- **`/etc/selinux/config`** — 2 dòng quan trọng: `SELINUX=enforcing|permissive|disabled` và `SELINUXTYPE=targeted|minimum|mls`. **Đổi xong PHẢI reboot** để có hiệu lực đầy đủ (permissive→enforcing có thể cần relabel).
- ⚠️ **Nguy hiểm nhất:** set `SELINUX=disabled` trong file này. Theo RHEL 9 docs, cách hiện đại được khuyến nghị là dùng kernel param (`grubby --update-kernel ALL --args selinux=0`) thay vì sửa config file, vì disable qua config làm hệ thống **bỏ qua hoàn toàn** việc label object mới → khi bật lại Enforcing sau này bắt buộc phải full relabel (`touch /.autorelabel && reboot`), tốn thời gian và có rủi ro boot fail nếu thiếu bước `fixfiles -F onboot` trước khi reboot.
- **`semanage fcontext`** — thứ hay bị hiểu sai nhất: đây chỉ là **khai báo policy** (persistent qua relabel), KHÔNG tự áp dụng lên file đang tồn tại. Phải chạy thêm `restorecon -Rv <path>` mới có hiệu lực thật. Quên bước này = tưởng đã fix nhưng vẫn denied.
- **Boolean mặc định nguy hiểm khi bật tuỳ tiện:** `httpd_unified`, `httpd_enable_homedirs`, `selinuxuser_execmod`... — mỗi boolean = nới rule cho hẳn 1 cụm hành vi, không phải chỉ 1 file. Đừng bật boolean theo kiểu "thử cho hết deny" — chỉ bật đúng boolean liên quan đến denial cụ thể (`audit2why` sẽ gợi ý đúng boolean nếu có).
- **Port context** phải add trước khi service start ở port khác chuẩn, nếu không service bind fail (`Permission denied`) chứ không chỉ là bug ứng dụng.

### 7. Security Considerations

- **Attack surface:** nằm ở (a) chính sách quá lỏng do lạm dụng boolean/module tự chế, (b) domain `unconfined_t`/`unconfined_service_t` — nhiều service 3rd-party không có policy riêng bị gán unconfined = SELinux gần như không bảo vệ gì cho nó, (c) label bị sai (mislabeled) do copy/move file sai cách khiến rule không match được mục đích.
- **Misconfiguration gây breach thực tế:**
  - Set toàn hệ thống Permissive/Disabled "cho dễ" rồi quên bật lại — biến MAC layer thành no-op vĩnh viễn.
  - `audit2allow -M x -a` chạy tự động, generate module từ **toàn bộ** log denial (bao gồm cả denial do exploit/scan) rồi `semodule -i` mà không review từng dòng → vô tình cấp quyền cho hành vi độc hại đã bị chặn.
  - Dùng `chcon` (đổi context tạm thời, không persistent) thay vì `semanage fcontext + restorecon` → tưởng đã "fix vĩnh viễn" nhưng sau lần relabel/update kế tiếp, context trở lại sai như cũ mà không ai biết.
  - Container: chạy `podman run --privileged` hoặc `--security-opt label=disable` tràn lan → tắt hẳn lớp cô lập SELinux cho container đó.
- **Hardening checklist tối thiểu:**
  - [ ] `getenforce` = `Enforcing` trên mọi server production.
  - [ ] `SELINUXTYPE=targeted` (hoặc `mls` nếu có yêu cầu compliance cao hơn).
  - [ ] Không có custom policy module nào được load mà không qua review (`semodule -l` định kỳ audit).
  - [ ] Không có domain nghiệp vụ quan trọng nào bị permissive vĩnh viễn (`semanage permissive -l` phải rỗng hoặc có lý do ghi chú rõ).
  - [ ] File context của thư mục dữ liệu custom đã khai báo đúng qua `semanage fcontext`, không dựa vào `chcon` tạm.
  - [ ] Audit log (`auditd`) đang chạy và có retention đủ để điều tra sự cố.

### 8. Ops Runbook — Production Notes

- **Health check nhanh:** `sestatus` (xem mode, policy, mode from config file có khớp mode hiện tại không — lệch nhau là dấu hiệu ai đó `setenforce` tạm mà quên sync config).
- **Log quan trọng:** `/var/log/audit/audit.log` (nguồn sự thật — audit record `type=AVC`), `journalctl -t setroubleshoot` nếu có setroubleshoot cài (không mặc định trên server minimal install — cần `dnf install setroubleshoot-server` mới có `sealert`/phân tích thân thiện).
- **Metric cần theo dõi:** tần suất AVC denial tăng đột biến sau mỗi lần deploy/update package — dấu hiệu policy chưa theo kịp thay đổi ứng dụng, hoặc dấu hiệu bị tấn công (nhiều denial lạ dồn dập từ 1 domain).
- **Quy trình xử lý denial chuẩn** (không phải "audit2allow -M rồi semodule -i" ngay): chi tiết đầy đủ từng bước → [[selinux--incident-runbook]]
- **Restart/rollback:** đổi mode không cần restart service (`setenforce 0/1` áp dụng ngay toàn hệ thống); đổi boolean cũng vậy (`setsebool` áp dụng ngay); nhưng đổi `SELINUXTYPE` hoặc bật lại từ Disabled thì bắt buộc **relabel + reboot**.

### 9. Gotchas & Lessons Learned

- **"Nó chạy được trên máy dev mà không chạy trên prod"** — 90% là do SELinux denial trên prod (Enforcing) trong khi dev để Permissive/Disabled. Luôn giữ SELinux mode giống nhau giữa các môi trường để bug lộ ra sớm.
- **Di chuyển file bằng `mv` giữ nguyên context cũ** (khác thư mục label khác) trong khi `cp` (không có `-a`/`--preserve=context`) thường label lại theo thư mục đích. Đây là nguồn gốc phổ biến của "tự nhiên bị denied" sau khi thao tác file tưởng như vô hại — copy web content vào `/var/www` bằng cách move từ `/tmp` mang theo `tmp_t` thay vì `httpd_sys_content_t`.
- **Restart service không tự fix label sai** — phải restorecon trước.
- **`setenforce 0` chỉ tạm thời (mất khi reboot)** — dễ tưởng nhầm là đã tắt permanent, rồi ngạc nhiên khi server reboot xong lại Enforcing (hoặc ngược lại, tưởng permanent nhưng thật ra chưa sửa `/etc/selinux/config`).
- **Không phải mọi lỗi "Permission denied" đều do SELinux** — luôn kiểm tra DAC (owner/group/mode) trước, SELinux là lớp thứ hai. Cách phân biệt nhanh: tắt tạm permissive (`setenforce 0`) rồi thử lại — nếu vẫn lỗi thì không phải SELinux.
- **`audit2allow` là công cụ gợi ý, không phải công cụ fix** — theo đúng RHEL docs, không nên dùng làm phương án đầu tiên; phải kiểm tra label/boolean/port context trước.
- **Container + SELinux:** quên gắn `:z`/`:Z` khi mount volume vào container là nguyên nhân denial phổ biến nhất khi vận hành Podman trên RHEL9 — container process (`container_t`) không có quyền lên file host mang label host gốc.

### 10. Resources

- [Using SELinux — RHEL 9 official guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_selinux/index) — nguồn chính, verify lại mọi command trước khi note.
- [Chapter 2 — Changing SELinux states and modes (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux)
- [Chapter 5 — Troubleshooting problems related to SELinux (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/troubleshooting-problems-related-to-selinux_using-selinux)
- [Chapter 9 — Creating SELinux policies for containers / udica (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/creating-selinux-policies-for-containers_using-selinux)
- [SELinux Project wiki](https://selinuxproject.org/) — tài liệu upstream, không phụ thuộc Red Hat.
- [Kubernetes — Configure a Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) — cho phần CKS.
- Child notes trong bộ note này (xem backlink `[[...]]` rải trong bài).

---
## Child notes (Tier 2)
- [[selinux--contexts-and-labeling]]
- [[selinux--modes-and-relabeling]]
- [[selinux--booleans]]
- [[selinux--troubleshooting]]
- [[selinux--policy-modules]]
- [[selinux--port-network-labeling]]
- [[selinux--mls-mcs-sandbox]]
- [[selinux--containers-kubernetes]]
- [[selinux--systemd-service-confinement]]
- [[selinux--incident-runbook]]
