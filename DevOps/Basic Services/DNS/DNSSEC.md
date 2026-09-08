---
title: DNSSEC
tags:
  - dns
  - dnssec
  - security
  - deep-dive
date: 2026-04-26
---

# DNSSEC — DNS Security Extensions

## Vấn đề DNSSEC giải quyết

DNS thuần không có authentication. Attacker có thể:
- **Cache poisoning (Kaminsky attack)**: inject fake records vào cache của resolver → client bị redirect đến server giả
- **MITM**: intercept DNS response và sửa IP trả về

DNSSEC thêm **chữ ký số** vào DNS response. Client verify chữ ký → biết response đến từ đúng authoritative server và không bị sửa đổi.

> DNSSEC **không encrypt** DNS query — vẫn plain text. Chỉ authenticate. Để encrypt dùng DoT/DoH.

---

## Chain of Trust

```
Root Zone (.)
  │  ký bởi IANA Root DNSKEY
  │  DS record trỏ đến .com
  ▼
.com TLD
  │  ký bởi Verisign
  │  DS record trỏ đến example.com
  ▼
example.com (zone của mình)
  │  ký bởi KSK của example.com
  │  ZSK ký tất cả records
  ▼
app.example.com  A  1.2.3.4
  └── RRSIG record đính kèm = chữ ký của ZSK
```

**Trust anchor**: Client/Resolver chỉ cần trust **1 root public key** (KSK của root zone) → có thể verify toàn bộ chain từ root xuống.

---

## Các record type mới trong DNSSEC

| Record | Vai trò |
|--------|---------|
| **DNSKEY** | Public key của zone — dùng để verify chữ ký |
| **RRSIG** | Chữ ký số của một RRset (Resource Record Set) |
| **DS** | Digest (hash) của KSK con — lưu ở zone cha để link chain |
| **NSEC / NSEC3** | Chứng minh record **không tồn tại** (authenticated denial) |

---

## KSK vs ZSK — Hai loại key

| | KSK (Key Signing Key) | ZSK (Zone Signing Key) |
|---|---|---|
| Ký cái gì | Ký DNSKEY RRset (ký ZSK) | Ký tất cả records khác (A, MX, ...) |
| Kích thước | Lớn hơn (2048-bit RSA hoặc P-256) | Nhỏ hơn (1024-bit RSA hoặc P-256) |
| Thay đổi | Hiếm — phải update DS ở zone cha | Thường xuyên hơn — không cần báo cha |
| DS record | Hash của KSK được lưu ở zone cha | Không có DS |

**Tại sao cần 2 key?**
- Thay ZSK thường xuyên → security tốt hơn
- Thay KSK phải notify zone cha (registrar) → chậm, tốn công
- Tách 2 key → có thể rollover ZSK mà không động đến chain cha

---

## RRSIG — Chữ ký chi tiết

Mỗi RRset có 1 RRSIG đính kèm:

```
app.example.com.  300  IN  A      1.2.3.4
app.example.com.  300  IN  RRSIG  A 13 3 300 (
                    20261231000000   ← signature expiry
                    20260101000000   ← signature inception
                    12345            ← key tag (link đến DNSKEY)
                    example.com.     ← signer name
                    Base64EncodedSignature== )
```

- **Algorithm 13** = ECDSA P-256 SHA-256 (khuyến nghị hiện tại)
- **Algorithm 8** = RSA SHA-256 (legacy nhưng vẫn dùng rộng rãi)
- Signature có **expiry date** → phải re-sign định kỳ (thường 30 ngày)

---

## NSEC vs NSEC3 — Authenticated Denial of Existence

**Vấn đề:** Nếu query record không tồn tại (`NXDOMAIN`), làm sao client biết đây là NXDOMAIN thật, không phải bị giả?

**NSEC:** Trả về "record tiếp theo trong thứ tự alphabetical" → chứng minh không có gì giữa 2 record đó.

```
; Zone có: app, db, www
app.example.com.  NSEC  db.example.com. A RRSIG NSEC
; Query "cache.example.com" → nằm giữa app và db → không tồn tại
```

**Nhược điểm NSEC:** **Zone enumeration** — attacker đi qua từng NSEC chain → biết toàn bộ records trong zone.

**NSEC3:** Hash tên trước khi liệt kê → không đọc được tên zone trực tiếp.

```
; NSEC3 dùng hash
<hash-of-app>.example.com.  NSEC3  1 0 10 AABBCC <hash-of-db> A RRSIG
```

**Khuyến nghị:** Dùng **NSEC3** cho public zone. NSEC đủ cho internal zone.

---

## Setup DNSSEC với PowerDNS Authoritative

### 1. Enable DNSSEC trong pdns.conf

```ini
# /etc/powerdns/pdns.conf
default-ksk-algorithm=ecdsa256
default-zsk-algorithm=ecdsa256
```

### 2. Secure zone

```bash
# Enable DNSSEC cho zone (tạo keys tự động)
pdnsutil secure-zone example.com

# Xem keys đã tạo
pdnsutil show-zone example.com
# Output:
# Zone is DNSSEC secured
# KSK: ID=1, Algorithm=ecdsa256, Active=true
# ZSK: ID=2, Algorithm=ecdsa256, Active=true
```

### 3. Lấy DS record để submit lên registrar

```bash
# Lấy DS record (gửi cho registrar/zone cha)
pdnsutil show-zone example.com | grep "^DS"
# DS record: example.com. 0 IN DS 12345 13 2 <hash>

# Hoặc format đẹp hơn
pdnsutil export-zone-ds example.com
```

### 4. Rectify zone (cần thiết sau khi thay đổi records)

```bash
# Rebuild NSEC/NSEC3 chain sau khi add/delete records
pdnsutil rectify-zone example.com

# Tự động rectify khi dùng API — set trong pdns.conf:
api-rectify=yes
```

### 5. Verify

```bash
# Check DNSSEC setup
pdnsutil check-zone example.com

# Test từ ngoài
dig +dnssec example.com A @127.0.0.1
# Phải thấy RRSIG record trong answer section

# Verify chain of trust
dig +trace +dnssec example.com
```

---

## ZSK Rollover — Thay key định kỳ

```bash
# Tạo ZSK mới (chưa active)
pdnsutil add-zone-key example.com zsk inactive ecdsa256

# Xem keys
pdnsutil show-zone example.com
# ZSK ID=2 active=true
# ZSK ID=3 active=false  ← mới tạo

# Pre-publish: publish key mới nhưng chưa sign (chờ TTL DNSKEY propagate)
pdnsutil activate-zone-key example.com 3

# Sau khi TTL DNSKEY hết hạn (ít nhất 1 TTL), deactivate key cũ
pdnsutil deactivate-zone-key example.com 2

# Remove key cũ sau khi đã propagate xong
pdnsutil remove-zone-key example.com 2
```

---

## DNSSEC Validation ở Recursor

```ini
# /etc/pdns-recursor/recursor.conf

# validate = bật validation, trả SERVFAIL nếu fail
# log-fail = chỉ log, không trả SERVFAIL
# off = tắt hoàn toàn
dnssec=validate

# Trust anchor cho root (có sẵn trong PowerDNS)
# Tự động load từ /usr/share/dns/root.key
```

**Khi DNSSEC validation fail:**
```bash
# SERVFAIL với AD=0 → validation fail
dig +dnssec example.com @127.0.0.1
# status: SERVFAIL

# Debug
dig +cd example.com @127.0.0.1   # +cd = checking disabled, bypass validation
# Nếu +cd trả về kết quả → validation fail
# Nếu +cd cũng fail → server issue

# Log recursor
journalctl -u pdns-recursor | grep "validation\|DNSSEC\|bogus"
```

---

## DNSSEC cho internal zone — có nên dùng không?

**Câu trả lời: thường là không cần.** Lý do:

- Internal zone không có zone cha public → không thể tạo chain of trust hoàn chỉnh
- Phải manually trust root của internal zone → quản lý phức tạp
- Lợi ích chính của DNSSEC (chống cache poisoning từ internet) ít apply cho internal

**Khi nên dùng DNSSEC cho internal:**
- Môi trường zero-trust, muốn verify toàn bộ DNS response
- Compliance requirement
- Multi-tenant infra, DNS server không được trust hoàn toàn

**Nếu muốn dùng:** Config recursor với trust anchor của internal KSK:

```ini
# recursor.conf
lua-config-file=/etc/pdns-recursor/config.lua
```

```lua
-- config.lua
addTA("internal.com.", "13 2 <DS hash của internal KSK>")
```

---

## Gotchas

- **Quên rectify sau khi thêm record** → NSEC chain sai → NXDOMAIN cho record tồn tại
- **RRSIG expiry**: PowerDNS tự re-sign định kỳ nếu chạy liên tục. Nếu server bị tắt lâu → RRSIG expire → validation fail toàn bộ zone
- **Clock skew**: RRSIG có inception/expiry timestamp → server và resolver phải đồng bộ giờ (NTP!) — lệch >5 phút có thể fail validation
- **`forward-zones` + DNSSEC**: recursor khi dùng `forward-zones` không thể validate DNSSEC cho zone đó (không có chain từ root). Dùng `forward-zones-recurse` thay thế nếu cần validation
- **Algorithm support**: client/resolver cũ không support ECDSA (algo 13/14) → dùng RSA (algo 8) cho compatibility tốt hơn

---

## Tóm tắt nhanh

```
Secure zone:    pdnsutil secure-zone example.com
Xem keys:       pdnsutil show-zone example.com
Lấy DS:         pdnsutil export-zone-ds example.com
Rectify:        pdnsutil rectify-zone example.com
Check:          pdnsutil check-zone example.com
Test:           dig +dnssec example.com @<server>
```
