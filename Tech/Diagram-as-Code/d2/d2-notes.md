# D2 — Ghi chú tra cứu nhanh

## Giới thiệu

D2 (declarative diagramming) là ngôn ngữ và CLI mã nguồn mở của Terrastruct, chuyên để vẽ diagram kỹ thuật (system architecture, infra, flow) từ text. Compile ra SVG/PNG/PDF, layout engine mạnh (Dagre, ELK, hoặc TALA trả phí) giúp diagram phức tạp vẫn gọn gàng.

### Ưu điểm
- Layout engine tốt hơn Mermaid rõ rệt cho diagram nhiều node/nested container (đặc biệt engine TALA) → phù hợp vẽ system architecture, network diagram phức tạp.
- Cú pháp ngắn gọn, hỗ trợ **icon** (AWS, GCP, Azure, Kubernetes...) và **shape** đa dạng có sẵn, rất hợp để vẽ kiến trúc hạ tầng.
- Hỗ trợ nested containers (khối lồng khối) trực quan, tốt cho vẽ network/VPC/service boundary.
- Có nhiều theme màu đẹp sẵn, dễ style bằng class tái sử dụng.
- Không phụ thuộc nền tảng cụ thể — chạy CLI/watch mode để tự render lại khi lưu file.

### Nhược điểm
- Không tích hợp sẵn (native render) trong GitHub/Notion như Mermaid — cần cài CLI hoặc dùng playground online (https://play.d2lang.com) để xem.
- Cộng đồng và tài liệu nhỏ hơn Mermaid, ít support có sẵn trong IDE/tool bên thứ ba.
- Không có sequence/class/state/ER diagram chuyên biệt mạnh như Mermaid — D2 thiên về architecture/flow hơn là UML đầy đủ (dù vẫn hỗ trợ sequence diagram cơ bản).
- Engine layout đẹp nhất (TALA) là tính năng trả phí; bản free dùng Dagre/ELK layout khả năng thẩm mỹ kém hơn.

---

## 1. Cú pháp cơ bản

### Shape / Node
```
Tên node đơn giản nhất chính là khai báo:
server
```
```
server: "Web Server"          # đổi label hiển thị
server.shape: rectangle       # định hình dạng
```

### Shape types (hình dạng)
| `shape:` | Hình dạng |
|---|---|
| `rectangle` (default) | Hình chữ nhật |
| `square` | Hình vuông |
| `circle` | Hình tròn |
| `oval` | Hình elip |
| `diamond` | Hình thoi (decision) |
| `cylinder` | Database (hình trụ) |
| `queue` | Hàng đợi |
| `package` | Gói/thư mục |
| `step` | Bước (mũi tên lõm) |
| `callout` | Chú thích bong bóng |
| `stored_data` | Dữ liệu lưu trữ |
| `person` | Hình người (actor) |
| `cloud` | Đám mây (thường dùng cho external/internet) |
| `hexagon` | Lục giác |
| `page` | Trang tài liệu |
| `parallelogram` | Hình bình hành |
| `document` | Tài liệu (đáy sóng) |
| `class` | UML class box |
| `sql_table` | Bảng SQL |
| `image` | Chèn ảnh (`icon: url`) |
| `sequence_diagram` | Container cho sequence diagram |

### Connection / Arrows (đường nối)
| Cú pháp | Ý nghĩa |
|---|---|
| `A -> B` | Mũi tên 1 chiều |
| `A <- B` | Mũi tên chiều ngược |
| `A <-> B` | Mũi tên 2 chiều |
| `A -- B` | Đường liền, không mũi tên |
| `A -> B: label` | Có nhãn trên đường nối |
| `A -> B -> C` | Nối chuỗi nhiều node |

### Style cho connection
```
A -> B: {
  style.stroke-dash: 3       # nét đứt
  style.stroke: red
  style.stroke-width: 2
  style.animated: true       # hiệu ứng chạy (chỉ SVG)
}
```

### Style cho shape
```
server: {
  shape: cylinder
  style: {
    fill: "#f5f5f5"
    stroke: "#333"
    stroke-width: 2
    stroke-dash: 4
    shadow: true
    3d: true
    multiple: true          # vẽ chồng nhiều bản (đại diện cluster)
    border-radius: 8
    font-color: blue
    bold: true
    italic: true
    opacity: 0.8
  }
}
```

---

## 2. Container / Nested (khối lồng khối) — mạnh nhất cho System Architecture

```
aws: {
  shape: cloud
  vpc: {
    label: "VPC"
    subnet_public: {
      web: "Web Server"
      lb: "Load Balancer"
    }
    subnet_private: {
      db: {
        shape: cylinder
        label: "PostgreSQL"
      }
    }
  }
}
aws.vpc.subnet_public.lb -> aws.vpc.subnet_public.web
aws.vpc.subnet_public.web -> aws.vpc.subnet_private.db
```
- Truy cập node lồng nhau bằng dấu chấm: `parent.child.grandchild`
- Container tự động render thành khung bo tròn bao quanh các node con.

---

## 3. Icon (đặc biệt mạnh cho Cloud/Infra Architecture)

D2 hỗ trợ icon built-in cho AWS/GCP/Azure/K8s qua thư viện icon chính thức:
```
server: {
  icon: https://icons.terrastruct.com/aws/Compute/AWS-EC2.svg
  shape: image
}
```
- Icon repo tham khảo: https://icons.terrastruct.com
- Có thể dùng icon cho bất kỳ shape nào (icon hiện góc trên bên trái shape, hoặc `shape: image` để icon chiếm toàn bộ).

---

## 4. Sequence Diagram trong D2

```
shape: sequence_diagram

Client: Client
Server: Server
DB: Database

Client -> Server: "Request data"
Server -> DB: "Query"
DB -> Server: "Result"
Server -> Client: "Response"
```
- Đặt `shape: sequence_diagram` ở đầu file/scope để bật chế độ sequence.
- Hỗ trợ nhóm bằng `group { ... }` để đóng khung nhiều bước (giống `alt`/`loop` của Mermaid).
- Activation/lifeline tự động vẽ khi có qua lại giữa 2 actor.

---

## 5. Class Diagram (UML)

```
Animal: {
  shape: class
  +name: string
  +age: int
  #makeSound(): void
}
Dog: {
  shape: class
}
Dog -> Animal: {
  style.stroke-dash: 3
}
```
- Ký hiệu visibility: `+` public, `-` private, `#` protected

---

## 6. SQL Table (ER-style)

```
users: {
  shape: sql_table
  id: int {constraint: primary_key}
  name: string
  email: string
}
orders: {
  shape: sql_table
  id: int {constraint: primary_key}
  user_id: int {constraint: foreign_key}
}
orders.user_id -> users.id
```

---

## 7. Markdown & Text trong node

```
explanation: |md
  # Tiêu đề
  Đây là **markdown** trong node.
|
```
```
note: |text
  Văn bản thuần, giữ nguyên định dạng.
|
```

---

## 8. Positioning & Layout hints

```
direction: right    # down (default), up, left, right — hướng tổng thể của diagram
```
```
a -> b {
  # ưu tiên đặt trong container để kiểm soát vị trí tương đối
}
```
- Layout engine: mặc định `dagre`. Đổi bằng flag CLI: `d2 --layout=elk` hoặc `--layout=tala` (bản trả phí, layout đẹp nhất cho diagram phức tạp).

---

## 9. Variables & Class (tái sử dụng style)

```
vars: {
  primary-color: "#4A90D9"
}

classes: {
  service: {
    style.fill: ${primary-color}
    style.stroke: black
    shape: rectangle
  }
}

api: {
  class: service
}
worker: {
  class: service
}
```

---

## 10. Comment & Import

```
# đây là comment
```
```
...@shared-styles.d2    # import/spread nội dung file khác vào scope hiện tại
```

---

## Ví dụ tổng hợp: System Architecture

```
direction: right

user: {
  shape: person
}

internet: {
  shape: cloud
}

infra: {
  label: "AWS"
  lb: {
    shape: hexagon
    label: "Load Balancer"
  }
  services: {
    api: "API Service"
    auth: "Auth Service"
  }
  data: {
    db: {
      shape: cylinder
      label: "PostgreSQL"
    }
    cache: {
      shape: cylinder
      label: "Redis"
    }
  }
}

user -> internet -> infra.lb
infra.lb -> infra.services.api
infra.lb -> infra.services.auth
infra.services.api -> infra.data.db
infra.services.api -> infra.data.cache
```

---

## Tool hỗ trợ
- Playground online: https://play.d2lang.com
- CLI cài đặt: `brew install d2` (macOS) hoặc script cài từ https://d2lang.com/tour/install
- Watch mode (tự render khi lưu file): `d2 --watch input.d2 output.svg`
- VS Code extension: "D2" (syntax highlight + preview)
- Export: `d2 input.d2 output.svg|png|pdf`
