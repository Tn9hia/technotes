<!--
TEMPLATE: setup / system configuration / lab runbook.
Usage: copy this whole file, fill in everything inside <...>, delete every HTML comment.
Section headings and prose are Vietnamese because they land in the output document verbatim.
If a section truly does not apply, write "Không áp dụng - <lý do>" instead of deleting the heading.
-->

# <Tên hệ thống hoặc công nghệ> - <Mục tiêu cụ thể của runbook>

<!--
Title = "what" + "what for". From the title alone the reader must know whether this is the document
they need.
Good : "Ansible GitOps - Recover VDI Machine kèm Persistent Data Disk"
Bad  : "Hướng dẫn Ansible" / "Setup hệ thống"

Then three bullets, in exactly this order. They decide whether the reader keeps reading.
-->

- **Bối cảnh và vấn đề**: <hệ thống hiện tại đang thế nào, cái gì không tự làm được, tại sao phải xử lý thủ công>
- **Cách giải quyết**: <dùng công nghệ/công cụ nào, làm những bước lớn nào để giải quyết vấn đề trên>
- **Kết quả sau khi hoàn thành**: <sau khi chạy xong runbook này thì vận hành thay đổi ra sao, người dùng cuối thấy gì>

> [!NOTE]
> <Context the reader should have before starting: an existing alternative solution, scope limits, or why
> this runbook picks its approach over the more common one. Write the callout body in Vietnamese.
> Delete this callout if there is nothing worth saying.>

## Prerequisites

<!--
What must already exist BEFORE step 1. Do not list things that the Installation section installs.
Every line must be something the reader can check as present or absent.
-->

- **Hạ tầng**: <các hệ thống phải đang chạy sẵn và ở trạng thái healthy>
- **Máy chủ / VM**: <số lượng, CPU/RAM/disk, OS và version cụ thể>
- **Tài khoản và quyền**: <service account cho từng hệ thống, kèm quyền tối thiểu cần thiết>
- **Mạng**: <chiều kết nối nào phải thông, port nào phải mở, có đi qua proxy không>
- **Kiến thức nền**: <những thứ runbook này giả định người đọc đã biết, không giải thích lại>

> [!WARNING]
> <If any step causes downtime, restarts a service, or affects users who are online, warn here and give
> an estimate of the impact window.>

## Thông tin Planning liên quan

<!--
One table holding every environment-dependent value. Purpose: the reader edits this table, does a
find-and-replace, and the runbook works in their environment. The Installation section must use exactly
the values or placeholders declared here.
-->

| Thành phần            | Giá trị                 | Ghi chú                      |
| --------------------- | ----------------------- | ---------------------------- |
| <Tên hệ thống/server> | <IP hoặc FQDN>          | <vai trò trong luồng>        |
| <Service account>     | `<username>`            | <quyền tối thiểu cần cấp>    |
| <Port / firewall>     | `<src> -> <dst>:<port>` | <giao thức, chiều kết nối>   |
| <Version>             | `<phần mềm x.y.z>`      | <lý do phải pin version này> |

## Diagram

<!--
Mandatory. The diagram exists to convey the flow, not to fully document the architecture.
Rules: flowchart TD (top-down, because this gets exported to A4 PDF), ~12 nodes max, number the edges
to show execution order, break label lines with <br/>.
Labels containing ( ) : ; or / must be wrapped in double quotes.
-->

```mermaid
flowchart TD
    Actor[<Người/hệ thống khởi tạo>] -- <hành động> --> A[<Component A><br/><IP>]
    A -- "1. <bước 1>" --> B[<Component B><br/><IP>]
    B -- "2. <bước 2>" --> C[<Component C>]
    C --> Result[<Trạng thái cuối mong muốn>]
```

---

## Installation

<!--
Each "###" is a complete milestone: once done, one component works and can be verified.
Order them by true dependency. If a milestone runs long, split it with "####".
For code/config runbooks (Ansible, Terraform, K8s), make the repo directory layout the first milestone
so the reader has the whole map before reading individual files.
-->

### <Bước 1 - Milestone name, start with a verb: "Cài đặt...", "Cấu hình...", "Khai báo...">

<!-- One or two sentences: what this milestone does and why it must come before the later ones. -->

- <Hành động 1, một việc duy nhất>

```bash
<lệnh chính xác, đã chạy thật>
```

- <Hành động 2>. Chỉnh sửa file cấu hình `<đường/dẫn/tuyệt/đối/file.conf>`:

```ini
<nội dung cần thêm hoặc sửa>
```

<!--
For every non-obvious value, give the reason right there — either as a comment inside the code block
or as a line just below it. This is the most valuable part of the runbook.
-->

> [!WARNING]
> <Explain the dangerous flag or parameter: what happens if it is set wrong. Example: this value stays
> `false` so the disk is only detached and the user's data is never deleted. Write it in Vietnamese.>

- Kiểm tra kết quả bước này:

```bash
<lệnh verify>
```

Kết quả mong đợi: <mô tả output đúng, hoặc dán output thật đã rút gọn>.

### <Bước 2 - ...>

<!-- Repeat the structure above for each milestone. Never write "làm tương tự bước 1" — repeat the full command. -->

### Khai báo thông tin nhạy cảm

<!--
Required for any runbook involving credentials. Never put a real secret in the document.
Describe: where the secret lives, how it is encrypted, how it is issued and rotated.
-->

- Khai báo các giá trị nhạy cảm trong `<đường/dẫn/secret-file>`:

```yaml
<ten_bien_1>: <StrongPassword>
<ten_bien_2>: <Token>
```

- Mã hoá file trước khi commit:

```bash
<lệnh mã hoá>
```

## Kiểm tra kết quả

<!--
End-to-end confirmation that the whole system works — distinct from the per-step verification above.
State clearly: what action triggers the flow, and what evidence proves it ran correctly.
Put screenshots here if there are any.
-->

- <Thao tác kích hoạt luồng end-to-end>. Kết quả mong đợi: <trạng thái đúng>.
- <Kiểm tra từ góc nhìn người dùng cuối>: <họ thấy gì khi mọi thứ chạy đúng>.

| Hạng mục cần kiểm tra | Cách kiểm tra | Kết quả đúng |
| --------------------- | ------------- | ------------ |
| <hạng mục 1>          | `<lệnh>`      | <output>     |

## Troubleshooting

<!--
Only list errors actually hit while doing this work. Do not speculate about errors that might happen.
If nothing went wrong, delete this whole section.
-->

| Triệu chứng       | Nguyên nhân   | Cách xử lý |
| ----------------- | ------------- | ---------- |
| `<thông báo lỗi>` | <nguyên nhân> | <cách fix> |

## Rollback

<!--
How to return to the original state if this fails halfway through.
Any step that cannot be rolled back must be called out, together with which step number it is.
-->

- <Bước hoàn tác 1 - theo thứ tự ngược lại với Installation>

```bash
<lệnh hoàn tác>
```

> [!CAUTION]
> <Irreversible operations, if any. Write the body in Vietnamese.>

## Reference

<!-- Only documentation actually used. Prefer official docs. Note the version when the docs are versioned. -->

- [<Tiêu đề tài liệu>](<url>)
- [<Tiêu đề tài liệu>](<url>)
