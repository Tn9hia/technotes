# Seccomp — Vận hành, Debug & Build Profile
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--profile-json]], [[seccomp--runtimes]], [[seccomp--kubernetes]]
Tags: #ops #debug #tooling
Last updated: 2026-09-06
Verified against: kubernetes.io tutorial, github.com/containers/oci-seccomp-bpf-hook, github.com/kubernetes-sigs/security-profiles-operator (v1.0.0, GA 2026)

## What it does

Quy trình + công cụ thực tế để: (1) biết node đang chặn/allow syscall gì, (2) build 1 fine-grained profile từ hành vi thật của app, (3) quản lý phân phối profile ra nhiều node ở quy mô cluster.

## Why it exists

Viết seccomp profile "từ trí nhớ" (đoán app cần syscall gì) gần như luôn sai — thiếu 1 syscall dùng ở 1 code path hiếm (error handling, signal, GC) sẽ gây crash không thể đoán trước lúc nào xảy ra trong production. Cách đúng luôn là: **quan sát hành vi thật rồi mới allowlist**, không phải suy đoán.

## Quy trình chuẩn build 1 fine-grained profile

```
1. Chạy app với profile "audit" (defaultAction: SCMP_ACT_LOG)
        │  → mọi syscall được LOG, KHÔNG bị chặn, app chạy bình thường
        ▼
2. Chạy app qua đầy đủ kịch bản thực tế (happy path + error path +
   restart + signal/graceful shutdown + healthcheck)
        │
        ▼
3. Thu thập log (xem mục Debug bên dưới), rút ra tập syscall duy nhất
        │
        ▼
4. Viết fine-grained profile: defaultAction=SCMP_ACT_ERRNO + allowlist
   = tập syscall thu được (+ biên độ an toàn nhỏ nếu chưa chắc)
        │
        ▼
5. Chạy lại app với profile "violation" chặn hết → xác nhận app THẬT SỰ
   crash (sanity check là profile có tác dụng, không phải audit sai chỗ)
        │
        ▼
6. Deploy fine-grained profile, canary trên 1 subset trước khi rollout toàn cluster
```

## Debug — công cụ theo lớp

### Lớp kernel/audit — biết chính xác syscall nào bị chặn

- Nếu profile dùng `SCMP_ACT_LOG` hoặc `SECCOMP_FILTER_FLAG_LOG`: kernel ghi event vào `audit.log` (cần `auditd` chạy trên node) với dòng dạng `type=SECCOMP ... syscall=<số> ...`.
- Map số syscall → tên: `ausyscall <số>` hoặc `ausyscall --dump` (in toàn bộ bảng ánh xạ theo arch của máy đang chạy — **quan trọng**: bảng số syscall khác nhau giữa x86_64/arm64/x86 32-bit, luôn map trên đúng arch đang audit).
- Container đang chạy, muốn xem log nhanh không qua auditd: `dmesg | grep -i seccomp` hoặc `journalctl -k | grep -i seccomp` tuỳ distro/logging driver.

### Lớp userspace — trace syscall trực tiếp từ process

- `strace -f -c <command>` chạy app **không có seccomp confine** để liệt kê toàn bộ syscall dùng qua (kèm số lần gọi) — cách đơn giản nhất, không cần quyền đặc biệt ngoài chạy được strace, nhưng chỉ đúng cho lần chạy đó (không cover mọi code path).
- `oci-seccomp-bpf-hook` ([containers/oci-seccomp-bpf-hook](https://github.com/containers/oci-seccomp-bpf-hook)): OCI prestart hook dùng eBPF (bám tracepoint `raw_syscalls:sys_enter`) để tự động sinh 1 file profile JSON allowlist từ hành vi thật của container khi nó chạy — tự động hoá bước 1-4 ở trên. Yêu cầu: chạy với quyền root (`CAP_SYS_ADMIN`), cần bcc toolchain + kernel-headers trên node, **không hoạt động với rootless container**. Có thể truyền 1 profile input làm baseline (`if:`), syscall đã bị chặn trong baseline sẽ **giữ nguyên bị chặn** dù có ghi nhận thêm trong lần trace mới — tránh vô tình nới lỏng profile khi merge.

### Lớp Kubernetes — lỗi thường gặp và cách đọc

```
Warning  Failed  kubelet  Error: setup seccomp: unable to load local profile
"/var/lib/kubelet/seccomp/nginx-1.25.3.json": open ...: no such file or directory
```
→ File chưa được copy ra node đó, hoặc sai path (nhớ path là tương đối trong thư mục gốc `/var/lib/kubelet/seccomp/`, xem [[seccomp--kubernetes]]).

`kubectl describe pod <name>` là lệnh đầu tiên nên chạy — event section luôn có nguyên văn lỗi runtime, đủ để phân biệt "thiếu file" vs "profile chặn syscall app cần" (trường hợp sau app sẽ start được container nhưng crash/exit code bất thường ngay sau đó, không phải `CreateContainerError`).

## Quản lý profile ở quy mô cluster — Security Profiles Operator (SPO)

[`kubernetes-sigs/security-profiles-operator`](https://github.com/kubernetes-sigs/security-profiles-operator) là project chính thức (SIG Security) để quản lý seccomp/SELinux/AppArmor profile như 1 Custom Resource thay vì file thủ công trên từng node.

- Đạt **v1.0.0 (stable API)** vào giữa 2026 sau hơn 4 năm ở `v1beta1` — nếu bạn thấy tài liệu cũ dùng `apiVersion` khác hoặc field như `recorder: logs` (chữ thường), đó là API cũ trước khi graduate lên `v1` (field đổi thành `Logs` viết hoa theo convention K8s enum).
- 2 CRD chính:
  - **`SeccompProfile`**: định nghĩa profile như 1 resource, operator tự distribute file ra đúng path trên mọi node (giải quyết triệt để gotcha "quên copy ra node mới" — operator chạy như DaemonSet, tự đồng bộ).
  - **`ProfileRecording`**: tự động **ghi lại** profile từ hành vi 1 workload đang chạy (dùng eBPF hoặc log-based recorder) — tương đương tự động hoá quy trình audit → fine-grained ở trên, nhưng tích hợp native vào Kubernetes thay vì chạy tool CLI riêng.
- Đây là công cụ nên cân nhắc dùng thật trong vận hành production thay vì tự chế script đồng bộ file qua DaemonSet/ConfigMap — đỡ phải tự xử lý các edge case (node mới, xoá node, version skew).

## Config gotchas

- `strace` một lần chạy **không cover hết code path** — đặc biệt code error-handling, retry logic, hoặc syscall chỉ xảy ra dưới tải cao (vd. epoll variant khác nhau tuỳ số lượng connection) dễ bị bỏ sót nếu chỉ test happy path.
- `oci-seccomp-bpf-hook` cần root trên **node**, không chạy được trong môi trường rootless hay managed K8s không cho SSH vào node (một số managed offering) — cần kiểm tra khả năng truy cập node trước khi lên kế hoạch dùng tool này.
- Audit log qua `auditd` có thể bị **rate-limited** hoặc rotate mất nếu volume log lớn (app gọi syscall rất nhiều lần/giây) — với workload cao tải, audit trong 1 khoảng thời gian ngắn rồi tắt, đừng để audit mode chạy dài hạn trong production (chi phí performance + log).

## Refs
- [Kubernetes — Restrict a Container's Syscalls with seccomp (quy trình audit → fine-grained, chính thức)](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [containers/oci-seccomp-bpf-hook](https://github.com/containers/oci-seccomp-bpf-hook)
- [kubernetes-sigs/security-profiles-operator](https://github.com/kubernetes-sigs/security-profiles-operator)
- [CNCF blog — Security Profiles Operator v1 (2026-06-26)](https://www.cncf.io/blog/2026/06/26/security-profiles-operator-v1-stable-apis-security-hardened-and-shaping-upstream-kubernetes/)
- `man ausyscall`, `man auditd`
