## Naming Convention

## 🧩 **1. Biến (variables)**

- **camelCase**
    
- Ngắn nhưng rõ ý, đừng essay.
    
- Biến export dùng **CapitalCase**.
    

**Ví dụ:**

```go
count := 0
userName := "nghia"
maxRetries := 3

// Exported
TotalUsers := 12

```

---

## 🧩 **2. Hàm (functions)**

- Tương tự biến: **camelCase**
    
- Export/Public: **CapitalCase**
    
- Tên hàm nên nói thẳng chức năng, Go không thích tham-số-dài-lòng-thòng.
    

**Ví dụ:**

```go
func sendEmail() {}
func calculateTotal() {}

// Export
func GetUser() {}
func NewServer() {}

```

---

## 🧩 **3. Struct**

- Tên struct là **danh từ**
    
- Export: CapitalCase
    
- Không nhét quá nhiều từ; Go thích cấu trúc chặt và minimalism.
    
```go
type User struct {
    Name string
    Age  int
}

type httpClient struct {  // unexported
    timeout int
}

```


---

## 🧩 **4. Interface**

- Go style: đặt tên theo hành vi, kết thúc bằng **-er** nếu hợp lý.
    

**Ví dụ:**

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Formatter interface {
    Format(v any) string
}

```

- Interface nhỏ, càng ít method càng tốt.
    

---

## 🧩 **5. Constants**

- **MixedCaps** hoặc **ALL_CAPS** đều được, nhưng chuẩn Go là **CamelCase**.
    
- Dùng `iota` cho enum-style.
    
```go
const DefaultTimeout = 30
const (
    StatusOK = iota
    StatusFailed
    StatusUnknown
)

```


---

## 🧩 **6. Package**

- **toàn chữ thường**
    
- Không dấu gạch nối, không camelCase, không viết tắt lung tung (trừ từ quen như `io`, `fmt`).
    

**Ví dụ chuẩn:**
```go
http
json
collector
router
database

```


**Ví dụ không nên:**

```go
HttpUtils        // ❌
JsonHandler      // ❌
my-super-package // ❌

```

---

## 🧩 **7. Receiver trong method**

- Cực ngắn, thường là **1 chữ**, lấy từ type.
    

```go
func (u *User) Save() {}
func (c *Client) Do() {}
func (s *Server) Start() {}

```

Đừng dùng `this` hoặc `self`, Go không thích drama vậy.

---

## 🧩 **8. Tên viết tắt**

Go có vibe rất đặc trưng: **viết tắt giữ nguyên chữ thường/hoa theo đúng position**.

**Ví dụ chuẩn:**

```go
userID        // not userId
APIClient     // not ApiClient
HTTPRequest   // not HttpRequest

```

---

## 🧩 **9. Error**

Tên error phải mô tả _điều gì đã sai_.  
Format thường dùng: `ErrSomething`.

```go
var ErrNotFound = errors.New("not found")
var ErrInvalidInput = errors.New("invalid input")

```

---

## 🧩 **10. File name**

- snake_case
    
- toàn chữ thường
    
- không dấu gạch ngang
    

```go
user_handler.go
http_client.go
config_loader.go

```

| Loại            | Convention                        |
| --------------- | --------------------------------- |
| package         | lowercase, no underscore          |
| variable        | camelCase                         |
| function        | camelCase (exported: CapitalCase) |
| struct          | CapitalCase                       |
| interface       | tên hành vi + "er" nếu hợp lý     |
| constant        | MixedCaps                         |
| file            | snake_case                        |
| method receiver | 1 ký tự                           |
