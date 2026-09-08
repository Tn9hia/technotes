```
mycli/
├── cmd/                 # Nơi chứa toàn bộ command & subcommand
│   ├── root.go          # Root command, khởi tạo viper, global flags
│   ├── serve.go         # Subcommand: `mycli serve`
│   ├── user.go          # Subcommand: `mycli user`
│   └── user_list.go     # Subcommand: `mycli user list`
│
├── internal/            # Logic bên trong (không export)
│   ├── server/
│   │   └── server.go    # Logic chạy server
│   └── user/
│       └── user.go      # Logic thao tác với user
│
├── pkg/                 # Public packages có thể tái sử dụng (optional)
│   └── util/
│       └── logger.go
│
├── config/              # File cấu hình (yaml, json…)
│   └── config.yaml
│
├── scripts/             # Script build, release, CI
│   └── build.sh
│
├── main.go              # Entry point → cmd.RootCmd.Execute()
├── go.mod
└── README.md

```
## Giải thích nhanh – theo kiểu “vào là hiểu liền”

### `cmd/` — nơi Cobra sống
- Mỗi file trong đây là **một command/subcommand**.
- `root.go` là “ông nội”, mọi thứ gắn vào nó.
- Tách file theo chức năng → nhìn vô là biết CLI support cái gì.
### `internal/` — business logic thật sự
- Còn `cmd/` chỉ gọi hàm thôi.
- Chỗ này mới là “não”, command chỉ là UI dòng lệnh.
- Go sẽ **không cho project khác import vùng internal** → bảo vệ code.
### `pkg/` — library công khai (tùy chọn)

- Nếu Nghĩa viết mấy hàm tái dùng: logger, utils, formatter,…
- Project khác _có thể_ import `pkg`.
### `config/` — config của Viper

- Viper tự đọc file ở đây.
- Tách riêng để sau build, dev/ops thay config nhanh.
### `scripts/` — automation
- build, release, generate docs, format,…
### `main.go`

- Rất mỏng, chỉ:

```go
func main() {
    cmd.Execute()
}

```
