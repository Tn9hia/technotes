# Talos Linux — v1.14.x
Tags: #k8s-provider #talos #node-os #bootstrap-provider #control-plane-provider
Related: [[CAPI]], [[Image-Pipeline]], [[Fleet-Ops]], [[Provider-Security]]
Last updated: 2026-10-03

> ⚠️ **Version note — CẦN VERIFY LẠI**: Số version (v1.14.2, 29/09/2026; v1.14.0, 04/09/2026; v1.13.0, 27/04/2026) lấy qua search engine, chưa đối chiếu trực tiếp CHANGELOG gốc. Verify bằng `talosctl version` hoặc https://github.com/siderolabs/talos/releases trước khi dùng để quyết định production.
>
> Tháng 9/2026, Talos công bố **một loạt advisory liên quan privilege-escalation/role bypass của chính API quản trị** (4 CVE cùng ngày 18/09). Chi tiết patched-version của từng CVE **chưa fetch được trực tiếp** từ GitHub (tool fetch trả 404) — trước khi áp dụng bất kỳ nhận định bảo mật nào trong note này vào hệ thống thật, mở trực tiếp https://github.com/siderolabs/talos/security/advisories để xác nhận lại.

---

### **1. What — Nó là cái gì?**

Talos Linux là Linux distro tối giản (chỉ khoảng 12 binary, **không shell, không SSH, không package manager**), thiết kế **chỉ để chạy Kubernetes** — toàn bộ OS là 1 SquashFS read-only image, mọi quản trị đi qua 1 gRPC API duy nhất (`talosctl`), không có cách truy cập node nào khác. Trong stack CAPI của mình, Talos đóng **2 vai trò provider cùng lúc**: Bootstrap Provider (CABPT — sinh machine config) và Control Plane Provider (CACPPT — quản lý lifecycle control-plane) → xem [[CAPI]].

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có Talos, để có node OS cho K8s phải chọn 1 trong các hướng kém hơn:
- **Ubuntu/Rocky + kubeadm thủ công/Ansible** — có package manager, SSH, shell ⇒ attack surface lớn hơn nhiều (phải tự patch CVE OS-level tách biệt khỏi K8s, user vẫn login sửa tay được gây config drift), và tự lo CIS hardening từ đầu.
- **Flatcar/Bottlerocket** — cũng minimal/immutable, nhưng không có 1 Control Plane Provider chính thức, cùng hệ sinh thái với bootstrap provider như Talos (Talos là 1 trong số ít OS thiết kế "API-only" mà chính công ty làm OS — Sidero Labs — cũng maintain luôn CAPI provider cho nó).
- **Tự build minimal OS (buildroot/custom)** — kiểm soát tối đa, nhưng tốn effort duy trì vá CVE kernel/runtime, Talos đã làm sẵn phần này và release đều.

Talos lấp khoảng trống: giảm attack surface node xuống mức tối thiểu (immutable, API-driven, không SSH) đồng thời expose đúng API cần cho automation — CAPI tạo/join/xoá node hoàn toàn qua machine config, không cần SSH vào máy làm bất cứ gì. Đây khớp hoàn hảo với mô hình "node là cattle, không phải pet" mà 1 KaaS provider cần để vận hành hàng loạt node của nhiều tenant mà không scale theo số người vận hành.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** muốn giảm attack surface + ops overhead ở tầng OS (không tự patch CVE OS riêng biệt khỏi K8s); build fleet lớn cần node hoàn toàn đồng nhất, reproducible (image + machine config quyết định toàn bộ, không có config drift qua SSH); đã chấp nhận quản trị 100% qua API/CAPI, không cần shell để debug thông thường.

**KHÔNG dùng khi:**
- Cần chạy agent/tooling đòi hỏi cài trực tiếp lên host theo kiểu truyền thống (SSH + package manager) — Talos chỉ cho thêm qua system extension ở **build-time** (Image Factory), không cài runtime được.
- Team chưa quen debug qua `talosctl` thay vì SSH — cần đổi thói quen vận hành, có learning curve thật dù tài liệu khá đầy đủ.
- Cần hardware/driver đặc thù (vd 1 số NIC/card riêng trên host KVM) chưa có extension tương ứng trên Image Factory và tự build extension custom chưa khả thi trong timeline — **cần verify** tình trạng hỗ trợ hardware cụ thể trước khi cam kết.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌───────────────────── Talos Node (control-plane hoặc worker) ──────────────────────┐
│                                                                                      │
│  SquashFS read-only root (/) — immutable, không SSH, không shell, không pkg mgr     │
│                                                                                      │
│  machined (PID 1)                                                                   │
│   - controller runtime: reconcile loop (actual state ↔ machine config desired state)│
│   - quản lý mọi system service: networking, kubelet, containerd, etcd (CP only)...  │
│        │                                                                             │
│        ▼                                                                             │
│  apid (gRPC API gateway, port 50000)  ◀── mTLS, client cert gắn "role"               │
│        ▲                                   (os:admin / os:reader / ... — xem mục 6)  │
│        │   cách DUY NHẤT để quản trị node                                           │
│  talosctl (máy quản trị, dùng talosconfig = client cert + CA)                        │
│                                                                                       │
│  Partition: EFI/BOOT (A/B, atomic upgrade) │ META │ STATE (machine config+secrets,   │
│             nhạy cảm nhất) │ EPHEMERAL (container image, etcd data, kubelet, log)    │
└───────────────────────────────────────────────────────────────────────────────────┘
          ▲
          │ machine config (YAML) — apply lúc boot, hoặc `talosctl apply-config`
          │ khi dùng CAPI: do CABPT tự sinh từ TalosControlPlane/TalosConfigTemplate
          ▼
   Management Cluster (CAPI core + CABPT + CACPPT) → [[CAPI]]
```

Dependency: cần Image Factory (hoặc tự host bản offline) để build image kèm extension đúng cho KVM/CloudStack/OpenStack trước khi node boot lần đầu — bước build-time này nằm ở nhóm [[Image-Pipeline]], không phải trong phạm vi note này (note này chỉ bàn OS runtime).

### **5. How — Cơ chế hoạt động**

- **`machined` (PID 1)** — trung tâm của Talos: boot, controller runtime (reconcile loop giữ actual state khớp machine config desired state), quản lý toàn bộ system service.
- **`apid`** — gRPC API gateway (port 50000), **cách duy nhất** để tương tác với node — không SSH/console nào khác tồn tại. Request đi qua `talosctl`, xác thực bằng client cert mTLS gắn 1 "role" (`os:admin`, `os:reader`...) quyết định hành động được phép. ⚠️ **Cần verify** danh sách role đầy đủ + quyền tương ứng tại version đang dùng thật — chưa tra kỹ bảng role-permission.
- **Machine Config** — 1 YAML document duy nhất định nghĩa toàn bộ node (network, disk, cluster join info, kubelet config...). Khi dùng CAPI, document này **không viết tay** — CABPT tự sinh từ `TalosControlPlane`/`TalosConfigTemplate`. Chi tiết luồng sinh/patch machine config qua CAPI đáng tách riêng khi thực hành thật → [[talos--capi-integration]] (chưa viết).
- **Partition layout** — `EFI`/`BOOT` (A/B, cho upgrade atomic — cài version mới vào partition không active, reboot vào đó, rollback tự động nếu fail); `META` (metadata nhỏ, đọc lúc boot); `STATE` (machine config đã apply + secrets/certificate — **phần nhạy cảm nhất của node**); `EPHEMERAL` (container image, etcd data, kubelet state, container log — phần chiếm dung lượng lớn nhất). Mỗi partition encrypt độc lập được (LUKS2) — xem mục 6/7.
- **Image Factory** — service build image Talos theo "schematic" (YAML khai báo system extension cần, vd driver GPU, `gvisor`, firmware...). Schematic ID là hash **content-addressable** của nội dung schematic — cùng input luôn ra cùng ID, reproducible. Đây là bước build-time trước khi image được đưa vào pipeline publish sang CloudStack Template/OpenStack Glance → [[Image-Pipeline]].
- **Upgrade** — atomic qua A/B boot partition, khác hoàn toàn cơ chế package-upgrade-từng-phần của distro thường. `talosctl upgrade` (đổi version Talos/OS) và `talosctl upgrade-k8s` (đổi version Kubernetes) là **2 lệnh, 2 quy trình độc lập** — dễ nhầm lẫn, xem mục 9.

### **6. Key Config — Cấu hình cần nhớ**

- Machine config có 2 nhóm field cần phân biệt: `.machine` (OS-level: network, disk, install) và `.cluster` (K8s-level: control-plane endpoint, CNI, token). Không phải mọi field đều hot-apply được — sửa nhầm field cần reboot/reset tưởng là apply live (`talosctl edit machineconfig` chỉ áp field hỗ trợ hot-apply, field khác im lặng chờ reboot hoặc lỗi) là nguồn nhầm lẫn phổ biến.
- **`EPHEMERAL` partition mặc định KHÔNG encrypt** — chứa etcd data + kubelet state + container log, là nơi dễ lộ dữ liệu nhạy cảm nhất nếu disk image bị lấy ra khỏi host (đáng chú ý khi chạy trên KVM với disk nằm trên shared storage của CloudStack/OpenStack — storage admin hoặc backup snapshot có thể đọc được). Phải tự enable `VolumeConfig` với KMS hoặc static key nếu cần.
- **KMS-based disk encryption**: node phải **reach được KMS lúc boot** để decrypt `STATE`/`EPHEMERAL` — KMS down = node không boot được. Trade-off thật: an toàn hơn static key, nhưng thêm 1 single point of failure vào đường boot, cần tính vào thiết kế HA của hạ tầng cung cấp KMS (đặc biệt nếu KMS nằm ngoài vùng mạng CloudStack/OpenStack).
- `ImageVerificationConfig` (feature mới từ v1.13, verify signature image container ở machine-wide level) có liên quan 1 advisory bypass (`GHSA-cfwm-jp72-j437`, 18/09/2026, Moderate — bypass qua mismatch cách normalize image reference) — nếu bật tính năng này, **cần verify đã patch** trước khi coi là enforce thật.
- `talosctl upgrade` cần đủ dung lượng trống cho partition B trong lúc upgrade (cơ chế A/B) — disk sát định mức ngay từ lúc tạo template CloudStack/OpenStack (vd set disk size tối thiểu để tiết kiệm) dễ khiến upgrade fail giữa đường.

### **7. Security Considerations**

- **Attack surface chính**: do không SSH/shell, mọi quyền truy cập node quy về đúng 1 điểm — `apid`/gRPC API. Quản lý `talosconfig` (chứa client cert + CA) phải nghiêm ngặt tương đương SSH private key của root trước đây: mất file này gần như mất quyền điều khiển node tương ứng.
- **Loạt CVE/advisory đáng chú ý** (⚠️ patched-version cụ thể của mỗi CVE **chưa verify trực tiếp**, xem banner đầu file):
  - `GHSA-rjwj-368c-f82r` (18/09/2026, **Critical**) — privilege escalation lên **Talos admin** qua "role-less certificate" + tiêm header `talos-role` — cho thấy ngay cơ chế phân quyền theo role của API cũng từng bị bypass hoàn toàn.
  - `GHSA-p28g-gwwg-jx6w` (18/09/2026, Moderate) — privilege escalation từ role `os:meta:writer` qua key META của staged-upgrade không được bảo vệ đúng.
  - `GHSA-m38g-vww2-mvgx` (01/05/2026, High) — local privilege escalation từ **untrusted workload** (1 Pod không có quyền đặc biệt vẫn leo thang ảnh hưởng tới host) — nhắc rằng "immutable OS" không đồng nghĩa container escape bất khả thi, container runtime vẫn là attack surface thật (xem thêm CVE-2024-21626, runc escape, patch ở Talos v1.5.6/v1.6.4).
  - `GHSA-cfwm-jp72-j437` (18/09/2026, Moderate) — bypass `ImageVerificationConfig`.
  - Lịch sử: `CVE-2022-36103` — worker join token từng bị lợi dụng để lấy quyền API cao hơn dự kiến, minh chứng sớm rằng role model của Talos API cần audit kỹ, không "trust by default".
- **Hardening tối thiểu**: (1) encrypt cả `STATE` + `EPHEMERAL` (ưu tiên KMS nếu đã có hạ tầng key management sẵn, static key nếu chưa); (2) phát `talosconfig` theo role hẹp nhất đủ dùng — không phát `os:admin` tràn lan cho CI/CD/automation; (3) theo dõi sát `github.com/siderolabs/talos/security/advisories` — mật độ CVE liên quan role/privilege-escalation riêng trong 09/2026 cho thấy đây là mảng đang bị soi tích cực, bản patch mới gần như luôn đáng áp ngay; (4) hạn chế workload chạy privileged/hostPath trên node Talos, vì container escape vẫn là đường tấn công thật dù OS layer đã giảm attack surface đáng kể.

### **8. Ops Runbook — Production Notes**

- **Health check**: `talosctl health` (check etcd quorum, kubelet, API server reachability toàn cluster); `talosctl get members` (xem node nào đã join). Không có tương đương `kubectl get nodes` ở tầng OS — Talos tách biệt "node OS healthy" khỏi "K8s node Ready", 2 trạng thái khác nhau cần check riêng.
- **Log quan trọng**: `talosctl logs <service>` (vd `kubelet`, `etcd`, `containerd`) và `talosctl dmesg` cho kernel log — không có `/var/log` truy cập qua SSH, mọi thứ qua API.
- **Debug bundle**: `talosctl support` xuất bundle (log + config + state) để phân tích. ⚠️ Lưu ý advisory `GHSA-7cxh-94cf-mfp6` (24/08/2026, Low) — support bundle **không redact credential registry** ở version bị ảnh hưởng, rà soát kỹ trước khi share bundle ra ngoài, kể cả nội bộ.
- **Backup**: etcd snapshot qua `talosctl etcd snapshot` (control-plane) — tách biệt hoàn toàn khỏi việc backup `STATE` partition (machine config + secrets). 2 thứ này là 2 nguồn backup độc lập cho disaster recovery → [[Fleet-Ops]].
- **Upgrade**: `talosctl upgrade` (Talos/OS version) và `talosctl upgrade-k8s` (Kubernetes version) — 2 lệnh, 2 quy trình độc lập, có thể upgrade cái này mà không đổi cái kia trong giới hạn support matrix.

### **9. Gotchas & Lessons Learned**

- Không SSH/shell nghĩa là **không "vào sửa tay"** được như ở distro thường khi gặp sự cố lạ — mọi workaround phải đi qua machine config hoặc `talosctl`. Nếu gặp tình huống chưa có API hỗ trợ, lựa chọn thực tế nhiều khi là reset/reprovision node, không phải debug sâu tại chỗ — cần chuẩn bị tâm lý và quy trình cho việc này trước khi lên production.
- `talosctl upgrade` và `talosctl upgrade-k8s` hay bị gộp chung thành "upgrade Talos" trong nhiều blog/tutorial — dễ hiểu lầm khi lập kế hoạch bảo trì, 2 lệnh này độc lập hoàn toàn.
- KMS-based disk encryption tạo dependency boot-time mới (node không tự đứng được nếu mất mạng tới KMS) — nếu lab trên KVM/CloudStack/OpenStack mà KMS nằm ngoài vùng mạng đó, có thể tạo circular dependency nguy hiểm lúc disaster recovery (cần KMS sống để boot node, nhưng node lại có thể là nơi chạy KMS).
- Riêng ngày 18/09/2026, Talos công bố liền 4 advisory liên quan role/privilege-escalation cùng lúc — dấu hiệu nên theo dõi sát security advisory feed hơn là tin "Talos an toàn vì immutable". Immutable chỉ giảm 1 lớp attack surface (OS-level persistence), không loại bỏ lớp API/role hay container-runtime.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với cluster Talos thật trên KVM — lesson learned thực chiến quan trọng hơn note lý thuyết)

### **10. Resources**

- Official docs: https://docs.siderolabs.com/talos (và bản theo version: https://www.talos.dev/)
- Image Factory: https://docs.siderolabs.com/talos/v1.14/learn-more/image-factory
- Disk Encryption guide: https://www.talos.dev/v1.11/talos-guides/configuration/disk-encryption/
- GitHub Releases: https://github.com/siderolabs/talos/releases
- Security Advisories (nguồn CVE chính thức): https://github.com/siderolabs/talos/security/advisories
- CABPT (bootstrap provider): https://github.com/siderolabs/cluster-api-bootstrap-provider-talos
- CACPPT (control-plane provider): https://github.com/siderolabs/cluster-api-control-plane-provider-talos
- **Lưu ý khi dùng note này**: version pin + danh sách CVE lấy qua search engine, đối chiếu 1 phần qua GitHub Advisory Database — riêng "patched ở version nào" của từng CVE **chưa fetch trực tiếp được** (tool fetch trả 404 khi thử truy cập advisory). Mở trực tiếp `github.com/siderolabs/talos/security/advisories` để xác nhận trước khi dùng cho quyết định patch thật.
