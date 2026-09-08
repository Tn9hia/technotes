## Tại sao HTTPS khó hơn HTTP?

```
HTTP (transparent proxy — dễ):
Client ──GET http://example.com──► Squid ──► example.com
Squid đọc được hết, modify được, cache được

HTTPS (vấn đề):
Client ──CONNECT example.com:443──► Squid ──► ???
Squid chỉ thấy "tunnel đến example.com:443"
Nội dung bên trong = encrypted = Squid mù tịt
```

Để inspect HTTPS, Squid phải làm **MitM hợp lệ** — đây là SSL Bump.

---

## SSL Bump hoạt động như thế nào?

```
WITHOUT SSL Bump:
Client ──TLS──► Squid ══tunnel══► Server
         (Squid không giải mã được)

WITH SSL Bump:
Client ◄──TLS──► Squid ◄──TLS──► Server
       (fake cert)    (real cert)
          ▲
          └── Squid tự ký cert giả cho client
              dùng internal CA mà client đã trust
```

```
Step by step:
1. Client → CONNECT google.com:443 → Squid
2. Squid → CONNECT google.com:443 → Google (real)
3. Google → trả TLS cert thật → Squid
4. Squid → tạo fake cert cho google.com, ký bằng internal CA
5. Squid → trả fake cert → Client
6. Client trust fake cert (vì đã import internal CA)
7. Squid giờ decrypt được cả 2 chiều → inspect → re-encrypt
```

---

## 3 Mode SSL Bump — Quan trọng nhất

```
┌─────────┬────────────────────────────────────────┐
│  peek   │ Squid xem SNI/CN của cert               │
│         │ KHÔNG decrypt content                   │
│         │ Dùng để quyết định bump hay splice      │
├─────────┼────────────────────────────────────────┤
│  bump   │ Full MitM — decrypt, inspect, re-encrypt│
│         │ Squid thấy hết content                  │
├─────────┼────────────────────────────────────────┤
│  splice │ Transparent tunnel — KHÔNG decrypt      │
│         │ Dùng cho banking, gov, certificate pin  │
└─────────┴────────────────────────────────────────┘
```

## Flow Tổng Quan

```
Client → CONNECT google.com:443 → Squid:3129
              │
         [peek step1]
         Squid xem SNI = "google.com"
              │
         Có trong no_bump_domains?
         ┌───┴───┐
        YES      NO
         │        │
      [splice]  [bump]
      tunnel    MitM
      qua thẳng decrypt+inspect
```

## Vấn đề Thường Gặp

```
❌ Certificate Pinning
   App mobile, Chrome (HPKP) pin cert cụ thể
   → SSL Bump sẽ bị reject
   → Phải splice các domain này

❌ TLS 1.3 + ESNI/ECH
   SNI bị encrypt → Squid không peek được hostname
   → Khó intercept, cần splice hoặc block

❌ OCSP Stapling
   Client check cert revocation
   → Fake cert của Squid không có OCSP
   → Một số app từ chối
```

## ACL Nên Splice (Không Bump)

```squid
acl no_bump ssl::server_name .apple.com        # certificate pinning
acl no_bump ssl::server_name .google.com       # HPKP
acl no_bump ssl::server_name .vietcombank.com.vn
acl no_bump ssl::server_name .techcombank.com.vn
acl no_bump ssl::server_name .momo.vn

ssl_bump splice no_bump
ssl_bump bump all
```

