**VMware Avi** (hiện được gọi là **VMware NSX Advanced Load Balancer**) là một giải pháp cân bằng tải (**Load Balancer**) và tường lửa ứng dụng web (**Web Application Firewall - WAF**) được thiết kế theo nguyên tắc phần mềm định nghĩa (software-defined). Đây là sản phẩm của Avi Networks, được VMware mua lại vào năm 2019, và nay là một phần của danh mục giải pháp mạng và bảo mật của VMware. VMware Avi cung cấp các dịch vụ phân phối ứng dụng (**Application Delivery Controller - ADC**) hiện đại, hỗ trợ triển khai trên nhiều nền tảng, từ trung tâm dữ liệu tại chỗ (on-premises) đến môi trường đám mây (public/private cloud) và Kubernetes.
## Thành phần

| Thành phần               | Chức năng                                                                                                                    |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Controller Cluster**   | Nơi điều khiển trung tâm, quản lý cấu hình, phân tích, theo dõi hiệu suất (analytics). Thường triển khai dạng HA với 3 node. |
| **Service Engines (SE)** | Thực hiện việc load balancing, SSL termination, caching,... Có thể scale out / in linh hoạt.                                 |
#### Service Engines (SEs) là gì?
- **Vai trò**: SEs là các thực thể xử lý lưu lượng thực tế trong hệ thống Avi. Chúng hoạt động như các máy chủ proxy (reverse proxy) để phân phối lưu lượng từ người dùng đến các máy chủ ứng dụng (application servers) theo các chính sách được định nghĩa.
- **Chức năng chính**:
    - **Cân bằng tải**: Phân phối lưu lượng đến các máy chủ back-end dựa trên thuật toán (Round Robin, Least Connections, v.v.).
    - **Bảo mật**: Xử lý SSL/TLS termination, áp dụng chính sách tường lửa ứng dụng web (WAF) để bảo vệ ứng dụng.
    - **Giám sát**: Thu thập dữ liệu thời gian thực về hiệu suất ứng dụng, sức khỏe máy chủ (health monitoring), và log giao dịch.
    - **Tối ưu hóa**: Hỗ trợ nén dữ liệu, caching, và tối ưu hóa TCP/HTTP để cải thiện hiệu suất.
- **Vị trí trong kiến trúc**:
    - SEs nhận cấu hình và lệnh từ **Avi Controller** (mặt phẳng điều khiển) và xử lý lưu lượng trong thời gian thực.
    - Chúng giao tiếp với cả mạng phía client (front-end) và mạng phía máy chủ ứng dụng (back-end).

#### Đặc điểm của Service Engines
- **Triển khai linh hoạt**:
    - SEs có thể được triển khai dưới dạng **máy ảo (VM)**, **container**, hoặc trên **bare-metal** tùy thuộc vào môi trường (vSphere, Kubernetes, AWS, Azure, v.v.).
    - Có thể tự động mở rộng (scale-out) hoặc thu hẹp (scale-in) dựa trên nhu cầu lưu lượng.
- **Tính độc lập**:
    - Mỗi SE hoạt động độc lập, nhưng được quản lý tập trung bởi Avi Controller.
    - Nhiều SE có thể được nhóm lại (Service Engine Group) để xử lý lưu lượng cho một Virtual Service.
- **Hiệu suất cao**: SEs được tối ưu để xử lý lưu lượng lớn, hỗ trợ hàng triệu kết nối đồng thời và giao dịch SSL/TLS.
## Chức năng chính

- **Cân bằng tải (Load Balancing)**: Hỗ trợ cân bằng tải cục bộ (Local) và toàn cầu (Global Server Load Balancing - GSLB), đảm bảo ứng dụng luôn sẵn sàng và tối ưu hiệu suất.
- **Tường lửa ứng dụng web (WAF)**: Bảo vệ ứng dụng khỏi các cuộc tấn công mạng như SQL Injection, DDoS tầng 4-7.
- **Tích hợp Kubernetes**: Cung cấp dịch vụ ingress và hỗ trợ Gateway API, giúp quản lý lưu lượng trong môi trường container.
- **Phân tích và tự động hóa**: Cung cấp khả năng phân tích hiệu suất ứng dụng thời gian thực, tự động mở rộng (predictive autoscaling) và tích hợp với các nền tảng như VMware vSphere, NSX, và Tanzu.
- **Hỗ trợ đa đám mây**: Hoạt động trên các môi trường VMware Cloud Foundation (VCF), vSphere, AWS, Azure, Google Cloud, OpenShift, v.v.

## Virtual Service Scaling
Virtual services may be scaled across one or more SEs using either native (L2 punting) or BGP-based load-balancing techniques.

When a native SE scaling is used, one SE will be the primary for a given virtual service and will advertise that virtual services IP address from the SEs own MAC address. The primary SE may either process and load balance a client connection itself, or it may forward the connection via Layer 2 to the MAC address of one of the secondary SEs having available capacity.

- Scaling out a virtual service distributes that virtual service to an additional SE. By default, Avi supports a maximum of four SEs per virtual service when native load balancing of SEs is in play. In BGP environments the maximum can be increased to 64. The max number supported is determined by the capabilities of the upstream router. 
- Scaling in a virtual service reduces the number of SEs over which its load is distributed. A virtual service will always require a minimum of one SE.
#### Process to scales out
Khi Virtual Service cần mở rộng, Avi **scale-out traffic** sang nhiều Service Engine (SE). Cách hoạt động:

1. **Primary SE** gửi GARP để nhận tất cả traffic từ client.
2. **Primary SE** có thể xử lý một phần traffic.
3. **Phần dư traffic** được chuyển tiếp ở **Layer 2** đến các **secondary SEs**.
4. Các secondary SEs xử lý traffic và **thay đổi source IP** sang chính IP của chúng.
5. Backend server trả lời về source IP này.
6. **Secondary SEs phản hồi trực tiếp đến client** (không qua primary SE).
#### Process to migrate
- **Migrate** cho phép di chuyển dịch vụ ảo (Virtual Service) từ SE (Service Engine) cũ sang SE mới một cách nhẹ nhàng.
- Sau **30 giây**, SE cũ sẽ bị gỡ bỏ khỏi dịch vụ ảo và SE mới sẽ tiếp nhận kết nối mới.

>[!note]
> Kết nối hiện tại có 30 giây để hoàn tất. Nếu không, kết nối sẽ bị ngắt và cần khởi động lại

