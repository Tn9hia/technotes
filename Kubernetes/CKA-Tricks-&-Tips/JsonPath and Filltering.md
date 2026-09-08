## JsonPath
Sử dụng `jq` để check cấu trúc json
```shell
k get pod -o json | jq .
```

### Công thức "Thần chú" JSONPath

Hầu hết các câu hỏi CKA chỉ xoay quanh cấu trúc sau: `kubectl get <resource> -o jsonpath='{.items[*].metadata.name}'`
- `{ }`: Bao quanh toàn bộ biểu thức.
- `.items[*]`: Duyệt qua tất cả các phần tử trong danh sách (vì khi bạn `get pods`, kết quả trả về là một `List`).
- `.metadata.name`: Đường dẫn đến trường bạn muốn lấy.
### Tuyệt chiêu "Custom Columns" (Dễ nhớ hơn)

Nếu đề bài chỉ yêu cầu xem thông tin (không yêu cầu lưu vào file định dạng cụ thể), `-o custom-columns` thường dễ viết hơn JSONPath: `kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP`

## Cách xử lý Filtering (Bộ lọc)

Phần khó nhất là lọc (ví dụ: lọc theo tên hoặc trạng thái). Hãy nhớ cấu trúc này: `[?(@.path.to.field == "value")]`

**Ví dụ: Lấy tên các Node có nhãn `disk=ssd`**
```shell
kubectl get nodes -o jsonpath='{.items[?(@.metadata.labels.disk=="ssd")].metadata.name}'
```

- `?()`: Ký hiệu của bộ lọc.
- `@`: Đại diện cho phần tử hiện tại đang được xét.

```shell
kubectl get pods -n prod -o jsonpath='{range .items[?(@.status.phase=="Running" && @.metadata.labels.app=="frontend")]}{.metadata.name} {.spec.nodeName}{"\n"}{end}'

kubectl get pods -n prod -o jsonpath='{range .items[?(@.status.phase=="Running" && @.metadata.labels.app=="frontend" && !@.metadata.labels.env)]}{.metadata.name} {.spec.nodeName}{"\n"}{end}'
```

## Duyệt qua các phần tử theo điều kiện

```shell
{range .items[?(@.field=="value")]} {.data1}{"\t"}{.data2}{"\n"} {end}
```
## Các trường phổ biến
| **Mục tiêu**                  | **Đường dẫn JSONPath**                               |
| ----------------------------- | ---------------------------------------------------- |
| **Tên Pod**                   | `.metadata.name`                                     |
| **IP của Pod**                | `.status.podIP`                                      |
| **Trạng thái Pod**            | `.status.phase`                                      |
| **IP của Node**               | `.status.addresses[?(@.type=="InternalIP")].address` |
| **Tên Node mà Pod đang chạy** | `.spec.nodeName`                                     |
| **Image của Container**       | `.spec.containers[*].image`                          |
## Chiến thuật làm bài thi

1. **Sử dụng tài liệu:** Bạn được phép truy cập [kubernetes.io](https://kubernetes.io/docs/reference/kubectl/jsonpath/). Hãy bookmark sẵn trang **JSONPath Support** trước khi thi.
2. **Test trước khi xuất file:** Luôn chạy lệnh để xem kết quả hiện ra màn hình trước. Khi thấy đúng danh sách tên cần thiết rồi mới thêm phần `> /opt/outputs/file.txt`.
3. **Đừng quá sa đà:** Nếu một câu JSONPath quá phức tạp và làm bạn mất hơn 5 phút, hãy `skip` và quay lại sau. Các câu về Troubleshooting hoặc Cluster Administration thường có điểm số cao hơn