Trong Kubernetes, **Pod QoS (Quality of Service)** có **3 mode**. Nói thẳng luôn: QoS quyết định _pod của bạn sống dai hay bay màu trước_ khi node thiếu tài nguyên 😄

---

## 1️⃣ Guaranteed – “con nhà người ta”

**Điều kiện**

- **TẤT CẢ containers** trong pod
- **request == limit** cho **CPU và Memory**
- Không được thiếu cái nào

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Đặc điểm**

- QoS cao nhất
- Ít bị OOMKill nhất
- Scheduler ưu tiên ổn định
- CPU được pin chắc hơn (cgroup chặt)

**Khi nào dùng**

- Database
- Core service
- App “chết là toang”

👉 Tóm gọn: _Muốn sống lâu, phải khai báo rõ ràng và nghiêm túc._

---

## 2️⃣ Burstable – “linh hoạt nhưng không bất tử”

**Điều kiện**

- Có **requests** (CPU hoặc memory)    
- Nhưng **request ≠ limit** **hoặc** chỉ khai báo một trong hai

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

**Đặc điểm**

- QoS trung bình
- Dùng tài nguyên vượt request nếu node còn dư
- Khi node thiếu RAM → bị giết **sau BestEffort**, **trước Guaranteed**

**Khi nào dùng**

- Web app thông thường
- API backend
- Service scale nhiều replica

👉 Đây là **mode phổ biến nhất trong đời thực**.

---

## 3️⃣ BestEffort – “vô gia cư”

**Điều kiện**

- **KHÔNG khai báo requests và limits** cho CPU & memory
```yaml
# không có resources
```

**Đặc điểm**

- QoS thấp nhất
- Không được guarantee gì cả
- Node thiếu RAM → **bị OOMKill đầu tiên**
- Scheduler coi như “ở ké”

**Khi nào dùng**

- Job test
- Debug tạm
- Tool linh tinh (curl, busybox)

👉 Nói thẳng: **đừng chạy production kiểu này**.

---

## 🧠 So sánh nhanh cho dễ nhớ

| QoS        | Request | Limit    | Độ ổn định   | Bị kill khi thiếu tài nguyên |
| ---------- | ------- | -------- | ------------ | ---------------------------- |
| Guaranteed | = limit | Có       | 🔥 Cao nhất  | Cuối cùng                    |
| Burstable  | Có      | Có/Không | ⚡ Trung bình | Ở giữa                       |
| BestEffort | Không   | Không    | 💀 Thấp      | Đầu tiên                     |

---

## 💡 Tip thực tế (không sách vở)

- **Không set limit CPU** → app ít bị throttle hơn
- **Luôn set request memory** → tránh pod chết oan
- Prod nghiêm túc: **Guaranteed hoặc Burstable**
- BestEffort chỉ để _nghịch_

Nếu bạn muốn, mình có thể:

- Check QoS của pod hiện tại
- Gợi ý resource cho app cụ thể
- Phân tích OOMKill / eviction log

Cứ bắn câu hỏi tiếp 👀