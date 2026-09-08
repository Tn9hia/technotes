### 🧩 HA là gì?

- VMware **High Availability (HA)** là tính năng cho phép **tự động khởi động lại máy ảo** khi một host trong cluster bị **lỗi hoặc mất kết nối**.
    
- Không yêu cầu can thiệp thủ công, giúp **tăng tính sẵn sàng** cho dịch vụ.
    

---

## ⚙️ Nguyên lý hoạt động

1. **vCenter** cấu hình và quản lý Cluster HA.
2. Một ESXi host được chọn làm **Master Host**, các host còn lại là **Slaves**.
3. Master theo dõi:
    - **Heartbeat** từ Slave hosts.
    - Trạng thái VM thông qua file lock trên datastore.
4. Khi một host không phản hồi:
    - Master xác nhận mất kết nối qua datastore heartbeat.
    - Tiến hành **khởi động lại VM** bị ảnh hưởng trên các host còn hoạt động.

---

## 🧠 Các thành phần chính

| Thành phần                  | Mô tả                                                                       |
| --------------------------- | --------------------------------------------------------------------------- |
| **Master Host**             | Quản lý cluster HA, quyết định failover khi có sự cố.                       |
| **Datastore Heartbeating**  | Dùng datastore làm kênh kiểm tra dự phòng nếu mất kết nối mạng.             |
| **Admission Control**       | Ngăn không cho bật thêm VM nếu không đủ tài nguyên dự phòng.                |
| **Host Isolation Response** | Quy định hành động khi host bị cô lập mạng (shutdown VM, giữ nguyên, v.v.). |
## 🔐 Admission Control trong VMware HA

### 🎯 Nhiệm vụ

- Đảm bảo **cluster luôn có đủ tài nguyên** để **restart VM** khi một (hoặc nhiều) host bị lỗi.
    
- Giúp **ngăn VM mới được bật lên** nếu tài nguyên dự phòng không đủ — tránh **quá tải** khi có sự cố.
    

---

### 📋 Các chính sách Admission Control

#### 1. **Host failures cluster tolerates**

- Xác định **số lượng host có thể bị lỗi** mà cluster vẫn đảm bảo restart toàn bộ VM.
    
- VMware tính toán theo **slot policy**:
    
    - **1 slot = lượng tài nguyên cần thiết lớn nhất của 1 VM (CPU + RAM)**.
        
    - Tổng slot tính được trên cluster → chia cho slot mỗi VM → ra số lượng VM có thể khởi động lại.
        

📌 _Ví dụ_: Nếu cluster có 3 host và cấu hình "Tolerate 1 host failure", thì luôn để dành tài nguyên tương đương 1 host.

---

#### 2. **Percentage of Cluster Resources Reserved**

- Thay vì dùng slot, bạn cấu hình **tỷ lệ % CPU và RAM dự phòng**.
    
- VMware đảm bảo số % tài nguyên đó luôn **được giữ lại**.
    

📌 _Ví dụ_: Bạn chọn 25% CPU, 25% RAM → VMware không cho bật thêm VM nếu cluster chỉ còn dưới mức này.

---

#### 3. **Disable Admission Control**

- Cho phép bật VM bất kể tài nguyên có đủ để khởi động lại khi sự cố hay không.
    
- Không khuyến nghị trong môi trường **production**.

## 🚨 Host Isolation Response trong VMware HA

### 🎯 Nhiệm vụ

- Xử lý tình huống khi một **host bị mất kết nối mạng**, nhưng **vẫn còn chạy VM**.
    
- HA cần quyết định: **làm gì với các VM trên host bị cô lập này?**
    

---

### ⚙️ Các lựa chọn Isolation Response

| Tùy chọn                      | Mô tả                                                                                           |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| **Leave Powered On**          | Giữ nguyên VM trên host bị cô lập. Dễ gây **split-brain** nếu host thực sự đã chết.             |
| **Power Off and Restart VMs** | Tắt VM trên host cô lập (nếu host còn phản hồi), sau đó HA **khởi động lại VM** trên host khác. |
| **Shutdown and Restart VMs**  | Shutdown mềm VM trước khi restart. Cần VMware Tools hoạt động bình thường trong VM.             |
| **Disabled**                  | Không thực hiện gì cả — để bạn tự xử lý.                                                        |