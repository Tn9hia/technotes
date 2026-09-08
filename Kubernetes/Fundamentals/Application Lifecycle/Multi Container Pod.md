There are 3 common patterns, when it comes to designing multi-container PODs. The first and what we just saw with the logging service example is known as a **side car** pattern. The others are the **adapter** and the **ambassador** pattern.

But these fall under the CKAD curriculum and are not required for the CKA exam. So we will be discuss these in more detail in the CKAD course.


|🧩 Loại Pod|🧍 Co-located Container Pod|🧑‍🔧 Sidecar Container|🚪 Init Container|
|---|---|---|---|
|**Mục đích chính**|Chạy nhiều container cùng lúc trong một Pod để chia sẻ tài nguyên|Hỗ trợ container chính bằng cách cung cấp chức năng phụ trợ|Chuẩn bị môi trường trước khi container chính chạy|
|**Thời điểm chạy**|Tất cả container chạy song song|Chạy song song với container chính|Chạy **trước** container chính, theo thứ tự|
|**Vòng đời**|Cùng tồn tại với Pod|Cùng tồn tại với Pod|Kết thúc trước khi container chính bắt đầu|
|**Ví dụ sử dụng**|Web server + log processor|Logging agent, proxy, metrics collector|Thiết lập cấu hình, tải dữ liệu, kiểm tra điều kiện|
|**Khả năng chia sẻ tài nguyên**|Có thể chia sẻ volume, network namespace|Có thể chia sẻ volume, network namespace|Có thể chia sẻ volume nhưng không chia sẻ network|
|**Restart khi lỗi**|Có thể được restart độc lập|Có thể được restart độc lập|Không restart, nếu lỗi thì Pod fail|
|**Số lượng container**|Không giới hạn cụ thể|Thường là 1 hoặc vài container phụ|Một hoặc nhiều container, chạy tuần tự|
|**Tính linh hoạt**|Cao, dùng cho các ứng dụng phức tạp|Rất hữu ích cho các chức năng phụ trợ|Rất phù hợp cho các bước khởi tạo bắt buộc|

---

### 🔍 Tóm tắt nhanh:

- **Co-located container**: Nhiều container cùng chạy, chia sẻ tài nguyên.
- **Sidecar**: Container phụ trợ, giúp container chính hoạt động tốt hơn.
- **Init container**: Container khởi tạo, đảm bảo môi trường sẵn sàng trước khi container chính chạy.

## Design pattern
### Co-located Containers

### Regular Init Container

### Sidecar Container



## Init Container
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```