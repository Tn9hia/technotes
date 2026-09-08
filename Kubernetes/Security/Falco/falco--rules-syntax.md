# Falco — Rules Syntax & Override
Tier: 2
Parent: [[Falco]]
Related: [[falco--drivers]], [[falco--falcoctl-plugins]]
Tags: #falco #rules #detection-engineering #cks

## What it does

Rule engine của Falco định nghĩa **cái gì bị coi là đáng ngờ** và **alert như thế nào**. Một rule gồm 3 phần bắt buộc: `condition` (biểu thức filter kiểu Sysdig để match event), `output` (template string hiển thị alert, có field `%proc.name`, `%container.id`...), `priority` (mức độ nghiêm trọng theo thang syslog). Ngoài rule còn có `macro` (đặt tên cho 1 đoạn condition để tái sử dụng) và `list` (mảng giá trị tái sử dụng, vd danh sách shell binary).

## Why it exists

Nếu không có rule engine có cấu trúc (macro/list/override), mỗi lần cần detect thêm hành vi mới sẽ phải viết lại toàn bộ condition dài dòng, và mỗi lần Falco upgrade rule mặc định thì custom change của mày sẽ **bị ghi đè mất** nếu sửa trực tiếp vào file gốc. Cơ chế override (load nhiều file rule theo thứ tự, rule trùng tên ở file sau override file trước) tồn tại chính là để tách biệt "rule của vendor" và "rule của mày" — upgrade an toàn mà không mất customization.

## How it works (flow/diagram)

```yaml
# Macro: định nghĩa 1 điều kiện tái sử dụng
- macro: spawned_process
  condition: evt.type = execve and evt.dir=<

# List: mảng giá trị tái dùng
- list: shell_binaries
  items: [bash, sh, zsh, csh, ksh]

# Rule: dùng macro + list trong condition
- rule: Terminal shell in container
  desc: A shell was spawned inside a container
  condition: >
    spawned_process and container.id != host
    and proc.name in (shell_binaries)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name image=%container.image.repository
    proc=%proc.cmdline parent=%proc.pname)
  priority: WARNING
  tags: [container, shell]
```

**Load order**: `falco.yaml` có `rules_file:` là 1 **danh sách file**, load tuần tự. Rule/macro/list trùng `name` ở file load **sau** sẽ ghi đè (theo cơ chế `override:`) hoặc thay thế hoàn toàn (nếu không dùng `override:`, default behavior là **replace toàn bộ** object cùng tên).

**Override syntax hiện tại (khuyến nghị, thay cho `append: true` đã deprecated từ 0.36):**

```yaml
- rule: Terminal shell in container
  override:
    condition: append   # nối thêm vào condition gốc bằng "and"
  condition: and not k8s.ns.name in (dev, staging)

- list: shell_binaries
  override:
    items: append        # thêm phần tử vào list gốc thay vì thay hết
  items: [nushell]
```

Key override được hỗ trợ:
- **List**: `items` (append hoặc replace)
- **Macro**: `condition` (append hoặc replace)
- **Rule**: `condition`, `output`, `desc`, `tags`, `exceptions` (append hoặc replace); `priority`, `enabled`, `warn_evttypes`, `skip-if-unknown-filter` (chỉ replace)

**`exceptions:`** — cách whitelist gọn hơn `override condition append + and not`, dùng khi muốn loại trừ theo field cụ thể mà không viết lại condition dài dòng:

```yaml
- rule: Terminal shell in container
  exceptions:
    - name: known_debug_pods
      fields: [k8s.pod.name]
      values:
        - [debug-toolbox]
```

## Config gotchas

- **`append: true` (cú pháp cũ) đã deprecated từ Falco 0.36, sẽ bị xoá ở 1.0.0** — viết rule mới nên dùng `override:` section ngay từ đầu để khỏi phải migrate sau.
- **Không sửa trực tiếp `falco_rules.yaml`** (rule mặc định do falcoctl quản lý) — mọi custom nên nằm ở file riêng load sau, dùng `override`/`exceptions`.
- **Rule engine compile TẤT CẢ rule thành filter tree lúc start** — 1 rule YAML sai syntax (indent sai, thiếu field bắt buộc) làm **Falco crash ngay khi khởi động**, không chạy nửa vời với rule lỗi bị skip. Luôn `falco --validate <file>` trước khi apply.
- **Priority không lọc alert tự động ở rule engine** — mọi rule match đều generate alert, `priority` chỉ là field để **output/filter phía sau** (log level threshold, hoặc filter ở SIEM). Set `priority: informational` cho rule test rồi quên không dọn = noise vĩnh viễn.
- **Macro/list không tự "cache" giữa các file** — nếu macro dùng field từ plugin (K8s audit, cloudtrail) mà quên enable plugin tương ứng, rule sẽ lỗi `skip-if-unknown-filter` hoặc crash tùy config.
- Rule mặc định dựa khá nhiều vào **path/binary name** (`/bin/bash`, `/usr/bin/curl`) — dễ bị evade bằng cách đổi tên binary hoặc dùng static binary tự build. Rule chất lượng cao nên bổ sung điều kiện theo **behavior** (proc.pname bất thường, syscall sequence lạ) chứ không chỉ tên file.

## Security notes

- Rule file chính là **policy bảo mật runtime** — bất kỳ ai có quyền sửa ConfigMap chứa rule (trên K8s) đều có thể **âm thầm tắt detection** cho 1 loại hành vi cụ thể mà không ai biết cho đến khi bị khai thác. RBAC cho ConfigMap Falco phải chặt tương đương RBAC cho Secret.
- `exceptions:`/`override condition append` dùng để whitelist theo namespace/pod — cẩn thận whitelist quá rộng (vd toàn bộ namespace `kube-system`) sẽ tạo ra **blind spot cố định** mà attacker có thể lợi dụng nếu biết được whitelist đó.
- Nên version control rule file (Git) riêng, review qua PR như review code — rule là detection logic, sai 1 dòng có thể làm mất khả năng phát hiện tấn công thật.

## Refs

- https://falco.org/docs/concepts/rules/overriding/
- https://falco.org/docs/reference/rules/supported-fields/ (danh sách field dùng trong condition/output)
- https://github.com/falcosecurity/rules (rule mặc định của falcosecurity)
