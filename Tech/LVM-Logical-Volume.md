### **Giới thiệu về LVM trong Linux**

#### 1️⃣ **LVM là gì?**

LVM (Logical Volume Manager) là một hệ thống quản lý ổ đĩa cho phép chia và quản lý dung lượng linh hoạt hơn so với các phân vùng truyền thống (`/dev/sda1`, `/dev/sdb1`...).

#### 2️⃣ **Tại sao nên dùng LVM?**

- **Co giãn linh hoạt**: Dễ dàng mở rộng hoặc thu nhỏ dung lượng mà không cần xóa dữ liệu.
    
- **Quản lý nhiều ổ đĩa**: Gộp nhiều ổ thành một nhóm, tạo logical volume theo nhu cầu.
    
- **Snapshot**: Tạo bản sao nhanh của ổ đĩa để backup hoặc thử nghiệm.
    

#### 3️⃣ **Các thành phần chính của LVM**

LVM hoạt động dựa trên 3 lớp chính:

✅ **Physical Volume (PV)** – Ổ đĩa vật lý (hoặc phân vùng) được đưa vào quản lý bởi LVM.  
✅ **Volume Group (VG)** – Nhóm tập hợp các PV lại thành một không gian lưu trữ chung.  
✅ **Logical Volume (LV)** – Phân vùng ảo bên trong VG, có thể sử dụng như một ổ đĩa bình thường.

**Ví dụ minh họa**:

- `/dev/sdb` và `/dev/sdc` được chuyển thành **PV**.
    
- PVs này được nhóm vào **VG1**.
    
- Từ VG1, ta tạo các **LV** như `/dev/VG1/root`, `/dev/VG1/home`, v.v.

## 1️⃣ Quản lý Physical Volume (PV) trong LVM
#### **Tạo Physical Volume (PV)**

Trước khi thêm ổ đĩa vào LVM, bạn cần chuyển đổi nó thành **Physical Volume (PV)**.  
👉 Lệnh để tạo PV:
```
pvcreate /dev/sdb /dev/sdc
```
Lệnh này biến `/dev/sdb` và `/dev/sdc` thành PV, sẵn sàng để sử dụng với LVM.

#### **Kiểm tra thông tin PV**

📌 **Liệt kê tất cả các PV đã tạo:**
```
pvdisplay
```
📌 **Liệt kê PV dưới dạng bảng ngắn gọn:**
```
pvs
```
📌 **Kiểm tra PV cụ thể:**
```
pvdisplay /dev/sdb
```
#### **Xóa một PV khỏi LVM**

👉 Nếu muốn loại bỏ một PV khỏi hệ thống:
```
pvremove /dev/sdb
```
⚠️ **Lưu ý:** Trước khi xóa, PV không được thuộc về bất kỳ Volume Group (VG) nào. Nếu nó đang thuộc VG, bạn cần xóa VG hoặc di chuyển dữ liệu trước.
## **2️⃣ Quản lý Volume Group (VG) trong LVM**

#### **Tạo Volume Group (VG)**

Sau khi có **Physical Volume (PV)**, bạn có thể tạo **Volume Group (VG)** để gom nhiều PV lại.  
👉 Lệnh tạo VG từ 2 PV:
```
vgcreate my_vg /dev/sdb /dev/sdc
```
📌 `my_vg` là tên của Volume Group.

#### **Kiểm tra thông tin VG**

📌 **Liệt kê tất cả VG**:
```
vgdisplay
```
📌 **Xem thông tin ngắn gọn về VG**:
```
vgs
```
📌 **Kiểm tra chi tiết một VG cụ thể**:
```
vgdisplay my_vg
```
#### **Thêm PV vào VG**

👉 Khi muốn mở rộng VG bằng cách thêm ổ đĩa mới:
```
vgextend my_vg /dev/sdd
```
#### **Xóa hoặc thu nhỏ VG**

👉 **Loại bỏ một PV khỏi VG** (nếu dữ liệu không còn trên đó):
```
vgreduce my_vg /dev/sdb
```
👉 **Xóa VG hoàn toàn**:
```
vgremove my_vg
```
⚠️ **Lưu ý:** Trước khi xóa VG, bạn cần xóa hết các Logical Volume (LV) bên trong.

## **3️⃣ Quản lý Logical Volume (LV) trong LVM**

#### **Tạo Logical Volume (LV)**

Logical Volume hoạt động như phân vùng thông thường.  
👉 Tạo một LV dung lượng 10GB từ VG:
```
lvcreate -L 10G -n my_lv my_vg
```
📌 `-L 10G`: Dung lượng của LV.  
📌 `-n my_lv`: Tên của LV.  
📌 `my_vg`: Volume Group chứa LV.

#### **Kiểm tra thông tin LV**

📌 **Liệt kê tất cả LV**:
```
lvdisplay
```
📌 **Xem thông tin ngắn gọn**:
```
lvs
```
📌 **Kiểm tra chi tiết một LV**:
```
lvdisplay /dev/my_vg/my_lv
```
#### **Mở rộng hoặc thu nhỏ LV**

👉 **Mở rộng LV thêm 5GB**:
```
lvextend -L +5G /dev/my_vg/my_lv
resize2fs /dev/my_vg/my_lv  # Áp dụng với ext4
```
👉 **Thu nhỏ LV xuống 8GB (chỉ áp dụng nếu chưa dùng hết dung lượng)**:
```
resize2fs /dev/my_vg/my_lv 8G  # Giảm kích thước file system trước
lvreduce -L 8G /dev/my_vg/my_lv
```
⚠️ **Thu nhỏ LV có thể gây mất dữ liệu, cần sao lưu trước!**

#### **Xóa LV**

👉 Khi không cần sử dụng LV nữa:
```
lvremove /dev/my_vg/my_lv
```
