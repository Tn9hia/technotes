## Basic Command

### Quản lý Repository

```bash
helm repo add <tên> <url>           # Thêm repo mới
helm repo list                       # Liệt kê các repo
helm repo update                     # Cập nhật thông tin từ các repo
helm repo remove <tên>               # Xóa repo
```

### Tìm kiếm Chart

```bash
helm search repo <keyword>           # Tìm chart trong các repo đã add
helm search hub <keyword>            # Tìm chart trên Artifact Hub
helm show chart <chart>              # Xem thông tin chart
helm show values <chart>             # Xem values mặc định
```

### Cài đặt và Nâng cấp

```bash
helm install <tên-release> <chart>                    # Cài đặt chart
helm install <tên> <chart> -f values.yaml             # Cài với custom values
helm install <tên> <chart> --set key=value            # Set giá trị trực tiếp
helm upgrade <tên-release> <chart>                    # Nâng cấp release
helm upgrade --install <tên> <chart>                  # Cài hoặc upgrade nếu đã tồn tại
```

### Quản lý Release

```bash
helm list                            # Liệt kê các release đang chạy
helm list --all                      # Liệt kê tất cả (kể cả failed)
helm status <tên-release>            # Xem trạng thái release
helm get values <tên-release>        # Xem values đang dùng
helm get manifest <tên-release>      # Xem manifest K8s
helm history <tên-release>           # Xem lịch sử versions
```

### Gỡ bỏ và Rollback

```bash
helm uninstall <tên-release>         # Gỡ release
helm uninstall <tên> --keep-history  # Gỡ nhưng giữ lịch sử
helm rollback <tên-release> <revision>  # Quay về version cũ
```

### Debug và Test

```bash
helm template <tên> <chart>          # Render template local (không deploy)
helm install --dry-run --debug <tên> <chart>  # Test trước khi cài
helm lint <chart-path>               # Kiểm tra lỗi chart
helm test <tên-release>              # Chạy tests của release
```

### Tạo Chart riêng

```bash
helm create <tên-chart>              # Tạo chart mới
helm package <chart-path>            # Đóng gói chart thành .tgz
helm dependency update               # Cập nhật dependencies
```

Một số tips hay dùng:

- Thêm `-n <namespace>` để chỉ định namespace
- Thêm `--version <version>` để cài phiên bản cụ thể
- Dùng `helm get all <release>` để xem toàn bộ thông tin release

### Chart development
```shell
# Create chart mới
helm create mychart

# Package chart thành .tgz
helm package ./mychart

# Install từ local .tgz
helm install my-release ./mychart-0.1.0.tgz

# Dependency management
helm dependency update ./mychart  # pull dependencies từ Chart.yaml
helm dependency list ./mychart
```

---

## Xem thêm

- [[Golang-Helm-Chart]] — đóng gói ứng dụng Golang thành Helm chart cho CI/CD
