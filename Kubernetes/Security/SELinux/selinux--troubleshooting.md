# SELinux — Troubleshooting Tools
Tier: 2
Parent: [[selinux]]
Related: [[selinux--contexts-and-labeling]], [[selinux--booleans]], [[selinux--policy-modules]], [[selinux--incident-runbook]]
Tags: #selinux #troubleshooting #audit

## What it does

Bộ công cụ để tìm, đọc, và diễn giải AVC denial (bản ghi kernel từ chối truy cập) — biến audit log khô khan thành hành động sửa cụ thể: sửa label, bật boolean, hay (hiếm khi) viết policy module.

## Why it exists

Audit log thô (`type=AVC`) khó đọc trực tiếp — chứa source/target context, syscall, permission bị chặn nhưng không nói thẳng "phải làm gì". Các tool này phân tích ngược từ log ra hành động khả dĩ, kèm mức độ tự tin.

## How it works (flow/diagram)

**Bước 1 — tìm denial liên quan (đúng theo ví dụ RHEL 9 docs):**
```bash
ausearch -m AVC,USER_AVC,SELINUX_ERR,USER_SELINUX_ERR -ts recent
```
`-ts recent` = 10 phút gần nhất; có thể thay bằng `-ts today` hoặc mốc thời gian cụ thể.

**Bước 2 — hỏi "tại sao bị chặn, cách nào hợp lý để mở":**
```bash
ausearch -m avc -ts recent | audit2why
```
`audit2why` diễn giải nguyên nhân: label sai? thiếu boolean? hay thật sự thiếu policy rule?

**Bước 3 (chỉ khi thật sự cần custom rule, không phải mặc định) — sinh module:**
```bash
ausearch -m avc -ts recent | audit2allow -M mymodule
semodule -i mymodule.pp
```
Ví dụ chính thức từ RHEL docs cho 1 binary cụ thể:
```bash
ausearch -x /usr/bin/passwd --raw | audit2allow -D -M my-passwd
```

**Phân tích thân thiện hơn (nếu có cài `setroubleshoot-server`):**
```bash
sealert -a /var/log/audit/audit.log
```
⚠️ `setroubleshoot`/`sealert` **không cài mặc định** trên RHEL 9 minimal/server install — phải `dnf install setroubleshoot-server` trước. Đừng ngạc nhiên khi `sealert: command not found` trên server mới cài.

## Config gotchas

- **RHEL 9 docs nói rõ**: "You should not use `audit2allow` to generate a local policy module as your first option" — thứ tự ưu tiên đúng là: (1) kiểm tra label file bằng `matchpathcon`/`ls -Z` → (2) kiểm tra boolean liên quan → (3) kiểm tra port context nếu là network → chỉ khi cả 3 đều ổn mới tính đến (4) custom policy module.
- `audit2allow` sinh module dựa trên **log đã có**, không phân biệt được denial nào "hợp lệ nên fix" và denial nào "đúng ra phải bị chặn" (VD: log của một lần bị quét cổng/khai thác). Luôn đọc kỹ từng dòng rule trước `semodule -i`, không pipe thẳng `audit2allow -M x -a | semodule -i` theo phản xạ.
- `ausearch` cần quyền root và `auditd` phải đang chạy — nếu tắt `auditd`, denial vẫn bị chặn (SELinux không phụ thuộc audit để enforce) nhưng sẽ không thấy log để điều tra.
- Theo Red Hat: viết custom SELinux policy tự tay "falls outside of the Production Support Scope of Coverage" trừ khi Red Hat cung cấp sẵn — cân nhắc mở support case thay vì tự chế module phức tạp cho hệ thống có support contract.

## Security notes

- Một denial bất thường, dồn dập, từ domain lạ hoặc nhắm vào file nhạy cảm (`shadow_t`, `etc_t` write attempt) nên được xem như **tín hiệu điều tra incident**, không chỉ là "bug cần fix cho hết log".
- Giữ lại `audit.log` đủ lâu (retention) để phục vụ điều tra sau sự cố — đây là nguồn bằng chứng access-control-level quan trọng.

## Refs

- [Chapter 5 — Troubleshooting problems related to SELinux (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/troubleshooting-problems-related-to-selinux_using-selinux)
