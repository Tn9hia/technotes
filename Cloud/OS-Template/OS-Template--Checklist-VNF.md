---
tags:
  - os-template
  - checklist
  - vnf
  - network-function
  - appliance
---

# OS Template - Checklist VNF / Virtualized Network Function

Parent: [[OS-Template--Checklist-Chung]]
Related: [[OS-Template--Index]]

> [!NOTE]
> Nhóm này khác biệt căn bản so với OS template thường: image là appliance đóng gói sẵn từ vendor (FortiGate, vRouter, load balancer ảo...). **KHÔNG generalize/sysprep** như Linux/Windows — mỗi VM deploy từ appliance image thường giữ nguyên license/config riêng, không phải "clone vô hạn từ 1 golden image".

Vì vậy phần lớn [[OS-Template--Checklist-Chung]] (mục generalize/seal) **không áp dụng** cho nhóm này. File này thay thế các mục đó bằng checklist riêng.

## 1. Nguồn image

- [ ] Image lấy trực tiếp từ vendor, đúng version compatible với KVM/libvirt version đang chạy — kiểm tra compatibility matrix của vendor trước khi import
- [ ] Không tự generalize/sysprep image vendor trừ khi tài liệu vendor xác nhận hỗ trợ (đa số appliance network KHÔNG hỗ trợ sysprep kiểu OS thường)
- [ ] Verify checksum/signature image từ vendor (áp dụng mục 1 của checklist chung bình thường)

## 2. Multi-NIC ordering

- [ ] Xác định đúng thứ tự NIC: NIC đầu tiên thường là management (mgmt0), các NIC sau là data plane — sai thứ tự khiến appliance nhận nhầm interface
- [ ] Đặt NIC trong platform đúng thứ tự PCI slot mà vendor mong đợi — kiểm tra tài liệu vendor, nhiều appliance rất nhạy với thứ tự này
- [ ] Nếu cần nhiều NIC data plane (ví dụ firewall inline 2-arm), xác nhận platform hỗ trợ đủ số NIC trong 1 VM — giới hạn số NIC khác nhau giữa CloudStack/OpenStack/OpenNebula

## 3. Performance tuning

- [ ] CPU model `host-passthrough` để lộ đầy đủ instruction set cho DPDK/vector processing nếu appliance dùng
- [ ] Hugepages bật nếu vendor yêu cầu (hầu hết appliance dùng DPDK cần hugepages)
- [ ] CPU pinning/NUMA alignment — pin vCPU đúng NUMA node chứa NIC PCI, đặc biệt quan trọng khi dùng SR-IOV
- [ ] virtio-net multiqueue, hoặc SR-IOV/PCI passthrough cho NIC data plane nếu cần throughput cao hơn virtio thường
- [ ] Tắt các tính năng ảnh hưởng latency nếu vendor khuyến cáo (ví dụ KSM, memory ballooning)

## 4. Day-0 config / bootstrap

- [ ] Xác định cơ chế inject config ban đầu: cloud-init user-data, config-drive, hay console/CLI thủ công — tuỳ vendor hỗ trợ gì
- [ ] License file/key: inject qua day-0 config hay nhập tay qua console sau khi boot — ghi rõ quy trình chuẩn
- [ ] Serial console access bắt buộc test được trước khi dùng thật — nhiều appliance chỉ expose CLI qua console ở lần boot đầu, chưa có network

## 5. HA / Availability

- [ ] Anti-affinity rule giữa các node trong 1 HA pair (không để 2 node cùng 1 compute host)
- [ ] Xác nhận appliance hỗ trợ HA mode nào (active-passive/active-active) và network requirement riêng cho heartbeat link

## 6. Versioning

- [ ] Naming convention ghi rõ vendor + version + build, ví dụ `fortigate-7.4.x-vXX`
- [ ] Theo dõi EOL/EOS (End of Support) của vendor cho version đang dùng — không giữ version đã hết hỗ trợ bảo mật

## Reference

- Tài liệu compatibility matrix của từng vendor (tham khảo trực tiếp docs chính thức của appliance đang dùng)
