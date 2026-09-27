---
tags:
  - kvm
  - kernel
  - memory
---

# Memory Virtualization — EPT & NPT

**EPT (Extended Page Tables, Intel)** và **NPT (Nested Page Tables, AMD — còn gọi RVI)** là cơ chế hardware cho phép CPU dịch địa chỉ **2 tầng** ngay trong MMU: guest virtual → guest physical (do guest OS quản lý, hypervisor không đụng vào) → host physical (do KVM quản lý). Đây là lý do KVM hiện đại **không cần shadow page table** — kỹ thuật cũ, chậm hơn nhiều, mà hypervisor phải tự duy trì bằng software.

> [!tip] So với vSAN/ESXi
> ESXi cũng dùng chính EPT/NPT của CPU (không phải công nghệ riêng của VMware) — đây là tính năng CPU chung cho mọi hypervisor loại 2-level translation. Điểm khác là cách mỗi hypervisor tận dụng thêm (VD large page mapping trong EPT, hoặc cách quản lý dirty page cho migration) mới là nơi tạo khác biệt hiệu năng.

## Why — vấn đề gì cần giải quyết?

Trước khi có EPT/NPT (CPU đời 2006-2008 trở về trước), hypervisor phải tự dựng **shadow page table**: một bản page table riêng do hypervisor giữ, ánh xạ trực tiếp guest virtual → host physical, rồi đồng bộ (liên tục "shadow") mỗi khi guest OS sửa page table của nó. Việc đồng bộ này cực tốn — mỗi lần guest OS switch context hoặc sửa page table đều trigger VMEXIT để hypervisor cập nhật shadow table tương ứng. EPT/NPT loại bỏ hoàn toàn bước đồng bộ này: CPU tự đi qua 2 tầng page table (guest page table + EPT/NPT table do hypervisor set up 1 lần) mà **không cần trap về hypervisor** cho mỗi lần switch.

## How — cơ chế hoạt động

```
Guest virtual address
   │
   ├─ Tầng 1: Guest page table (guest OS tự quản lý, hypervisor không biết nội dung)
   │  → ra Guest Physical Address (GPA)
   │
   ├─ Tầng 2: EPT/NPT table (KVM quản lý, ánh xạ GPA → Host Physical Address)
   │  → ra Host Physical Address (HPA) thật
   │
   └─ CPU (hardware MMU) tự đi qua CẢ 2 tầng trong 1 lần page walk
      (không có software trap nào ở giữa, trừ khi gặp EPT violation)
```

Chỉ khi xảy ra **EPT violation** (GPA chưa được map trong EPT table — VD lần đầu truy cập một trang RAM mới cấp cho guest) mới trap về KVM để KVM cấp phát page vật lý và cập nhật EPT entry. Sau lần đầu đó, mọi truy cập tiếp theo tới cùng trang nhớ **không tốn thêm overhead nào** — CPU cache lại kết quả dịch địa chỉ 2 tầng này ngay trong TLB (dùng thêm trường tag để phân biệt theo VM, gọi là VPID/ASID).

## Key Config — cần nhớ

```bash
# Kiểm tra EPT có được CPU hỗ trợ và KVM có dùng không
cat /proc/cpuinfo | grep -o ept          # Intel: xuất hiện "ept" trong flags nếu CPU hỗ trợ
cat /sys/module/kvm_intel/parameters/ept # Y = KVM đang dùng EPT (mặc định Y nếu CPU hỗ trợ)

# AMD tương ứng
cat /sys/module/kvm_amd/parameters/npt
```

| Tham số | Ảnh hưởng | Ghi chú |
|---|---|---|
| `ept=Y` / `npt=1` | Bật second-level translation bằng hardware | Gần như luôn nên để mặc định — tắt chỉ để debug/test shadow paging cũ |
| `kvm_intel.eptad` | Bật Accessed/Dirty bit tracking trong EPT | Quan trọng cho live migration — KVM dùng dirty bit để biết trang nào cần gửi lại, xem [[Live Migration]] |
| Hugepages (transparent hoặc static) | EPT table nhỏ hơn, giảm số tầng page walk | Xem [[Hugepages & Memory Tuning]] — tác động rõ nhất với workload có working set lớn |

## Performance Considerations

> [!tip] Đòn bẩy hiệu năng dễ bị bỏ qua: hugepages giảm áp lực lên EPT
> Page walk qua EPT vẫn tốn nhiều bước hơn page walk thường (guest table 4 tầng + EPT table 4 tầng = tối đa 24 lần truy cập memory trong trường hợp xấu nhất trên x86_64, dù có TLB cache giảm bớt rất nhiều trong thực tế). Dùng **hugepages** (2MB hoặc 1GB) giảm số tầng cần đi qua và giảm TLB miss rate đáng kể — đây là lý do mọi khuyến nghị tuning KVM cho workload database/latency-sensitive đều bắt đầu bằng hugepages, không phải chỉnh CPU pinning trước.

## Gotchas & Lessons Learned

> [!warning] EPT violation lần đầu luôn có "cold start cost"
> Khi VM mới khởi động hoặc sau khi memory bị balloon ra rồi cấp lại (xem virtio-balloon ở [[Virtio Devices]]), mỗi trang nhớ "mới" đều cần ít nhất 1 EPT violation để KVM map lần đầu — với VM có RAM lớn, hiện tượng này thể hiện rõ như "VM chạy chậm vài phút đầu rồi mới ổn định". Không phải bug — là chi phí warm-up tự nhiên của cơ chế lazy-mapping.

## Resources

- Intel SDM Volume 3C, Chapter 28 (EPT) — tài liệu gốc
- AMD64 Architecture Programmer's Manual Volume 2, Chapter 15 (NPT)

---
*Xem thêm: [[KVM Kernel Module & Hardware Virtualization Extensions]] | [[Hugepages & Memory Tuning]] | [[Live Migration]] | [[Kvm-virtualization|KVM Virtualization]]*
