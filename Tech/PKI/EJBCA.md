# EJBCA — Enterprise Java Beans Certificate Authority
Tags: #pki #ejbca #infra
Last updated: 2026-08-16

---

### **1. What — Nó là cái gì?**

EJBCA là phần mềm CA (Certificate Authority) mã nguồn mở, hiện do **Keyfactor** phát triển (trước đây là PrimeKey), viết trên nền Java EE. Nó hiện thực hoá gần như toàn bộ các khái niệm PKI (xem [[PKI]]) thành 1 sản phẩm chạy được thực tế: quản lý CA, cấp/thu hồi cert, publish CRL/OCSP, quản lý end entity, RBAC, API tự động hoá. Có bản **Community** (mã nguồn mở, miễn phí) và **Enterprise** (thêm HA, hỗ trợ chính thức, tính năng nâng cao như RA node tách rời, ACME, validation nâng cao...).

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Tự build 1 CA từ đầu (dùng OpenSSL script thủ công) không scale: không có RBAC, không audit log chuẩn, không tự động publish CRL/OCSP, không quản lý được hàng nghìn end entity, không tích hợp HSM chuẩn hoá, không có API cho tự động hoá. EJBCA đóng gói tất cả thành 1 hệ thống có UI quản trị, API, và tuân thủ được các chuẩn compliance (WebTrust, eIDAS, Common Criteria tuỳ bản Enterprise) — thứ mà công ty cần khi PKI trở thành hạ tầng production thực sự, không còn là vài cert tự ký cho dev.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:**
- Cần vận hành CA nội bộ lâu dài, số lượng cert lớn, nhiều team/hệ thống cần enrollment tự động (mTLS service-to-service, VPN, 802.1x, IoT).
- Cần audit trail, RBAC, approval workflow cho việc cấp/thu hồi cert (yêu cầu compliance).
- Cần hỗ trợ nhiều giao thức enrollment chuẩn (ACME/SCEP/EST/CMP/REST) cho nhiều loại client khác nhau.

**KHÔNG dùng / cân nhắc thay thế khi:**
- Chỉ cần vài chục cert nội bộ, không cần tự động hoá — 1 script OpenSSL hoặc `step-ca`/`smallstep` đơn giản hơn nhiều để vận hành.
- Hạ tầng đã dùng cloud-native CA (AWS Private CA, HashiCorp Vault PKI engine) và không có lý do compliance/kỹ thuật để tự vận hành CA riêng — quản lý EJBCA (app server, DB, HSM, patching) là gánh nặng vận hành thực sự, không nên đánh giá thấp.
- Cần short-lived cert cực nhanh cho service mesh nội bộ (giây/phút) — SPIFFE/SPIRE hoặc Vault PKI thường nhẹ và nhanh hơn cho use-case đó, dù EJBCA vẫn có thể làm được qua ACME.

*(Ở đây công ty đã chọn EJBCA làm CA chính — phần này để hiểu bối cảnh khi so sánh với hệ thống khác, không phải để đề xuất đổi.)*

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
                    ┌─────────────────────────────────────────┐
                    │           EJBCA Node(s)                  │
                    │  (WildFly/JBoss app server, hoặc container│
                    │   EJBCA Enterprise từ bản 7.x trở đi)      │
                    │                                            │
                    │  ┌───────────┐  ┌───────────┐             │
                    │  │ Admin Web │  │  RA Web    │             │
                    │  │ (full cfg,│  │ (enrollment│             │
                    │  │  client   │  │  đơn giản  │             │
                    │  │  cert auth)│  │  cho RA)   │             │
                    │  └───────────┘  └───────────┘             │
                    │  ┌────────────────────────────┐            │
                    │  │ REST API / EST / SCEP / CMP │            │
                    │  │ / ACME endpoints              │          │
                    │  └────────────────────────────┘            │
                    └───────────┬───────────────┬────────────────┘
                                │               │
                     ┌──────────▼─────┐  ┌──────▼─────────┐
                     │   Database      │  │  Crypto Token   │
                     │ (MariaDB/       │  │ (soft keystore  │
                     │  PostgreSQL/    │  │  hoặc HSM qua   │
                     │  Oracle...)     │  │  PKCS#11)       │
                     │ lưu cert, CA    │  │ lưu private key │
                     │ metadata, EE,   │  │ của CA           │
                     │ audit log       │  │                 │
                     └─────────────────┘  └─────────────────┘
                                │
                     ┌──────────▼──────────────────┐
                     │  Publishers (đẩy cert/CRL ra  │
                     │  LDAP, AD, external DB...)     │
                     │  Services (CRL gen, expire      │
                     │  notify — chạy theo lịch)        │
                     └──────────────────────────────┘
```

- **Vị trí trong hệ thống:** EJBCA là trung tâm phát hành/quản lý cert nội bộ — mọi service cần cert (TLS, mTLS, VPN, code signing...) enroll thông qua nó, trực tiếp (Admin/RA Web, CLI) hoặc gián tiếp (ACME/SCEP/EST/CMP/REST tự động).
- **Component chính:** App server (chạy toàn bộ logic CA), Database (lưu mọi thứ trừ private key nếu dùng HSM), Crypto Token/HSM (bảo vệ private key CA — xem [[pki--hsm-key-protection]]), Publishers (đồng bộ ra hệ thống khác), Services (tác vụ nền theo lịch).
- **Traffic/data flow:** Client → gửi CSR/enrollment request (qua UI hoặc protocol) → EJBCA xác thực + áp Profile → CA ký (gọi tới Crypto Token/HSM) → lưu cert vào DB → publish (nếu có Publisher cấu hình) → trả cert về client.
- **Dependency:** Database phải luôn sẵn sàng (EJBCA gần như không hoạt động được nếu DB down), Crypto Token/HSM phải kết nối được để ký, app server cần đủ tài nguyên cho load enrollment cao điểm.

### **5. How — Cơ chế hoạt động**

Core concepts đặc thù EJBCA cần nắm:

- **Architecture & Crypto Token** — cách EJBCA tổ chức CA, DB, HSM. → [[ejbca--architecture-components]]
- **Profiles (CA / Certificate / End Entity)** — 3 tầng cấu hình quyết định "cert được cấp ra sao". → [[ejbca--profiles]]
- **End Entity & RA workflow** — đơn vị "người/thiết bị được cấp cert" và quy trình duyệt. → [[ejbca--end-entities-ra]]
- **Publishers & Services** — đồng bộ dữ liệu ra ngoài + tác vụ nền (CRL, expire notify). → [[ejbca--publishers-services-crl-ocsp]]
- **RBAC (Roles & Access Rules)** — phân quyền admin, xác thực bằng client cert. → [[ejbca--rbac-admin-roles]]
- **REST API / CLI / Peer connectors** — tự động hoá và tích hợp. → [[ejbca--api-cli]]
- **Ops runbook** — vận hành hàng ngày thực tế. → [[ejbca--ops-runbook]]

**Request lifecycle tóm tắt (issue 1 cert mới qua RA):**
1. Admin/RA tạo **End Entity** mới (chọn End Entity Profile → quyết định field nào cần điền, CA nào được dùng, Certificate Profile nào được áp).
2. Entity đó nhận credential đăng ký 1 lần (enrollment code) hoặc admin generate keystore trực tiếp.
3. Entity enroll (qua Web/CLI/API/protocol) → EJBCA verify → CA (được chỉ định trong End Entity) ký cert theo đúng Certificate Profile.
4. Cert lưu DB, publish theo Publisher đã cấu hình (nếu có), trả về entity.

### **6. Key Config — Cấu hình cần nhớ**

- **Crypto Token cho mỗi CA** — CA không có Crypto Token hợp lệ (mất kết nối HSM, hoặc soft keystore bị khoá) thì **không ký được cert mới**, dù toàn bộ hệ thống còn lại chạy bình thường. Đây là điểm fail phổ biến nhất khi debug "sao không cấp cert được".
- **End Entity Profile giới hạn field nào bắt buộc/tuỳ chọn** — cấu hình quá lỏng (cho phép RA tự nhập bất kỳ SAN nào) là lỗ hổng thực tế: RA/end user có thể enroll cert cho domain họ không sở hữu nếu không có kiểm soát chặt ở tầng này.
- **Certificate Profile mặc định của EJBCA** (`ENDUSER`, `SUBCA`, `ROOTCA`...) chỉ nên dùng làm điểm khởi đầu để clone, không nên dùng thẳng cho production — luôn tạo Profile riêng theo đúng CP/CPS nội bộ (xem [[pki--cp-cps-policy]]).
- **Approval Profile** — thao tác nhạy cảm (tạo CA, revoke hàng loạt...) nên bắt buộc multi-person approval, không để 1 admin tự làm 1 mình.

### **7. Security Considerations**

- **Attack surface:** Admin Web (xác thực bằng client certificate, không phải username/password thuần — cấu hình sai phần này là rủi ro lớn), RA Web/enrollment endpoint public-facing (nếu mở ra ngoài), REST API key/credential.
- **Misconfiguration gây breach:** End Entity Profile cho phép tự set SAN tuỳ ý; Crypto Token dùng soft keystore (không HSM) cho Root/Issuing CA production; Role/Access Rule cấp quyền "Super Administrator" cho quá nhiều người thay vì theo nguyên tắc least-privilege.
- **Hardening checklist tối thiểu:**
  - Root CA (nếu chạy trong EJBCA) nên offline sau khi ký Issuing CA — hoặc tốt hơn, giữ Root CA ở hệ thống tách biệt hoàn toàn.
  - HSM cho ít nhất Issuing CA production, không dùng soft keystore.
  - Admin Web chỉ truy cập được từ mạng quản trị (không public), xác thực bằng client cert cấp riêng cho từng admin.
  - Bật audit log đầy đủ, xuất log ra hệ thống SIEM ngoài EJBCA (đừng chỉ tin audit log nằm trong chính DB của EJBCA).

### **8. Ops Runbook — Production Notes**

Xem chi tiết đầy đủ ở [[ejbca--ops-runbook]]. Tóm tắt nhanh:
- Health check: CA còn ký được không (Crypto Token connectivity), DB connection pool, dung lượng ổ đĩa (audit log + CRL tích luỹ).
- Log quan trọng: audit log (ai làm gì), application log của app server (WildFly), log kết nối HSM.
- Metric cần alert: cert của chính EJBCA CA sắp hết hạn, CRL sắp/đã quá `nextUpdate`, Service (worker) job fail (vd CRL generation service).
- Backup: Database + Crypto Token (nếu soft keystore, backup cực kỳ nhạy cảm) + cấu hình (Profiles, Roles) — restore-test định kỳ, đừng chỉ backup mà không thử restore.

### **9. Gotchas & Lessons Learned**

- *(điền dần trong quá trình tiếp nhận thực tế — vd: thứ tự áp dụng field giữa End Entity Profile và Certificate Profile khi xung đột, hành vi khi Crypto Token mất kết nối giữa chừng 1 batch issue, downtime khi upgrade version...)*

### **10. Resources**

- Official docs: docs.keyfactor.com (EJBCA documentation, cả Community lẫn Enterprise).
- EJBCA Community source code trên GitHub (Keyfactor/ejbca-ce) — hữu ích khi cần hiểu chính xác hành vi 1 tính năng.
- [[PKI]] — nền tảng lý thuyết đứng sau mọi khái niệm EJBCA hiện thực hoá.
