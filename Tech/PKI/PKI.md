# PKI — Public Key Infrastructure
Tags: #pki #security #infra #crypto
Last updated: 2026-08-16

---

### **1. What — Nó là cái gì?**

PKI là tập hợp người, chính sách, quy trình và hệ thống dùng để tạo, phát hành, quản lý, phân phối và thu hồi **certificate số** (digital certificate) — thứ dùng để gắn một public key với một danh tính (server, người dùng, thiết bị, service). Nói đơn giản: PKI là "cơ quan cấp CMND điện tử" cho máy móc và người dùng trong hệ thống, để hai bên không quen biết nhau vẫn có thể xác thực lẫn nhau và trao đổi dữ liệu mã hoá.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Không có PKI, mình chỉ còn 2 lựa chọn tệ: (1) trust-on-first-use kiểu SSH (chấp nhận rủi ro MITM ở lần kết nối đầu), hoặc (2) tự tay trao đổi shared secret với từng bên trước khi nói chuyện — không scale được với internet hay hệ thống có hàng nghìn service/thiết bị.

PKI giải quyết 2 bài toán cùng lúc:
- **Xác thực danh tính** — chứng minh "public key này thực sự thuộc về domain X / service Y" thông qua chữ ký của một bên thứ ba đáng tin (CA).
- **Trao đổi khoá an toàn không cần gặp trước** — nhờ asymmetric crypto (xem [[pki--asymmetric-crypto]]), hai bên lạ vẫn thiết lập được kênh mã hoá (TLS handshake là ví dụ điển hình).

Nếu không có PKI: không có HTTPS đáng tin, không có mTLS giữa service-to-service, không ký code/document được, không xác thực VPN/802.1x bằng cert được.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:**
- Cần xác thực danh tính hai chiều hoặc một chiều giữa các bên không có trust sẵn (TLS server/client cert, mTLS giữa microservices).
- Cần ký số (code signing, document signing, email S/MIME) để đảm bảo tính toàn vẹn + non-repudiation.
- Cần cấp danh tính máy cho thiết bị số lượng lớn (IoT, 802.1x network access, VPN client cert) — thứ mà password không scale nổi.

**KHÔNG dùng / cân nhắc thay thế khi:**
- Hệ thống nhỏ, nội bộ, đã có mTLS layer do service mesh tự quản lý (vd Istio tự sinh cert ngắn hạn) — lúc đó PKI vẫn tồn tại nhưng nằm ẩn trong mesh, không cần build riêng.
- Chỉ cần xác thực người dùng cuối (human) — OAuth2/OIDC + password/MFA thường rẻ và dễ vận hành hơn certificate-based auth.
- Không có ai vận hành/rotate key được — một CA bị bỏ quên (không renew, không revoke kịp) nguy hiểm hơn là không có PKI, vì nó tạo ảo giác an toàn.
- Thời gian sống của định danh cực ngắn và tự động hoàn toàn (vd container ngắn hạn) — nên cân nhắc short-lived cert tự động qua ACME/SPIFFE thay vì cấp cert thủ công dài hạn.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
                         ┌───────────────────────────┐
                         │        Root CA            │  ← offline, ký 1 lần rồi cất (HSM/air-gap)
                         │  (self-signed, trust anchor)│
                         └─────────────┬─────────────┘
                                       │ ký
                         ┌─────────────▼─────────────┐
                         │     Intermediate/Issuing CA│  ← online, cấp cert hàng ngày (EJBCA chạy ở đây)
                         └─────────────┬─────────────┘
                     ┌─────────────────┼─────────────────┐
                     │                 │                 │
              ┌──────▼─────┐   ┌───────▼──────┐   ┌──────▼──────┐
              │  RA         │   │  End Entity  │   │  VA (OCSP/  │
              │ (Registration│   │ (server/user/│   │  CRL)       │
              │  Authority)  │   │  device)     │   │             │
              └──────────────┘   └──────────────┘   └─────────────┘

Client kết nối → nhận cert của End Entity → build chain lên tới
Root CA (đã có sẵn trong trust store OS/browser) → validate chữ ký
từng mắt xích → check revocation (CRL/OCSP) → trust hay không.
```

- **Vị trí trong hệ thống:** PKI là lớp hạ tầng nền (giống DNS) — hầu như mọi kết nối TLS, mTLS, VPN, 802.1x, code signing đều phụ thuộc vào nó, nhưng bản thân nó thường "vô hình" khi hoạt động đúng.
- **Component chính:** CA (Certificate Authority), RA (Registration Authority — xác minh danh tính trước khi CA ký), VA (Validation Authority — trả lời cert còn hiệu lực không, qua CRL/OCSP), Repository (nơi publish cert/CRL công khai), End Entity (chủ thể được cấp cert).
- **Traffic/data flow:** Entity tạo keypair → gửi CSR cho RA → RA verify danh tính → CA ký → cert được publish/trả về entity → entity dùng cert khi handshake → bên nhận verify chain + check revocation.
- **Dependency:** đồng hồ hệ thống phải đúng (NTP) vì cert có `notBefore`/`notAfter`; trust store của client/OS phải có đúng root CA; HSM (nếu có) để bảo vệ private key của CA.

### **5. How — Cơ chế hoạt động**

Core concepts cần nắm (chi tiết ở các note con):

- **Asymmetric key pair** — nền tảng toán học của toàn bộ PKI. → [[pki--asymmetric-crypto]]
- **X.509 certificate** — định dạng chuẩn buộc public key với danh tính + chữ ký CA. → [[pki--x509-certificate]]
- **CA hierarchy & chain of trust** — tại sao có nhiều tầng CA, root CA vì sao phải offline. → [[pki--ca-hierarchy-trust-chain]]
- **CSR & enrollment** — cách một entity xin cấp cert. → [[pki--csr-enrollment]]
- **Certificate lifecycle** — issue → renew → rekey → expire. → [[pki--certificate-lifecycle]]
- **Revocation (CRL/OCSP)** — cert bị "thu hồi trước hạn" khi nào và bằng cách nào. → [[pki--revocation-crl-ocsp]]
- **Enrollment protocols** — ACME/SCEP/EST/CMP, tự động hoá việc xin cấp cert. → [[pki--enrollment-protocols]]
- **HSM & key protection** — bảo vệ private key của CA, thứ quan trọng nhất toàn hệ thống. → [[pki--hsm-key-protection]]
- **CP/CPS** — chính sách quản trị, ai được cấp cert loại gì, theo quy trình nào. → [[pki--cp-cps-policy]]

**Chain validation flow (khi client nhận 1 cert):**
1. Đọc `Issuer` field trên leaf cert → tìm cert của CA đã ký nó.
2. Verify chữ ký leaf bằng public key của CA đó.
3. Lặp lại lên tầng trên cho tới khi chạm 1 cert có trong local trust store (trust anchor).
4. Check từng cert trong chain: còn hạn (notBefore/notAfter), chưa bị revoke (CRL/OCSP), đúng `KeyUsage`/`BasicConstraints` (vd CA cert phải có `CA:TRUE`).
5. Nếu build chain thất bại hoặc 1 mắt xích fail check → reject.

### **6. Key Config — Cấu hình cần nhớ**

- **Key length / algorithm** — RSA 2048 là baseline tối thiểu hiện nay (RSA 1024 đã unsafe), ECC P-256 phổ biến hơn cho hiệu năng. Root CA nên dùng key mạnh hơn (RSA 4096 hoặc ECC P-384) vì thời gian sống rất dài.
- **Validity period** — Root CA: 15-25 năm. Intermediate CA: 5-10 năm. Leaf/end-entity cert: tối đa 398 ngày theo yêu cầu CA/Browser Forum cho TLS public, nội bộ có thể tự set nhưng ngắn hơn = an toàn hơn (giảm blast radius khi key leak).
- **BasicConstraints `CA:TRUE`** — default nguy hiểm nhất: nếu vô tình set `CA:TRUE` cho 1 leaf cert, cert đó có thể tự ký cert khác → toàn bộ chain trust bị phá vỡ.
- **KeyUsage / ExtendedKeyUsage** — phải giới hạn đúng mục đích (`serverAuth`, `clientAuth`, `codeSigning`...); để trống hoặc quá rộng là lỗi cấu hình phổ biến gây lạm dụng cert.
- **SAN (Subject Alternative Name)** — bắt buộc cho TLS hiện đại, browser không còn tin `CN` field nữa.

### **7. Security Considerations**

- **Attack surface:** private key của CA (đặc biệt Root CA) là tài sản quan trọng nhất — lộ key này = attacker có thể mint cert giả mạo bất kỳ ai. RA/enrollment endpoint là nơi dễ bị tấn công để xin cấp cert giả danh tính người khác.
- **Misconfiguration gây breach:** Root CA online 24/7 thay vì offline; thiếu revocation check ở client (chain hợp lệ nhưng cert đã bị thu hồi vẫn được chấp nhận); path length constraint thiếu → intermediate CA có thể tạo ra vô số sub-CA ngoài kiểm soát.
- **Hardening checklist tối thiểu:**
  - Root CA phải offline / air-gapped, chỉ bật lên khi ký intermediate cert.
  - Private key CA lưu trong HSM, không bao giờ export dạng plaintext.
  - Có quy trình revoke rõ ràng và test thử (đừng để lần đầu thực hành là lúc sự cố thật).
  - Giới hạn thời hạn cert càng ngắn càng tốt nếu có tự động hoá renew.
  - Audit log đầy đủ mọi request issue/revoke.

### **8. Ops Runbook — Production Notes**

- **Health check:** CA service còn issue được cert mới không, VA (OCSP responder) trả response đúng không, CRL có publish đúng chu kỳ không (CRL quá hạn = client có thể fail-open hoặc fail-closed tuỳ config, cả 2 đều nguy hiểm).
- **Log quan trọng:** log issue/revoke certificate (ai, khi nào, cho ai), log truy cập RA/enrollment endpoint, log truy cập HSM.
- **Metric cần alert:** CRL/cert của CA sắp hết hạn, dung lượng CRL tăng bất thường (dấu hiệu revoke hàng loạt), OCSP responder latency/error rate, HSM connectivity.
- **Xem thêm vận hành cụ thể ở** [[EJBCA]] và [[ejbca--ops-runbook]].

### **9. Gotchas & Lessons Learned**

- *(điền dần khi thực tế vận hành EJBCA — vd: renew intermediate cert quên update ở downstream trust store, CRL quá lớn làm client timeout, quên đồng bộ giờ NTP gây fail validate...)*

### **10. Resources**

- RFC 5280 — X.509 PKI Certificate and CRL Profile (chuẩn gốc, tra khi cần biết chính xác 1 field nghĩa là gì).
- CA/Browser Forum Baseline Requirements — quy định thực tế mà các CA công khai phải tuân theo (tham khảo để hiểu best practice dù mình chạy CA nội bộ).
- [[EJBCA]] — công cụ PKI công ty đang dùng, xem chi tiết ở đó.

---
## Glossary — Thuật ngữ hay gặp (tra nhanh)

| Term | Nghĩa ngắn gọn |
|---|---|
| **CA** (Certificate Authority) | Bên ký và phát hành certificate |
| **RA** (Registration Authority) | Bên xác minh danh tính trước khi CA ký (có thể tách riêng hoặc gộp vào CA) |
| **VA** (Validation Authority) | Hệ thống trả lời trạng thái revocation (OCSP responder) |
| **Root CA** | CA gốc, tự ký chính nó (self-signed), là trust anchor |
| **Intermediate/Issuing CA** | CA con, được Root CA ký, dùng để cấp cert hàng ngày |
| **Trust anchor** | Cert được client/OS tin tưởng sẵn (nằm trong trust store) |
| **Chain of trust** | Chuỗi cert từ leaf lên tới trust anchor |
| **CSR** (Certificate Signing Request) | Yêu cầu xin cấp cert, chứa public key + thông tin danh tính, ký bằng private key tương ứng |
| **X.509** | Chuẩn định dạng certificate phổ biến nhất |
| **SAN** (Subject Alternative Name) | Danh sách domain/IP mà cert đại diện |
| **KeyUsage / EKU** | Giới hạn mục đích sử dụng key (server auth, client auth, code signing...) |
| **CRL** (Certificate Revocation List) | Danh sách cert bị thu hồi, do CA publish định kỳ |
| **OCSP** (Online Certificate Status Protocol) | Giao thức hỏi real-time 1 cert còn hiệu lực hay không |
| **OCSP stapling** | Server tự đính kèm OCSP response vào TLS handshake, đỡ client phải tự query |
| **HSM** (Hardware Security Module) | Thiết bị phần cứng lưu và xử lý private key, không cho export ra ngoài |
| **CP/CPS** | Certificate Policy / Certification Practice Statement — tài liệu chính sách quản trị CA |
| **PKCS#10 / PKCS#12** | Chuẩn định dạng CSR (#10) và bundle cert+key có password (#12) |
| **ACME/SCEP/EST/CMP** | Các giao thức tự động hoá enrollment cert (xem [[pki--enrollment-protocols]]) |
| **End Entity** | Chủ thể cuối được cấp cert (server, user, device) — thuật ngữ EJBCA hay dùng |
