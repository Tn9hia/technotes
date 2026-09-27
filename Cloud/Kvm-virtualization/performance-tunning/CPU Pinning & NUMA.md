---
tags:
  - performance
  - numa
---

# CPU Pinning & NUMA

**NUMA (Non-Uniform Memory Access)** là kiến trúc của mọi CPU server multi-socket hiện đại — mỗi socket CPU có 1 "node" RAM gắn gần nó (latency thấp), truy cập RAM của node khác chậm hơn (phải đi qua interconnect giữa socket, VD Intel UPI hoặc AMD Infinity Fabric). Nếu VM không được "ràng buộc" vào 1 NUMA node cụ thể, vCPU thread có thể chạy trên CPU thuộc node A trong khi memory của nó nằm trên node B — mọi truy cập RAM đều tốn thêm latency qua interconnect.

> [!tip] So với NUMA-aware scheduling của ESXi
> ESXi có DRS/scheduler tự động NUMA-aware (cố gắng giữ vCPU + memory VM trong cùng node) mà không cần admin cấu hình gì trong nhiều trường hợp. KVM/libvirt **không tự động** làm điều này ở mức tối ưu — cần cấu hình tường minh qua `vcpupin` + `numatune` trong domain XML. Đây là lý do rất nhiều cluster KVM "chạy được nhưng chậm hơn kỳ vọng" chỉ vì chưa từng đụng tới NUMA tuning.

## Where — kiểm tra topology NUMA của host trước khi tune

```bash
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 64509 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 64483 MB

lscpu | grep -i numa
virsh nodeinfo    # thông tin NUMA từ góc nhìn libvirt
virsh capabilities | grep -A20 "<topology>"   # chi tiết NUMA topology XML
```

> [!warning] Xác định topology TRƯỚC khi tune, không đoán
> Số socket vật lý **không luôn bằng** số NUMA node — một số CPU AMD (kiến trúc chiplet, VD EPYC) có nhiều NUMA node **trong cùng 1 socket vật lý** (NPS setting trong BIOS: NPS1/NPS2/NPS4). Luôn chạy `numactl --hardware` thật trên host cụ thể, không giả định dựa trên số socket.

## How — pin vCPU + memory vào cùng 1 NUMA node

```xml
<domain>
  <vcpu placement='static'>8</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='5'/>
    <vcpupin vcpu='6' cpuset='6'/>
    <vcpupin vcpu='7' cpuset='7'/>
    <emulatorpin cpuset='0-7'/>
  </cputune>
  <numatune>
    <memory mode='strict' nodeset='0'/>
  </numatune>
  <cpu>
    <numa>
      <cell id='0' cpus='0-7' memory='16777216' unit='KiB'/>
    </numa>
  </cpu>
</domain>
```

| Thẻ | Ý nghĩa |
|---|---|
| `<vcpupin>` | Gán từng vCPU thread vào CPU logic cụ thể — **phải nằm trong cùng NUMA node** với memory đã chọn |
| `<numatune><memory mode='strict' nodeset='0'>` | Bắt buộc VM chỉ cấp phát RAM từ node 0 — `strict` fail nếu node đó không đủ RAM (an toàn hơn `preferred` vốn cho phép fallback sang node khác âm thầm) |
| `<cpu><numa><cell>` | Khai báo topology NUMA **ảo** cho guest thấy — quan trọng với VM lớn (VD 8+ vCPU) để guest OS bên trong (Linux/Windows) tự tối ưu lịch trình theo NUMA nội bộ |

> [!info] VM nhỏ hơn 1 NUMA node vs VM lớn hơn 1 NUMA node
> Nếu VM cần **ít vCPU/RAM hơn** dung lượng 1 NUMA node (trường hợp phổ biến nhất) → luôn pin gọn cả vCPU và memory vào **đúng 1 node duy nhất**, đơn giản và tối ưu nhất. Nếu VM cần **nhiều hơn** 1 node cung cấp được (VM khổng lồ, nhiều vCPU/RAM hơn cả 1 socket) → buộc phải trải qua nhiều node, lúc này khai báo `<cpu><numa><cell>` cho guest biết rõ topology thật để guest OS tự tối ưu (đừng để guest "tưởng" mình là UMA trong khi thực tế NUMA).

## Key Config — kiểm tra kết quả sau khi tune

```bash
virsh vcpuinfo vm01              # xem vCPU đang chạy trên CPU nào — so khớp với pin đã set
numastat -p $(pgrep -f vm01)     # % memory access local vs remote NUMA node — mục tiêu: gần 100% local
```

## Gotchas & Lessons Learned

> [!warning] Lesson learned: pin vCPU nhưng quên `numatune` → vẫn bị remote memory access
> Chỉ set `vcpupin` mà không set `numatune` là lỗi rất phổ biến — vCPU chạy đúng CPU mong muốn, nhưng memory VM đã được cấp phát **trước đó** (lúc VM start, theo policy mặc định của kernel host) có thể nằm rải trên nhiều node. Luôn set cả 2 cùng lúc, và tốt nhất set từ domain XML **trước khi** VM start lần đầu (đổi `numatune` khi VM đang chạy không re-tổ chức lại memory đã cấp phát).

> [!warning] Không nên pin `emulatorpin` chung CPU với `vcpupin` nếu có dư CPU
> Nếu host đủ CPU, tách riêng CPU cho `emulatorpin` (QEMU main loop, I/O thread) khỏi dải CPU dành cho `vcpupin` (vCPU chính) — tránh 2 loại thread cạnh tranh cùng lõi khi hệ thống dưới tải cao. Với host ít CPU (VD lab 4-core), việc tách này không thực tế — chấp nhận share.

## Resources

- `man numactl`, `man virsh` (phần cputune/numatune)
- Libvirt NUMA tuning docs: https://libvirt.org/formatdomain.html#cpu-tuning

---
*Xem thêm: [[vCPU Threads & Scheduling]] | [[Memory Virtualization - EPT & NPT]] | [[Hugepages & Memory Tuning]] | [[Kvm-virtualization|KVM Virtualization]]*
