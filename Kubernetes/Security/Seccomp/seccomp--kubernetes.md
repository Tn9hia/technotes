# Seccomp — Kubernetes API & Pod Security Standards
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--profile-json]], [[seccomp--runtimes]], [[seccomp--cks-exam]]
Tags: #kubernetes #api
Last updated: 2026-09-06
Verified against: kubernetes.io/docs/reference/node/seccomp/ và /docs/tutorials/security/seccomp/ (bản v1.34–v1.37), KEP-2413 (kubernetes/enhancements)

## What it does

Tầng API của Kubernetes để khai báo seccomp profile cho Pod/Container, cộng với cơ chế đổi default toàn cluster (`SeccompDefault`) và ràng buộc trong Pod Security Standards.

## Why it exists

Không có tầng này, seccomp là chuyện của OCI runtime config — engineer phải tự viết `config.json` tay cho từng container, không thể quản lý qua Pod spec, không thể set policy toàn cluster, không tích hợp được với Pod Security Admission.

## How it works (API fields)

### `seccompProfile.type` — 3 giá trị

| Type | Ý nghĩa |
|---|---|
| `Unconfined` | Không áp seccomp — full syscall access |
| `RuntimeDefault` | Dùng profile mặc định của container runtime (containerd/CRI-O/...) — nội dung khác nhau giữa runtime, xem [[seccomp--runtimes]] |
| `Localhost` | Dùng file JSON custom, cần thêm field `localhostProfile` |

### 4 cấp khai báo, container-level override pod-level

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:                 # pod-level: baseline cho mọi container không tự khai
    seccompProfile:
      type: RuntimeDefault
  initContainers:
  - name: init
    securityContext:
      seccompProfile: {type: RuntimeDefault}
  containers:
  - name: app
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/my-app.json   # đường dẫn tương đối trong /var/lib/kubelet/seccomp/
  ephemeralContainers:
  - name: debug
    securityContext:
      seccompProfile: {type: RuntimeDefault}
```

Container không tự khai `seccompProfile` → kế thừa giá trị pod-level. **Ngoại lệ duy nhất**: container `privileged: true` luôn chạy `Unconfined` bất kể khai gì (kể cả pod-level) — runtime bỏ qua field seccomp khi privileged.

### `localhostProfile` — path convention

- Path là **tương đối** so với thư mục gốc `/var/lib/kubelet/seccomp/` trên **từng node** (không phải trên control plane, không phải trong image).
- Runtime kiểm tra sự tồn tại file **tại thời điểm tạo container**. Thiếu file → event `CreateContainerError`, pod kẹt ở trạng thái không chạy được (không phải `Pending` do scheduling, mà lỗi runtime sau khi đã schedule).
- Từ K8s v1.29 (CRI-O), có hướng đi thay thế: phân phối profile qua OCI registry (OCI artifact) thay vì đặt file thủ công trên từng node → xem [[seccomp--ops-debug]].

### `SeccompDefault` — đổi default toàn node

| Version | Trạng thái |
|---|---|
| v1.22 | Alpha (feature gate `SeccompDefault`, tắt mặc định) |
| v1.25 | Beta |
| v1.27 | GA (Stable) |

Cách bật: kubelet flag `--seccomp-default=true`, hoặc trong `KubeletConfiguration`:
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
seccompDefault: true
```
Khi bật: pod/container **không khai** `seccompProfile` sẽ chạy `RuntimeDefault` thay vì `Unconfined`. Đây là setting cấp **node**, áp dụng cho toàn bộ pod chạy trên node đó — không phải cấp cluster tự động đồng bộ, phải set trên từng node (qua node config / machine config nếu dùng managed K8s).

> ⚠️ Nếu cluster của bạn không set flag này và không phải bản K8s ≥1.27 với default bật sẵn, hành vi mặc định lịch sử vẫn là **`Unconfined`** cho pod không khai gì — đây là điều đáng kiểm tra đầu tiên khi audit 1 cluster có sẵn (đúng bối cảnh "nhận bàn giao hệ thống" của bạn).

### Pod Security Standards — mức `restricted`

Mức `restricted` yêu cầu control `seccompProfile`: field `type` ở pod-level hoặc container-level phải là `RuntimeDefault` hoặc `Localhost` — **không được `Unconfined`**, và không được để hoàn toàn trống ở cả 2 cấp (phải có ít nhất pod-level set, container kế thừa được). Yêu cầu này là 1 phần của Pod Security Standards, ổn định từ v1.25 (thời điểm Pod Security Admission GA thay thế PodSecurityPolicy đã bị xoá).

Mức `baseline` **không** ràng buộc gì về seccomp (baseline chủ yếu chặn các thứ nguy hiểm rõ ràng như privileged, hostNetwork...).

## Config gotchas

- Set `Localhost` + `localhostProfile` nhưng quên copy file ra node mới (đặc biệt với cluster autoscaler / node pool mới) → `CreateContainerError` chỉ xảy ra trên node mới, rất khó reproduce nếu bạn test trên node cũ đã có file.
- `SeccompDefault=true` là **node-level flag**, không phải Admission Controller — nếu chỉ set trên 1 số node (vd. quên set khi thêm node pool mới bằng tool riêng), hành vi sẽ không đồng nhất giữa các node dù cùng 1 pod spec.
- PSA (Pod Security Admission) chỉ **enforce** field `seccompProfile.type` đúng giá trị cho phép, nó **không** tự động điền `RuntimeDefault` cho bạn — nếu muốn auto-default, phải dùng `SeccompDefault` kubelet flag hoặc 1 mutating webhook/Security Profiles Operator riêng.
- `ephemeralContainers` (dùng bởi `kubectl debug`) cũng nằm trong scope PSA `restricted` — debug container vào 1 namespace `restricted` cũng phải tuân seccomp, có thể khiến `strace`/`ptrace` trong debug container bị chặn (do `RuntimeDefault` cấm `ptrace` trừ khi có `CAP_SYS_PTRACE`).

## Security notes

- Đây là control **admission-time + runtime-time**: PSA chặn ở API server (không cho pod tạo nếu vi phạm), còn seccomp filter thật sự chặn ở kernel khi container chạy. Một cluster chỉ bật PSA `restricted` mà không thật sự có `RuntimeDefault`/`Localhost` hoạt động đúng ở tầng runtime vẫn có thể có lỗ hổng nếu runtime bug (hiếm nhưng về lý thuyết PSA chỉ validate field tồn tại, không validate runtime enforce đúng).
- Không nhầm `seccompProfile` (Pod Security Context) với `RuntimeClassName` — 2 cơ chế độc lập, có thể dùng cùng lúc (RuntimeClass chọn sandbox runtime như gVisor/Kata, seccomp giới hạn syscall trong sandbox đó) → [[seccomp--sandbox-runtimeclass]].

## Refs
- [Kubernetes — Seccomp reference (chính thức)](https://kubernetes.io/docs/reference/node/seccomp/)
- [Kubernetes — Restrict a Container's Syscalls with seccomp (tutorial)](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Kubernetes — Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [KEP-2413 — Kubelet default seccomp profile](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2413-seccomp-by-default/README.md)
