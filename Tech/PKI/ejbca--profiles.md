# EJBCA — Profiles (CA / Certificate / End Entity)
Tier: 2
Parent: [[EJBCA]]
Related: [[pki--x509-certificate]], [[pki--cp-cps-policy]], [[ejbca--end-entities-ra]]
Tags: #ejbca #profiles

## What it does

EJBCA tách cấu hình "1 cert được cấp ra sao" thành **3 tầng profile** kết hợp với nhau, thay vì 1 config duy nhất:

1. **CA** — chính thực thể ký (gắn với Crypto Token/key riêng).
2. **Certificate Profile** — định nghĩa **nội dung/extension** của cert sinh ra (validity, KeyUsage, EKU, có cho phép SAN không, thuật toán chữ ký...).
3. **End Entity Profile** — định nghĩa **quyền hạn khi enrollment**: field nào RA/entity được điền, CA nào được chọn, Certificate Profile nào được áp, token type nào (P12, JKS, chỉ CSR...).

Đây là khái niệm đặc thù và quan trọng nhất cần hiểu để làm việc với EJBCA — hầu hết lỗi cấu hình thực tế nằm ở tầng tương tác giữa 3 profile này.

## Why it exists

Nếu gộp chung 1 config, sẽ không tách được "cert này *có thể chứa gì*" khỏi "*ai* được phép enrollment loại cert đó". Tách 3 tầng cho phép: 1 Certificate Profile (vd "TLS Server Cert 1 năm") được tái sử dụng bởi nhiều End Entity Profile khác nhau (vd "Server nội bộ team A" và "Server nội bộ team B"), mỗi End Entity Profile lại giới hạn quyền khác nhau (team A không được set SAN ngoài domain của họ) — linh hoạt nhưng vẫn kiểm soát chặt.

## How it works (flow/diagram)

```
                    ┌───────────────────┐
                    │   CA (vd:         │  ← ký thực tế, gắn Crypto Token
                    │   "Issuing-CA-01")│
                    └─────────┬─────────┘
                              │ được chọn bởi
                    ┌─────────▼─────────────┐
                    │  End Entity Profile     │  ← "ai được enroll gì"
                    │  (vd "TLS-Server-EEP") │     - CA nào được dùng
                    │                        │     - Certificate Profile
                    │                        │       nào được áp
                    │                        │     - field nào bắt buộc/
                    │                        │       tuỳ chọn/cố định
                    └─────────┬──────────────┘
                              │ áp dụng
                    ┌─────────▼──────────────┐
                    │  Certificate Profile     │  ← "cert chứa gì"
                    │  (vd "TLS-Server-CP")   │     - validity period
                    │                          │     - KeyUsage/EKU
                    │                          │     - cho phép SAN không
                    │                          │     - thuật toán chữ ký
                    └──────────────────────────┘
```

**Quy tắc quan trọng khi xung đột:** End Entity Profile là nơi kiểm soát "cái gì được phép nhập vào", nhưng field cuối cùng xuất hiện trong cert phải nằm trong giới hạn mà Certificate Profile cho phép — nếu End Entity Profile cho phép nhập SAN tuỳ ý nhưng Certificate Profile không bật extension SAN, cert sinh ra sẽ **không có SAN** dù RA đã nhập.

## Config gotchas

- **Certificate Profile mặc định (`ENDUSER`, `SUBCA`, `ROOTCA`)** chỉ nên dùng để clone/tham khảo, không dùng thẳng cho production — chúng có giá trị mặc định chung chung, không khớp CP/CPS nội bộ.
- **End Entity Profile cho phép "Modifiable" quá nhiều field** (vd cho phép RA tự nhập bất kỳ CN/SAN nào) là lỗ hổng thực tế phổ biến nhất — nên set giá trị cố định hoặc giới hạn pattern bất cứ khi nào có thể, đặc biệt với SAN.
- **1 End Entity Profile trỏ nhầm Certificate Profile** (vd EEP dành cho server lại trỏ Certificate Profile có EKU `clientAuth` thay vì `serverAuth`) — lỗi im lặng, chỉ phát hiện khi client thực tế bị TLS handshake reject.
- **Thay đổi Certificate Profile của 1 profile đang dùng production** không ảnh hưởng cert đã cấp trước đó (cert cũ giữ nguyên nội dung) nhưng ảnh hưởng mọi cert cấp **sau** thời điểm sửa — cẩn thận khi sửa profile đang active, nên test trên profile clone trước.

## Security notes

- Coi End Entity Profile là lớp "authorization" thực sự của hệ thống — review định kỳ xem field nào đang để mở quá rộng, đặc biệt các field liên quan SAN, KeyUsage, EKU, validity.
- Certificate Profile validity dài bất thường so với chính sách chung (vd 1 profile lỡ để 10 năm cho leaf cert) là dấu hiệu cấu hình sai cần rà soát.

## Refs

- [[pki--x509-certificate]] — ý nghĩa từng field/extension mà Certificate Profile điều khiển.
- [[pki--cp-cps-policy]] — Certificate Profile nên map 1-1 với 1 CP nội bộ đã định nghĩa.
- [[ejbca--end-entities-ra]] — cách End Entity Profile được dùng thực tế khi tạo End Entity.
