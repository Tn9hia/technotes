## Architecture
![[Pasted image 20240812224803.png]]

## PromQL
- **PromQL** (Prometheus Query Language) là ngôn ngữ truy vấn của Prometheus, dùng để lấy và phân tích dữ liệu time-series từ cơ sở dữ liệu Prometheus.
- Dữ liệu trong Prometheus được lưu dưới dạng **metrics** (chỉ số), ví dụ:
    - node_cpu_seconds_total: Thời gian CPU sử dụng.
    - node_memory_MemAvailable_bytes: Dung lượng bộ nhớ trống.
- PromQL giúp bạn:
    - Lấy dữ liệu thô (raw data).
    - Tính toán (tỷ lệ, tổng hợp, trung bình, v.v.).
    - Tạo biểu đồ trong Grafana dựa trên dữ liệu này.
