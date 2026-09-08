# Falco — Falcoctl & Plugin Architecture
Tier: 2
Parent: [[Falco]]
Related: [[falco--rules-syntax]], [[falco--kubernetes-deployment]]
Tags: #falco #plugins #falcoctl #oci

## What it does

**Falcoctl** là công cụ quản lý artifact (rule + plugin) cho Falco, tương tự cách `helm` quản lý chart hoặc `docker` quản lý image — tải rule/plugin đóng gói dưới dạng **OCI artifact** từ registry (mặc định GHCR/GitHub Packages) về node, và tự cập nhật định kỳ.

**Plugin framework** là cơ chế mở rộng Falco ra ngoài phạm vi syscall Linux thuần túy. Có 2 loại plugin:
- **Source plugin**: sinh ra một nguồn event hoàn toàn mới (không phải syscall) — vd `k8saudit` (K8s Audit Log), `cloudtrail` (AWS CloudTrail), `okta` (Okta system log).
- **Extractor plugin**: không sinh event mới, chỉ thêm field để rule có thể filter trên event đã có sẵn.

## Why it exists

Nếu Falco chỉ hard-code hỗ trợ syscall, mỗi khi cần thêm 1 nguồn dữ liệu mới (audit log, cloud API log) sẽ phải sửa core code của Falco. Kiến trúc plugin (từ Falco 0.36 trở đi theo hướng "plugin-first") tách nguồn event ra khỏi core engine — ai cũng có thể viết plugin riêng để đưa dữ liệu bất kỳ vào rule engine chung của Falco, không cần đợi upstream merge code.

Falcoctl tồn tại vì quản lý version rule + plugin thủ công (tải file, copy vào đúng path, restart) không scale khi có nhiều node và rule cần update thường xuyên để theo kịp threat mới — cần 1 cơ chế kiểu package manager với ký số (cosign) để đảm bảo integrity.

## How it works (flow/diagram)

```
falcoctl index add <name> <url>      # thêm nguồn index (danh sách artifact khả dụng)
falcoctl artifact install <ref>      # tải rule/plugin về, verify chữ ký (cosign)
falcoctl artifact follow <ref>       # tự động theo dõi + cập nhật khi có version mới
                    │
                    ▼
        OCI Registry (vd ghcr.io/falcosecurity/plugins/...)
                    │
                    ▼
     Rule/Plugin file được đặt vào path Falco đọc (rules_files / plugins config)
                    │
                    ▼
        Falco load lúc start (hoặc reload nếu hỗ trợ hot-reload)
```

Plugin cần khai báo trong `falco.yaml`:
```yaml
plugins:
  - name: k8saudit
    library_path: libk8saudit.so
    init_config: ""
    open_params: "http://:9765/k8s-audit"

load_plugins: [k8saudit]
```

## Config gotchas

- **`falcoctl artifact follow` = auto-update rule/plugin** — tiện cho việc luôn có rule mới nhất chống threat mới, nhưng cũng là **rủi ro thay đổi ngoài ý muốn**: rule tự động update có thể đổi behavior detection (tighter hoặc looser) mà không qua review nội bộ. Cân nhắc pin version cụ thể trong môi trường production có compliance yêu cầu change control chặt.
- Plugin và rule phải khớp **`required_plugin_versions`** khai báo trong rule file — rule viết cho version plugin mới hơn plugin đang cài sẽ fail load lúc start (không silent skip).
- Air-gapped environment: `falcoctl` cần internet để reach OCI registry mặc định — phải tự host private registry/mirror và trỏ `index add` vào đó.
- Signature verification (cosign) mặc định bật cho artifact chính thức của falcosecurity — nếu build/host plugin riêng mà không ký, cần disable verify hoặc tự ký, nếu không falcoctl sẽ từ chối cài.
- Nhầm lẫn phổ biến: cài `k8saudit` **plugin** không tự động nghĩa là K8s Audit Log đã chảy vào Falco — vẫn cần cấu hình control plane gửi audit event tới plugin (xem [[falco--kubernetes-deployment]]).

## Security notes

- Plugin chạy trong cùng process với Falco (`.so`/`.dll` load native) — plugin bên thứ ba không đáng tin cậy tương đương chạy code tùy ý trong context privileged của Falco. Chỉ dùng plugin từ nguồn đã audit/ký chính thức, hoặc tự review source nếu tự build.
- Cosign signature verification là lớp bảo vệ chính chống supply-chain attack (artifact bị thay thế trong registry) — không nên tắt verify trừ khi có lý do rất cụ thể (air-gapped + đã tự kiểm soát nguồn artifact).
- `open_params` của source plugin (vd endpoint webhook k8saudit) là attack surface network — cần giới hạn network policy chỉ cho phép API server (hoặc log forwarder) gọi tới, không mở rộng ra toàn cluster.

## Refs

- https://github.com/falcosecurity/falcoctl
- https://falco.org/blog/falcoctl-install-manage-rules-plugins/
- https://github.com/falcosecurity/plugins (danh sách plugin chính thức)
- https://falco.org/blog/sign-verify-plugins-rules/
