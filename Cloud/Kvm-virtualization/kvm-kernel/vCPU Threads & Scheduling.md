---
tags:
  - kvm
  - kernel
  - scheduling
---

# vCPU Threads & Scheduling

Mỗi **vCPU** trong KVM chính là **1 thread Linux thông thường** bên trong QEMU process — không có khái niệm "vCPU" nào đặc biệt ở tầng kernel scheduler. Linux CFS (Completely Fair Scheduler) đối xử với thread vCPU giống bất kỳ thread nào khác: xếp lịch lên CPU vật lý theo cùng cơ chế dùng cho `nginx`, `postgres`, hay bất kỳ process nào.

> [!tip] So với ESXi
> ESXi có một scheduler riêng biệt (VMkernel scheduler) được thiết kế đặc thù cho việc xếp lịch vCPU, hiểu rõ khái niệm "world" và NUMA-aware ngay từ lõi. KVM **tái sử dụng nguyên CFS của Linux** — về cơ bản đơn giản và ít code hơn, nhưng cũng có nghĩa: nếu không tune (CPU pinning, cgroups, NUMA), scheduler sẽ đối xử vCPU thread như mọi thread khác trên hệ thống — kể cả cạnh tranh CPU với QEMU's I/O thread hoặc process khác của host.

## How — mối quan hệ giữa QEMU process, vCPU thread, và KVM_RUN

```
QEMU process (1 process = 1 VM)
   │
   ├─ Thread chính (main loop QEMU — xử lý QMP, timer, một số I/O)
   ├─ vCPU thread 0  ──ioctl(KVM_RUN)──> chạy guest CPU 0
   ├─ vCPU thread 1  ──ioctl(KVM_RUN)──> chạy guest CPU 1
   ├─ ...
   ├─ IOThread(s)     — tách riêng I/O disk khỏi vCPU thread (xem IO Tuning)
   └─ vhost-net kernel thread(s) — nếu dùng vhost-net, offload virtio-net vào kernel
```

Vì mỗi vCPU là 1 thread thường, `ps -eLf | grep qemu` hoặc `top -H -p <qemu_pid>` cho thấy rõ từng vCPU thread riêng — đây cũng là cách nhanh nhất kiểm tra 1 vCPU cụ thể có đang "nóng" (100% CPU) hay không.

```bash
# Xem tất cả thread của 1 QEMU process, map ra tên rõ ràng
virsh qemu-monitor-command <domain> --hmp "info cpus"   # map vCPU logic -> thread id (tid)

# Hoặc xem trực tiếp qua /proc
ps -T -p $(pgrep -f "guid.*<domain>") -o tid,psr,pcpu,comm
```

## Key Config — vcpupin & emulatorpin

Domain XML cho phép **pin** (gán cố định) từng vCPU thread vào 1 CPU vật lý (hoặc dải CPU) cụ thể, tương tự `taskset`:

```xml
<domain>
  <vcpu placement='static'>4</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='2'/>
    <vcpupin vcpu='1' cpuset='3'/>
    <vcpupin vcpu='2' cpuset='4'/>
    <vcpupin vcpu='3' cpuset='5'/>
    <emulatorpin cpuset='0-1'/>
  </cputune>
</domain>
```

| Thẻ | Ý nghĩa | Khi nào cần |
|---|---|---|
| `vcpupin` | Gán vCPU thread cụ thể vào CPU vật lý cụ thể | Workload latency-sensitive, cần tránh CPU migration giữa các lõi |
| `emulatorpin` | Gán thread QEMU main loop + các thread phụ (không phải vCPU) | Tách hẳn "overhead" QEMU khỏi CPU đang chạy vCPU chính |
| `<numatune>` | Ràng buộc VM chỉ dùng memory từ 1 NUMA node cụ thể | Bắt buộc đi kèm vcpupin nếu host multi-socket, xem [[CPU Pinning & NUMA]] |

Chi tiết đầy đủ về NUMA-aware pinning, xem [[CPU Pinning & NUMA]] — note này chỉ tập trung vào cơ chế thread/scheduling nền tảng.

## Security/Isolation Considerations

- Không pin vCPU đồng nghĩa scheduler CFS tự do di chuyển thread giữa các lõi — điều này **không phải lỗ hổng an ninh**, nhưng có thể gây side-channel timing khác biệt giữa các lần chạy (ít liên quan thực tế, chỉ đáng chú ý trong môi trường cực kỳ nhạy về side-channel).
- Nhiều VM share cùng physical core (over-commit vCPU) làm tăng khả năng nhiễu chéo hiệu năng (noisy neighbor) — không phải vấn đề bảo mật trực tiếp nhưng ảnh hưởng SLA.

## Gotchas & Lessons Learned

> [!warning] Lesson learned: over-provisioning vCPU không "miễn phí" như over-provisioning RAM
> Cấp 8 vCPU cho VM trên host chỉ có 4 core vật lý **vẫn chạy được** (CFS time-share giữa các vCPU thread), nhưng với workload nhạy độ trễ (database, VoIP...) hiện tượng "vCPU steal time" xuất hiện — guest OS bên trong thấy CPU nhưng thực ra đang chờ tới lượt. Kiểm tra bằng `top` trong guest (cột `%st`) hoặc từ host bằng cách so `vcpu_time` qua `virsh domstats <domain> --cpu-total`. Không có ngưỡng cố định "bao nhiêu vCPU/core là an toàn" — phụ thuộc hoàn toàn vào workload, nhưng tỷ lệ 1:1 hoặc thấp hơn (dedicated) là bắt buộc cho workload production nhạy độ trễ.

> [!tip] `virsh vcpuinfo` là lệnh đầu tiên nên chạy khi nghi ngờ CPU là nguyên nhân chậm
> Cho biết ngay vCPU nào đang chạy trên CPU vật lý nào tại thời điểm hỏi, và tổng CPU time đã dùng — nhanh hơn nhiều so với đi vòng qua `ps`/`top` để tự map.

## Resources

- `man virsh` — phần `vcpupin`, `emulatorpin`, `numatune`
- Linux CFS scheduler documentation: `Documentation/scheduler/sched-design-CFS.rst`

---
*Xem thêm: [[KVM Kernel Module & Hardware Virtualization Extensions]] | [[CPU Pinning & NUMA]] | [[Kvm-virtualization|KVM Virtualization]]*
