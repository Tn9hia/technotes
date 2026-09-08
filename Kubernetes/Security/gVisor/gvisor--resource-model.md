# gVisor — Resource Model (process/memory/cgroup accounting)
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--platforms]], [[gvisor--debugging-observability]]
Tags: #gvisor #cgroup #memory #ops

## What it does

Định nghĩa cách gVisor sandbox tiêu thụ và báo cáo tài nguyên host (CPU, RAM, PID, thời gian) — quan trọng để đọc đúng dashboard/monitoring khi vận hành.

## Why it exists

gVisor cố tình **không giả định số vCPU/RAM cố định** như VM — nó để host quyết định phân bổ tài nguyên vật lý (giống container thường), nhờ đó sandbox scale linh hoạt (dùng nhiều core/RAM khi bận, trả lại khi rảnh). Nhưng vì có 1 lớp Sentry đứng giữa, cách đo/account resource **khác với container thường (runc)** — nếu không biết điều này, engineer dễ đọc sai dashboard.

## How it works (flow/diagram)

- **Process**: process bên trong sandbox **không hiện trên host** (`top` trên host không thấy). Muốn tương tác process-level phải "vào" sandbox (vd. `docker exec`).
- **Threads**: mỗi task thread trong Sentry là 1 **goroutine** (green thread) — không nhất thiết map 1-1 với host thread. Host thread được tạo thêm tùy theo số thread app đang active; app càng bận thì hội tụ về đúng số OS thread cần dùng.
- **Time**: Sentry có vDSO + time-keeping riêng, khởi tạo từ host clock nhưng sau đó **độc lập hoàn toàn**, không share state với host. Khi mọi thread trong app idle, Sentry tắt timer (giống tickless kernel) → gần như 0% CPU khi app không làm gì.
- **Memory**: toàn bộ memory của app được backing bởi **1 memfd** (không phải anonymous memory thường). Sentry lazily populate mapping, để host tự demand-page/reclaim/swap như bình thường. Sentry **không demand-page từng trang lẻ** mà chọn theo heuristic theo vùng — nên số liệu memory qua `/proc` bên trong sandbox **là ước lượng (approximation)**, không chính xác tuyệt đối.
- **Address space**: 1 số platform (tuỳ implementation) tạo thêm process "stub" trên host để hỗ trợ address space — các stub này **vẫn bị tính vào giới hạn PID limit** ở cấp sandbox.

## Config gotchas / Vận hành

- **🔴 Bẫy lớn nhất khi đọc cgroup**: vì application memory backing bởi `memfd`, kernel host tính nó là **`shmem`** trong `memory.stat`/`memory.current`, **KHÔNG phải `anon`**. Nếu dashboard/alerting của bạn theo dõi field `anon` để phát hiện memory leak hay tính usage thật của app, nó **sẽ luôn thấp** dù app đang ăn hết RAM — false negative nguy hiểm. Phải đọc `memory.current` (tổng, đáng tin) hoặc field `shmem`/`file` (tuỳ kernel version), hoặc tốt nhất dùng `runsc usage <container-id>` để lấy breakdown chính xác từ chính Sentry.
- Cgroup **CPU/memory bên trong sandbox chỉ dùng để accounting**, KHÔNG enforce limit giữa các process **trong cùng 1 sandbox** — đây là gap đã biết (xem thêm compatibility.md). Muốn giới hạn resource thật của cả sandbox, phải đặt gVisor trong 1 cgroup Linux-native ở tầng host (Docker/K8s làm việc này tự động qua container cgroup như bình thường).
- `madvise()` app gọi để đánh dấu vùng nhớ không cần nữa → Sentry **trả ngay về host** thay vì giữ lại để tự phục vụ request khác — đánh đổi: có thể chậm hơn về mặt cục bộ (phải re-fault lại nếu cần), nhưng giúp host multiplex tài nguyên hiệu quả hơn ở tầm toàn cục. Không phải bug nếu bạn thấy memory bị trả về host "sớm hơn mong đợi".
- Sentry có thể tự lắng nghe pressure signal của cgroup chứa nó để chủ động purge cache nội bộ khi bị áp lực memory.

## Security notes

- Vì memory tất cả app share `memfd`, về lý thuyết cùng vùng host memory này *có thể* dùng chung giữa nhiều sandbox (nếu file mapping cùng backing) — cơ chế này **không loại trừ khả năng side-channel**, vẫn nằm trong phạm vi "gVisor không chống side-channel phần cứng" (xem [[gvisor--security-model]]).

## Refs
- `g3doc/architecture_guide/resources.md` (repo `google/gvisor`, nhánh `master`)
- `runsc usage <container-id>` — subcommand chính thức để lấy accounting chính xác từ Sentry
