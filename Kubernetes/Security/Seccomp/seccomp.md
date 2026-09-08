# Seccomp — Secure Computing Mode
Tags: #kernel #container #kubernetes #security #cks
Last updated: 2026-09-06
Verified against: Linux man-pages seccomp(2) (man7.org), kubernetes.io docs v1.34–v1.37, CKS Curriculum v1.34 (cncf/curriculum, chính thức)

---

### **1. What — Nó là cái gì?**

Seccomp (secure computing mode) là một cơ chế trong kernel Linux cho phép một process tự giới hạn tập syscall mà nó (và các thread con) được phép gọi. Ở dạng hiện đại (seccomp-BPF), giới hạn này là một chương trình BPF chạy trong kernel, được đánh giá mỗi khi process gọi syscall, trả về một "action" (allow/block/kill/log/...).

Trong container world, seccomp là 1 trong 3 lớp kernel-hardening đứng cạnh nhau: **Capabilities** (giới hạn *quyền* — được làm gì), **AppArmor/SELinux** (giới hạn *tài nguyên* — file/network nào được đụng), **Seccomp** (giới hạn *cổng vào kernel* — syscall nào được gọi). Cả 3 áp dụng cùng lúc, không thay thế nhau.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có seccomp: một container bị chiếm quyền (RCE trong app) thì attacker có full syscall surface của kernel — bao gồm cả syscall hiếm khi dùng nhưng cực kỳ nguy hiểm để escape/escalate: `ptrace`, `mount`, `keyctl`, `add_key`, `perf_event_open`, `unshare`, `clone` với `CLONE_NEWUSER`, các syscall khai thác lỗ hổng kernel 0-day (rất nhiều CVE kernel nằm ở syscall ít test kỹ, ví dụ các lỗ hổng qua `io_uring`, `bpf()`).

Seccomp thu hẹp attack surface đó lại: container chỉ có ~300-350 syscall thường dùng (profile mặc định của Docker/containerd), thay vì ~450+ syscall tồn tại trên kernel hiện đại. Đây là lớp phòng thủ "giảm thiệt hại nếu app đã bị compromise", không phải lớp ngăn compromise xảy ra — tư duy đúng là **defense in depth**, không phải silver bullet.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng:**
- Mặc định luôn nên bật `RuntimeDefault` cho mọi container (baseline hợp lý, hầu như không cần tune gì).
- Custom/fine-grained profile (`Localhost`) cho workload nhạy cảm, chạy trên node đa tenant, hoặc khi audit yêu cầu allowlist syscall tối thiểu.
- Bắt buộc theo Pod Security Standard mức `restricted` (xem [[seccomp--kubernetes]]).

**KHÔNG dùng / cẩn trọng:**
- Không set `Unconfined` như một cách "sửa lỗi tạm" khi container crash — đây là bẫy phổ biến nhất (xem mục 9). Root cause thường là thiếu 1-2 syscall cụ thể, không phải "seccomp không hợp với app này".
- Không tự viết fine-grained profile cho app mà bạn không maintain lâu dài — profile quá khít với 1 snapshot hành vi sẽ vỡ khi lib/runtime version đổi (Go runtime, glibc, JIT của ngôn ngữ nào đó gọi syscall mới).
- Không áp seccomp cho container `privileged: true` — kernel/runtime sẽ bỏ qua, container luôn chạy `Unconfined` bất kể bạn khai gì trong manifest.
- Seccomp không thay thế network policy, RBAC, hay image scanning — nó chỉ chặn ở lớp syscall.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
User process (container)
      │  gọi syscall (ví dụ open(), mount())
      ▼
┌─────────────────────────────┐
│  Kernel syscall entry        │
│  ┌─────────────────────────┐ │
│  │ Seccomp-BPF filter chain │ │ ← attach qua prctl()/seccomp() syscall
│  │ (đánh giá theo thứ tự    │ │
│  │  filter cuối cùng gắn    │ │
│  │  trước, ưu tiên action   │ │
│  │  nghiêm khắc nhất thắng) │ │
│  └─────────────────────────┘ │
│         │ action             │
│  ALLOW ─┼─ ERRNO/KILL/TRAP/  │
│         │  LOG/TRACE/NOTIFY  │
└─────────┼─────────────────────┘
          ▼
  Syscall thực thi (nếu ALLOW/LOG) hoặc bị chặn
```

**Vị trí trong container stack:**

```
Pod spec (securityContext.seccompProfile)
        │
        ▼
kubelet ──► CRI (containerd / CRI-O)
        │        │
        │        ▼
        │   OCI runtime spec (config.json → linux.seccomp)
        │        │
        │        ▼
        │   runc / crun (dùng libseccomp để compile JSON → BPF program)
        │        │
        │        ▼
        └──►  prctl(PR_SET_SECCOMP) / seccomp(2) trước khi exec() container process
```

**Dependency:** kernel `CONFIG_SECCOMP` + `CONFIG_SECCOMP_FILTER` (bật mặc định trên hầu hết distro production hiện đại), `libseccomp` (thư viện userspace mà runc/crun/Docker/CRI-O dùng để build BPF filter từ JSON profile).

### **5. How — Cơ chế hoạt động**

Core concepts quan trọng nhất:

1. **2 mode**: `SECCOMP_SET_MODE_STRICT` (chỉ cho `read/write/exit/sigreturn`, gần như không ai dùng trong container) và `SECCOMP_SET_MODE_FILTER` (dùng BPF, đây là mode mọi container runtime dùng). Chi tiết → [[seccomp--kernel-bpf]]
2. **Filter là chương trình BPF cổ điển**, nhận input là `struct seccomp_data` (syscall nr + arch + 6 args), trả về 1 action. Nhiều filter có thể stack (ví dụ container runtime filter + app tự áp thêm) — **action nghiêm khắc nhất trong toàn bộ chain luôn thắng**, không phải filter gắn sau. → [[seccomp--kernel-bpf]]
3. **Profile JSON (OCI format)** là thứ bạn thực sự viết tay — `defaultAction` + danh sách `syscalls` với action riêng, có thể lọc theo argument. → [[seccomp--profile-json]]
4. **RuntimeDefault vs Localhost vs Unconfined** ở tầng Kubernetes — quyết định profile nào được nạp cho container. → [[seccomp--kubernetes]]
5. **Mỗi container runtime tự maintain 1 default profile riêng** (Docker/moby, containerd, CRI-O) — không có 1 chuẩn "RuntimeDefault" chung, nội dung khác nhau giữa các runtime và có thể khác nhau giữa các version. → [[seccomp--runtimes]]

Request lifecycle khi tạo container (K8s): kubelet đọc `seccompProfile` từ pod spec → truyền cho CRI runtime qua CRI API → CRI runtime ghi vào OCI `config.json` (`linux.seccomp`) hoặc set annotation `RuntimeDefault` → runc/crun gọi libseccomp compile JSON thành BPF bytecode → gọi `seccomp(SECCOMP_SET_MODE_FILTER, ...)` **ngay trước `execve()`** container entrypoint (thứ tự này quan trọng: filter phải có trước khi process đầu tiên chạy, và filter được kế thừa qua `fork()/clone()` — con luôn strict-hơn-hoặc-bằng cha, không bao giờ lỏng hơn).

### **6. Key Config — Cấu hình cần nhớ**

| Config | Ý nghĩa | Rủi ro nếu sai |
|---|---|---|
| `securityContext.seccompProfile.type: Unconfined` | Tắt hoàn toàn seccomp | **Default value nguy hiểm nhất** — nếu không set gì và cluster không bật `SeccompDefault`, hành vi mặc định lịch sử là `Unconfined` |
| `type: RuntimeDefault` | Dùng profile mặc định của container runtime | An toàn baseline, đủ cho >95% workload |
| `type: Localhost` + `localhostProfile` | Dùng profile custom trên node | Path sai / file không tồn tại → `CreateContainerError`, pod Pending mãi — lỗi vận hành hay gặp nhất |
| `localhostProfile` path | `/var/lib/kubelet/seccomp/<path>` | Phải tồn tại **trên từng node** trước khi pod schedule tới — dễ quên khi thêm node mới hoặc dùng autoscaler |
| kubelet `--seccomp-default` (hoặc `seccompDefault: true` trong KubeletConfiguration) | Đổi default toàn node từ `Unconfined` → `RuntimeDefault` khi pod không khai gì | GA từ K8s v1.27; **không set thì cluster cũ vẫn Unconfined-by-default** dù bạn tưởng an toàn |
| `defaultAction` trong profile JSON | Hành động khi syscall không match rule nào | Set `SCMP_ACT_ALLOW` = mất hết ý nghĩa của việc viết profile (allowlist ngược) |

### **7. Security Considerations**

- **Attack surface**: syscall interface của kernel — càng nhiều syscall được allow, càng nhiều bug kernel tiềm ẩn có thể bị khai thác để escape container/escalate privilege. Lịch sử có nhiều CVE kernel-escape đi qua syscall hiếm (vd. các lỗ hổng liên quan `io_uring`, một số lỗ hổng cgroup/namespace qua `unshare`/`clone`).
- **Misconfiguration gây breach**: `Unconfined` mặc định (cluster cũ chưa bật `SeccompDefault`), hoặc profile custom set `defaultAction: SCMP_ACT_ALLOW` (chỉ block vài syscall thay vì allowlist).
- **Privileged container luôn Unconfined** — team hay quên điều này, tưởng set `Localhost` là đủ trong khi pod cũng đang chạy `privileged: true`.
- **Seccomp KHÔNG chặn được logic-level attack** (SSRF, injection trong app) — nó chỉ chặn syscall, không hiểu ngữ nghĩa request.
- **TOCTOU với argument-based filter**: seccomp-BPF chỉ đọc giá trị con trỏ (address), không đọc nội dung buffer tại thời điểm filter chạy — filter theo path string (`open("/etc/passwd")`) là **không thể** làm đúng bằng seccomp thuần (cần LSM như AppArmor/SELinux cho việc đó).
- **Hardening checklist tối thiểu**:
  - [ ] Bật `SeccompDefault` (hoặc PSS `restricted`) ở cấp cluster, đừng phụ thuộc từng pod tự khai.
  - [ ] Không có pod nào chạy `Unconfined` trừ khi có lý do document rõ (debug container, workload cần syscall lạ đã review).
  - [ ] Không có pod nào `privileged: true` trừ khi thực sự cần (system daemon, CNI...).
  - [ ] Kết hợp Pod Security Standard `restricted` + seccomp, không dùng seccomp đơn độc.
  - [ ] Nếu dùng `Localhost` profile, có pipeline sync profile ra mọi node (đừng copy tay).

### **8. Ops Runbook — Production Notes**

- **Health check**: seccomp không có "health endpoint" — dấu hiệu vấn đề là container `CrashLoopBackOff` hoặc `CreateContainerError` ngay sau khi áp profile mới.
- **Log quan trọng cần theo dõi**:
  - `kubectl describe pod` → event `CreateContainerError: unable to load local profile ... no such file or directory` (profile thiếu trên node).
  - Kernel `audit.log` (nếu auditd chạy) → dòng `type=SECCOMP ... syscall=<nr> ...` khi syscall bị kill/log — cần map số syscall → tên bằng `ausyscall <nr>` hoặc `ausyscall --dump`.
  - Nếu dùng profile với `SCMP_ACT_LOG`, syscall bị log nhưng **vẫn chạy** — dùng để audit trước khi chuyển sang `SCMP_ACT_ERRNO`.
- **Metric cần alert**: số lượng `CreateContainerError`/`CrashLoopBackOff` tăng đột biến sau khi rollout seccomp profile mới — nên rollout theo canary, không áp đồng loạt toàn cluster.
- **Restart/rollback**: rollback = đổi `seccompProfile.type` về giá trị cũ (hoặc `RuntimeDefault`) rồi rolling-restart deployment. Không có "hot reload" — seccomp filter gắn tại thời điểm `execve()`, đổi profile bắt buộc phải tạo lại container.
- Chi tiết quy trình build/debug profile → [[seccomp--ops-debug]]

### **9. Gotchas & Lessons Learned**

> Phần này là kinh nghiệm thực tế / case study đã research, không phải suy đoán — nhưng bạn nên tự verify lại khi gặp trong môi trường cụ thể.

- **"Tắt seccomp để fix uptime" là cái bẫy kinh điển**: khi 1 service crash sau khi bật seccomp, phản xạ sai là set `Unconfined` để "cứu hỏa" rồi quên đổi lại. Root cause thường là 1-2 syscall thiếu (ví dụ Go runtime dùng `clone3`, `futex`, hoặc syscall mới do libc/kernel version đổi) — nên fix bằng cách thêm syscall vào allowlist, không phải tắt hẳn.
- **Library update âm thầm đổi syscall dùng** → profile fine-grained viết cách đây 6 tháng có thể vỡ khi bump base image hoặc runtime version, dù code app không đổi dòng nào. Đây là lý do "custom profile" chỉ nên viết nếu có người thực sự maintain lâu dài; nếu không, `RuntimeDefault` là lựa chọn bền hơn.
- **Quên check `arch`**: nếu tự viết BPF filter/level thấp mà không kiểm tra field `arch` trong `seccomp_data`, syscall gọi qua 32-bit compat ABI (số thứ tự syscall khác hoàn toàn) có thể bypass filter viết cho 64-bit. Profile JSON chuẩn (libseccomp) xử lý việc này qua `"architectures"` — đừng tự chế BPF tay trừ khi hiểu rõ.
- **Profile thiếu trên node mới** khi dùng `Localhost`: cluster autoscaler thêm node mới không tự có file profile trong `/var/lib/kubelet/seccomp/` nếu bạn distribute bằng tay (scp/DaemonSet copy) — pod schedule vào node mới sẽ bị `CreateContainerError`. Cách bền hơn: dùng Security Profiles Operator hoặc CRI-O OCI-artifact profile (K8s 1.29+ với CRI-O) → xem [[seccomp--ops-debug]].
- **`RuntimeDefault` không phải 1 chuẩn cố định**: nội dung khác nhau giữa containerd/CRI-O/Docker, và có thể đổi giữa các version runtime. Đừng document "RuntimeDefault allow đúng 300 syscall X, Y, Z" trong runbook nội bộ — nó sẽ lỗi thời. Thay vào đó, trỏ tới cách tự kiểm tra trên node đang chạy (xem [[seccomp--runtimes]]).
- **Debug container / sidecar cần `ptrace`, `strace` bị chặn bởi default profile** — đây là lý do ephemeral debug container (`kubectl debug`) đôi khi cần chạy với profile khác hoặc `Unconfined` tạm thời, có kiểm soát, không phải để mặc định.

### **10. Resources**

- [Kubernetes — Seccomp reference](https://kubernetes.io/docs/reference/node/seccomp/) — chính thức, tài liệu API/GA status
- [Kubernetes — Restrict a Container's Syscalls with seccomp (tutorial)](https://kubernetes.io/docs/tutorials/security/seccomp/) — workflow build profile từ audit → fine-grained
- [Kubernetes — Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) — yêu cầu seccomp ở mức `restricted`
- [man7.org — seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html) — nguồn chuẩn cho cơ chế kernel
- [KEP-2413 — seccomp-by-default](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2413-seccomp-by-default/README.md) — lịch sử alpha/beta/GA của `SeccompDefault`
- [CKS Curriculum (chính thức, cncf/curriculum)](https://github.com/cncf/curriculum) — seccomp nằm trong domain **System Hardening (10%)**, sandboxing (gVisor/Kata) nằm trong domain **Minimize Microservice Vulnerabilities (20%)**
- Backlinks: [[seccomp--kernel-bpf]] · [[seccomp--profile-json]] · [[seccomp--kubernetes]] · [[seccomp--runtimes]] · [[seccomp--ops-debug]] · [[seccomp--sandbox-runtimeclass]] · [[seccomp--cks-exam]]
