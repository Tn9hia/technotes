---
tags:
  - kvm
  - prerequisites
---

# KVM Prerequisites

Kiến thức nền nên có **trước khi** đọc sâu vào các note KVM khác trong vault này. Không phải danh sách "phải học hết mới được đọc tiếp" — nhưng thiếu các mục dưới sẽ khiến nhiều chỗ ở [[KVM Kernel Module & Hardware Virtualization Extensions]], [[CPU Pinning & NUMA]], [[Device Passthrough - VFIO & GPU]] khó hiểu vì sao lại làm vậy.

## 1. Linux fundamentals

- **Process vs thread**: một QEMU process có nhiều thread (mỗi vCPU = 1 thread) — nếu chưa quen `ps -eLf`, `taskset`, `/proc/<pid>/task/`, nên ôn lại trước khi vào [[vCPU Threads & Scheduling]].
- **cgroups**: libvirt dùng cgroups (v1 hoặc v2 tùy distro) để giới hạn CPU/memory/IO của từng VM. Không cần biết viết cgroup tay, nhưng cần biết nó tồn tại khi debug "VM bị throttle mà không rõ vì sao".
- **udev & sysfs**: PCI passthrough (VFIO) và SR-IOV thao tác trực tiếp qua `/sys/bus/pci/devices/...` — cần quen đọc sysfs cơ bản.
- **systemd**: `libvirtd.service` (hoặc `virtqemud.socket` ở bản mới), log qua `journalctl -u libvirtd`.

## 2. x86 virtualization cơ bản (không cần chuyên sâu CPU design)

- **Ring 0-3 & vì sao trap-and-emulate cũ chậm**: hiểu sơ lược tại sao x86 truyền thống không "virtualizable" thuần túy (một số instruction nhạy cảm không trap được ở ring khác) — đây là lý do Intel/AMD phải thêm VT-x/AMD-V thay vì để software (như QEMU thời chưa có KVM, hay Xen HVM cũ) tự xử lý bằng binary translation.
- **VMX root mode vs non-root mode** (Intel) / tương tự ở AMD-V: khái niệm tối thiểu cần nắm trước khi đọc [[KVM Kernel Module & Hardware Virtualization Extensions]].
- **Virtual memory & page table cơ bản**: TLB, page walk — bắt buộc để hiểu vì sao EPT/NPT (xem [[Memory Virtualization - EPT & NPT]]) là bước tiến so với shadow page table.

## 3. Networking cơ bản

- Linux bridge (`brctl`/`ip link`), VLAN tagging, NAT/iptables cơ bản — nền cho [[Linux Bridge & NAT Networking]].
- Khái niệm TAP/TUN device (`/dev/net/tun`) — QEMU dùng tap interface để nối VM vào bridge host.

## 4. Storage cơ bản

- LVM (PV/VG/LV), khác biệt raw block device vs filesystem-backed image — nền cho [[Storage Backends Overview]].
- Khái niệm sparse file, copy-on-write — cần trước khi đọc [[Disk Image Formats - qcow2 vs raw]].

## 5. Nếu đã biết VMware/ESXi — mapping tư duy nhanh

| Khái niệm ESXi/vCenter | Tương đương gần nhất bên KVM | Khác biệt cần chú ý |
|---|---|---|
| ESXi host (VMkernel) | Linux host + kvm.ko | KVM dùng chung kernel Linux, không phải OS riêng |
| vCenter | libvirt (`virsh`) hoặc orchestrator (CloudStack/OpenStack) | libvirt không có UI tập trung sẵn — cần thêm virt-manager/CloudStack |
| VMX file | Domain XML | XML khai báo tường minh hơn, không có GUI wizard mặc định |
| vmxnet3 driver | virtio-net | Cả 2 đều là paravirtualized driver, cùng ý tưởng |
| VMFS/vSAN datastore | libvirt storage pool (LVM/NFS/RBD...) | libvirt storage pool chỉ là lớp abstraction mỏng, backend thật đa dạng hơn |
| vMotion | KVM/libvirt Live Migration | Yêu cầu tự thiết lập shared storage + network, không "tự động" như vMotion tích hợp sẵn |

## 6. Công cụ nên cài sẵn trên máy học/lab

```bash
# Debian/Ubuntu
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils virt-manager

# RHEL/Rocky/Alma
sudo dnf install qemu-kvm libvirt virt-install bridge-utils virt-manager

# Kiểm tra nested virtualization nếu lab trong VM (VMware Workstation/Fusion, cloud instance...)
egrep -c '(vmx|svm)' /proc/cpuinfo
```

> [!warning] Lesson learned: học KVM trong VM lồng VM (nested) dễ gây hiểu sai hiệu năng
> Nested virtualization (chạy KVM trong 1 VM VMware/cloud) hoạt động được để **học cú pháp/API**, nhưng số liệu hiệu năng (latency, IOPS) sẽ **sai lệch nặng** — không dùng kết quả benchmark từ môi trường nested để đưa ra quyết định sizing/tuning cho production bare-metal.

---
*Xem thêm: [[Kvm-virtualization|KVM Virtualization]]*
