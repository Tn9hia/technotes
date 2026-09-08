# Kyverno — Policy Exceptions
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--security-cves-hardening]], [[kyverno--policy-types-cel-vs-legacy]]
Tags: #kyverno #security #exceptions #cve

## What it does

`PolicyException` (legacy, `kyverno.io/v2beta1`) cho phép miễn trừ 1 resource cụ thể khỏi 1 rule cụ thể của 1 policy, **mà không sửa policy gốc**. Ví dụ: policy `disallow-host-path` áp `enforce` toàn cluster, nhưng team hạ tầng cần 1 DaemonSet cụ thể dùng hostPath — thay vì nới lỏng cả policy, tạo `PolicyException` trỏ đúng resource đó.

## Why it exists

Không có PolicyException, muốn miễn trừ 1 case đặc biệt buộc phải sửa policy gốc (thêm điều kiện loại trừ ngay trong rule) — làm policy phình to, khó review, và mọi thay đổi ngoại lệ đều phải qua chủ sở hữu policy. PolicyException tách quyền: policy owner viết rule chung, còn ai đó (có RBAC tạo PolicyException) có thể xin miễn trừ theo case cụ thể — tách biệt về mặt tổ chức. Đây cũng chính là lý do nó trở thành attack surface: tách quyền không đi kèm giới hạn namespace theo mặc định.

## How it works (flow/diagram)

```
ClusterPolicy (enforce) ──match──▶ resource X
                                        │
                    có PolicyException nào exceptions.match resource X không?
                                        │
                          ┌─────────────┴─────────────┐
                         Có                           Không
                          │                             │
                 rule đó bị SKIP cho resource X    enforce như bình thường
                 (dù policy vẫn enforce cho
                  resource khác)
```

Vấn đề (đã là CVE thật, xem Security notes): nếu có **2 PolicyException cùng match** 1 resource — 1 cái match chặt (theo tên cụ thể), 1 cái match lỏng (theo pattern/wildcard) — engine không đảm bảo áp dụng cái chặt hơn; kẻ tấn công có thể **cố ý đặt tên resource** để rơi vào phạm vi exception lỏng hơn, bypass hoàn toàn enforce dù không được cấp quyền cho case đó.

## Config gotchas

- Mặc định `PolicyException` **tạo được ở bất kỳ namespace nào**, không giới hạn theo namespace của policy — nếu không dùng flag `--enablePolicyException`/webhook restriction đi kèm cấu hình namespace whitelist (`--exceptionNamespace`), user có quyền tạo resource trong namespace bất kỳ có thể tự miễn trừ chính họ khỏi policy áp lên họ.
- Tránh dùng wildcard hoặc pattern lỏng trong `spec.exceptions[].match` — luôn chỉ định tên resource cụ thể hoặc label selector chặt, không dùng pattern có thể bị "khớp trúng" bởi tên resource do attacker tự đặt.
- Legacy `PolicyException` (`kyverno.io`) cũng nằm trong nhóm deprecated từ v1.19 — kiểm tra tài liệu policy type CEL mới nhất để biết cơ chế exception tương đương (chưa verify chi tiết cơ chế exception cho CEL policy — cần tự kiểm tra khi thực hành với version thật).

## Security notes — CVE thực tế (đã xác thực qua GitHub Security Advisories API)

- **GHSA-gg4x-fgg2-h9w9** — "Bypassing Kyverno Policies via Double Policy Exceptions", **CVSS: Critical**. Ảnh hưởng `v1.9.0 – v1.12.7`. 2 PolicyException chồng nhau (1 chặt hơn, 1 lỏng hơn) → Kyverno áp cái lỏng hơn bất kể → bypass hoàn toàn `enforce` (PoC gốc: bypass `disallow-hostPath`). **Chưa có `first_patched_version` được ghi rõ trong advisory** — verify version fix thật bằng CHANGELOG/release notes khi áp dụng, đừng suy đoán.
- **CVE-2024-48921** (GHSA-qjvc-p88j-j9rm) — "PolicyException objects can be created in any namespace by default", severity **Medium**. Fixed: `< 1.13.0` bị ảnh hưởng. Cluster user có quyền tạo resource trong 1 namespace bất kỳ có thể tự tạo PolicyException miễn trừ chính họ khỏi ClusterPolicy áp lên namespace đó → escalate (PoC gốc dẫn tới chạy privileged container).
- Bài học chung: **PolicyException là bề mặt tấn công độc lập với chính policy nó áp dụng** — review PolicyException phải khắt khe tương đương review policy gốc, không nên coi là "config phụ, ít rủi ro".

## Refs

- Policy Exceptions guide: https://kyverno.io/docs/guides/exceptions/
- GHSA-gg4x-fgg2-h9w9: https://github.com/kyverno/kyverno/security/advisories/GHSA-gg4x-fgg2-h9w9
- CVE-2024-48921 / GHSA-qjvc-p88j-j9rm: https://github.com/kyverno/kyverno/security/advisories/GHSA-qjvc-p88j-j9rm
