---
tags:
  - performance
  - memory
---

# Hugepages & Memory Tuning

Trang nhớ (page) mặc định của Linux là **4KB** — với VM có RAM lớn (VD 32GB), số page table entry cần quản lý cực nhiều, gây áp lực TLB miss cao (đặc biệt qua 2 tầng dịch địa chỉ của EPT/NPT, xem [[Memory Virtualization - EPT & NPT]]). **Hugepages** (2MB hoặc 1GB mỗi trang) giảm mạnh số entry cần thiết — ít TLB miss hơn, page walk ngắn hơn, giảm overhead memory management nói chung.

> [!tip] Đây là tuning có ROI cao nhất, nên làm trước CPU pinning
> Nhiều hướng dẫn tuning KVM online liệt kê CPU pinning trước hugepages, nhưng thực tế với workload RAM-intensive (database, in-memory cache), **hugepages thường mang lại cải thiện rõ rệt hơn** so với riêng CPU pinning — vì nó tác động trực tiếp lên tần suất page walk, thứ luôn xảy ra bất kể vCPU chạy ở CPU nào.

## Where — 2 loại hugepages

| Loại | Kích thước | Cấp phát | Đặc điểm |
|---|---|---|---|
| **Transparent HugePages (THP)** | 2MB (tự động, kernel quản lý) | Tự động, không cần cấu hình gì | Tiện nhưng không đảm bảo — kernel có thể "defragment" hoặc gộp/tách page linh hoạt, đôi khi gây latency spike khó đoán (compaction) |
| **Static HugePages (hugetlbfs)** | 2MB hoặc 1GB, cấp phát cố định trước | Cần đặt số lượng trước lúc boot hoặc runtime (nếu đủ RAM liền mạch) | Đảm bảo tuyệt đối — VM được cấp đúng số hugepage đã reserve, không bị kernel "lấy lại" bất ngờ |

> [!warning] THP không phải "hugepages thật" cho production nghiêm túc
> THP tiện vì không cần cấu hình, nhưng cơ chế `khugepaged` (defragment nền) có thể gây **latency spike bất ngờ** khi kernel cố gắng gộp page thành hugepage giữa lúc hệ thống đang chạy — với workload nhạy latency (database, real-time), khuyến nghị chuẩn là **tắt THP** và dùng **static hugepages** khai báo tường minh cho VM.

## How — cấp static hugepages

```bash
# 1. Kiểm tra kích thước hugepage hệ thống hỗ trợ
cat /proc/meminfo | grep Huge

# 2. Reserve 1GB hugepages (VD 16 trang x 1GB = 16GB dành cho VM)
echo 16 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Để giữ qua reboot — thêm vào /etc/sysctl.conf hoặc kernel boot param
# GRUB: default_hugepagesz=1G hugepagesz=1G hugepages=16
```

```xml
<domain>
  <memoryBacking>
    <hugepages>
      <page size='1048576' unit='KiB' nodeset='0'/>
    </hugepages>
  </memoryBacking>
  <memory unit='GiB'>16</memory>
</domain>
```

> [!warning] Reserve hugepages TRƯỚC khi RAM bị fragment — làm ngay sau boot, không phải giữa lúc host đã chạy lâu
> Hugepages 1GB cần **1GB RAM vật lý liên tục** (contiguous) — sau khi host chạy một thời gian, RAM bị fragment bởi các allocation nhỏ khác, `echo N > nr_hugepages` có thể **fail thầm lặng** (trả về số hugepage thực tế cấp được ít hơn yêu cầu, không báo lỗi rõ ràng). Luôn set qua kernel boot parameter (GRUB) để kernel reserve **ngay lúc boot**, trước khi bất kỳ allocation nào khác diễn ra.

## Tắt THP nếu chọn dùng static hugepages

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# Thêm vào rc.local hoặc systemd service để giữ qua reboot
```

## Key Config — kiểm tra kết quả

```bash
cat /proc/meminfo | grep -i huge
# HugePages_Total, HugePages_Free, HugePages_Rsvd — Rsvd tăng khi VM start dùng hugepages

virsh dumpxml vm01 | grep -A3 memoryBacking
```

## Gotchas & Lessons Learned

> [!warning] Lesson learned: reserve hugepages quá nhiều RAM host → host thiếu RAM cho chính nó (OOM)
> Hugepages đã reserve **không thể dùng chung** với bất kỳ tiến trình nào khác ngoài VM được gán — nếu tổng hugepages reserve gần hết RAM host, các process host thường (libvirtd, monitoring agent, SSH...) có thể bị OOM-killed khi RAM "thường" cạn, dù `free -h` vẫn hiển thị RAM "còn" (thực ra đã bị khóa cho hugepages, không hiển thị đúng trong `free` theo cách trực quan). Luôn để dư ít nhất 2-4GB RAM thường cho host, không reserve hugepages sát 100% tổng RAM.

## Resources

- Kernel hugetlbpage documentation: `Documentation/admin-guide/mm/hugetlbpage.rst`
- Libvirt memory backing docs: https://libvirt.org/formatdomain.html#memory-backing

---
*Xem thêm: [[Memory Virtualization - EPT & NPT]] | [[CPU Pinning & NUMA]] | [[Kvm-virtualization|KVM Virtualization]]*
