# SELinux — Containers & Kubernetes (CKS-relevant)
Tier: 2
Parent: [[selinux]]
Related: [[selinux--mls-mcs-sandbox]], [[selinux--policy-modules]]
Tags: #selinux #kubernetes #cks #podman #containers

> Verify: Kubernetes docs (kubernetes.io/docs/tasks/configure-pod-container/security-context) + Kubernetes v1.37 "Garhwal" release blog (08/2026), truy cập 2026-09-06. **Kiểm tra lại `kubectl version` trên cluster thật** — hành vi mặc định của `seLinuxChangePolicy` phụ thuộc version cluster, không phải version note này.

## What it does

Hai tầng riêng biệt cần phân biệt rõ:
1. **Tầng OS/container runtime** (Podman/Docker/CRI-O/containerd trên node) — dùng SELinux (qua sVirt/MCS, xem [[selinux--mls-mcs-sandbox]]) làm lớp cô lập bổ sung ngoài namespace/cgroup.
2. **Tầng Kubernetes API** — field `seLinuxOptions` trong `securityContext` cho phép Pod/Container **chỉ định label SELinux cụ thể** thay vì để container runtime tự sinh ngẫu nhiên.

## Why it exists

Namespace/cgroup cô lập *view* và *resource limit*, không phải access control — một process root trong container vẫn có thể đụng vào file host nếu chỉ dựa vào namespace (đặc biệt qua hostPath volume hoặc kernel bug). SELinux/MCS thêm lớp MAC độc lập với namespace: dù escape được namespace, process vẫn bị chặn bởi type/category không khớp. Đây là lý do CKS domain "Minimize Microservice Vulnerabilities" liệt kê SELinux/AppArmor/seccomp cùng nhóm "kernel/OS hardening cho workload".

## How it works (flow/diagram)

### Podman/Docker (tầng node)

```bash
podman run -v /host/data:/data:z ...   # 'z' = shared label, nhiều container cùng dùng volume này
podman run -v /host/data:/data:Z ...   # 'Z' = private label, CHỈ container này được đụng vào
```
Cả hai đều relabel file host thành `container_file_t`; khác biệt là `:Z` gắn thêm MCS category riêng cho container đó (xem [[selinux--mls-mcs-sandbox]]) → container khác dù cũng `container_file_t` vẫn bị chặn vì category không khớp.

⚠️ **Docker: SELinux support tồn tại nhưng KHÔNG bật mặc định** — khác với Podman (bật mặc định khi host có SELinux Enforcing). Nếu đang chuyển từ Docker sang Podman (phổ biến trên RHEL 9, vì Docker CE không official support), đây là khác biệt hành vi cần biết trước khi bàn giao.

**Custom policy cho container cụ thể (khi default `container_t` quá chặt hoặc quá lỏng cho nhu cầu riêng):**
```bash
podman inspect <container_id> | udica my_container_policy
semodule -i my_container_policy.cil /usr/share/udica/templates/{base_container.cil,net_container.cil,home_container.cil}
podman run --security-opt label=type:my_container_policy.process ...
```

### Kubernetes API (tầng cluster)

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"     # thường chỉ cần set level cho multi-tenant; user/role/type ít khi cần custom
    seLinuxChangePolicy: Recursive   # hoặc "MountOption" — xem gotcha bên dưới
  containers:
  - name: app
    securityContext:
      seLinuxOptions:
        level: "s0:c789,c999"   # override ở container-level nếu cần khác pod-level
```

`seLinuxOptions` có 4 field: `user`, `role`, `type`, `level` — set ở `spec.securityContext` (áp dụng mọi container trong Pod) hoặc `spec.containers[].securityContext` (override riêng container đó).

**`spec.securityContext.seLinuxChangePolicy`** (field riêng, cùng cấp với `seLinuxOptions`, không nằm trong nó) — quyết định cách gắn label lên **volume**:
- `MountOption` — mount volume với `-o context=<label>`, nhanh, nhưng yêu cầu mọi Pod share cùng volume phải dùng **cùng** SELinux label.
- `Recursive` — container runtime relabel đệ quy toàn bộ file trong volume (chậm với volume lớn), nhưng cho phép trộn Pod privileged/unprivileged share cùng volume trên cùng node.

## Config gotchas

- **Breaking change cần biết cho vận hành 2026+:** theo Kubernetes blog 04/2026, feature `SELinuxMount`/`SELinuxChangePolicy` graduate lên **Stable/GA ở Kubernetes v1.37** (phát hành 08/2026) và **bật mặc định** — nghĩa là hành vi mount volume SELinux có thể đổi từ "luôn recursive relabel" (hành vi cũ) sang "`MountOption` khi hạ tầng hỗ trợ". Nếu cluster đang/sắp lên v1.37+, **audit lại các Pod share chung volume giữa nhiều SELinux label khác nhau trước khi upgrade** — đây chính là khuyến nghị chính thức từ Kubernetes SIG-storage. *(Ghi chú: chi tiết đầy đủ về giá trị default chính xác và danh sách CSI driver hỗ trợ `-o context` nên tự kiểm chứng lại trên `kubernetes.io` tại thời điểm upgrade thật, vì đây là vùng đang thay đổi nhanh.)*
- `seLinuxOptions` **không tự động cấp quyền** — nó chỉ đổi *label* mà container process/volume mang; container vẫn phải chạy trên node có `selinux-policy` cho phép domain đó làm điều nó cần. Set sai `type` (VD: gõ nhầm type không tồn tại trong policy node) khiến container không start được, lỗi thường thấy ở tầng CRI (`CreateContainerError` liên quan SELinux label) chứ không phải lỗi YAML.
- Trên node **không bật SELinux** (`getenforce` = Disabled), toàn bộ field `seLinuxOptions`/`seLinuxChangePolicy` **vô hiệu lực, không báo lỗi** — Kubernetes docs nói rõ field này "no effect on nodes that do not support SELinux". Đây là bẫy phổ biến khi test trên node dev (thường tắt SELinux) rồi deploy thẳng lên prod node (Enforcing) mà không kiểm tra lại.
- CKS lưu ý: nội dung thi hiện tại (theo khảo sát 2026) nhấn mạnh seccomp/AppArmor trong domain "System Hardening" nhiều hơn SELinux; `seLinuxOptions` chủ yếu rơi vào domain "Minimize Microservice Vulnerabilities" (Security Context nói chung). Đừng học lệch chỉ SELinux mà bỏ qua seccomp/AppArmor khi ôn thi — nên đối chiếu lại curriculum chính thức mới nhất từ Linux Foundation/CNCF trước ngày thi vì phạm vi hay được cập nhật theo từng phiên bản Kubernetes exam.

## Security notes

- `--security-opt label=disable` (Podman) hoặc chạy container `--privileged` tắt hẳn lớp cô lập SELinux cho container đó — chỉ chấp nhận khi có kiểm soát bù đắp rõ ràng (VD: container đó chính là 1 sandbox/debug tool có kiểm soát network riêng).
- Trong multi-tenant Kubernetes cluster dùng chung node, `seLinuxOptions.level` set thủ công (thay vì để runtime tự sinh) chỉ an toàn nếu có quy trình đảm bảo **không trùng category** giữa các tenant — trùng category vô hiệu hoá cô lập MCS giữa các workload đó.

## Refs

- [Kubernetes — Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes v1.37 "Garhwal" release notes](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/)
- [SELinux Volume Label Changes goes GA (breaking changes blog, 04/2026)](https://kubernetes.io/blog/2026/04/22/breaking-changes-in-selinux-volume-labeling/)
- [Chapter 9 — Creating SELinux policies for containers / udica (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/creating-selinux-policies-for-containers_using-selinux)
- [udica GitHub repo](https://github.com/containers/udica)
