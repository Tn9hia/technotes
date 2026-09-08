### DRS là gì?

- DRS là tính năng thuộc **vSphere Cluster** cho phép **phân bổ tài nguyên tự động** (CPU, RAM) giữa các máy ảo dựa trên mức độ sử dụng tài nguyên thực tế.
- Tự động **di chuyển máy ảo (vMotion)** để tối ưu tài nguyên trên các host ESXi trong cluster.

---

### 🧠 Cách hoạt động

1. **Theo dõi liên tục** mức sử dụng tài nguyên của các host và VM.
2. Dựa vào các **thuật toán load balancing**, DRS đánh giá có nên di chuyển VM không.
3. Nếu cần, thực hiện **vMotion tự động** hoặc đưa ra **gợi ý thủ công** (tùy theo chế độ).

---

### ⚙️ Các chế độ hoạt động

|Chế độ|Mô tả|
|---|---|
|Manual|Chỉ gợi ý di chuyển VM, admin tự thực hiện.|
|Partially Automated|Tự động chọn host khi VM được bật, còn lại gợi ý di chuyển.|
|Fully Automated|Hoàn toàn tự động vMotion để cân bằng tài nguyên.|