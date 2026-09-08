# gVisor — Security Model & CVE Taxonomy
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--platforms]], [[gvisor--kubernetes-containerd]]
Tags: #gvisor #security #cks #threat-model

## What it does

Định nghĩa: gVisor bảo vệ chống lại việc **exploit host kernel/hypervisor thông qua System API** (syscall). Nó KHÔNG phải giải pháp chống mọi loại tấn công.

## Why it exists

Vì hầu hết privilege escalation thực tế đi qua con đường: mở file/socket → truyền argument dị dạng → race condition đa luồng để trúng code path lỗi trong kernel (ví dụ kinh điển: Dirty Cow). gVisor chặn đường này bằng cách **không bao giờ để app chạm trực tiếp vào syscall interface thật của host** — mọi syscall được Sentry (viết bằng Go, memory-safe) diễn giải và trả lời độc lập.

## How it works (flow/diagram)

**4 loại attack vector** mà tài liệu chính thức liệt kê (để hiểu phạm vi bảo vệ):
1. **System API** — bug trong syscall/trap chính thức. **Đây là thứ gVisor tập trung giảm thiểu nhất.**
2. **System ABI** — exploit qua hành vi ngầm của phần cứng/kernel khi phản ứng với trap/interrupt (vd. CVE-2018-8897 "POPSS", ảnh hưởng cả Xen hypervisor) — ngoài phạm vi bảo vệ trực tiếp của gVisor vì đây không phải syscall path.
3. **Side channel** phần cứng (Spectre-class, L1TF) — **gVisor không chống được**, dựa vào host kernel/firmware mitigation (retpoline, frame poisoning...).
4. **Other vectors** (network-accessible service bug, physical access...) — ngoài phạm vi. Quote chính thức: *"A sandbox is not a substitute for a secure architecture."*

**2 nguyên tắc thiết kế chính:**
- Sentry **implement lại toàn bộ System API** thay vì để app tương tác trực tiếp với host → chặn exploit trực tiếp.
- Syscall mà **chính Sentry** được phép gọi xuống host bị giới hạn tối thiểu → chặn exploit gián tiếp (kể cả nếu Sentry bị compromise, nó cũng không làm được nhiều).

**3 kiểu tương tác duy nhất Sentry được phép làm với host** (ghi rõ trong security.md, đáng nhớ thuộc lòng khi audit):
1. Giao tiếp với Gofer process qua socket đã connect sẵn (không tự mở file/socket mới, trừ khi bật host networking hoặc directfs).
2. Một tập syscall tối thiểu: dup/close FD, sync, timer, signal management.
3. Đọc/ghi packet qua 1 virtual ethernet device (bỏ qua nếu dùng host networking hoặc network=none).

**Nguyên tắc kỹ thuật đảm bảo điều trên** (engineering constraint, không chỉ policy):
- Không syscall nào pass-through — mọi syscall support có implementation độc lập trong Sentry.
- Chỉ implement chức năng phổ quát — **không** implement extended attribute đặc thù, raw socket, ioctl chuyên biệt của filesystem/device lạ.
- Toàn bộ `unsafe` Go code bị cô lập trong file có hậu tố `unsafe.go` để dễ audit; **cấm CGo hoàn toàn**; hạn chế external import trong core package.
- Sentry được **fuzz liên tục**; crash ở production được ghi nhận và triage.

## Config gotchas

- Không có config nào "bật thêm security" ở đây — đây là kiến trúc cố định. Cái sysadmin cần làm là **không vô tình tắt lớp phòng thủ thứ 2** (seccomp/namespace/pivot_root áp lên chính Sentry) bằng cách chạy gVisor với quyền/host access rộng hơn cần thiết (vd. `--host-uds=all`, `--network=host` khi không cần).
- `runsc do` KHÔNG đại diện cho mô hình bảo mật thật khi test — nó cho sandbox quyền đọc toàn bộ host filesystem (read-only) để tiện demo. Muốn test đúng góc độ security phải dùng Docker/K8s với OCI spec giới hạn rõ ràng, không dùng `runsc do`.

## Security notes — Phân loại mức độ nghiêm trọng (từ `SECURITY.md`)

gVisor chỉ cấp CVE khi issue: **cross sandbox boundary** + **attacker không tự control sandbox config từ đầu** + **đặc thù gVisor** (không phải bug chung của mọi sandbox).

Từ nghiêm trọng nhất đến ít nhất (vượt ra ngoài sandbox boundary):
- `Escape` — container escape, chạy được code trên host. Do kiến trúc 2 lớp (Sentry + Linux security primitives), escape thường cần **chain nhiều exploit cùng lúc** — 1 bug trong Sentry KHÔNG đủ để escape theo thiết kế.
- `HostLeak` — đọc được file/metadata host ngoài phạm vi được cấp.
- `Exfil` — data exfiltration ra ngoài dù config sandbox cấm (vd. network=none nhưng vẫn connect ra ngoài được).
- `Lateral` — di chuyển ngang, chạy code trong sandbox khác trên cùng host.
- `HostDoS` / `PeerDoS` — DoS ảnh hưởng host kernel / sandbox khác.

Ở dưới, **KHÔNG bị coi trọng bằng** (giới hạn trong 1 sandbox, thường không cấp CVE riêng):
- `InternalEsc` — leo quyền lên root **trong sandbox**.
- `InternalRead` — đọc được thứ mà root-trong-sandbox đọc được.

**Điểm quan trọng cho vận hành thực tế**: chính sách CVE của gVisor-the-project coi việc kiểm soát OCI spec là "out of scope" (nếu attacker tự control config từ đầu thì không tính là lỗ hổng của gVisor). Nhưng môi trường production cụ thể (vd. GKE Autopilot) **có policy riêng enforce lên PodSpec** trước khi đưa vào gVisor — nên 1 vấn đề "không phải CVE gVisor" vẫn có thể là **incident thật sự** trong deployment cụ thể của bạn nếu nó phá vỡ policy đó. Khi report/đánh giá 1 vấn đề bảo mật, phải phân biệt rõ 2 ngữ cảnh này.

## Refs
- `g3doc/architecture_guide/security.md`, `g3doc/architecture_guide/intro_to_gvisor.md` (repo `google/gvisor`)
- `SECURITY.md` (root repo) — chính sách CVE, taxonomy đầy đủ, kênh report: gvisor-security@googlegroups.com
- CVE tham chiếu trong doc gốc: Dirty Cow (không CVE cụ thể ghi), CVE-2018-8897 (POPSS), CVE-2018-12904 (nested virtualization)
