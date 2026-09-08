# Seccomp — OCI Profile JSON Syntax
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--kernel-bpf]], [[seccomp--runtimes]], [[seccomp--ops-debug]]
Tags: #config #oci
Last updated: 2026-09-06
Verified against: kubernetes.io tutorial "Restrict a Container's Syscalls with seccomp", OCI runtime-spec (config-linux.md#seccomp)

## What it does

Đây là format JSON mà bạn thực sự viết tay khi làm `Localhost` profile. Runc/crun dùng libseccomp để parse JSON này và compile ra BPF program (chi tiết BPF → [[seccomp--kernel-bpf]]).

## Why it exists

Viết BPF tay là bất khả thi cho người thường (phải tự tính offset struct, xử lý 64-bit arg, tính syscall number theo arch...). JSON schema này là lớp trừu tượng chuẩn hoá do OCI runtime-spec định nghĩa, được Docker/containerd/CRI-O/Kubernetes dùng chung.

## How it works (cấu trúc)

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": ["read", "write", "close", "futex", "epoll_wait"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["mount", "umount2", "unshare", "setns"],
      "action": "SCMP_ACT_ALLOW",
      "includes": { "caps": ["CAP_SYS_ADMIN"] }
    },
    {
      "names": ["personality"],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        { "index": 0, "value": 0, "op": "SCMP_CMP_EQ" }
      ]
    }
  ]
}
```

### Các field cấp cao nhất

| Field | Bắt buộc | Ý nghĩa |
|---|---|---|
| `defaultAction` | Có | Action áp dụng cho mọi syscall **không** match rule nào trong `syscalls[]`. Gần như luôn nên là `SCMP_ACT_ERRNO` (deny-by-default). Nếu để `SCMP_ACT_ALLOW`, toàn bộ profile trở thành denylist thay vì allowlist — **đảo ngược hoàn toàn mục đích**. |
| `defaultErrnoRet` | Không | Mã errno trả về khi `defaultAction: SCMP_ACT_ERRNO` (mặc định `EPERM`=1 nếu không set). |
| `architectures` / `archMap` | Không | Kiến trúc filter áp dụng — thiếu field này trên hệ multi-arch có thể để lọt syscall qua ABI khác (xem gotcha ở [[seccomp--kernel-bpf]]). |
| `syscalls[]` | Có | Danh sách rule, mỗi rule có `names[]`, `action`, và tuỳ chọn `args[]` để lọc theo giá trị argument, `includes`/`excludes` để rule chỉ áp dụng có điều kiện (theo capability, theo arch, theo min kernel version). |

### Action strings (dùng trong JSON, map 1-1 với `SECCOMP_RET_*` ở tầng kernel)

`SCMP_ACT_KILL_PROCESS`, `SCMP_ACT_KILL` / `SCMP_ACT_KILL_THREAD`, `SCMP_ACT_TRAP`, `SCMP_ACT_ERRNO`, `SCMP_ACT_TRACE`, `SCMP_ACT_LOG`, `SCMP_ACT_ALLOW`, `SCMP_ACT_NOTIFY`.

**Lưu ý**: không phải mọi action đều được mọi runtime hỗ trợ như nhau (`SCMP_ACT_NOTIFY` phụ thuộc runc/crun version + kernel ≥5.0 — xem [[seccomp--kernel-bpf]]).

### Argument filtering (`args[]`)

Cho phép rule chỉ match khi 1 argument cụ thể của syscall thoả điều kiện — hữu ích để allow syscall "nguy hiểm" nhưng chỉ với 1 số giá trị an toàn (ví dụ ví dụ `personality` ở trên: chỉ cho giá trị `0` tức `PER_LINUX`, chặn các personality khác có thể dùng để bypass ASLR).

```json
{ "index": 0, "value": 0, "valueTwo": 0, "op": "SCMP_CMP_EQ" }
```
`op` có thể là `SCMP_CMP_EQ`, `NE`, `LT`, `LE`, `GT`, `GE`, `MASKED_EQ`.

### 3 profile mẫu chính thức của Kubernetes (dùng để học workflow build profile)

| File | `defaultAction` | Dùng để |
|---|---|---|
| `audit.json` | `SCMP_ACT_LOG` | Log **mọi** syscall, không chặn gì — chạy trước để quan sát app cần syscall gì |
| `violation.json` | `SCMP_ACT_ERRNO` (không có `syscalls[]`) | Chặn tuyệt đối mọi syscall — dùng để demo/test |
| `fine-grained.json` | `SCMP_ACT_ERRNO` + allowlist cụ thể | Ví dụ thực tế 1 allowlist tối thiểu cho 1 network service (~50 syscall) |

## Config gotchas

- Quên `defaultAction: SCMP_ACT_ERRNO` → allowlist trở thành vô nghĩa (đã nói ở trên) — đây là lỗi review-code hay gặp nhất khi merge PR sửa seccomp profile.
- `includes.caps` chỉ có tác dụng nếu runtime bạn dùng hỗ trợ evaluate theo capability tại thời điểm build filter (libseccomp làm việc này tại "compile time" dựa trên capability set của container, không phải runtime check) — nghĩa là đổi capability của container sau khi tạo **không** tự động đổi seccomp filter.
- Không có "diff" hay "merge" chuẩn giữa 2 file profile — nếu muốn kế thừa từ `RuntimeDefault` rồi thêm/bớt vài syscall, phải copy toàn bộ nội dung `RuntimeDefault` hiện tại của runtime đang chạy ra rồi sửa tay (xem [[seccomp--runtimes]] cách lấy nội dung đó), hoặc dùng Security Profiles Operator để merge ([[seccomp--ops-debug]]).

## Security notes

- Argument-based filtering (`args[]`) vẫn có giới hạn TOCTOU nếu argument là con trỏ (không filter được nội dung trỏ tới, chỉ filter được giá trị con trỏ/số nguyên trực tiếp) — không dùng để filter theo nội dung path/buffer.
- Một profile allowlist "quá khít" theo 1 lần chạy audit không đảm bảo đúng cho mọi code path (error handling path, signal handler, GC của runtime...) — luôn test qua nhiều kịch bản trước khi enforce production.

## Refs
- [OCI runtime-spec — config-linux.md#seccomp](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#seccomp)
- [Kubernetes seccomp tutorial — profile examples](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [libseccomp](https://github.com/seccomp/libseccomp)
