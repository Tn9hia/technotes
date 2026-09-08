# Mermaid — Ghi chú tra cứu nhanh

## Giới thiệu

Mermaid là công cụ Diagram-as-Code dùng cú pháp giống Markdown để vẽ diagram, render bằng JavaScript ngay trên trình duyệt. Được tích hợp sẵn (native) trong GitHub, GitLab, Notion, Obsidian, VS Code, Confluence... nên không cần cài đặt hay tool riêng để xem kết quả.

### Ưu điểm
- Tích hợp sẵn trong rất nhiều nền tảng phổ biến (GitHub markdown, Notion, Obsidian) → viết xong là thấy ngay, không cần build/export.
- Cú pháp đơn giản, dễ học, gần giống ngôn ngữ tự nhiên.
- Hỗ trợ nhiều loại diagram: flowchart, sequence, class, state, ER, Gantt, pie, mindmap, gitgraph...
- Cộng đồng lớn, tài liệu nhiều, có [Mermaid Live Editor](https://mermaid.live) để preview trực tiếp.

### Nhược điểm
- Layout engine tự động, khó tùy chỉnh vị trí/thẩm mỹ chi tiết như D2 hay Excalidraw.
- Diagram phức tạp (nhiều node) dễ bị rối, layout tự động không tối ưu.
- Styling (màu sắc, theme) phải viết thêm cú pháp riêng, không trực quan bằng CSS.
- Không mạnh cho vẽ kiến trúc hệ thống/network phức tạp (icon, container lồng nhau) so với các tool chuyên biệt (D2, draw.io).

---

## 1. Flowchart (Flow chart / Process)

### Khai báo & hướng
```
flowchart TD    %% hoặc graph TD
```
- `TD` / `TB`: Top → Down/Bottom
- `BT`: Bottom → Top
- `LR`: Left → Right
- `RL`: Right → Left

### Node shapes (hình dạng ô)
| Cú pháp | Hình dạng |
|---|---|
| `A[Text]` | Hình chữ nhật (process) |
| `A(Text)` | Hình bo tròn (rounded) |
| `A([Text])` | Stadium / viên thuốc |
| `A[[Text]]` | Subroutine (2 viền dọc) |
| `A[(Text)]` | Database / cylinder |
| `A((Text))` | Hình tròn (circle) |
| `A>Text]` | Asymmetric (flag) |
| `A{Text}` | Hình thoi (decision/rhombus) |
| `A{{Text}}` | Hexagon |
| `A[/Text/]` | Parallelogram (hình bình hành) |
| `A[\Text\]` | Parallelogram ngược |
| `A[/Text\]` | Trapezoid |
| `A[\Text/]` | Trapezoid ngược |

### Đường nối (edges/arrows)
| Cú pháp | Ý nghĩa |
|---|---|
| `A --> B` | Mũi tên liền |
| `A --- B` | Đường liền, không mũi tên |
| `A -.-> B` | Mũi tên nét đứt |
| `A -.- B` | Đường nét đứt, không mũi tên |
| `A ==> B` | Mũi tên đậm (thick) |
| `A === B` | Đường đậm, không mũi tên |
| `A --> B\|Label\|` hoặc `A -->\|Label\| B` | Mũi tên có nhãn |
| `A -- Label --> B` | Cách khác để ghi nhãn |
| `A --o B` | Kết thúc bằng vòng tròn |
| `A --x B` | Kết thúc bằng dấu x |
| `A <--> B` | Mũi tên 2 chiều |
| `A ~~~ B` | Đường vô hình (invisible link, dùng để căn layout) |

### Subgraph (nhóm/khối con)
```
flowchart TD
  subgraph Group1 [Tên nhóm]
    A --> B
  end
  Group1 --> C
```

### Style & class
```
style A fill:#f9f,stroke:#333,stroke-width:2px
classDef important fill:#f96,stroke:#333;
class A,B important
```

### Ví dụ đầy đủ
```
flowchart LR
  Start([Bắt đầu]) --> Input[/Nhập dữ liệu/]
  Input --> Check{Hợp lệ?}
  Check -- Có --> Process[Xử lý]
  Check -- Không --> Error[[Báo lỗi]]
  Process --> DB[(Lưu DB)]
  DB --> End([Kết thúc])
```

---

## 2. Sequence Diagram

### Cú pháp cơ bản
```
sequenceDiagram
  participant A as Client
  participant B as Server
```

### Message types (mũi tên)
| Cú pháp | Ý nghĩa |
|---|---|
| `A->>B: msg` | Gọi đồng bộ (solid + mũi tên đặc) |
| `A-->>B: msg` | Phản hồi (dashed + mũi tên đặc) |
| `A->B: msg` | Đường liền, mũi tên mở (open arrow) |
| `A-->B: msg` | Đường đứt, mũi tên mở |
| `A-)B: msg` | Async message (mũi tên mở, không chờ) |
| `A--)B: msg` | Async response |
| `A-xB: msg` | Message bị mất/lỗi (kết thúc bằng x) |

### Activation (kích hoạt lifeline)
```
A->>B: request
activate B
B-->>A: response
deactivate B
%% hoặc viết tắt:
A->>+B: request
B-->>-A: response
```

### Notes
```
Note left of A: ghi chú bên trái
Note right of B: ghi chú bên phải
Note over A,B: ghi chú bao trùm 2 actor
```

### Control flow
```
loop Mỗi 5 giây
  A->>B: ping
end

alt Thành công
  B-->>A: OK
else Thất bại
  B-->>A: Error
end

opt Điều kiện phụ
  A->>B: optional call
end

par Song song
  A->>B: task1
and
  A->>C: task2
end
```

### Autonumber
```
sequenceDiagram
  autonumber
  A->>B: bước 1
  B->>A: bước 2
```

---

## 3. Class Diagram (mô hình OOP)

```
classDiagram
  class Animal {
    +String name
    +int age
    +makeSound() void
  }
  class Dog
  Animal <|-- Dog : kế thừa (inheritance)
```

### Quan hệ (relationship arrows)
| Cú pháp | Ý nghĩa |
|---|---|
| `<\|--` | Inheritance (kế thừa) |
| `*--` | Composition (sở hữu chặt, con chết theo cha) |
| `o--` | Aggregation (sở hữu lỏng) |
| `-->` | Association |
| `--` | Link (solid, không hướng) |
| `..>` | Dependency (phụ thuộc) |
| `..\|>` | Realization (implement interface) |
| `.. ` | Dashed link |

### Visibility (phạm vi truy cập)
`+` public, `-` private, `#` protected, `~` package

---

## 4. State Diagram

```
stateDiagram-v2
  [*] --> Idle
  Idle --> Running : start
  Running --> Idle : stop
  Running --> [*] : done
```
- `[*]` là điểm bắt đầu/kết thúc (start/end state).
- Composite state: 
```
state Running {
  [*] --> SubA
  SubA --> SubB
}
```
- Fork/Join (song song):
```
state fork_state <<fork>>
state join_state <<join>>
```

---

## 5. ER Diagram (Entity Relationship)

```
erDiagram
  CUSTOMER ||--o{ ORDER : places
  ORDER ||--|{ LINE_ITEM : contains
```

### Ký hiệu Crow's foot (số lượng quan hệ)
| Ký hiệu | Ý nghĩa |
|---|---|
| `\|o` | Zero or one |
| `\|\|` | Exactly one |
| `}o` | Zero or many |
| `}\|` | One or many |

Ghép 2 bên bằng `--` (không định danh) hoặc `..` (định danh yếu). Ví dụ đọc `||--o{`: bên trái "exactly one", bên phải "zero or many".

---

## 6. Gantt Chart

```
gantt
  title Kế hoạch dự án
  dateFormat YYYY-MM-DD
  section Giai đoạn 1
    Task 1 :a1, 2026-01-01, 30d
    Task 2 :after a1, 20d
  section Giai đoạn 2
    Task 3 :crit, 2026-02-15, 10d
```
- `crit`: đánh dấu task quan trọng (critical, tô đỏ)
- `done` / `active`: trạng thái task
- `after a1`: bắt đầu sau khi task `a1` xong

---

## 7. Các loại khác (tham khảo nhanh)

- **Pie chart**: `pie title Tên biểu đồ` rồi `"Label" : 45`
- **Mindmap**: `mindmap` + thụt lề để tạo cây phân cấp
- **Git graph**: `gitGraph` với `commit`, `branch`, `checkout`, `merge`
- **C4 Diagram** (System context/container/component): dùng `C4Context`, `C4Container`... với `Person()`, `System()`, `Rel()`

---

## Lưu ý về System Architecture

Mermaid **không có loại diagram chuyên dụng** cho System Architecture/Infrastructure (không có icon cloud, DB, load balancer sẵn). Cách phổ biến:
- Dùng **flowchart** + subgraph để nhóm theo layer (Frontend, Backend, Database, Infra) và shape `[(...)]` cho DB, `[[...]]` cho service.
- Hoặc dùng cú pháp thử nghiệm `architecture-beta` (Mermaid bản mới) có icon `service`, `group`, `edge` — cần kiểm tra version hỗ trợ trước khi dùng.

```
flowchart TB
  subgraph Client
    Browser
  end
  subgraph Backend
    API[API Gateway]
    Auth[Auth Service]
  end
  subgraph Data
    DB[(PostgreSQL)]
    Cache[(Redis)]
  end
  Browser --> API
  API --> Auth
  API --> DB
  API --> Cache
```

---

## Tool hỗ trợ
- Live editor: https://mermaid.live
- VS Code extension: "Markdown Preview Mermaid Support" hoặc "Mermaid Chart"
- CLI: `mmdc` (mermaid-cli) để export PNG/SVG/PDF
