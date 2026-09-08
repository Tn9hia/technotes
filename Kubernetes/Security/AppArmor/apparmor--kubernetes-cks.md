# AppArmor — Kubernetes Integration (CKS)
Tier: 2
Parent: [[AppArmor]]
Related: [[apparmor--profile-syntax]], [[apparmor--debugging-runbook]]
Tags: #apparmor #kubernetes #cks #exam

## What it does
Cách Kubernetes gán 1 AppArmor profile (đã load sẵn trên node) cho container của pod, qua annotation (cũ) hoặc field `securityContext.appArmorProfile` (mới, GA từ v1.30).

## Why it exists
CKS domain "Minimize Microservice Vulnerabilities" yêu cầu biết dùng AppArmor/Seccomp/SecurityContext để giảm attack surface container — AppArmor là 1 trong các bài thực hành phổ biến (tạo profile, load lên node, gán vào pod, verify enforcement thật).

## How it works (flow)

```
 Node (bạn/DaemonSet control)          API Server              kubelet trên Node
┌──────────────────────────┐        ┌────────────────┐      ┌──────────────────────────┐
│ /etc/apparmor.d/my-prof   │        │  Pod spec:      │      │ đọc field/annotation      │
│ apparmor_parser -r ...    │───────▶│  securityContext│─────▶│ truyền profile NAME cho   │
│ (PHẢI làm TRƯỚC khi pod   │  (K8s  │  .appArmorProfile│      │ container runtime (CRI)   │
│  chạy trên node này)      │ không  │  .type: Localhost│      │ containerd/CRI-O attach   │
│                            │ tự làm)│  .localhostProfile│     │ profile khi tạo container │
└──────────────────────────┘        └────────────────┘      └──────────────────────────┘
```

**Điểm mấu chốt: kubelet KHÔNG load profile lên node.** Phải tự đảm bảo profile đã tồn tại trên node (bake vào image, DaemonSet, config management) trước khi pod được schedule tới đó. Nếu thiếu → pod stuck ở `CreateContainerError`, không phải bị từ chối lúc schedule.

### 2 cách khai báo (biết cả 2 vì đề CKS/cluster thực tế có thể dùng version khác nhau)

**Cách cũ — annotation (deprecated từ 1.30, vẫn còn ở nhiều cluster/exam bản cũ):**
```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/<profile-name>
    # hoặc: runtime/default | unconfined
```

**Cách mới — native field (GA từ K8s 1.30):**
```yaml
spec:
  securityContext:            # đặt được ở pod-level hoặc container-level
    appArmorProfile:
      type: Localhost          # RuntimeDefault | Localhost | Unconfined
      localhostProfile: my-profile   # bắt buộc nếu type=Localhost, PHẢI trùng TÊN profile đã load, không phải tên file
```

### 3 giá trị `type`
| Type | Ý nghĩa |
|---|---|
| `RuntimeDefault` | Dùng profile mặc định của container runtime (vd `docker-default`/containerd default) — KHÔNG phải unconfined, vẫn có chặn cơ bản |
| `Localhost` | Dùng custom profile đã load sẵn trên node, chỉ định qua `localhostProfile` |
| `Unconfined` | Tắt hẳn AppArmor cho container này — dùng có chủ đích, không phải default |

## Config gotchas
- **Không set field/annotation gì cả ≠ Unconfined** — container runtime tự áp default profile tương đương `RuntimeDefault` cho container thường (tuỳ version/runtime), khác hẳn suy nghĩ "không khai báo = chạy trần". Luôn verify bằng `aa-status` trên node hoặc thử hành vi bị chặn qua `kubectl exec`, đừng suy đoán từ manifest.
- `localhostProfile`/annotation value phải khớp **tên profile khai báo bên trong file** (`profile <name> {`), KHÔNG phải tên file trong `/etc/apparmor.d/`. Nhầm lẫn 2 cái này là lỗi hay gặp nhất khi thao tác/thi.
- Field ở pod-level áp cho container không tự override riêng — nếu cần khác nhau giữa các container cùng pod, đặt `securityContext` riêng ở container-level.
- Muốn chắc pod chỉ schedule vào node đã có profile: dùng `nodeSelector`/`nodeAffinity` theo label tự gắn — K8s không tự biết node nào có profile gì.
- Node phải chạy OS bật AppArmor kernel module (Ubuntu/Debian node pool); trên node RHEL/SELinux-based, field/annotation này gần như vô nghĩa — luôn test thực tế trước, đừng giả định hành vi theo doc.
- Kiểm tra nhanh OS của node: `kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.osImage}'`; kiểm tra AppArmor thật sự chạy: exec/`kubectl debug node/<name>` rồi chạy `aa-status`.

## Security notes
- Đây là 1 trong 4 lớp phòng thủ container hay được hỏi chung trong CKS: **Seccomp** (lọc syscall) + **AppArmor/SELinux** (lọc resource access theo path/label) + **Capabilities** (drop ALL rồi add cụ thể cần thiết) + **PodSecurity Standards/securityContext** (runAsNonRoot, readOnlyRootFilesystem,...). AppArmor không thay thế 3 lớp còn lại — luôn kết hợp.
- Cluster multi-tenant: không nên cho phép tự do set `type: Unconfined` — cân nhắc policy engine (Kyverno/OPA Gatekeeper) chặn pod đặt Unconfined trừ namespace whitelist.

## Cách verify nhanh (dùng khi thao tác thật hoặc lúc thi)
```bash
# 1. Load profile lên node (ssh/console vào worker node)
sudo cp my-profile /etc/apparmor.d/
sudo apparmor_parser -r /etc/apparmor.d/my-profile
aa-status | grep my-profile     # confirm đã load, đúng mode

# 2. Apply pod với field/annotation trỏ đúng TÊN profile (không phải tên file)

# 3. Verify enforcement thật, đừng chỉ tin manifest
kubectl exec -it <pod> -- cat /etc/shadow            # kỳ vọng: Permission denied nếu profile deny
kubectl exec -it <pod> -- cat /proc/1/attr/current   # xem tên profile + mode đang áp cho process (vd "my-profile (enforce)")
```

## Refs
- https://kubernetes.io/docs/tutorials/security/apparmor/
- KEP native `appArmorProfile` field (K8s 1.30 graduation)
- CKS curriculum — domain "Minimize Microservice Vulnerabilities"
