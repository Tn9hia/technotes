---
title: Minimize Microservice Vulnerabilities
tags:
  - kubernetes
  - security
  - cks
  - microservice-vulnerabilities
date: 2026-08-18
---

# Minimize Microservice Vulnerabilities

Đây là domain lớn nhất của CKS (20% trọng số thi, ngang với "Cluster Setup"). Nội dung xoay quanh cách cô lập (isolate) workload trong môi trường multi-tenant, kiểm soát request đi vào API server (admission control, OPA), chuẩn hoá security posture của Pod (Security Context, Pod Security Standards), và bảo vệ dữ liệu nhạy cảm cả khi lưu trữ (Secrets, encryption at rest) lẫn khi truyền đi (mTLS, Cilium encryption).

## Multi-tenancy & Isolation

### Multi-tenancy là gì

Multi-tenancy nghĩa là nhiều team/khách hàng cùng chia sẻ một cluster Kubernetes duy nhất thay vì mỗi bên một cluster riêng. Ẩn dụ quen thuộc: cluster giống một toà nhà văn phòng nhiều tầng — mỗi namespace là một tầng riêng cho một team, nhưng vẫn dùng chung hạ tầng (thang máy, bãi đỗ xe ~ network, storage, node).

Anti-pattern phổ biến là "mỗi tenant một cluster riêng" — có vẻ cô lập tuyệt đối nhưng không scale được vì chi phí vận hành (patch, upgrade, giám sát N cluster) tăng tuyến tính theo số tenant. Lợi ích của multi-tenancy trên một cluster:

| Lợi ích | Mô tả |
|---|---|
| Cost Efficiency | Giảm chi phí hạ tầng/vận hành so với việc tạo cluster riêng cho từng tenant |
| Enhanced Resource Usage | Tận dụng tốt hơn compute/storage/network dùng chung |
| Simplified Management | Quản lý, giám sát, troubleshoot tập trung |
| Robust Security | Namespace, RBAC, NetworkPolicy ngăn tenant này ảnh hưởng tenant khác |

Rủi ro nếu không kiểm soát chặt: rò rỉ dữ liệu chéo tenant, một tenant "chiếm dụng" tài nguyên (noisy neighbor) làm ảnh hưởng hiệu năng toàn cluster, khó tuân thủ quy định (GDPR, HIPAA...) khi dữ liệu không phân tách rõ ràng.

### Multi-team vs Multi-customer tenancy

| Tiêu chí | Multi-Team | Multi-Customer |
|---|---|---|
| Đối tượng | Các team/dự án nội bộ cùng tổ chức | Khách hàng/tổ chức bên ngoài (mô hình SaaS) |
| Truy cập cluster | Team nội bộ thường truy cập trực tiếp qua `kubectl`/GitOps controller | Khách hàng bên ngoài **không** có quyền truy cập trực tiếp cluster; mọi thao tác diễn ra "phía sau" |
| Yêu cầu bảo mật | RBAC + resource quota theo namespace là đủ | Yêu cầu cao hơn hẳn — thường phải tuân thủ chuẩn pháp lý (GDPR, HIPAA...) |

Cả hai mô hình đều dựa trên namespace làm đơn vị cô lập cơ bản, nhưng multi-customer đòi hỏi kiểm soát nghiêm ngặt hơn nhiều vì tenant bên ngoài không được tin tưởng như nhân viên nội bộ.

### Các lớp isolation: Namespace → Pod → Network → Node

Kubernetes cô lập tenant ở cả **Control Plane** (namespace + RBAC) lẫn **Data Plane** (network policy, storage isolation, node isolation).

- **Namespace isolation**: Chia cluster thành các "văn phòng" logic; tên resource có thể trùng nhau giữa các namespace khác nhau mà không xung đột. Phần lớn chính sách bảo mật Kubernetes (RBAC Role, NetworkPolicy...) đều scope theo namespace.
- **Pod isolation**: Trong một namespace, mỗi Pod là một "chỗ làm việc" riêng — các container trong cùng Pod chia sẻ tài nguyên chặt (network namespace, có thể chung volume), nhưng các Pod khác nhau thì cô lập với nhau.
- **Network isolation**: Dùng NetworkPolicy để giới hạn Pod nào được giao tiếp với Pod nào — giống việc chỉ cho phép nhân viên team A vào phòng họp/network segment dành riêng cho team A.
- **Node isolation**: Node giống một tầng của toà nhà — có thể dùng chung cho nhiều team dưới kiểm soát chặt, hoặc dành hẳn cho một team/khách hàng (dùng taints/tolerations).

### Hard isolation vs Soft isolation

- **Hard isolation**: Dành hẳn hạ tầng vật lý/ảo hoá (compute, storage, network) riêng cho mỗi tenant — cô lập tuyệt đối nhưng tốn kém, ít tận dụng chung tài nguyên.
- **Soft isolation**: Tenant chia sẻ chung hạ tầng bên dưới, cô lập bằng namespace + ResourceQuota/LimitRange + NetworkPolicy. Rẻ hơn, linh hoạt hơn, nhưng phụ thuộc vào việc cấu hình đúng các cơ chế cô lập logic.

### Control Plane Isolation (Namespace + RBAC)

Tạo namespace cho từng team:

```bash
kubectl create namespace namespaceA
kubectl create namespace namespaceB
```

Nguyên tắc cốt lõi là **least privilege**: mỗi team chỉ nên có quyền trên namespace/resource của chính họ. Ví dụ Role + RoleBinding giới hạn user `pranjal` chỉ thao tác được trên pods/services trong namespace `development`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
  - kind: User
    name: pranjal
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### Data Plane Isolation (tổng quan)

Data plane là nơi workload thực sự chạy — network, storage, compute. Ba cơ chế chính để cô lập data plane:

- **Network Policies**: kiểm soát traffic pod-to-pod.
- **Storage Isolation**: tách biệt storage class/PV/PVC theo tenant.
- **Taints & Tolerations**: chỉ cho phép Pod có toleration phù hợp được schedule lên node đã taint.

### Data Plane Isolation — Network (NetworkPolicy)

NetworkPolicy định nghĩa rule dựa trên **Pod Selector**, **Namespace Selector**, **Port**, **Protocol**. Ví dụ: chỉ cho phép Pod có label `app: backend` trong namespace `tenant_a` nhận traffic TCP port 8080 từ các Pod thuộc namespace có label `tenant: tenant_b`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-tenant-a-to-tenant-b
  namespace: tenant_a
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: tenant_b
  ports:
  - protocol: TCP
    port: 8080
```

### Data Plane Isolation — Storage (StorageClass)

Tạo StorageClass riêng theo mức hiệu năng để tenant "hạng nặng" dùng storage IOPS cao, tenant thường dùng storage tiêu chuẩn:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1                          # AWS io1 disk hỗ trợ IOPS cao
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

PVC nhắm tới workload cần IOPS cao sẽ bind trực tiếp vào StorageClass này; tenant thường dùng StorageClass tiêu chuẩn khác.

### Node Pools + Taints/Tolerations cho node isolation

Dùng để tránh "noisy neighbor" bằng cách dành hẳn node cho một tenant. Chỉ Pod có toleration khớp mới được schedule lên node đã taint.

```bash
kubectl taint nodes nodeA customer=customerA:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: customer-a-pod
  namespace: customer_a
spec:
  containers:
  - name: customer-a-container
    image: nginx
  tolerations:
  - key: "customer"
    operator: "Equal"
    value: "customerA"
    effect: "NoSchedule"
```

Lưu ý: toleration chỉ **cho phép** Pod schedule lên node đã taint, chứ không **ép** Pod phải chạy ở đó — muốn ép Pod chỉ chạy trên node dành riêng đó (tránh Pod khác "lạc" vào node chưa taint), phải kết hợp thêm `nodeSelector`/`nodeAffinity`.

### DNS trong môi trường multi-tenant

Mặc định CoreDNS cho phép resolve FQDN xuyên namespace, ví dụ Pod ở namespace B vẫn resolve được `backend.namespace-a.svc.cluster.local`. Điều này thuận tiện nhưng làm giảm cô lập tenant. Có thể giới hạn resolve chỉ trong cùng namespace bằng directive `fallthrough in-namespace`:

```bash
kubectl edit configmap coredns -n kube-system
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods verified
            fallthrough in-namespace
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Kiểm tra bằng cách chạy pod test ở namespace A và cố resolve service ở namespace B:

```bash
kubectl run test-pod --rm -i --tty --image=busybox --restart=Never --namespace=namespace-a -- nslookup backend.namespace-b.svc.cluster.local
```

Nếu cấu hình đúng, lookup xuyên namespace sẽ thất bại. Lưu ý đây chỉ là hạn chế **DNS discovery** — nếu muốn thực sự chặn traffic xuyên namespace phải kết hợp thêm NetworkPolicy (chặn DNS chỉ ẩn tên, không tự động chặn kết nối IP trực tiếp).

### API Priority and Fairness & Pod Priority/Preemption

Hai cơ chế khác nhau nhưng dễ gây nhầm lẫn trong đề thi vì cùng liên quan đến "priority":

- **API Priority and Fairness (APF)**: kiểm soát cách **API server** xử lý request (list, create, update...) — đảm bảo tenant/namespace quan trọng không bị nghẽn request vì tenant khác gửi traffic API dồn dập. Dùng hai object: `PriorityLevelConfiguration` (định nghĩa mức concurrency) và `FlowSchema` (map request/subject vào priority level).
- **Pod Priority and Preemption**: kiểm soát việc **schedule Pod lên node** — khi thiếu tài nguyên, Pod priority thấp có thể bị preempt (evict) để nhường chỗ cho Pod priority cao. Dùng object `PriorityClass` + field `priorityClassName` trên Pod.

Ví dụ cấu hình APF (API group `flowcontrol.apiserver.k8s.io/v1beta3`):

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: high-priority
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 10  # priority cao hơn -> nhiều concurrency hơn
    limitResponse:
      type: Queue
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: low-priority
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 1
    limitResponse:
      type: Queue
```

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: high-priority-namespace-a
spec:
  priorityLevelConfiguration:
    name: high-priority
    matchingPrecedence: 1000
  rules:
    - subjects:
        - kind: ServiceAccount
          name: "system-account"
          namespace: "namespace-a"
      resourceRules:
        - verbs: ["*"]
          apiGroups: ["*"]
          resources: ["*"]
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: low-priority-namespace-b
spec:
  priorityLevelConfiguration:
    name: low-priority
    matchingPrecedence: 2000
  rules:
    - subjects:
        - kind: Group
          name: "regular-users"
        - kind: ServiceAccount
          name: "default"
          namespace: "namespace-b"
      resourceRules:
        - verbs: ["*"]
          apiGroups: ["*"]
          resources: ["*"]
```

Ví dụ Pod Priority and Preemption:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "This priority class is for critical production workloads."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "This priority class is for non-critical development workloads."
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  namespace: namespace-a
spec:
  priorityClassName: high-priority
  containers:
  - name: app-container
    image: nginx
    resources:
      requests:
        memory: "500Mi"
        cpu: "500m"
      limits:
        memory: "500Mi"
        cpu: "500m"
```

## Container Sandboxing / Runtime Isolation

### VM vs Container: khác biệt căn bản

| Khía cạnh | Virtual Machine | Container |
|---|---|---|
| Kernel riêng cho mỗi workload | Có — guest kernel riêng | Không — dùng chung host kernel |
| Cơ chế cô lập | Hypervisor | Namespaces, cgroups, LSM (AppArmor/SELinux) |
| Use case điển hình | Cô lập mạnh trong multi-tenant | Microservice nhẹ, mật độ cao |
| Rủi ro escape nếu kernel bị khai thác | Thấp hơn (guest kernel tách biệt) | Cao hơn (dùng chung kernel) |

Vì container dùng chung kernel host, một lỗ hổng leo thang đặc quyền cấp kernel (ví dụ Dirty COW) có thể cho phép process trong container ảnh hưởng tới host và các container khác.

### PID namespace — ví dụ minh hoạ cô lập process

```bash
# Chạy container sleep
$ docker run -d --name sleeping-container busybox sleep 1000

# Trong container: process sleep là PID 1
$ docker exec -ti sleeping-container ps -ef
PID   USER     TIME  COMMAND
1     root     0:00  sleep 1000
11    root     0:00  ps -ef

# Trên host: cùng process nhưng PID khác
$ ps -ef | grep sleep | grep -vi grep
root     7902  7871  0 21:39 ?        00:00:00 sleep 1000
```

Vì process host thật vẫn tồn tại, kill PID trên host sẽ giết luôn process trong container — chứng minh namespace chỉ cô lập ở mức user-space, kernel dùng chung vẫn là điểm kiểm soát cuối cùng.

### seccomp và AppArmor

- **seccomp**: giới hạn tập syscall mà process được phép gọi (mô hình **whitelist** — mặc định deny, chỉ allow syscall cần thiết).
- **AppArmor**/SELinux: enforce theo path và capability (thường dùng theo mô hình **blacklist** — mặc định allow, chặn hành vi cụ thể).

Ví dụ seccomp profile (default deny, chỉ allow một số syscall):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": ["execve", "brk", "access", "capset", "clone"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Ví dụ AppArmor profile (blacklist — chặn ghi vào `/proc`):

```text
profile apparmor-deny-write flags=(attach_disconnected) {
    #include <abstractions/base>

    /usr/bin/your-binary ixr,
    /lib/** r,
    /usr/lib/** r,
    /etc/** r,

    deny /proc/** w,
}
```

| Mô hình | Mô tả | Ưu điểm | Đánh đổi |
|---|---|---|---|
| Whitelist (seccomp) | Default deny, allow tường minh | Bề mặt tấn công kernel tối thiểu, bảo mật cao | Cần hiểu rõ app cần syscall nào |
| Blacklist (AppArmor) | Default allow, chặn hành vi cụ thể | Dễ áp dụng cho nhiều loại app | Dễ sót vector tấn công, kém chặt hơn |

Khuyến nghị: ưu tiên whitelist (seccomp) khi khả thi; với fleet lớn/đa dạng dùng cách tiếp cận nhiều lớp (namespaces + cgroups + capability drop + seccomp + LSM) và tự động hoá sinh/rollout profile; luôn áp dụng least privilege (drop capability không cần, hạn chế filesystem/network).

### Kỹ thuật sandbox nâng cao (microVM)

Khi mô hình "chung kernel" không đủ an toàn cho threat model của bạn:

| Công nghệ | Mô tả | Use case |
|---|---|---|
| gVisor | User-space kernel chặn và giả lập syscall | Tăng cô lập mà không cần VM nặng |
| Kata Containers | Chạy container trong lightweight VM riêng | Cô lập cấp VM |
| Firecracker | MicroVM khởi động cực nhanh, overhead thấp | Serverless, multi-tenant cấp VM |

### gVisor

gVisor chèn thêm một lớp trung gian giữa container và Linux kernel, chặn (intercept) syscall thay vì để container gọi thẳng kernel host. Gồm 2 thành phần:

- **Sentry**: một "application-level kernel" độc lập, xử lý syscall của container. Vì chỉ hỗ trợ tập chức năng giới hạn (không phải full Linux kernel), bề mặt tấn công nhỏ hơn nhiều.
- **Gofer**: khi container cần truy cập file, Sentry không gọi thẳng kernel mà chuyển qua Gofer — một process proxy file riêng, đóng vai trò trung gian.

gVisor còn dùng network stack riêng (không đụng trực tiếp network code của OS host). Mỗi container có kernel gVisor cô lập riêng — nếu một instance gVisor bị lỗi/compromise, các container khác không bị ảnh hưởng. Đánh đổi: không phải app nào cũng tương thích 100%, và việc chặn syscall qua trung gian có thể gây overhead hiệu năng nhẹ — cần test kỹ trước khi dùng production.

Sơ đồ luồng syscall khi có gVisor chen giữa:

```
Container Process
      │ syscall
      ▼
   gVisor Sentry (user-space kernel, viết bằng Go)
      │ filtered/translated — chỉ forward syscall thực sự cần
      ▼
   Host Kernel (bề mặt tấn công nhỏ hơn nhiều so với gọi thẳng)
```

Cài gVisor (`runsc`) trên node và cấu hình containerd nhận diện runtime handler:

```bash
# Cài gVisor
curl -fsSL https://gvisor.dev/archive.key | gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" > /etc/apt/sources.list.d/gvisor.list
apt-get update && apt-get install -y runsc

# Khai báo runtime handler "runsc" trong containerd
cat >> /etc/containerd/config.toml <<EOF
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF

systemctl restart containerd
```

Kiểm chứng Pod thực sự chạy trong gVisor (kernel version trả về sẽ khác kernel host):

```bash
kubectl run gvisor-test \
  --image=nginx \
  --overrides='{"spec":{"runtimeClassName":"gvisor"}}' \
  --rm -it -- uname -r
```

Đánh đổi tổng quan của gVisor:

| Khía cạnh | Đánh giá |
|---|---|
| Security | Cô lập rất tốt — Sentry chỉ hỗ trợ tập syscall giới hạn nên attack surface nhỏ |
| Compatibility | ~90% syscall coverage — một số workload (database nặng I/O, truy cập GPU trực tiếp) có thể không chạy được |
| Performance | Overhead 10-30% cho ứng dụng I/O-intensive do syscall bị chặn/dịch qua Sentry |

### Kata Containers

Khác với gVisor, Kata Containers chạy **mỗi container trong một lightweight VM riêng**, mỗi container có kernel riêng thay vì dùng chung kernel host — lỗi ở một container không ảnh hưởng container khác hay host. Đánh đổi là overhead về memory/compute do phải chạy VM riêng cho từng container.

Yêu cầu quan trọng: Kata Containers cần **hardware virtualization support**. Trên cloud, compute instance thường vốn đã là VM, nên chạy Kata bên trong nghĩa là **nested virtualization** — phần lớn cloud provider không hỗ trợ (một số ngoại lệ như Google Cloud có hỗ trợ nhưng cần cấu hình thủ công và hiệu năng không tối ưu). Muốn dùng Kata hiệu quả nhất nên chạy trên bare-metal/physical server.

Cài Kata Containers và khai báo runtime handler trong containerd (luồng thực thi: Container → Kata Agent → QEMU/KVM microVM → Host Kernel):

```bash
apt-get install -y kata-containers
```

```toml
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

Đánh đổi tổng quan của Kata:

| Khía cạnh | Đánh giá |
|---|---|
| Security | Cô lập mạnh nhất (cấp VM) — phù hợp workload không tin cậy |
| Startup | Chậm hơn hẳn (1-2 giây thay vì mili-giây) vì phải khởi động microVM |
| Resource overhead | Tốn thêm memory/CPU cho mỗi VM riêng theo từng container |

### Khi nào dùng gVisor vs Kata vs standard?

| Workload | Runtime nên dùng | Lý do |
|---|---|---|
| Web app, API thông thường | gVisor | Cân bằng giữa bảo mật và hiệu năng |
| Database, stateful workload | Standard + seccomp/AppArmor | Cần tương thích syscall đầy đủ và hiệu năng I/O cao |
| CI/CD job runner (chạy code không tin cậy) | Kata | Cô lập cấp VM cho code chưa được kiểm chứng |
| Multi-tenant SaaS (khách hàng ngoài) | Kata | Cô lập tenant ở mức VM, chấp nhận overhead đổi lấy an toàn |
| Batch job đã tin cậy | Standard + seccomp RuntimeDefault | Ưu tiên hiệu năng, rủi ro thấp |

### RuntimeClass — cách gán runtime sandbox cho Pod

Để dùng gVisor hay Kata cho một Pod cụ thể, cluster phải có sẵn `RuntimeClass` object trỏ tới container runtime handler tương ứng (đã cài trên node), sau đó Pod khai báo `runtimeClassName`:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc          # tên handler containerd/CRI-O đã cấu hình cho gVisor
---
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-pod
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx
```

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata            # handler đã cấu hình cho Kata Containers
```

## Admission Control

### Request lifecycle: Authentication → Authorization → Admission

Mọi request tới API server (kể cả từ `kubectl`) đi qua 3 bước:

1. **Authentication**: xác thực danh tính (ví dụ certificate trong kubeconfig).
2. **Authorization**: RBAC quyết định user có được phép thực hiện hành động không.
3. **Admission Control**: kiểm tra/sửa đổi request **sau khi** đã auth+authz nhưng **trước khi** ghi vào etcd.

RBAC chỉ trả lời "được phép làm hành động X trên resource Y hay không" — không đủ để validate nội dung chi tiết như "image có từ registry tin cậy không", "container có chạy root không". Admission Controller lấp khoảng trống này.

Ví dụ Role RBAC cơ bản và Role giới hạn theo `resourceNames`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create"]
  resourceNames: ["blue", "orange"]
```

Pod dưới đây RBAC không chặn được (có quyền tạo pod) nhưng lại nguy hiểm về mặt security posture — đây chính là việc admission controller cần xử lý:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 0
        capabilities:
          add: ["MAC_ADMIN"]
```

### Built-in admission controllers

Một số admission controller có sẵn: `AlwaysPullImages`, `DefaultStorageClass`, `EventRateLimit`, `NamespaceExists`...

Xem admission controller nào đang bật:

```bash
kube-apiserver -h | grep enable-admission-plugins
# hoặc trong cluster kubeadm:
kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

Bật thêm admission plugin qua flag `--enable-admission-plugins` (kubeadm: sửa manifest `/etc/kubernetes/manifests/kube-apiserver.yaml`):

```text
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --runtime-config=api/all \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --v=2 \
  --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```

Tắt plugin dùng `--disable-admission-plugins` tương tự.

**Lưu ý deprecation quan trọng**: `NamespaceAutoProvision` và `NamespaceExists` đã bị **thay thế** bởi `NamespaceLifecycle` — một admission controller duy nhất vừa reject request nhắm vào namespace không tồn tại, vừa bảo vệ các namespace hệ thống (`default`, `kube-system`, `kube-public`) khỏi bị xoá.

### Validating vs Mutating admission controllers

- **Validating**: chỉ kiểm tra và allow/deny request (ví dụ `NamespaceLifecycle`).
- **Mutating**: sửa đổi request trước khi lưu (ví dụ `DefaultStorageClass` tự thêm `storageClassName` vào PVC nếu chưa chỉ định).

Ví dụ PVC không khai báo storage class:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Sau khi qua `DefaultStorageClass` (mutating), request trở thành:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: default
```

**Thứ tự chạy quan trọng: mutating controller chạy trước validating controller** — để các thay đổi (mutation) kịp được validate. Ví dụ `NamespaceAutoProvision` (mutating, tự tạo namespace) phải chạy trước `NamespaceExists`/`NamespaceLifecycle` (validating); nếu đảo ngược thứ tự thì request tới namespace chưa tồn tại sẽ bị reject trước khi kịp auto-provision.

### Admission Webhooks (mở rộng bằng logic tuỳ biến)

Ngoài built-in controller, Kubernetes hỗ trợ 2 loại webhook để chạy logic tuỳ biến bên ngoài: **Mutating Admission Webhook** và **Validating Admission Webhook**. API server gửi một `AdmissionReview` object (JSON) tới webhook server, webhook trả lời allowed/denied (và với mutating webhook, trả thêm JSON patch).

Ví dụ request gửi tới webhook:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-4201aa800002",
    "kind": {"group": "autoscaling", "version": "v1", "kind": "Scale"},
    "resource": {"group": "apps", "version": "v1", "resource": "deployments"},
    "subResource": "scale"
  }
}
```

Webhook server ví dụ bằng Python/Flask, một route validate, một route mutate:

```python
from flask import Flask, request, jsonify
import base64

app = Flask(__name__)

@app.route("/validate", methods=["POST"])
def validate():
    object_name = request.json["request"]["object"]["metadata"]["name"]
    user_name = request.json["request"]["userInfo"]["name"]
    status = True
    message = ""
    if object_name == user_name:
        message = "You can't create objects with your own name"
        status = False
    return jsonify({
        "response": {
            "allowed": status,
            "uid": request.json["request"]["uid"],
            "status": {"message": message},
        }
    })

@app.route("/mutate", methods=["POST"])
def mutate():
    user_name = request.json["request"]["userInfo"]["name"]
    patch = [{"op": "add", "path": "/metadata/labels/users", "value": user_name}]
    patch_encoded = base64.b64encode(str(patch).encode()).decode()
    return jsonify({
        "response": {
            "allowed": True,
            "uid": request.json["request"]["uid"],
            "patch": patch_encoded,
            "patchType": "JSONPatch",
        }
    })
```

Đăng ký webhook với cluster (bắt buộc TLS/`caBundle`):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
  - name: "pod-policy.example.com"
    clientConfig:
      service:
        namespace: "webhook-namespace"
        name: "webhook-service"
      caBundle: "CiOtLS0tQk......tLS0K"
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
```

Với mutating webhook, cấu trúc tương tự nhưng `kind: MutatingWebhookConfiguration`. Nếu bất kỳ admission controller/webhook nào trong chuỗi trả về reject, toàn bộ request thất bại và trả lỗi cho user.

## Policy Engines (OPA & Kyverno)

### OPA thuần (không có Kubernetes)

OPA (Open Policy Agent) là policy engine tập trung, tách logic authorization ra khỏi application code — thay vì mỗi service tự viết `if user != "john": return 401`, tất cả service query OPA qua API để xin quyết định allow/deny.

Chạy OPA server (mặc định port 8181, API mở không auth — cần network security riêng khi production):

```bash
curl -L -o opa https://github.com/open-policy-agent/opa/releases/download/v0.11.0/opa_linux_amd64
chmod 755 ./opa
./opa run -s
```

Policy viết bằng ngôn ngữ **Rego**:

```rego
package httpapi.authz

import input

default allow = false

allow {
    input.path == "home"
    input.user == "john"
}
```

Nạp policy vào OPA:

```bash
curl -X PUT --data-binary @example.rego http://localhost:8181/v1/policies/example1
curl http://localhost:8181/v1/policies   # liệt kê policy
```

Application (Flask) gọi OPA để xin quyết định thay vì tự check:

```python
@app.route('/home')
def hello_world():
    user = request.args.get("user")
    input_dict = {"input": {"user": user, "path": "home"}}
    rsp = requests.post("http://127.0.0.1:8181/v1/data/httpapi/authz", json=input_dict)
    if not rsp.json()["result"]["allow"]:
        return 'Unauthorized!', 401
    return 'Welcome Home!', 200
```

OPA có framework test riêng bằng Rego:

```rego
package authz

test_post_allowed {
    allow with input as {"path": ["users"], "method": "POST"}
}

test_get_anonymous_denied {
    not allow with input as {"path": ["users"], "method": "GET"}
}
```

```bash
opa test -v
```

Có thể thử nghiệm Rego trực tuyến tại `play.openpolicyagent.org`.

### OPA trong Kubernetes — Gatekeeper

**OPA Gatekeeper** tích hợp OPA vào Kubernetes bằng cách kết hợp admission webhook với **CRD-based policy** (Constraint Framework) — giúp policy dễ chia sẻ, dễ audit hơn so với quản lý file Rego rời rạc.

Cài Gatekeeper:

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper
kubectl get all -n gatekeeper-system
```

Luồng hoạt động: khi có request tạo Pod, admission controller (Gatekeeper webhook) lấy label của Pod → kiểm tra label bắt buộc (ví dụ `billing`) có tồn tại không → nếu thiếu, trả lỗi.

Rego kiểm tra label bắt buộc:

```rego
package systemrequiredlabels

import data.lib.helpers

violation[{"msg": msg, "details": {"missing_labels": missing}}] {
  provided := {label | input.request.object.metadata.labels[label]}
  required := {label | label == input.parameters.labels[_]}
  missing = required - provided
  count(missing) > 0
  msg = sprintf("you must provide labels: %v", [missing])
}
```

Quan hệ **ConstraintTemplate ↔ Constraint**: `ConstraintTemplate` định nghĩa CRD mới (`kind` tuỳ chọn, ví dụ `SystemRequiredLabel`) và nhúng Rego logic, đồng thời khai báo schema cho `parameters`. `Constraint` là **instance** của CRD đó — áp dụng policy cho namespace/label cụ thể bằng cách truyền `parameters`.

`requiredlabels-template.yaml`:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: systemrequiredlabels
spec:
  crd:
    spec:
      names:
        kind: SystemRequiredLabel
      validation:
        # schema cho field 'parameters'
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package systemrequiredlabels

        import data.lib.helpers

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.request.object.metadata.labels[label]}
          required := {label | label == input.parameters.labels[_]}
          missing = required - provided
          count(missing) > 0
          msg = sprintf("you must provide labels: %v", [missing])
        }
```

`require-label-billing.yaml` (Constraint — instance của `SystemRequiredLabel`):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: SystemRequiredLabel
metadata:
  name: require-billing-label
spec:
  match:
    namespaces: ["expensive"]
  parameters:
    labels: ["billing"]
```

Có thể tạo thêm Constraint khác dùng chung ConstraintTemplate cho namespace/label khác:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: SystemRequiredLabel
metadata:
  name: require-tech-label
spec:
  match:
    namespaces: ["engineering"]
  parameters:
    labels: ["tech"]
```

Áp dụng (thứ tự bắt buộc: ConstraintTemplate trước, Constraint sau):

```bash
kubectl apply -f requiredlabels-template.yaml
kubectl apply -f require-label-billing.yaml
```

### Policy thường dùng khác — cấm tag `:latest`, bắt buộc resource limits

Cấm dùng image tag `:latest` (kể cả trường hợp không khai tag nào — mặc định coi như `:latest`):

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowedtags
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedTags
      validation:
        openAPIV3Schema:
          type: object
          properties:
            tags:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowedtags

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          tag := split(container.image, ":")[1]
          tag == input.parameters.tags[_]
          msg := sprintf("Container '%v' uses disallowed tag '%v'", [container.name, tag])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, ":")
          msg := sprintf("Container '%v' has no tag (defaults to latest)", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: no-latest-tag
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    tags: ["latest"]
```

Bắt buộc container phải khai `resources.limits`:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' missing CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' missing memory limit", [container.name])
        }
```

### Audit mode — kiểm tra resource đang vi phạm

Ngoài chặn tại admission, Gatekeeper còn định kỳ audit các resource **đã tồn tại** trong cluster và ghi violation vào `status` của Constraint:

```bash
# Xem toàn bộ violation hiện có (mọi constraint)
kubectl get constraint -o json | jq '.items[].status.violations'

# Xem violation của một constraint cụ thể
kubectl describe k8srequiredlabels require-app-labels
# → Status.Violations: danh sách resource đang vi phạm
```

Lưu ý: audit **không hồi tố (not retroactive)** — resource được tạo trước khi Constraint tồn tại sẽ không tự bị chặn/xoá, chỉ được liệt kê là vi phạm để người vận hành xử lý thủ công.

### Kyverno

Kyverno là policy engine "Kubernetes-native" — viết policy hoàn toàn bằng **YAML**, không cần học Rego như OPA. Nhờ vậy dễ tiếp cận hơn cho phần lớn policy liên quan riêng tới Kubernetes, dù Rego/OPA vẫn linh hoạt hơn cho policy cross-resource phức tạp.

Cài Kyverno qua Helm:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set replicaCount=3   # production: 3 replicas cho HA
```

**Validate policy** — ví dụ cấm privileged container:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
  annotations:
    policies.kyverno.io/title: Disallow Privileged Containers
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce   # Enforce | Audit
  background: true   # đồng thời quét (audit) resource đã tồn tại
  rules:
    - name: privileged-containers
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
      validate:
        message: "Privileged mode is not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

Ví dụ bắt buộc chạy non-root:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-non-root
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-containers
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Containers must run as non-root user."
        anyPattern:
          - spec:
              securityContext:
                runAsNonRoot: true
          - spec:
              containers:
                - securityContext:
                    runAsNonRoot: true
```

**Mutate policy** — tự động sửa/bổ sung resource thay vì chỉ chặn. Ví dụ tự thêm `seccompProfile` nếu Pod chưa khai:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-seccomp
spec:
  rules:
    - name: add-seccomp
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        patchStrategicMerge:
          spec:
            securityContext:
              +(seccompProfile):    # dấu + = chỉ thêm nếu chưa có field này
                type: RuntimeDefault
```

Tự động thêm resource limits mặc định cho mọi container chưa khai:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-resources
spec:
  rules:
    - name: add-limits
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        foreach:
          - list: "request.object.spec.containers"
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    resources:
                      limits:
                        +(cpu): "500m"
                        +(memory): "256Mi"
```

**Generate policy** — tự động tạo resource mới khi có sự kiện trigger. Ví dụ tự tạo `NetworkPolicy` default-deny cho mọi namespace mới:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-network-policy
spec:
  rules:
    - name: default-deny-all
      match:
        any:
          - resources:
              kinds: [Namespace]
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-all
        namespace: "{{request.object.metadata.name}}"
        synchronize: true    # giữ đồng bộ, xoá theo nếu policy bị xoá
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              - Egress
```

**Kyverno CLI** — test policy offline trước khi apply vào cluster:

```bash
brew install kyverno

# Test policy với một manifest cụ thể
kyverno apply disallow-privileged.yaml --resource pod.yaml

# Chạy bộ test unit (đọc kyverno-test.yaml trong thư mục hiện tại)
kyverno test .

# Dry-run trực tiếp với cluster đang chạy
kyverno apply ./policies/ --cluster
```

Format file test (`kyverno-test.yaml`):

```yaml
name: Test disallow privileged
policies:
  - disallow-privileged-containers.yaml
resources:
  - test-pod-privileged.yaml
  - test-pod-secure.yaml
results:
  - policy: disallow-privileged-containers
    rule: privileged-containers
    resource: test-pod-privileged
    kind: Pod
    result: fail
  - policy: disallow-privileged-containers
    rule: privileged-containers
    resource: test-pod-secure
    kind: Pod
    result: pass
```

### So Sánh PSA vs OPA vs Kyverno

| | PSA | OPA Gatekeeper | Kyverno |
|---|---|---|---|
| Cài đặt | Built-in (Admission Controller) | `kubectl apply` manifest | Helm |
| Ngôn ngữ policy | YAML label trên namespace | Rego | YAML |
| Phạm vi | Chỉ Pod spec | Mọi loại resource | Mọi loại resource |
| Mutate | Không | Không | Có |
| Generate resource mới | Không | Không | Có |
| Audit resource đã tồn tại | Không | Có | Có |
| Độ khó học | Thấp | Cao (phải biết Rego) | Trung bình |

**Khuyến nghị production**: dùng PSA cho baseline bảo mật Pod (đơn giản, built-in, gán qua namespace label); dùng Kyverno cho policy tuỳ biến cần mutate/generate (auto-inject seccomp, auto-tạo NetworkPolicy...); cân nhắc OPA khi team đã quen Rego hoặc cần policy cross-resource phức tạp mà Kyverno khó biểu diễn.

## Pod Security Standards / Pod Security Admission (PSA)

### PSA thay thế PSP

Pod Security Admission (PSA) và Pod Security Standards được định nghĩa trong KEP-2579, ra đời để thay thế Pod Security Policy (PSP) — hướng tới an toàn để bật mặc định, dễ dùng, và mở rộng được. Các cấu hình phức tạp/tuỳ biến hơn có thể giao cho giải pháp bên thứ ba như Kyverno hay OPA Gatekeeper.

PSA hoạt động như một Admission Controller, **bật sẵn theo mặc định** trong cluster hiện đại — không cần thêm bước enable. Kiểm tra:

```bash
kubectl exec -n kube-system kube-apiserver-controlplane -it -- kube-apiserver -h | grep enable-admission-plugins
```

PSA áp dụng ở cấp **namespace** thông qua label.

### 3 profile bảo mật

| Profile | Mô tả |
|---|---|
| **Privileged** | Gần như không giới hạn, cho phép privilege escalation |
| **Baseline** | Cân bằng — ngăn escalation trái phép trong khi vẫn hỗ trợ hầu hết ứng dụng thông thường |
| **Restricted** | Siết chặt nhất theo pod-hardening best practice; bảo mật cao nhưng có thể gây khó tương thích |

### 3 mode xử lý vi phạm

| Mode | Hành vi khi vi phạm |
|---|---|
| **enforce** | Từ chối tạo Pod không tuân thủ |
| **audit** | Cho phép tạo Pod nhưng ghi log vi phạm vào audit log |
| **warn** | Cho phép tạo Pod nhưng cảnh báo user |

Ví dụ cấu hình 3 namespace khác nhau:

| Namespace | Mode | Profile |
|---|---|---|
| payroll | enforce | restricted |
| hr | enforce | baseline |
| dev | warn | restricted |

```bash
kubectl label ns payroll pod-security.kubernetes.io/enforce=restricted
kubectl label ns hr pod-security.kubernetes.io/enforce=baseline
kubectl label ns dev pod-security.kubernetes.io/warn=restricted
```

Khác biệt quan trọng so với PSP: **PSA/Pod Security Standards không mutate** (không tự thêm giá trị mặc định vào Pod spec) — nó chỉ validate/warn/audit, không sửa đổi request như PSP từng làm.

### So sánh chi tiết 3 profile theo từng control

| Control | Privileged | Baseline | Restricted |
|---|---|---|---|
| `hostPID`/`hostIPC`/`hostNetwork` | Cho phép | Cấm | Cấm |
| `hostPorts` | Cho phép | Cấm | Cấm |
| `privileged: true` | Cho phép | Cấm | Cấm |
| `allowPrivilegeEscalation` | Cho phép | Cho phép | Bắt buộc `false` |
| `capabilities.add` | Bất kỳ | Chỉ `NET_BIND_SERVICE` | Không được add (bắt buộc `drop: ["ALL"]`) |
| `runAsUser: 0` (root) | Cho phép | Cho phép | Cấm — bắt buộc `runAsNonRoot: true` |
| `seccompProfile` | Bất kỳ | Bất kỳ | Bắt buộc `RuntimeDefault` hoặc `Localhost` |
| `volumes` | Bất kỳ | Không `hostPath` | Chỉ `configMap`/`secret`/`projected`/`emptyDir`/`csi`/`persistentVolumeClaim`/`downwardAPI` |
| `seLinux`/`apparmor` override | Bất kỳ | Bất kỳ | Bất kỳ (không giới hạn thêm) |

Pod mẫu tuân thủ đầy đủ **Restricted**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault    # bắt buộc ở restricted
  containers:
    - name: app
      image: myapp:v1.2.3     # tag cố định, không dùng :latest
      securityContext:
        allowPrivilegeEscalation: false   # bắt buộc ở restricted
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]                   # bắt buộc ở restricted
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - mountPath: /tmp
          name: tmp-dir        # writable tmpfs thay vì ghi vào root filesystem
  volumes:
    - name: tmp-dir
      emptyDir: {}             # emptyDir được phép ở restricted
```

### Cấu hình PSA mặc định toàn cluster (AdmissionConfiguration)

Thay vì gán label cho từng namespace, có thể set **default profile/mode toàn cluster** qua file cấu hình admission controller — hữu ích khi muốn baseline chung và chỉ liệt kê namespace ngoại lệ:

```yaml
# /etc/kubernetes/admission/psa-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: "baseline"
        enforce-version: "latest"
        audit: "restricted"
        audit-version: "latest"
        warn: "restricted"
        warn-version: "latest"
      exemptions:
        # Namespace không bị PSA áp dụng
        namespaces:
          - kube-system
          - monitoring       # ví dụ: prometheus-node-exporter cần hostPID
        # RuntimeClass được miễn (ví dụ gVisor cần ngoại lệ so với restricted mặc định)
        runtimeClasses: []
        # User/ServiceAccount được miễn
        usernames: []
```

Nạp file này vào kube-apiserver qua flag:

```text
--admission-control-config-file=/etc/kubernetes/admission/psa-config.yaml
```

### Pod Security Policies (PSP) — kiến thức lịch sử

> PSP đã bị **deprecated từ Kubernetes 1.21** và **removed hoàn toàn từ 1.25**. Exam hiện tại không yêu cầu cấu hình PSP — chỉ cần hiểu khái niệm cơ bản để nhận diện câu hỏi liên quan cluster cũ.

Pod mẫu với cấu hình nguy hiểm (dùng để minh hoạ PSP sẽ chặn gì):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        privileged: True
        runAsUser: 0
        capabilities:
          add: ["CAP_SYS_BOOT"]
  volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

PSP hoạt động như một Admission Controller cần bật tường minh:

```text
--enable-admission-plugins=PodSecurityPolicy
```

Định nghĩa PSP disallow privileged container:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example-psp
spec:
  privileged: false
```

PSP nâng cao hơn — cấm chạy root, bắt buộc drop `CAP_SYS_BOOT`, thêm default capability, giới hạn loại volume:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example-psp
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  requiredDropCapabilities:
    - CAP_SYS_BOOT
  defaultAddCapabilities:
    - CAP_SYS_TIME
  volumes:
    - persistentVolumeClaim
```

Điểm khác biệt lớn: **PSP có thể mutate** Pod spec (thêm default capability) — đây là khả năng mà PSA/Pod Security Standards **không** có.

Để Pod (qua ServiceAccount) được phép "dùng" một PSP, cần Role + RoleBinding cấp quyền `use` trên chính PSP đó:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: psp-example-role
rules:
  - apiGroups: ["policy"]
    resources: ["podsecuritypolicies"]
    resourceNames: ["example-psp"]
    verbs: ["use"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-example-rolebinding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: Role
  name: psp-example-role
  apiGroup: rbac.authorization.k8s.io
```

Rủi ro vận hành lớn của PSP: nếu bật admission controller mà chưa có PSP + Role/RoleBinding đúng, **toàn bộ request tạo Pod có thể bị chặn** (kể cả của controller như Deployment) — đây chính là lý do PSP bị coi là phức tạp và khó vận hành, dẫn tới việc bị loại bỏ khỏi Kubernetes.

## Security Contexts

Tương tự flag Docker (`docker run --user=1001`, `docker run --cap-add MAC_ADMIN`), Kubernetes cho khai báo `securityContext` ở **cả cấp Pod lẫn cấp container**. Cấu hình ở Pod-level tự động áp dụng cho mọi container trong Pod; nếu container tự khai báo `securityContext` riêng thì **cấu hình container-level ghi đè (override) cấu hình pod-level**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

Lưu ý: `capabilities` chỉ có thể khai báo ở **container-level**, không có ở pod-level (vì Linux capabilities gắn với process/container, không có khái niệm ở cấp pod).

## Secrets Management

### Vấn đề: hardcode credential trong code

```python
import os
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def main():
    # Không an toàn: hardcode host/user/password
    mysql.connector.connect(host="mysql", database="mysql",
                            user="root", password="paswrd")
    return render_template('hello.html', color=fetchcolor())
```

Dữ liệu không nhạy cảm (hostname, tên service...) có thể để trong ConfigMap, nhưng **password/credential phải dùng Secret**.

### Tạo Secret — cách imperative

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswrd

# hoặc từ file
kubectl create secret generic app-secret --from-file=app_secret.properties

# các biến thể khác:
kubectl create secret generic my-secret --from-file=path/to/bar
kubectl create secret generic my-secret --from-file=ssh-privatekey=path/to/id_rsa --from-file=ssh-publickey=path/to/id_rsa.pub
kubectl create secret generic my-secret --from-file=ssh-privatekey=path/to/id_rsa --from-literal=passphrase=topsecret
kubectl create secret generic my-secret --from-env-file=path/to/foo.env --from-env-file=path/to/bar.env
```

### Tạo Secret — cách declarative (giá trị phải base64)

```bash
echo -n 'mysql' | base64     # bXlzcWw=
echo -n 'root' | base64      # cm9vdA==
echo -n 'paswrd' | base64    # cGFzd3Jk
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

```bash
kubectl create -f secret-data.yaml
```

Xem/giải mã Secret:

```bash
kubectl get secrets
kubectl describe secrets app-secret       # chỉ hiện kích thước, không lộ giá trị
kubectl get secret app-secret -o yaml     # hiện giá trị base64-encoded
echo -n 'bXlzcWw=' | base64 --decode      # -> mysql
```

### Inject Secret vào Pod

Cách 1 — làm biến môi trường (toàn bộ key-value trong Secret thành env var):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      envFrom:
        - secretRef:
            name: app-secret
```

Cách 2 — mount thành volume (mỗi key thành 1 file):

```yaml
volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```

```bash
ls /opt/app-secret-volumes
cat /opt/app-secret-volumes/DB_Password   # -> paswrd
```

### Secret chỉ encode, không encrypt — vì sao cần Encryption at Rest

Mặc định, Secret lưu trong etcd dưới dạng **base64 (chỉ encode, không mã hoá)**. Ai có quyền truy cập etcd trực tiếp đều đọc được dữ liệu gốc.

Kiểm tra dữ liệu thô trong etcd bằng `etcdctl` (API v3):

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C
```

Nếu chưa mã hoá, output sẽ lộ rõ giá trị plaintext của secret (ví dụ "supersecret"). Nếu thiếu `etcdctl`:

```bash
apt-get install etcd-client
```

### Bật Encryption at Rest cho Secret

Kiểm tra xem API server đã có flag `--encryption-provider-config` chưa (nếu chưa, mọi Secret vẫn chỉ base64):

```bash
ps aux | grep kube-api
```

Tạo file `enc.yaml` (cấu hình tối thiểu — 1 provider mã hoá + `identity` làm fallback):

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
  providers:
    - aescbc:
        keys:
          - name: key1
            secret: y0xTt+U6xgRdNxe4nDYYsij0GgRDoUYc+wAwOKeNfPs=
    - identity: {}
```

Key AES-CBC phải là chuỗi base64 của 32 byte ngẫu nhiên:

```bash
head -c 32 /dev/urandom | base64
```

Ví dụ đầy đủ hơn với nhiều provider hỗ trợ (aesgcm, aescbc, secretbox — có thể khai nhiều key để hỗ trợ key rotation):

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
providers:
  - identity: {}
  - aesgcm:
      keys:
        - name: key1
          secret: c2VjcmV0IGlzIHN1bXdlcnZlZQ==
        - name: key2
          secret: dGhpcPyBcyBwYXNzd29yZA==
  - aescbc:
      keys:
        - name: key1
          secret: c2VjcmV0IGlzIHN1bXdlcnZlZQ==
        - name: key2
          secret: dGhpcPyBcyBwYXNzd29yZA==
  - secretbox:
      keys:
        - name: key1
          secret: YWjZGVmZ2hpamtsbW5vcHyc3R1nd4eXokMjY=
```

Đưa file này vào cluster và cập nhật manifest kube-apiserver:

```bash
mkdir /etc/kubernetes/enc
mv enc.yaml /etc/kubernetes/enc/
```

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    volumeMounts:
    ...
    - name: enc
      mountPath: /etc/kubernetes/enc
      readOnly: true
  volumes:
  ...
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```

API server tự restart và nạp cấu hình mới (vì đây là static pod, kubelet phát hiện thay đổi manifest và tái tạo pod). Tạo secret mới để kiểm chứng đã mã hoá:

```bash
kubectl create secret generic my-secret-2 --from-literal=key2=topsecret
```

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret-2 | hexdump -C
```

Output sẽ không còn plaintext "topsecret" — xác nhận đã mã hoá.

**Secret cũ tạo trước khi bật encryption sẽ KHÔNG tự động được mã hoá** — phải ghi lại (rewrite) tường minh:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### Best practice khác cho Secrets

- Không commit file định nghĩa Secret vào version control (Git).
- Ai có quyền tạo Pod/Deployment trong cùng namespace có thể mount/đọc Secret trong namespace đó — RBAC phải siết chặt quyền tạo Pod, không chỉ quyền đọc Secret.
- Cân nhắc secret provider bên thứ ba (AWS/Azure/GCP Secrets provider, HashiCorp Vault) để lưu secret ngoài etcd hoàn toàn.

### Sealed Secrets — commit secret an toàn vào Git (GitOps)

Encryption at rest bảo vệ dữ liệu trong etcd, nhưng không giải quyết việc **commit file Secret vào Git** (base64 dễ decode). Sealed Secrets giải quyết đúng vấn đề này: mã hoá secret bằng public key ngay trên máy dev, chỉ controller trong cluster (giữ private key) mới decrypt được — file `SealedSecret` kết quả an toàn để commit vào Git.

```
Developer → kubeseal (public key) → SealedSecret (đã mã hoá)
                                           │ commit vào git
                                           ▼
                             sealed-secrets-controller decrypt
                                           │
                                           ▼
                                  K8s Secret (trong cluster)
```

```bash
# Cài controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Cài CLI kubeseal
brew install kubeseal

# Lấy public key từ cluster
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system > pub-cert.pem

# Seal một Secret — tạo ra file SealedSecret an toàn để commit
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-cert.pem --format yaml > db-creds-sealed.yaml

# Scope mặc định là namespace-scoped, có thể mở rộng cluster-wide
kubeseal --scope cluster-wide --cert pub-cert.pem ...
```

```yaml
# db-creds-sealed.yaml — an toàn để commit vào Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # đã mã hoá, không decode được nếu thiếu private key
  template:
    type: Opaque
```

**Hạn chế**: key rotation phức tạp — nếu rotate private key của controller, toàn bộ `SealedSecret` cũ phải được **re-seal lại** bằng public key mới, không tự động migrate.

### External Secrets Operator (ESO) — đồng bộ từ external secret store

ESO đồng bộ secret từ kho bên ngoài (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault) vào K8s Secret — đây là hướng tiếp cận **được khuyến nghị cho production** vì secret gốc không bao giờ nằm trong Git, và hỗ trợ auto-rotation.

```
External Store           ESO Controller              K8s Secret
(AWS SM, Vault)  ──►  SecretStore/ClusterSecretStore  ──►  ExternalSecret  ──►  Secret
                       (cấu hình auth)                     (khai secret nào cần sync)   (tự tạo/cập nhật)
```

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --set installCRDs=true
```

Ví dụ với AWS Secrets Manager (IRSA — IAM Role for Service Account trên EKS):

```yaml
# ClusterSecretStore — dùng chung cho nhiều namespace
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-sa
            namespace: external-secrets
---
# ServiceAccount gắn IAM Role qua IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-sa
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/ESO-Role
---
# ExternalSecret — khai secret nào cần sync và map field nào vào key nào
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h       # tự sync lại — cơ chế rotation
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-creds
    creationPolicy: Owner    # ESO sở hữu vòng đời Secret này
  data:
    - secretKey: password
      remoteRef:
        key: production/db/credentials
        property: password
    - secretKey: username
      remoteRef:
        key: production/db/credentials
        property: username
```

Ví dụ với HashiCorp Vault (auth qua Kubernetes ServiceAccount):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: https://vault.internal:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso-production
          serviceAccountRef:
            name: eso-sa
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault
    kind: ClusterSecretStore
  target:
    name: app-secrets
  dataFrom:
    - extract:
        key: production/myapp/secrets
```

`PushSecret` đảo ngược chiều đồng bộ — đẩy một K8s Secret hiện có lên external store (ví dụ để chia sẻ secret giữa nhiều cluster):

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-app-secret
  namespace: production
spec:
  updatePolicy: Replace
  secretStoreRefs:
    - name: aws-secrets-manager
      kind: ClusterSecretStore
  selector:
    secret:
      name: app-generated-secret
  data:
    - match:
        secretKey: api-key
        remoteRef:
          remoteKey: production/app/generated-secret
          property: api-key
```

### HashiCorp Vault — tích hợp trực tiếp (không qua K8s Secret)

Ngoài việc dùng ESO để đồng bộ vào K8s Secret, Vault còn hỗ trợ **Vault Agent Injector** — inject secret thẳng vào filesystem của Pod (tmpfs trong bộ nhớ) qua sidecar, hoàn toàn không tạo K8s Secret trung gian:

```
Pod → Vault Agent Sidecar → Vault Server
              │
              ▼
    /vault/secrets/config.txt (tmpfs, chỉ tồn tại trong bộ nhớ)
```

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  -n vault --create-namespace \
  --set "server.ha.enabled=true" \
  --set "server.ha.replicas=3" \
  --set "injector.enabled=true"

# Bật Kubernetes auth method
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Policy — giới hạn path được đọc
vault policy write myapp-policy - <<EOF
path "secret/data/production/myapp/*" {
  capabilities = ["read"]
}
EOF

# Role — bind policy vào đúng ServiceAccount/namespace
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h
```

Pod chỉ cần thêm annotation, sidecar tự inject secret — không cần thay đổi application code:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp-role"
    vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/production/myapp/config"
    vault.hashicorp.com/agent-inject-template-config.txt: |
      {{- with secret "secret/data/production/myapp/config" -}}
      DATABASE_URL=postgres://{{ .Data.data.username }}:{{ .Data.data.password }}@{{ .Data.data.host }}/mydb
      API_KEY={{ .Data.data.api_key }}
      {{- end }}
spec:
  serviceAccountName: myapp
  containers:
    - name: app
      image: myapp:v1
      command: ["sh", "-c", "source /vault/secrets/config.txt && ./myapp"]
```

**Vault Dynamic Secrets** đi xa hơn nữa: thay vì lưu một password DB tĩnh, Vault tự tạo credential DB **ngắn hạn, tự động rotate**:

```bash
vault secrets enable database

vault write database/config/myapp-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles=myapp-role \
  connection_url="postgresql://{{username}}:{{password}}@db.internal:5432/myapp" \
  username=vault-root \
  password=vault-root-pass

vault write database/roles/myapp-role \
  db_name=myapp-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl=1h \
  max_ttl=24h

# Vault tự tạo user DB mới với password ngắn hạn, tự revoke khi hết TTL
vault read database/creds/myapp-role
# → username: v-k8s-myapp-role-AbCdEf
# → password: A1B2C3D4-E5F6
# → lease_duration: 1h
```

### Secret Best Practices — RBAC theo secret cụ thể & rotation tự động

RBAC nên giới hạn quyền `get` chỉ trên **đúng tên Secret** cần thiết thay vì cấp quyền `list`/`get` toàn bộ Secret trong namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-creds"]    # chỉ đúng Secret này
    verbs: ["get"]
    # Không cấp: list, watch, create, update, delete
```

Secret cập nhật qua file mount tự động refresh trong Pod sau khoảng 1 phút (kubelet sync), nhưng **env var thì không tự update** — cần restart Pod. **Stakater Reloader** tự động phát hiện Secret/ConfigMap thay đổi và rolling-restart Deployment liên quan:

```yaml
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "db-creds,api-keys"
```

```bash
helm install reloader stakater/reloader -n kube-system
# Sau khi Secret thay đổi → Deployment tương ứng tự động rolling restart
```

### So sánh các phương án quản lý Secret

| Phương án | Độ phức tạp | Bảo mật | Rotation | GitOps-friendly |
|---|---|---|---|---|
| K8s Secret (thuần) | Thấp | Thấp (chỉ base64) | Thủ công | Không |
| K8s Secret + encryption at rest | Trung bình | Trung bình | Thủ công | Không |
| Sealed Secrets | Thấp | Trung bình | Re-seal thủ công | Có |
| ESO + AWS SM/GCP SM | Trung bình | Cao | Tự động | Một phần |
| ESO + Vault | Cao | Rất cao | Tự động + dynamic | Một phần |
| Vault Agent Injector | Cao | Rất cao | Tự động + dynamic | Có |

**Khuyến nghị production**: trên EKS ưu tiên ESO + AWS Secrets Manager + IRSA (không cần quản lý key thủ công); tự vận hành hạ tầng thì ESO + Vault (có HA, audit trail, dynamic secrets); nếu ưu tiên GitOps đơn giản, Sealed Secrets hoặc ESO với `ExternalSecret` YAML commit vào Git (không chứa giá trị secret thật) đều phù hợp.

## TLS / mTLS giữa các Pod

### One-way SSL vs Mutual TLS

| Đặc điểm | One-way SSL | Mutual TLS (mTLS) |
|---|---|---|
| Xác thực certificate | Chỉ client verify certificate của server | Cả hai bên verify certificate lẫn nhau |
| Use case điển hình | Dịch vụ web hướng người dùng (online banking, email, mạng xã hội) | Giao tiếp hệ thống-với-hệ thống tự động (B2B, service-to-service) |
| Mức bảo mật | An toàn cho kênh truyền; xác thực user xử lý riêng (password...) | Bảo mật cao hơn nhờ xác thực hai chiều |

**One-way SSL** — luồng xử lý:
1. Client nhận certificate công khai của server.
2. Browser verify certificate qua CA trong trust store.
3. Browser dùng public certificate của server để mã hoá symmetric key, gửi cho server.
4. Server giải mã symmetric key bằng private key riêng.

Chỉ server được xác thực; danh tính client dựa vào cơ chế khác (username/password...).

**Mutual TLS** — luồng xử lý (ví dụ client `mybank.com` gọi server `abc-financials`):
1. Client yêu cầu certificate của server.
2. Server trả certificate, đồng thời yêu cầu ngược lại certificate của client.
3. Client verify certificate server qua CA trong trust store.
4. Client gửi certificate của mình kèm symmetric key đã mã hoá bằng public key server.
5. Server verify certificate client qua CA để xác nhận đúng là `mybank.com`.

Sau khi cả hai xác thực lẫn nhau, dùng chung symmetric key để mã hoá phần còn lại của phiên giao tiếp.

### Pod-to-Pod Encryption — vì sao cần

Mặc định giao tiếp giữa các Pod trong Kubernetes **không mã hoá**. Nếu dữ liệu nhạy cảm (số thẻ, thông tin cá nhân) truyền dạng plaintext giữa front-end và back-end, kẻ tấn công thực hiện man-in-the-middle có thể đọc/sửa dữ liệu. Mã hoá pod-to-pod còn hỗ trợ tuân thủ GDPR/HIPAA, giảm rủi ro insider threat, và phù hợp mô hình zero-trust (mọi kết nối coi là không tin cậy cho tới khi được xác thực).

Ba hướng triển khai phổ biến:

| Phương pháp | Cơ chế |
|---|---|
| Mutual TLS (mTLS) | Thường qua service mesh: Istio, Linkerd |
| Cilium Encryption | IPsec hoặc WireGuard |
| Calico Encryption | IPsec |

### Triển khai mTLS bằng Service Mesh (Istio)

Áp dụng mTLS thủ công cho từng service (kiểu MySQL tự bật SSL) dễ dẫn tới không đồng nhất thuật toán/cấu hình giữa các service. Giải pháp phổ biến hơn: **service mesh** (Istio, Linkerd) đảm nhiệm mã hoá ở network layer, tách khỏi application code, tự động deploy sidecar container cạnh mỗi Pod.

Luồng xác thực mTLS giữa 2 Pod (qua sidecar):
1. Pod A yêu cầu certificate của Pod B.
2. Pod B trả certificate, đồng thời yêu cầu certificate của Pod A.
3. Sau khi verify certificate của B, Pod A gửi certificate của mình kèm symmetric key.
4. Sau khi verify certificate của A, cả hai chuyển sang dùng symmetric key để mã hoá giao tiếp.

Istio hỗ trợ 2 mode:

| Mode | Hành vi |
|---|---|
| **Permissive (Opportunistic)** | Mã hoá khi có thể; vẫn cho phép traffic không mã hoá với service ngoài/không hỗ trợ mTLS |
| **Strict (Enforced)** | Bắt buộc toàn bộ traffic dùng mTLS — mọi service tham gia đều phải hỗ trợ mTLS, nếu không sẽ mất kết nối |

Sidecar Istio chặn traffic gửi đi, mã hoá; sidecar bên nhận giải mã trước khi chuyển cho ứng dụng — hoàn toàn trong suốt với code ứng dụng.

### Vấn đề service-to-service trust khi chưa có mTLS

```
Không có mTLS:
frontend → backend (plaintext hoặc TLS một chiều)
Attacker đứng trên network của cluster → intercept/MITM → đọc được traffic

Có mTLS:
frontend → backend (mutual TLS)
- frontend verify certificate của backend
- backend verify certificate của frontend
- traffic mã hoá end-to-end
- xác thực dựa trên identity (không chỉ dựa vào việc "traffic tới từ trong cluster")
```

### PeerAuthentication — bật mTLS theo namespace

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT    # STRICT: chỉ chấp nhận traffic mTLS
                     # PERMISSIVE: chấp nhận cả mTLS lẫn plaintext (dùng khi đang migrate)
                     # DISABLE: tắt mTLS
```

### AuthorizationPolicy — RBAC service-to-service theo identity

Ngoài mã hoá, mTLS còn cho phép áp policy dựa trên **identity của service gọi tới** (principal) thay vì chỉ dựa vào network — ví dụ chỉ cho phép `frontend` ServiceAccount gọi `backend`:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"    # chỉ cho phép ServiceAccount frontend
    - to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

### Kiểm chứng bằng istioctl

```bash
# Xem trạng thái mTLS của một Pod
istioctl x describe pod <pod-name> -n production
# → mTLS: Yes

# Xem certificate đang dùng
istioctl proxy-config secret <pod-name> -n production

# Test AuthorizationPolicy — request từ service được phép
kubectl exec -n production deployment/frontend -- \
  curl http://backend:8080/api/v1/data
# → 200 OK

# Test từ service không nằm trong danh sách principals — phải bị chặn
kubectl exec -n production deployment/some-other-service -- \
  curl http://backend:8080/api/v1/data
# → 403 Forbidden
```

## Cilium (CNI / eBPF)

### Kiến trúc Cilium

Cilium là CNI dựa trên **eBPF** — chạy trực tiếp trong Linux kernel để đạt hiệu năng cao với overhead thấp. Ngoài việc quản lý pod networking, Cilium cung cấp:

- **Network Policies**: kiểm soát giao tiếp pod-to-pod, chỉ cho phép traffic hợp lệ.
- **Services & Load Balancing**: định tuyến và cân bằng tải hiệu quả.
- **Bandwidth Management**: giới hạn băng thông theo pod, tránh nghẽn mạng.
- **Flow & Policy Logging**: theo dõi traffic thời gian thực, quan sát việc enforce policy.
- **Security & Operational Metrics**: visibility sâu về security posture và hiệu năng cluster.

### Vì sao chọn Cilium thay vì NetworkPolicy chuẩn?

`NetworkPolicy` chuẩn (dùng CNI bất kỳ) chỉ hoạt động ở tầng L3/L4 (IP, port) — đủ cho phần lớn nhu cầu cô lập cơ bản nhưng không đáp ứng được các yêu cầu bảo mật tinh hơn:

| Khả năng | NetworkPolicy chuẩn | Cilium |
|---|---|---|
| L3/L4 (IP, port) | Có | Có |
| L7 (HTTP method/path, gRPC service, Kafka topic) | Không | Có |
| Policy dựa trên DNS/FQDN | Không | Có |
| Egress gateway cho service ngoài cluster | Không | Có |
| Mã hoá trong suốt (WireGuard) | Không | Có |
| Network observability (Hubble) | Không | Có |

### Cilium Pod-to-Pod Encryption

Cilium mã hoá traffic pod-to-pod trực tiếp qua eBPF — hoàn toàn trong suốt với application code (không cần đổi code app). Các điểm nổi bật:

- **Flexible Encryption Options**: hỗ trợ nhiều thuật toán mã hoá.
- **End-to-End Security**: mã hoá xuyên suốt hành trình traffic trong mạng.
- **Policy-Driven Control**: định nghĩa policy mã hoá theo namespace/label.

Các bước thiết lập tổng quát: (1) Cài đặt Cilium (Helm hoặc manifest chuẩn) → (2) Bật tính năng encryption sau khi cài → (3) Quản lý key (dùng cơ chế key management gốc của Cilium hoặc tích hợp hạ tầng sẵn có) → (4) Định nghĩa policy mã hoá theo namespace/label.

### Viết CiliumNetworkPolicy cho encrypted traffic

`CiliumNetworkPolicy` tương tự NetworkPolicy chuẩn nhưng thêm khả năng kiểm soát traffic đã mã hoá. Ví dụ policy chỉ cho phép traffic egress port 80/TCP giữa các Pod cùng label `app: myapp` (traffic này sẽ được Cilium mã hoá nếu encryption đã bật ở cấp cluster):

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-encrypted-traffic
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  egress:
  - toEndpoints:
    - matchLabels:
        app: myapp
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

Kiểm chứng traffic đã mã hoá bằng `tcpdump` trong Pod:

```bash
kubectl exec -it <pod-name> -- /bin/bash
apt-get update && apt-get install -y tcpdump
tcpdump -i eth0 -nn
```

Nếu mã hoá hoạt động đúng, packet capture sẽ không hiển thị payload dạng plaintext.

### CiliumNetworkPolicy tầng L7 — kiểm soát theo HTTP method/path

Vì hoạt động ở tầng L7, Cilium có thể chỉ allow đúng method + path cụ thể thay vì mở toàn bộ port — ví dụ chỉ cho phép `GET /api/*` và `POST /api/v1/orders`, mọi request khác (kể cả trên đúng port 8080) đều bị deny:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: /api/.*
              - method: POST
                path: /api/v1/orders
              # Mọi path/method khác đều bị deny
```

### DNS-based egress policy (`toFQDNs`)

Khi cần cho phép egress tới dịch vụ ngoài cluster nhưng IP của dịch vụ đó có thể đổi (CDN, SaaS API), dùng `toFQDNs` thay vì `ipBlock` — Cilium tự resolve DNS và cập nhật rule theo IP hiện tại:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  egress:
    - toFQDNs:
        - matchName: api.payment-gateway.com
        - matchPattern: "*.stripe.com"
      toPorts:
        - ports:
            - port: "443"
```

### Hubble — network observability

Hubble là thành phần quan sát traffic tích hợp sẵn với Cilium — cho phép xem real-time flow, tìm nhanh traffic bị NetworkPolicy chặn mà không cần `tcpdump` thủ công.

```bash
# Bật Hubble relay + UI khi cài/upgrade Cilium
helm upgrade cilium cilium/cilium --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Port-forward để mở Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# CLI — xem 100 flow gần nhất trong namespace production
hubble observe --namespace production --last 100

# Chỉ xem flow bị DROP (vi phạm NetworkPolicy)
hubble observe --namespace production --verdict DROPPED

# Trích xuất nhanh src/dst/port của các flow bị drop để tìm NetworkPolicy đang chặn nhầm
hubble observe --namespace production \
  --verdict DROPPED \
  --output json | jq '{src: .source.pod_name, dst: .destination.pod_name, port: .l4.TCP.destination_port}'
```

## Resource Management

### Quality of Service (QoS) classes

Kubernetes tự động gán QoS class cho Pod dựa trên cách khai báo `requests`/`limits` — **không có field để set trực tiếp QoS**.

| QoS class | Điều kiện | Hành vi khi thiếu tài nguyên |
|---|---|---|
| **Guaranteed** | `requests == limits` cho **cả** CPU và memory, ở **mọi** container trong Pod | Không bị evict trừ khi node bất ổn |
| **Burstable** | `requests < limits` (có set ít nhất một trong hai) | Đảm bảo mức tối thiểu, được burst thêm nếu còn tài nguyên; evict sau Best-Effort |
| **Best-Effort** | Không set `requests`/`limits` | Không đảm bảo gì, evict đầu tiên khi thiếu tài nguyên |

Ví dụ Guaranteed:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
  namespace: namespace-a
spec:
  containers:
    - name: critical-container
      image: nginx
      resources:
        requests:
          memory: "500Mi"
          cpu: "500m"
        limits:
          memory: "500Mi"
          cpu: "500m"
```

Ví dụ Burstable:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-app
  namespace: namespace-b
spec:
  containers:
    - name: burstable-container
      image: nginx
      resources:
        requests:
          memory: "200Mi"
          cpu: "200m"
        limits:
          memory: "16Gi"
          cpu: "1"
```

### Network QoS (qua CNI, không phải tính năng lõi K8s)

Kubernetes không có Network QoS built-in — cần CNI hỗ trợ (Calico) hoặc Linux traffic control. Ví dụ giới hạn băng thông bằng Calico NetworkPolicy:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: tenant-a-network-policy
  namespace: namespace-a
spec:
  selector: all()
  ingress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [80]
      limits:
        rate: 10Mbps
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports: [80]
      limits:
        rate: 10Mbps
```

### Storage QoS

Kubernetes không kiểm soát trực tiếp IOPS — dựa vào StorageClass + khả năng QoS của storage backend bên dưới:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

### ResourceQuota — giới hạn tài nguyên theo namespace

`ResourceQuota` giới hạn tổng tài nguyên (CPU, memory, storage...) mà một namespace được dùng — ngăn một tenant chiếm dụng quá nhiều tài nguyên cluster.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-memory-quota
  namespace: namespace_a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 16Gi
    limits.cpu: "8"
    limits.memory: 32Gi
```

Pod tuân thủ giới hạn trên:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limited-pod
  namespace: namespace_a
spec:
  containers:
  - name: resource-limited-container
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
```

Lưu ý: khi namespace đã có `ResourceQuota` áp cho `requests.cpu`/`requests.memory`, mọi Pod tạo trong namespace đó **bắt buộc phải khai báo** `requests`/`limits` tương ứng — nếu không, request tạo Pod sẽ bị admission reject. Kết hợp với `LimitRange` để tự động gán giá trị mặc định cho Pod không khai báo tường minh.

## Gotchas

- **PSP đã bị deprecated (1.21) và removed hoàn toàn (1.25)** — CKS hiện tại chỉ kiểm tra Pod Security Admission/Pod Security Standards; học PSP chỉ để hiểu bối cảnh lịch sử, không cần thuộc chi tiết field.
- Khác biệt cốt lõi PSP vs PSA: **PSP có thể mutate** Pod spec (ví dụ `defaultAddCapabilities`), còn **PSA/Pod Security Standards chỉ validate/warn/audit, không mutate**.
- `EncryptionConfiguration`: thứ tự `providers` quan trọng — provider **đầu tiên** trong danh sách được dùng để mã hoá dữ liệu **mới**; nếu đặt `identity` (không mã hoá) lên đầu, dữ liệu mới sẽ **không** được mã hoá dù có khai `aescbc`/`aesgcm` phía sau.
- Bật `EncryptionConfiguration` **không tự động mã hoá lại Secret cũ** đã tồn tại trước đó — phải chạy `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` để rewrite.
- Key AES-CBC/AES-GCM trong `EncryptionConfiguration` phải base64-encode đúng 32 byte (`head -c 32 /dev/urandom | base64`); sai độ dài sẽ khiến API server không khởi động được.
- Cập nhật flag `--encryption-provider-config` trên static pod kube-apiserver: nhớ mount đúng `volumeMounts`/`volumes` trỏ tới `hostPath` chứa file `enc.yaml`, nếu không API server sẽ crash loop.
- Thứ tự admission controller: **mutating luôn chạy trước validating** — mutation phải hoàn tất trước khi validate, nếu không các object được mutate (như PVC được gán default storage class) có thể bị validate sai.
- `NamespaceAutoProvision` + `NamespaceExists` đã bị thay bằng **`NamespaceLifecycle`** (một controller duy nhất, vừa reject namespace không tồn tại vừa bảo vệ namespace hệ thống khỏi bị xoá).
- OPA Gatekeeper: phải apply **`ConstraintTemplate` trước, `Constraint` sau** — `Constraint.kind` phải khớp với `crd.spec.names.kind` khai trong `ConstraintTemplate`; `parameters` trong Constraint được đọc trong Rego qua `input.parameters`.
- `runtimeClassName` trên Pod chỉ hoạt động nếu: (1) object `RuntimeClass` tương ứng đã tồn tại trong cluster, và (2) node được schedule tới đã cài đặt/cấu hình đúng runtime handler (`runsc` cho gVisor, `kata` cho Kata Containers) trong containerd/CRI-O — thiếu một trong hai, Pod sẽ lỗi khi tạo/chạy.
- Kata Containers cần **hardware virtualization** — chạy trên cloud VM thông thường nghĩa là nested virtualization, phần lớn provider không hỗ trợ (trừ một số ngoại lệ như GCP, cần cấu hình thủ công); ưu tiên bare-metal khi có thể.
- seccomp mặc định theo mô hình **whitelist** (`SCMP_ACT_ERRNO` = deny tất cả trừ khi allow tường minh), AppArmor thường dùng theo mô hình **blacklist** (allow mặc định, chặn hành vi cụ thể) — dễ nhầm chiều mặc định giữa hai công cụ.
- QoS class **không thể set trực tiếp** — Kubernetes tự suy ra từ `requests`/`limits`; muốn Guaranteed, **cả** CPU và memory của **mọi** container trong Pod phải có `requests == limits`, thiếu một container không khai báo cũng làm cả Pod tụt xuống Burstable.
- Khi namespace có `ResourceQuota` áp cho `requests.cpu`/`requests.memory`, Pod **không khai báo** `requests`/`limits` sẽ bị admission controller từ chối tạo — kết hợp `LimitRange` để cấp default tự động.
- `fallthrough in-namespace` trong CoreDNS chỉ chặn **DNS resolution** xuyên namespace — không tự động chặn kết nối mạng trực tiếp qua IP; muốn cô lập network thật sự vẫn cần NetworkPolicy riêng.
- Toleration chỉ **cho phép** Pod được schedule lên node đã taint, **không ép buộc** Pod phải chạy ở đó — muốn đảm bảo Pod chỉ chạy trên node dành riêng, phải kết hợp thêm `nodeSelector`/`nodeAffinity`.
- Dễ nhầm giữa hai cơ chế "priority": **API Priority and Fairness** (`PriorityLevelConfiguration` + `FlowSchema`) kiểm soát cách **API server** xử lý request, còn **Pod Priority and Preemption** (`PriorityClass` + `priorityClassName`) kiểm soát **scheduling/eviction** Pod ở node — hai cơ chế độc lập, không thay thế nhau được.
- Istio **Strict mTLS mode** yêu cầu **toàn bộ** service trong mesh phải hỗ trợ mTLS — bật strict mode khi còn service chưa tương thích sẽ làm mất kết nối; nên test kỹ trước khi chuyển từ Permissive sang Strict.
- Security context ở **container-level ghi đè (override) pod-level**; field `capabilities` chỉ tồn tại ở container-level, không có ở pod-level.
- Secret trong Kubernetes chỉ **base64-encode**, không mã hoá — ai đọc được YAML/etcd đều decode được; đồng thời bất kỳ ai có quyền **tạo Pod** trong namespace đó cũng có thể mount và đọc được Secret trong namespace — kiểm soát chặt quyền tạo Pod, không chỉ quyền đọc Secret.
- PSA **mặc định miễn trừ (exempt) namespace `kube-system`** — nhiều Pod hệ thống (fluentd, node-exporter, CNI plugin) cần `hostPID`/`privileged` nên không thể ép `restricted`; quên khai exemption tương tự cho namespace hệ thống khác có thể làm cluster hỏng.
- Gatekeeper vừa validate lúc admission vừa audit định kỳ resource đã tồn tại, nhưng **audit không hồi tố** — resource tạo trước khi Constraint tồn tại không tự bị chặn hay xoá, chỉ được liệt kê là vi phạm trong `status`.
- Kyverno: `background: true` scan cả resource đã tồn tại (audit), `background: false` chỉ kiểm tra lúc admission — nhầm giữa hai chế độ dễ khiến resource vi phạm cũ "lọt lưới" vì tưởng đã được quét.
- ESO: khi xoá `ExternalSecret`, K8s Secret tương ứng **mặc định vẫn tồn tại** trừ khi `creationPolicy: Owner` ràng buộc vòng đời — cần rà soát để tránh secret mồ côi (orphan) còn sót lại trong cluster.
- Vault Agent tự renew dynamic secret trước khi hết hạn, nhưng nếu Vault server không reachable thì lease hết hạn mà không renew được → credential invalid → ứng dụng lỗi; Vault production cần chạy **HA** để tránh single point of failure này.
- Private key của **Sealed Secrets controller** là tài sản phải backup cẩn thận — mất private key đồng nghĩa **không thể decrypt** bất kỳ `SealedSecret` nào đã commit trước đó, buộc phải re-seal lại toàn bộ secret với key mới.
- Audit log Kubernetes ở level `RequestResponse` ghi lại **toàn bộ body request/response**, kể cả giá trị base64-encoded của Secret khi tạo/update — vô tình làm rò rỉ secret vào audit log; nên dùng level `Request` hoặc `Metadata` cho resource `secrets`.
