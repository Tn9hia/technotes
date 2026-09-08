# gVisor — release-20260831.0 (latest tại 2026-09-06)
Tags: #container #sandbox #security #infra #runtime #cks
Last updated: 2026-09-06

> ⚠️ **Lưu ý về "version"**: gVisor **không dùng semver** (không có v1.2.3). Release là theo **ngày** (`release-YYYYMMDD.rc`), thường ra hàng tuần/vài tuần. "Latest" đổi liên tục. Khi note này nói "hiện tại", nghĩa là tại ngày 2026-09-06, bản mới nhất trên [GitHub Releases](https://github.com/google/gvisor/releases) là `release-20260831.0`. Luôn tự kiểm tra lại `github.com/google/gvisor/releases` trước khi áp dụng vào production, đừng tin số version cứng trong note cũ (kể cả note này sau vài tháng).

---

### **1. What — Nó là cái gì?**

gVisor là một **application kernel** viết bằng Go, chạy trong userspace, đóng vai trò "kernel giả" cho container. Nó **không phải VM**, **không phải seccomp wrapper**, mà là một **third approach**: intercept toàn bộ syscall + page fault của process trong sandbox và tự implement lại logic (process, memory, filesystem, network...) thay vì forward xuống host kernel. Ship dưới dạng OCI runtime tên `runsc`, cắm được vào Docker, containerd, CRI-O, Kubernetes.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Container thường (runc) chỉ cách ly bằng namespace + cgroup + seccomp — nhưng tất cả process vẫn **share 1 host kernel**. Một bug trong host kernel (Dirty Cow, v.v.) = 1 syscall là thoát ra ngoài. Nếu không có gVisor, để cô lập workload không tin cậy, bạn phải hoặc (a) dùng VM riêng cho mỗi workload (tốn tài nguyên, boot chậm), hoặc (b) tự viết seccomp profile cực kỳ chi tiết cho từng app (dễ sai, dễ thiếu, không generic được). gVisor cho phép **giữ mô hình process/container nhẹ** (start nhanh, density cao) nhưng **thêm 1 lớp kernel riêng biệt** giữa app và host kernel — attacker phải exploit đồng thời gVisor Sentry (Go, memory-safe) **và** host kernel thật thì mới thoát ra được.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Nên dùng khi:**
- Endpoint expose ra ngoài (load balancer, API công khai, web server) — theo khuyến nghị chính thức trong [Production guide](https://gvisor.dev/docs/user_guide/production/).
- Multi-tenant: chạy code/container của nhiều khách hàng khác nhau trên cùng host (vd. App Platform, Cloud Run, CI runner cho code lạ).
- Cần defense-in-depth cho workload nhạy cảm (payment, PII) hoặc có yêu cầu compliance.
- Cần tính năng đi kèm: intrusion detection (runtime monitoring), checkpoint/restore.

**KHÔNG nên dùng khi (quan trọng hơn phần trên):**
- Workload đã **tin cậy hoàn toàn** (code nội bộ, đã audit) — sandbox không giúp gì nếu chính app đó đã có quyền truy cập data nhạy cảm, vì "user data đã ở trong sandbox rồi".
- Workload **I/O-heavy hoặc network-heavy** (database, workload ghi file nhiều, iperf-like) — đây là nơi overhead lớn nhất (xem [[gvisor--resource-model]] và performance guide).
- Cần **mount block device filesystem** (ext4, fat32 trực tiếp trong sandbox), cần **KVM lồng trong sandbox**, cần **iptables/nftables đầy đủ**, hoặc cần **device file cho hardware custom** (ngoại trừ GPU NVIDIA/TPU đã hỗ trợ) — gVisor không support các case này (xem [[gvisor--common-errors-lessons]]).
- Coi sandbox là **thay thế cho kiến trúc bảo mật tốt** — quote chính thức: *"A sandbox is not a substitute for a secure architecture."* Nó không chống được: tấn công vào tầng cao hơn container runtime (vd. bug trong containerd khiến nó chạy container mà không qua gVisor), side-channel CPU (Spectre-class), hay exploit *bên trong* chính app đã sandbox (app vẫn có quyền truy cập những gì sandbox được cấu hình cho phép).

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌─────────────────────────── Host Linux Kernel ───────────────────────────┐
│                                                                          │
│   ┌──────────────── gVisor Sandbox (1 process opaque với host) ─────┐   │
│   │                                                                  │  │
│   │   App process(es)  <-- syscall/page fault --> [ Sentry ]        │  │
│   │   (chạy code KHÔNG sửa đổi, ELF x86_64/ARM64)     │              │  │
│   │                                                    │ re-impl:    │  │
│   │                                                    │ - syscall   │  │
│   │                                                    │ - mm        │  │
│   │                                                    │ - netstack  │  │
│   │                                                    │ - fs (VFS)  │  │
│   │                                                    │ - proc mgmt │  │
│   │                                                    ▼             │  │
│   └──────────── [platform: systrap|KVM|ptrace] ──────────────────────┘  │
│                          │ (giới hạn syscall qua seccomp-bpf)           │
│                          │ (AF_PACKET socket cho network)               │
│                          ▼                                             │
│                    ┌───────────┐   LISAFS protocol                     │
│                    │   Gofer   │◄──────────────────────► host FS thật  │
│                    │ (process  │   (file descriptor qua SCM_RIGHTS,    │
│                    │  riêng)   │    hoặc directfs FD donation)         │
│                    └───────────┘                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Component chính**: `runsc` (CLI/OCI runtime) → spawn ra 2 tiến trình host thật: **Sentry** (kernel giả, xử lý syscall) và **Gofer** (proxy filesystem, ít quyền hơn 1 chút). Optional: `containerd-shim-runsc-v1` khi tích hợp containerd/K8s.
- **Traffic/data flow**: App → syscall → Sentry (không bao giờ pass-through host) → nếu cần I/O thật → Gofer (qua LISAFS) hoặc trực tiếp qua FD đã donate (directfs) → host kernel thật.
- **Dependency**: Linux kernel ≥ 5.6, kiến trúc x86_64 hoặc ARM64. Cần `seccomp-bpf` (systrap platform) hoặc KVM module (`/dev/kvm`, nhóm `kvm`) nếu chọn platform KVM.
- Backlinks: [[gvisor--platforms]], [[gvisor--filesystem-gofer]], [[gvisor--networking]]

### **5. How — Cơ chế hoạt động**

Core concepts (5 cái quan trọng nhất):

1. **Sentry = kernel giả trong userspace.** Mọi syscall app gọi (`getpid`, `read`, `open`...) được Sentry intercept và **tự trả lời bằng logic Go riêng** — ví dụ `getpid()` trả về PID trong bảng PID nội bộ của Sentry, không phải PID thật trên host (`top` trên host không thấy process trong sandbox). **Không syscall nào được pass-through** — nếu 1 kernel feature chưa được implement trong Sentry, app trong sandbox không dùng được, dù host kernel có support.
2. **Platform = cơ chế intercept.** gVisor cần 1 "Platform" để bắt syscall + page fault + context switch. 3 lựa chọn: `systrap` (default từ giữa 2023, dùng `SECCOMP_RET_TRAP`), `KVM` (dùng virtualization extension, nhanh nhất trên bare-metal), `ptrace` (chạy mọi nơi nhưng chậm, **deprecated**, sẽ bị xoá). Chi tiết: [[gvisor--platforms]].
3. **Gofer = proxy filesystem ít đặc quyền hơn.** Sentry **không tự mở file trên host** (trừ khi bật `directfs` để nhận FD donate sẵn). Mọi thao tác filesystem thật đi qua Gofer process riêng, giao tiếp bằng giao thức **LISAFS**. Chi tiết: [[gvisor--filesystem-gofer]].
4. **Netstack = network stack riêng viết lại từ đầu.** TCP/IP nằm hoàn toàn trong Sentry, Sentry chỉ có 1 `AF_PACKET` socket raw để gửi/nhận packet ở tầng link — Sentry **không tự tạo socket host**. Chi tiết: [[gvisor--networking]].
5. **Defense-in-depth 2 lớp thật sự** (không phải marketing): Lớp 1 = Sentry (Go, memory-safe, tách biệt code với host kernel hoàn toàn). Lớp 2 = seccomp-bpf + user namespace + `pivot_root` + capability tối thiểu áp lên **chính process Sentry** khi nó cần chạm host. Muốn escape phải phá cả 2 lớp cùng lúc. Chi tiết: [[gvisor--security-model]].

Test nhanh xem có đang chạy trong gVisor không: `dmesg` trong container sẽ in log giả hài hước của Sentry (không phải log kernel thật) — đây là cách chính thức để verify.

### **6. Key Config — Cấu hình cần nhớ**

- `--platform=systrap|kvm|ptrace` — quyết định hiệu năng lớn nhất. **Bare-metal → KVM. Chạy trong VM → systrap** (KVM lồng trong VM chậm hơn systrap do overhead nested virtualization). GKE Sandbox dùng platform custom tối ưu riêng, không phải chọn thủ công. Xem [[gvisor--platforms]].
- `--network=sandbox|host|none` — mặc định `sandbox` (netstack, cô lập tốt nhất). `host` = bỏ cô lập network để lấy performance (semi-trusted workload). `none` = cô lập tuyệt đối, chỉ còn loopback nội bộ.
- `--overlay2=root:self` (default) — root filesystem có overlay tmpfs ghi vào chính rootfs, để không đụng vào ảnh gốc. **Nguy hiểm nếu bạn tưởng thay đổi filesystem sẽ propagate ra host** — nó không propagate trừ khi tắt overlay (`--overlay2=none`) hoặc dùng shared mode.
- `--directfs=true` (default) — sandbox nhận FD donate trực tiếp từ Gofer để tăng tốc, thay vì round-trip Gofer mỗi lần I/O. Tắt đi (`--directfs=false`) để tăng cô lập (giảm bề mặt tấn công) nhưng giảm performance.
- `--file-access=shared|exclusive` (rootfs) và `--file-access-mounts=shared|exclusive` (bind mount khác) — `exclusive` cache aggressive, nhanh hơn nhiều, nhưng **data corruption nếu có process khác ngoài sandbox sửa file đó**. Đừng bật exclusive cho volume bị mount ở nơi khác.
- `--rootless` — chạy không cần `sudo`, nhưng **mất Netstack** (phải dùng host network), **mất save/restore**, `create` command không hỗ trợ. Chi tiết [[gvisor--filesystem-gofer]] phần rootless.
- `--metric-server`, `--profile`, `--debug --strace --debug-log=` — xem [[gvisor--debugging-observability]]. **Lưu ý: bật `--profile` sẽ nới lỏng seccomp filter — không bật ở production.**
- **Default value nguy hiểm nhất cần biết**: `runsc do` (chế độ test nhanh) mặc định cho sandbox **read-only access vào TOÀN BỘ filesystem host** — chỉ dùng để demo/test, **không phải mô hình security thật**. Trong OCI runtime thật (Docker/K8s), filesystem bị giới hạn đúng theo spec.

### **7. Security Considerations**

- **Attack surface**: 2 chỗ chính — (1) syscall interface mà Sentry expose cho app trong sandbox (bug tại đây = "in-sandbox escalation", nghiêm trọng thấp hơn), (2) syscall/tài nguyên mà **chính Sentry/Gofer** dùng từ host (bug tại đây kết hợp bug (1) mới thành **escape** thật sự). gVisor phân loại rõ mức độ nghiêm trọng trong `SECURITY.md`: `Escape` > `HostLeak` > `Exfil` > `Lateral` > `HostDoS`/`PeerDoS` (ra ngoài sandbox) vs `InternalEsc`/`InternalRead` (chỉ trong sandbox, ít nghiêm trọng hơn nhiều — **KHÔNG được cấp CVE riêng nếu chỉ dừng ở đây**).
- **Không được bảo vệ (ghi rõ để khỏi ảo tưởng)**: side-channel phần cứng (Spectre/L1TF-class) — vẫn phải patch host kernel/firmware; resource exhaustion/DoS — dựa hoàn toàn vào cgroup của host, sandbox tự nó không enforce được resource limit **giữa các process trong cùng 1 sandbox**; network-level policy — vẫn cần network policy ở tầng container/cluster.
- **Misconfiguration dễ gây breach**:
  - Bật `--network=host` cho workload không hoàn toàn tin cậy → mất toàn bộ cô lập network.
  - Bật `--file-access-mounts=exclusive` trên mount bị process ngoài sandbox sửa đổi → không chỉ là bug hiệu năng, mà là đọc data cũ/sai (có thể dẫn đến logic bypass).
  - Bật `--profile` ở production → seccomp bị nới lỏng.
  - Chạy `runsc do` trực tiếp ngoài production nhưng lại tưởng nó an toàn như setup Docker/K8s thật (nó cho read-only cả host fs).
  - SELinux enforcing + chạy gVisor lồng trong container khác mà quên gán label `container_engine_t` → lỗi mount, nhưng nếu "fix" bằng cách tắt SELinux tùy tiện thay vì gán đúng label thì lại giảm bảo mật ngoài ý muốn.
- **Hardening checklist tối thiểu**:
  1. Dùng `systrap` (default) hoặc `KVM` — không dùng `ptrace` (deprecated, sắp gỡ, chậm).
  2. Giữ `--network=sandbox` (default) trừ khi có lý do performance rõ ràng và workload semi-trusted.
  3. Giữ `directfs` bật (default) trừ khi cần độ cô lập filesystem tối đa hơn nữa và chấp nhận đánh đổi performance.
  4. Không tắt seccomp/AppArmor ở layer ngoài (Docker/K8s) chỉ vì đã có gVisor — đây vẫn là 1 lớp trong "defense-in-depth", không phải thay thế.
  5. Trên GKE Sandbox / Kubernetes: không cấp `NET_RAW`, không dùng privileged container, không dùng hostPath volume — các thứ này **không tương thích hoặc bị chặn có chủ đích** với sandbox (xem [[gvisor--kubernetes-containerd]]).
  6. Theo dõi `gvisor-security@googlegroups.com` / SECURITY.md để biết CVE liên quan escape thật sự.

Chi tiết đầy đủ: [[gvisor--security-model]]

### **8. Ops Runbook — Production Notes**

- **Health check / verify đang chạy trong gVisor**: `docker exec <c> dmesg | head` → phải thấy log giả kiểu "Starting gVisor...". Hoặc `sandbox_running` metric (xem dưới).
- **Log quan trọng cần biết**: bật `--debug --debug-log=/tmp/runsc/ --strace` → file `.boot` chứa strace của app (tìm syscall thiếu/lỗi), file `.create` chứa lý do container không start được. Dùng biến `%ID%`/`%COMMAND%` trong path để tránh nhiều sandbox ghi đè log lẫn nhau.
- **Metric cần alert** (qua `runsc metric-server`, Prometheus-compatible, chạy **unsandboxed** như sidecar):
  - `sandbox_presence` + `sandbox_running` → so sánh 2 cái để phát hiện sandbox "biết nhưng không chạy" (crash im lặng).
  - `num_sandboxes_broken_metrics` → sandbox mà metric server không lấy được dữ liệu (dấu hiệu bất thường).
  - `runsc_fs_*`, `runsc_fs_read_wait` → phát hiện bottleneck filesystem/Gofer.
  - **Runtime Monitoring** (khác với metric server ở trên) dùng cho intrusion detection — stream mọi syscall/event ra process giám sát ngoài (vd. tích hợp Falco). Không dùng chung mục đích với `metric-server`.
- **Restart/rollback**: đổi platform/flag → sửa `runtimeArgs` trong `/etc/docker/daemon.json` (hoặc `runsc.toml` cho containerd/CRI-O) → `systemctl restart docker` (hoặc `containerd`/`crio`). Container đang chạy **không tự áp dụng** config mới, phải tạo lại.
- **Checkpoint/restore** dùng cho migration hoặc fast-restart (đặc biệt hữu ích cho workload lớn như LLM serving nhờ `--background` restore + `--exclude-committed-zero-pages`). Chi tiết: [[gvisor--checkpoint-restore]].
- **Cgroup/memory accounting có bẫy lớn**: memory của app trong sandbox được backing bởi `memfd`, kernel host tính nó là **shmem, không phải anon** trong `memory.stat`. Nếu bạn dashboard theo `anon` để phát hiện memory leak, bạn sẽ **không thấy gì** dù app ăn hết RAM. Phải nhìn `memory.current` (tổng) hoặc dùng `runsc usage <container-id>` để breakdown đúng. Chi tiết: [[gvisor--resource-model]].
- Link runbook chi tiết: [[gvisor--debugging-observability]], [[gvisor--common-errors-lessons]]

### **9. Gotchas & Lessons Learned**

(Xem đầy đủ, có nguồn dẫn, tại [[gvisor--common-errors-lessons]] — dưới đây là top-of-mind khi vận hành)

- **⏰ Deadline vận hành khẩn cấp (tại thời điểm viết note, 2026-09-06)**: gVisor đang chuyển từ mô hình "1 binary `runsc` chứa hết mọi thứ" sang mô hình nhiều file (`runsc` + `containerd-shim-runsc-v1` + thư mục `gvisor-bin/`). Nếu hệ thống cũ dùng cơ chế auto-download (runsc tự tải phần thiếu khi thiếu binary), **cơ chế này sẽ bị gỡ bỏ cuối tháng 9/2026**. Nếu bạn sắp nhận bàn giao 1 hệ thống gVisor cũ, đây là việc đầu tiên phải kiểm tra: `runsc --version`, cách cài đặt hiện tại (apt repo hay tarball thủ công), và di chuyển sang apt repo hoặc tarball đầy đủ trước hạn. Nguồn: `g3doc/user_guide/install.md` (mục "Migrating from legacy runsc-binary-only installations").
- `docker cp` vào container gVisor xong `ls` không thấy file mới → không phải bug filesystem, là **dentry cache** của gofer chưa invalidate. Trick: tạo 1 file bất kỳ trong thư mục đó để force refresh, hoặc bật shared root filesystem. `kubectl cp` **không bị vấn đề này** vì nó copy qua `exec`.
- Lỗi `RuntimeHandler "runsc" not supported` trên K8s dùng `kubeadm` → khả năng cao Docker vẫn được cài song song và kubeadm ưu tiên Docker thay vì containerd đã cấu hình runsc. Phải set rõ `--cri-socket` khi `kubeadm init`.
- DNS lookup tên container khác lỗi trên Docker user-defined bridge → do embedded DNS server của Docker bind vào `127.0.0.10` trên **host** network namespace, mà gVisor cô lập network stack nên không với tới được. Dùng default bridge + `--link`, hoặc dùng IP thẳng, hoặc chuyển sang Kubernetes (không bị vấn đề này).
- Lỗi liên quan `memfd_create` khi start container → kernel host quá cũ, không hỗ trợ syscall này.
- Lỗi mount `/etc/hostname` input/output error → bug kernel Linux 5.1–5.3.15/5.4.2/5.5 cụ thể, fix bằng nâng kernel hoặc set `LimitMEMLOCK=infinity` cho containerd.service.

### **10. Resources**
- Official docs (index): https://gvisor.dev/docs/
- Source docs (markdown gốc, luôn mới nhất, dùng để verify lại note này): `https://github.com/google/gvisor/tree/master/g3doc`
- Release mới nhất: https://github.com/google/gvisor/releases (kiểm tra lại mỗi lần trước khi note "phiên bản hiện tại")
- Security policy / CVE taxonomy: `https://github.com/google/gvisor/blob/master/SECURITY.md`
- GKE Sandbox (dùng gVisor trong GKE, có limitation riêng khác với gVisor "trần"): https://docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods
- Blog kỹ thuật đáng đọc (không phải Hello World): Systrap release post, directfs post — link trong [[gvisor--platforms]] và [[gvisor--filesystem-gofer]]
