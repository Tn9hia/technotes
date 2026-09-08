# Falco — Drivers (Kernel Module / eBPF / Modern eBPF)
Tier: 2
Parent: [[Falco]]
Related: [[falco--performance-tuning-drops]], [[falco--kubernetes-deployment]]
Tags: #falco #ebpf #kernel #driver

## What it does

Driver là component chạy trong kernel (hoặc gần kernel nhất) làm nhiệm vụ **bắt syscall** và đẩy event lên userspace cho Falco xử lý. Không có driver thì Falco không thấy được gì cả — mọi rule engine phía trên đều vô nghĩa nếu driver không load được.

3 loại driver, chọn 1 qua config `engine.kind`:

| Driver | Cơ chế | Cần gì | Ghi chú |
|---|---|---|---|
| **Kernel module** (`kmod`) | `.ko` module load vào kernel | Build theo đúng kernel version, hoặc dùng driver kit build sẵn | Cũ nhất, mature nhất, nhưng attack surface kernel thật sự |
| **Legacy eBPF** (`ebpf`) | eBPF program, không cần build theo kernel version nhưng cần compile-time info | Kernel headers hoặc BTF | An toàn hơn kmod nhưng vẫn cần driver builder cho kernel lạ |
| **Modern eBPF** (`modern_ebpf`) | eBPF CO-RE (Compile Once Run Everywhere), **embedded sẵn trong Falco binary** | Kernel ≥ 5.8 + BTF support | **Default từ Falco 0.35+**, không cần build/download gì thêm |

## Why it exists

Falco cần visibility ở tầng syscall nhưng kernel mỗi distro/version một khác. Ba driver tồn tại vì trade-off giữa **compatibility** (kmod build theo từng kernel = chạy được cả kernel cũ không có BTF) và **an toàn + zero-maintenance** (modern eBPF không cần build lại nhưng đòi hỏi kernel mới hơn). Modern eBPF ra đời chính là để giải quyết nỗi đau lớn nhất của kmod/legacy eBPF: **driver không tương thích sau khi node kernel được patch**.

## How it works (flow/diagram)

```
Falco start
    │
    ▼
engine.kind = ? ──kmod──▶ insmod driver.ko (khớp đúng `uname -r`)
    │                         │
    │                         └─ nếu kernel version không khớp → load FAIL
    │
    ├──ebpf────▶ load eBPF object file (cần .o build sẵn hoặc build tại chỗ
    │                nhờ kernel headers) → verifier check → attach tracepoint
    │
    └──modern_ebpf (default)──▶ eBPF CO-RE object nhúng sẵn trong binary Falco
                                    → dùng BTF của kernel để tự resolve struct offset
                                    → không cần build lại theo kernel version
```

Modern eBPF dùng BTF (BPF Type Format) — thông tin kiểu dữ liệu kernel được nhúng sẵn (`/sys/kernel/btf/vmlinux`) để chương trình eBPF tự "hiểu" struct layout của kernel đang chạy, thay vì phải compile riêng cho từng kernel như legacy eBPF/kmod.

## Config gotchas

- **Kernel upgrade là nguyên nhân #1 khiến driver chết** với kmod/legacy eBPF — node patch kernel tự động (managed K8s node pool rotation, unattended-upgrades) mà driver không rebuild kịp → Falco pod CrashLoopBackOff hoặc chạy nhưng **không bắt được event nào** (im lặng, dễ bị bỏ sót nếu không alert riêng cho driver load error).
- **Modern eBPF không miễn nhiễm 100%**: một số custom kernel (đặc biệt distro tự build, hoặc kernel cũ patch backport) **thiếu BTF info** → modern eBPF vẫn fail load. Kiểm tra bằng `ls /sys/kernel/btf/vmlinux` trên node trước khi chọn driver.
- Driver builder (driverkit) cần internet access để tải/build driver cho kmod/legacy eBPF nếu không có sẵn trong driver registry — trong môi trường air-gapped phải pre-build và pre-load driver thủ công.
- Container chạy driver cần **privileged** hoặc capability cụ thể (`SYS_MODULE` cho kmod, `SYS_ADMIN`/`SYS_RESOURCE`/`SYS_PTRACE`/`NET_ADMIN`/`BPF` cho eBPF tùy kernel version) — set thiếu capability sẽ fail load với lỗi permission denied, dễ nhầm tưởng là driver không tương thích.
- Khi troubleshoot, log lỗi `Unable to load the driver` không luôn nói rõ nguyên nhân (kernel version mismatch vs thiếu capability vs thiếu BTF) — phải check thêm `dmesg`/kernel log trên node.

## Security notes

- **Kernel module = attack surface kernel thật sự.** Module có bug (dù là của Falco) chạy trong kernel space, exploit được thì có thể full compromise host, không chỉ crash. Cần ký số module (`modsign`) nếu cluster enforce Secure Boot / kernel lockdown.
- **eBPF (cả legacy và modern) an toàn hơn về nguyên tắc** vì mọi program phải pass qua **eBPF verifier** của kernel trước khi được load — verifier chặn được phần lớn class lỗi (out-of-bounds, infinite loop) mà kernel module không có cơ chế tương đương.
- Ưu tiên **modern eBPF** trừ khi có lý do kỹ thuật cụ thể (kernel quá cũ không đủ BTF/version) buộc phải dùng kmod.
- Trong môi trường có kernel lockdown mode (Secure Boot bật) — kmod thường **bị chặn hoàn toàn** trừ khi ký số hợp lệ, đây là lý do thực tế bắt buộc chuyển sang eBPF chứ không chỉ là best practice.

## Refs

- https://falco.org/docs/concepts/event-sources/kernel/
- https://falco.org/blog/falco-modern-bpf-0-35-0/
- https://github.com/falcosecurity/driverkit (build driver cho kernel cụ thể)
