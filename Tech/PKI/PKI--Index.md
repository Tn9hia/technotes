# PKI — Index (MOC)
Tags: #moc #pki
Last updated: 2026-08-16

Map of Content cho toàn bộ note PKI/EJBCA. Đọc theo thứ tự gợi ý bên dưới nếu đang học từ đầu để chuẩn bị bàn giao; hoặc dùng để tra nhanh khi cần.

## Gợi ý thứ tự đọc (từ cơ bản → nâng cao)

1. [[PKI]] — bắt đầu ở đây, có glossary tra nhanh toàn bộ thuật ngữ.
2. [[pki--asymmetric-crypto]]
3. [[pki--x509-certificate]]
4. [[pki--ca-hierarchy-trust-chain]]
5. [[pki--csr-enrollment]]
6. [[pki--certificate-lifecycle]]
7. [[pki--revocation-crl-ocsp]]
8. [[pki--enrollment-protocols]]
9. [[pki--hsm-key-protection]]
10. [[pki--cp-cps-policy]]
11. [[EJBCA]] — chuyển sang phần công cụ thực tế công ty đang dùng.
12. [[ejbca--architecture-components]]
13. [[ejbca--profiles]] — khái niệm quan trọng nhất của EJBCA, đọc kỹ.
14. [[ejbca--end-entities-ra]]
15. [[ejbca--publishers-services-crl-ocsp]]
16. [[ejbca--rbac-admin-roles]]
17. [[ejbca--api-cli]]
18. [[ejbca--ops-runbook]] — vận hành thực tế hàng ngày.

## Theo chủ đề

**Nền tảng crypto & chuẩn:**
- [[pki--asymmetric-crypto]]
- [[pki--x509-certificate]]

**Mô hình trust & tổ chức CA:**
- [[pki--ca-hierarchy-trust-chain]]
- [[pki--hsm-key-protection]]
- [[pki--cp-cps-policy]]

**Vòng đời certificate:**
- [[pki--csr-enrollment]]
- [[pki--certificate-lifecycle]]
- [[pki--revocation-crl-ocsp]]
- [[pki--enrollment-protocols]]

**EJBCA — kiến trúc & vận hành:**
- [[EJBCA]]
- [[ejbca--architecture-components]]
- [[ejbca--profiles]]
- [[ejbca--end-entities-ra]]
- [[ejbca--publishers-services-crl-ocsp]]
- [[ejbca--rbac-admin-roles]]
- [[ejbca--api-cli]]
- [[ejbca--ops-runbook]]

## Ghi chú

Toàn bộ note theo template ở `CLAUDE-researcher.md` trong cùng thư mục — file root (`PKI.md`, `EJBCA.md`) dùng cấu trúc 10 mục đầy đủ, file con (`tech--concept.md`) dùng cấu trúc Tier 2 gọn hơn. Mục **Gotchas & Lessons Learned** ở 2 file root đang để trống — nên tự điền dần khi vào thực tế vận hành EJBCA, đó là phần giá trị nhất sau này.
