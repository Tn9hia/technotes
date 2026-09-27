---
tags:
  - performance
  - storage
---

# IO Tuning — Cache Modes & IOThreads

**Cache mode** quyết định QEMU có dùng page cache của host để buffer I/O disk hay không — ảnh hưởng trực tiếp tới data safety và hiệu năng. **IOThreads** tách xử lý I/O disk ra khỏi vCPU thread chính, để 1 vCPU bận nặng không làm nghẽn toàn bộ I/O của VM.

## Cache Modes — bảng quyết định

| Mode | Host page cache? | Data safety khi host crash | Hiệu năng | Khi nào dùng |
|---|---|---|---|---|
| `none` | Không (O_DIRECT — bypass page cache host) | An toàn nhất trong nhóm — guest tự chịu trách nhiệm flush, không có tầng cache host "nói dối" đã ghi xong | Cao, ổn định, dự đoán được | **Khuyến nghị mặc định cho production**, đặc biệt với shared storage (NFS/RBD/iSCSI) |
| `writethrough` | Có, nhưng flush ngay mỗi write | An toàn (write chỉ "xong" sau khi thật sự ghi xuống) | Chậm hơn `none` (mỗi write đều sync) | Hiếm dùng — an toàn cao nhưng đánh đổi hiệu năng không cần thiết ở hầu hết trường hợp |
| `writeback` | Có, không flush ngay (giống cache thường của Linux) | **Rủi ro mất dữ liệu** nếu host crash trước khi cache được flush xuống disk thật | Cao nhất | Chỉ dùng khi đã hiểu rõ rủi ro và có UPS/backup đủ tốt, hoặc storage backend tự đảm bảo (VD RBD với writeback cache riêng đã cấu hình cẩn thận) |
| `directsync` | Không, tương tự `none` nhưng sync mọi write | An toàn nhất, chậm nhất | Thấp | Hiếm dùng, chỉ khi cần đảm bảo tuyệt đối |

> [!warning] Lesson learned: `writeback` "nhanh" trong benchmark nhưng là quả bom nổ chậm cho production
> Rất dễ bị cuốn theo số liệu benchmark đẹp của `writeback` (đôi khi nhanh hơn `none` đáng kể với workload write nhiều) — nhưng nếu host mất điện/crash đột ngột, dữ liệu đang "cache" ở host (mà guest OS tin tưởng đã ghi xong) **biến mất**, guest filesystem có thể corrupt theo cách khó phát hiện ngay (lỗi xuất hiện sau, khi đọc lại vùng dữ liệu đó). Trừ khi có lý do rất cụ thể và đã accept rủi ro, luôn dùng `none` cho production.

## IOThreads — tách I/O khỏi vCPU thread

```xml
<domain>
  <iothreads>2</iothreads>
  <devices>
    <disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' iothread='1'/>
      ...
    </disk>
  </devices>
</domain>
```

Không có IOThread, xử lý I/O disk mặc định chạy trên chính vCPU thread (hoặc main QEMU thread) — nếu vCPU đang bận 100% xử lý tính toán, request I/O phải chờ tới lượt. Với IOThread riêng, disk I/O được xử lý trên 1 thread độc lập hoàn toàn, không bị chặn bởi tải CPU của vCPU chính.

> [!tip] Số lượng IOThread hợp lý
> Không cần 1 IOThread cho mỗi disk — thường 1-2 IOThread đủ cho đa số VM (trừ khi VM có rất nhiều disk hiệu năng cao chạy song song). Gán nhiều disk cùng dùng chung 1-2 IOThread là bình thường, không phải lỗi cấu hình.

## `io` mode — native vs threads

| Mode | Cơ chế | Yêu cầu | Hiệu năng |
|---|---|---|---|
| `threads` | QEMU dùng thread pool userspace để giả lập async I/O | Không yêu cầu gì đặc biệt | Trung bình |
| `native` | Dùng Linux AIO thật (kernel-level async I/O) | Bắt buộc đi kèm `cache='none'` (yêu cầu O_DIRECT) | Cao hơn `threads`, khuyến nghị khi đã chọn `cache=none` |
| `io_uring` | Dùng `io_uring` (kernel 5.1+) — API async I/O hiện đại nhất | Kernel host + QEMU đủ mới | Cao nhất trong 3 mode, đặc biệt với queue depth lớn |

```xml
<driver name='qemu' type='raw' cache='none' io='io_uring' iothread='1'/>
```

## Ops Runbook — benchmark trước khi chốt cấu hình

```bash
# Chạy trong GUEST, không chạy trên host — cần đo đúng góc nhìn của VM
fio --name=randwrite --ioengine=libaio --rw=randwrite --bs=4k \
    --size=1G --numjobs=4 --runtime=60 --direct=1 --group_reporting
```

> [!warning] Không kết luận cấu hình tốt/xấu chỉ dựa vào 1 loại benchmark
> `fio` với `randwrite` cho ra kết luận khác hẳn `seqread` — cache mode và io mode có ROI khác nhau tùy pattern I/O thật của ứng dụng (database random I/O nhỏ khác hẳn video streaming sequential lớn). Luôn benchmark với pattern **gần giống workload thật** của ứng dụng sẽ chạy, không dùng benchmark tổng quát rồi áp dụng kết luận cho mọi VM.

## Resources

- QEMU disk I/O documentation: `docs/qemu-block-drivers.txt` trong source QEMU
- `man fio`

---
*Xem thêm: [[Virtio Devices]] | [[Disk Image Formats - qcow2 vs raw]] | [[Ceph RBD with KVM]] | [[Kvm-virtualization|KVM Virtualization]]*
