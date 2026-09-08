# SELinux — Modes, Boot Flow & Relabeling
Tier: 2
Parent: [[selinux]]
Related: [[selinux--contexts-and-labeling]], [[selinux--incident-runbook]]
Tags: #selinux #boot #relabel

## What it does

Quản lý 3 trạng thái vận hành của SELinux (Enforcing/Permissive/Disabled) và quy trình relabel toàn hệ thống khi chuyển trạng thái hoặc khi label bị hỏng diện rộng.

## Why it exists

Enforcing/Permissive tách biệt "thi hành policy" khỏi "quan sát policy" — cho phép debug an toàn (Permissive vẫn log AVC nhưng không chặn) mà không phải tắt hẳn MAC. Relabel tồn tại vì label sống trong xattr của filesystem — nếu SELinux từng Disabled hoặc policy thay đổi lớn, label trên đĩa không còn khớp với policy mới, cần quét lại toàn bộ.

## How it works (flow/diagram)

**Kiểm tra trạng thái:**
```bash
getenforce          # Enforcing | Permissive | Disabled — trạng thái RUNTIME
sestatus            # chi tiết hơn: current mode, mode from config file, policy type
```
`sestatus` quan trọng hơn `getenforce` khi audit vì nó cho thấy **lệch pha** giữa runtime mode và config file (ai đó `setenforce` tạm rồi quên đồng bộ).

**Đổi mode tạm thời (mất khi reboot), chỉ áp dụng Enforcing ⇄ Permissive:**
```bash
setenforce 0   # → Permissive
setenforce 1   # → Enforcing
```
Không thể `setenforce` sang/từ Disabled — Disabled chỉ set được qua boot param, cần reboot.

**Đổi permanent:** sửa `/etc/selinux/config` → `SELINUX=enforcing|permissive|disabled` → **reboot**.

**Cô lập 1 domain ở permissive mà không hạ cả hệ thống** (kỹ thuật debug đúng cách, ưu tiên hơn `setenforce 0` toàn cục):
```bash
semanage permissive -a httpd_t     # chỉ domain httpd_t chạy permissive
semanage permissive -l             # xem domain nào đang permissive
semanage permissive -d httpd_t     # gỡ lại enforcing cho domain đó
```

**Disable SELinux — cách hiện đại theo RHEL 9 docs** (ưu tiên hơn sửa config file):
```bash
grubby --update-kernel ALL --args selinux=0
reboot
```
Boot param liên quan khác:
- `enforcing=0` — boot vào Permissive (dùng khi nghi ngờ hệ thống không boot được do SELinux, để chẩn đoán mà vẫn còn log).
- `autorelabel=1` — force relabel toàn bộ ở lần boot kế tiếp, tương đương `touch /.autorelabel && reboot`.

**Relabel toàn hệ thống (chuyển Disabled → Enforcing, hoặc label bị hỏng diện rộng):**
```bash
touch /.autorelabel
reboot
```
Hoặc không cần reboot ngay (chạy trực tiếp, chậm hơn, cẩn thận với hệ thống đang live):
```bash
fixfiles -F onboot     # đánh dấu relabel vào lần boot kế tiếp — khuyến nghị trước khi bật lại Enforcing
```

## Config gotchas

- **Chuyển từ Disabled → Enforcing/Permissive luôn kích hoạt autorelabel tự động** theo RHEL docs — hãy chủ động chạy `fixfiles -F onboot` trước khi reboot để tránh boot dừng giữa chừng do thiếu label trên file mà systemd cần sớm trong quá trình boot.
- Nên **boot thử ở Permissive trước** (`enforcing=0`) khi vừa bật lại SELinux sau một thời gian dài Disabled — nếu có sai sót nghiêm trọng, hệ thống vẫn boot được (chỉ log, không chặn) thay vì treo boot.
- Relabel toàn bộ filesystem lớn (`/`) có thể mất hàng chục phút tuỳ dung lượng — lên kế hoạch maintenance window, đừng chạy bất ngờ trên production.
- Permissive mode **chỉ log denial đầu tiên** trong một chuỗi denial lặp lại giống hệt nhau (theo cơ chế AVC cache) — đừng ngạc nhiên nếu log không show đủ số lần request thực tế.

## Security notes

- Disabled mode không label object mới được tạo trong thời gian đó → nợ kỹ thuật (technical debt) tăng dần, càng để lâu càng khó bật lại Enforcing an toàn.
- `semanage permissive -a <domain>` là công cụ hợp pháp cho debug nhưng **dễ bị lạm dụng thành "permanent workaround"** — luôn có ticket/note nhắc dọn lại, và định kỳ review `semanage permissive -l` (mục checklist trong [[selinux]] section 7).

## Refs

- [Chapter 2 — Changing SELinux states and modes (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux)
