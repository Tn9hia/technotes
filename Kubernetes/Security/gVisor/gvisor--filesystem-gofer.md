# gVisor — Filesystem & Gofer (overlay, directfs, dentry cache, EROFS)
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--security-model]], [[gvisor--resource-model]], [[gvisor--common-errors-lessons]]
Tags: #gvisor #filesystem #performance

## What it does

gVisor truy cập filesystem thông qua 1 process proxy riêng gọi là **Gofer**, giao tiếp với Sentry qua giao thức **LISAFS**. Đây là nơi có nhiều option tuning nhất trong toàn bộ hệ thống, vì I/O là chi phí (implementation cost) lớn nhất của gVisor.

## Why it exists

Nếu Sentry tự mở file trên host trực tiếp, nó sẽ cần quyền host filesystem access rộng — phá vỡ nguyên tắc "host surface tối thiểu". Gofer là process **tách biệt, ít đặc quyền hơn một chút**, đứng giữa để Sentry không bao giờ cần quyền filesystem access trực tiếp (trừ khi tự nguyện bật `directfs` để đổi lấy performance với 1 mức độ rủi ro đã được kiểm soát).

## How it works (flow/diagram)

```
App (Sentry) --syscall filesystem--> Sentry logic (VFS nội bộ)
                                          │
                          directfs=true?  │  directfs=false?
                          (donate FD)     │  (RPC mọi lần)
                                          ▼
                                       Gofer  <--LISAFS--> host filesystem thật
```

- **Directfs (default: bật)**: Gofer donate FD của toàn bộ mount point cho sandbox 1 lần, sandbox sau đó tự gọi `openat(2)`, `fchownat(2)`... trực tiếp trên FD đó — **bỏ qua round-trip RPC tới Gofer** mỗi lần I/O → nhanh hơn nhiều. Sandbox vẫn chỉ thao tác được trên cây filesystem mà Gofer đã expose, không đụng được gì khác trên host. Có thêm ràng buộc security: enforce `O_NOFOLLOW` qua seccomp, đảm bảo không leak FD khi khởi động.
- **Directfs tắt**: sandbox chạy với seccomp filter chặt hơn, ít capability hơn, **không tự làm filesystem op được** — mọi thứ phải qua RPC tới Gofer. An toàn hơn, chậm hơn.
- **Overlay** (`--overlay2`): đặt 1 lớp tmpfs ghi đè lên trên mount (kể cả để biến filesystem read-only như EROFS thành ghi được). Mọi thay đổi nằm ở overlay, filesystem gốc giữ nguyên. 3 backing medium: `memory` (RAM, tốn RAM), `self` (file ẩn trong chính mount, **default cho rootfs**), `dir=/path` (file trên 1 path host chỉ định).
- **Dentry cache**: Gofer client giữ cây dentry để tăng tốc path resolution. Cache là LRU cho các dentry hết reference (leaf node không ai giữ). Default size 1000/mount, chỉnh bằng `--dcache` (global) hoặc mount option `dcache=N` (per-mount).
- **Shared vs exclusive bind mount** (`--file-access-mounts`): mặc định `shared` — Gofer liên tục re-validate dentry tree so với host filesystem thật vì giả định có process khác cũng đụng vào mount đó. `exclusive` bật cache aggressive (không re-validate) → nhanh hơn nhiều nhưng **chỉ an toàn nếu chắc chắn không ai khác đụng vào mount đó**.
- **EROFS**: filesystem read-only hiệu năng cao, được mmap thẳng vào Sentry, **không cần chạm host syscall để đọc** — dùng làm lower layer của overlay rootfs là lý tưởng, và cho phép chạy **gofer-less mode** nếu không còn gofer mount nào khác.
- **Custom Gofer extension**: có thể tự build runsc với extension riêng để serve mount từ backend khác (network storage, encrypted fs...) qua interface `extension.Extension`, song song với stock gofer (LISAFS thường) cho các mount khác.

## Config gotchas

- `--overlay2=root:self` là **default** — đổi filesystem bên trong sandbox **không tự propagate ra host image gốc**. Nếu bạn cần `docker cp` ra rồi mong đổi phản ánh lên host, phải hiểu rõ cơ chế này hoặc tắt overlay (`--overlay2=none`).
- Bật `--file-access-mounts=exclusive` hoặc `--file-access=shared` (rootfs) đúng ngữ cảnh: `exclusive` cho static data/dedicated storage (ML model, dataset không đổi); tuyệt đối không dùng cho volume bị mutate từ bên ngoài — **warning chính thức**: có thể dẫn tới data corruption/undefined behavior vì sandbox có thể làm việc với dữ liệu stale.
- `docker cp` file mới vào container xong `ls` không thấy → do dentry cache của thư mục cha chưa invalidate (bug đã biết, tracked #4). Workaround: tạo 1 file trong thư mục đó để force refresh, hoặc bật shared root filesystem (`--file-access=shared`, chậm hơn do check nhiều hơn). `kubectl cp` không bị vấn đề này vì nó exec vào trong sandbox để copy (Sentry tự biết có file mới).
- EROFS mount cần annotation `dev.gvisor.spec.rootfs.type=erofs` + `source` trỏ tới image `.erofs` — 2 field này bắt buộc, `overlay`/`options` optional.

## Security notes

- Dù directfs bật hay tắt, **sandbox mount namespace luôn rỗng** và **filesystem luôn do Gofer sở hữu** — sandbox không bao giờ tự ý thấy toàn bộ host fs, chỉ thấy đúng cây được Gofer expose. Đây là bất biến bảo mật, không phụ thuộc config directfs.
- directfs đánh đổi 1 phần bảo mật (sandbox tự làm filesystem syscall) lấy performance — nếu ưu tiên bảo mật tuyệt đối hơn performance (vd. chạy code hoàn toàn không tin cậy), cân nhắc tắt (`--directfs=false`).

## Refs
- `g3doc/user_guide/filesystem.md`, `g3doc/user_guide/rootless.md` (repo `google/gvisor`, nhánh `master`)
- Blog directfs: https://gvisor.dev/blog/2023/06/27/directfs/
- Bug tracker dentry cache/docker cp: gvisor issue #4
