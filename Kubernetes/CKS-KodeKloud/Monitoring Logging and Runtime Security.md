---
title: Monitoring, Logging and Runtime Security
tags:
  - kubernetes
  - security
  - cks
  - runtime-security
  - falco
date: 2026-08-18
---

# Monitoring, Logging and Runtime Security

## Immutable Infrastructure

### Mutable vs Immutable

**Mutable infrastructure**: sửa trực tiếp server/resource đang chạy khi cần update. Ví dụ: nâng cấp Nginx từ 1.17 → 1.18 → 1.19 ngay trên server đang chạy (thủ công hoặc bằng script/Ansible). Vấn đề lớn nhất của cách này là **configuration drift**: nếu một server trong pool (ví dụ server 3) thiếu dependency, upgrade thất bại và server đó bị kẹt ở version cũ trong khi các server khác đã lên version mới — dẫn đến một pool server không đồng nhất, khó troubleshoot và khó lên kế hoạch update tiếp theo.

**Immutable infrastructure**: không sửa server đang chạy. Thay vào đó, provision server mới với version mới, rồi decommission server cũ. Một khi đã deploy, server/container không bao giờ bị chỉnh sửa trong vòng đời của nó — mọi thay đổi đều đòi hỏi tạo mới. Cách này giảm thiểu configuration drift vì mọi instance đều xuất phát từ cùng một cấu hình chuẩn.

Với container, mô hình immutable áp dụng tự nhiên: container được sinh ra từ image, nên mọi update (ví dụ Nginx 1.18 → 1.19) phải thực hiện trên **image** trước, sau đó rolling update để không downtime:

```dockerfile
FROM nginx:1.19
COPY nginx.conf /etc/nginx
ENTRYPOINT ["sh", "entrypoint.sh"]
```

Về mặt kỹ thuật vẫn có thể sửa một container đang chạy (copy file thẳng vào filesystem, hoặc exec vào shell rồi sửa tay), nhưng làm vậy tăng rủi ro bảo mật (unauthorized access, thay đổi không kiểm soát) và phá vỡ nguyên tắc immutability — cần ngăn chặn việc này ở runtime.

### Ép buộc Immutability tại Runtime

Container về bản chất được thiết kế là immutable, nhưng vẫn có thể bị sửa in-place (copy file vào pod, `kubectl exec` lấy shell rồi sửa). Có 2 cơ chế chính để ngăn việc này:

**1. Read-only root filesystem** — set trong `securityContext` của pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      readOnlyRootFilesystem: true
```

`readOnlyRootFilesystem: true` khiến root filesystem của container chỉ đọc ngay từ lúc start, ngăn copy/ghi file trái phép. **Cảnh báo quan trọng**: cấu hình này rất dễ làm ứng dụng crash nếu nó cần ghi vào filesystem lúc runtime. Ví dụ Nginx cần ghi vào `/var/run` (runtime data) và `/var/cache/nginx` (cache) — nếu áp readOnlyRootFilesystem mà không xử lý, pod sẽ vào trạng thái `Error`.

**2. Mount `emptyDir` volume cho đúng những thư mục cần ghi** — để giữ toàn bộ filesystem read-only nhưng vẫn cho phép ghi có kiểm soát vào các đường dẫn cụ thể:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: cache-volume
      mountPath: /var/cache/nginx
    - name: runtime-volume
      mountPath: /var/run
  volumes:
  - name: cache-volume
    emptyDir: {}
  - name: runtime-volume
    emptyDir: {}
```

Dùng `emptyDir` vì dữ liệu này không cần persist sau khi pod bị xoá. Sau khi mount đúng các thư mục, pod sẽ chạy `Running` bình thường.

**Read-only root filesystem vẫn có hiệu lực kể cả khi container chạy `privileged: true`** — đây là điểm hay bị hỏi trong exam. Test minh hoạ: dù pod chạy privileged, `kubectl exec -ti nginx -- apt update` vẫn fail với lỗi `Read-only file system` vì apt cần ghi vào `/var/lib/apt/lists/partial`. Tuy vậy, vẫn nên **tránh dùng privileged flag** trừ khi thực sự bắt buộc, vì container privileged có thể thay đổi các giá trị ở `/proc` (ví dụ swappiness) làm ảnh hưởng trực tiếp đến host — bất chấp root filesystem của container là read-only.

**Best practice tổng hợp cho container immutability:**

| Best Practice | Mô tả |
|---|---|
| Read-Only Root File System | Set container với root filesystem read-only để ngăn sửa in-place |
| Limited Write Volumes | Chỉ mount volume (`emptyDir` hoặc PV) vào đúng thư mục cần ghi |
| Avoid Privileged Mode | Tránh dùng privileged flag để giới hạn tác động lên host |
| Non-Root Containers | Chạy container với non-root user khi có thể |
| Enforce Security Policies | Dùng Pod Security Policy (hoặc cơ chế thay thế như Pod Security Admission/OPA Gatekeeper ở phiên bản K8s mới) để enforce các nguyên tắc trên |

Ví dụ PSP minh hoạ (lưu ý: PodSecurityPolicy đã bị **loại bỏ khỏi Kubernetes từ v1.25**, exam CKS hiện dùng Pod Security Admission/Standards thay thế, nhưng ý tưởng enforcement vẫn tương tự):

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: RunAsNonRoot
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
```

## Falco (Runtime Threat Detection)

### Nguyên lý hoạt động

Falco (của Sysdig) giám sát runtime bằng cách bắt các **syscall** từ ứng dụng ở user-space đi vào Linux kernel, rồi đưa qua policy engine đánh giá theo các rule đã định nghĩa để phát hiện hành vi bất thường. Khi phát hiện anomaly, Falco alert qua nhiều kênh: syslog, stdout, Slack, email, v.v.

Falco có 2 cách tương tác với kernel:

1. **Kernel module**: chèn thêm code vào kernel — hiệu quả nhưng intrusive; nhiều managed Kubernetes provider cấm dùng kernel module vì lý do bảo mật.
2. **eBPF (Extended Berkeley Packet Filter)**: cách ít xâm lấn hơn, được nhiều provider ưa chuộng hơn vì ít ảnh hưởng đến tính toàn vẹn hệ thống.

Dù dùng cách nào, syscall sau khi bắt được đi qua user-space syscall library rồi được lọc bởi Falco policy engine dựa trên Falco rules để sinh alert.

**Lưu ý triển khai**: cài Falco trực tiếp lên node dưới dạng service (thay vì chỉ chạy trong cluster) giúp Falco cô lập khỏi chính Kubernetes — nếu cluster/pod bị compromise, Falco (chạy ở tầng host) vẫn tiếp tục hoạt động và phát hiện hành vi bất thường.

### Cài đặt Falco

**Cài trực tiếp trên node** (vì Falco tương tác trực tiếp với kernel nên cần cài kèm kernel module tương ứng):

```bash
# 1. Thêm public key và repo
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | tee -a /etc/apt/sources.list.d/falcosecurity.list

# 2. Cài kernel headers tương ứng + Falco, rồi start service
apt update -y
apt-get install -y linux-headers-$(uname -r)
apt install -y falco
systemctl start falco
```

**Deploy dưới dạng DaemonSet** (nếu không cài trực tiếp lên node được): cách dễ nhất là dùng Helm chart để chạy Falco trên mọi node của cluster.

Kiểm tra Falco đã chạy (nếu deploy dạng DaemonSet):

```bash
kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
falco-7grdt   1/1     Running   0          2m21s
falco-tmq28   1/1     Running   0          2m21s
```

Nếu cài như package trên node, verify bằng:

```bash
systemctl status falco
```

### File cấu hình Falco

File cấu hình chính: `/etc/falco/falco.yaml`. Có thể xác nhận Falco đang dùng file config nào qua unit file của service (flag `-c`) hoặc qua log (`journalctl -fu falco`):

```
Apr 13 21:45:36 node01 falco[9817]: Falco initialized with configuration file /etc/falco/falco.yaml
```

File này định nghĩa: vị trí các rule file, format log/output, và các output channel.

**Load rule file** — qua field `rules_file` (danh sách file, thứ tự quan trọng):

```yaml
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/k8s_audit_rules.yaml
  - /etc/falco/rules.d/
```

- `/etc/falco/falco_rules.yaml` chứa rule/list/macro **built-in**, luôn là file đầu tiên trong danh sách.
- **Nếu cùng một rule xuất hiện ở nhiều file, định nghĩa ở file đứng sau (load sau) sẽ ghi đè (override) định nghĩa trước đó** — đây là điểm hay bị hỏi trong exam.
- Custom rule/chỉnh sửa nên luôn để trong `/etc/falco/falco_rules.local.yaml`, **không sửa trực tiếp** `falco_rules.yaml` vì file này có thể bị ghi đè khi package Falco được update.

Các option cấu hình khác thường gặp:

```yaml
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/k8s_audit_rules.yaml
  - /etc/falco/rules.d
json_output: false
log_stderr: true
log_syslog: true
log_level: info
```

**Output channel** — Falco mặc định log ra stdout, nhưng có thể bật thêm nhiều channel song song:

```yaml
stdout_output:
  enabled: true

file_output:
  enabled: true
  filename: /opt/falco/events.txt

program_output:
  enabled: true
  program: "jq '{text: .output}' | curl -d @- -X POST https://hooks.slack.com/services/XXXX"

http_output:
  enabled: true
  url: http://some.url/some/path/
```

### Cấu trúc một Falco rule

Rule mẫu (built-in) — phát hiện shell được spawn trong container có terminal:

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal (user=%user.name user_loginuid=%user.loginuid %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE
```

Một rule có **5 key bắt buộc**:

- `rule`: tên duy nhất của rule
- `desc`: mô tả mục đích rule
- `condition`: biểu thức filter áp lên event đến (syscall)
- `output`: message log khi rule trigger (hỗ trợ field động qua `%field.name`)
- `priority`: mức độ nghiêm trọng (từ `debug` thấp nhất đến `emergency` cao nhất)

Ví dụ chỉnh priority của rule built-in (từ NOTICE lên WARNING) và thêm custom rule mới (đặt trong `falco_rules.local.yaml`), phát alert CRITICAL khi có file read bất thường trong container webapp:

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal (user=%user.name user_loginuid=%user.loginuid %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: WARNING

- rule: Anomalous read in kodekloud/webapp pod
  desc: Detect suspicious file reads in a custom webapp container.
  condition: >
    open_read and container
    and container.image.repository == "kodekloud/simple-webapp"
    and fd.directory != "/opt/app"
  output: >
    A file was opened and read outside the /opt/app directory (user=%user.name user_loginuid=%user.loginuid
    container_id=%container.id image=%container.image.repository)
  priority: CRITICAL
```

Một ví dụ custom rule khác, dùng **macro** và **list** để rule gọn và dễ maintain hơn — phát hiện shell mở trong container:

```yaml
- rule: Detect Shell inside a container
  desc: Alert if a shell such as bash is open inside a container
  condition: container and proc.name in (linux_shells)
  output: Bash Opened (user=%user.name container=%container.id)
  priority: WARNING

- list: linux_shells
  items: [bash, zsh, ksh, sh, csh]

- macro: container
  condition: container.id != host
```

- `list` (`linux_shells`): gom danh sách giá trị dùng lại được (ở đây là tên các shell) để condition gọn hơn, tránh liệt kê lặp lại ở nhiều rule.
- `macro` (`container`): đặt tên cho một biểu thức condition con (`container.id != host`) để tái sử dụng ở nhiều rule khác nhau, tăng khả năng đọc và maintain.

### Hot reload cấu hình

Không cần restart toàn bộ service để áp dụng thay đổi config/rule — gửi tín hiệu `SIGHUP` (kill -1) tới process Falco. Khi chạy qua systemd, PID được lưu ở `/var/run/falco.pid`:

```bash
cat /var/run/falco.pid
7183
kill -1 $(cat /var/run/falco.pid)
```

Falco sẽ reload config và restart engine mà không cần full service restart.

### Demo thực hành phát hiện threat

1. Verify Falco chạy: `systemctl status falco`.
2. Deploy pod test: `kubectl run nginx --image=nginx`, xác định node nó chạy trên bằng `kubectl get pods -o wide`.
3. SSH vào đúng node đó, stream log Falco real-time: `journalctl -fu falco` (bỏ qua các log cũ, chỉ chú ý log mới sinh ra).
4. Từ terminal khác, exec vào container: `kubectl exec -ti nginx -- bash` → Falco log ngay một alert "shell spawned" kèm container ID, image, namespace.
5. Trong shell đó, chạy `cat /etc/shadow` → Falco log thêm một alert vì đây là truy cập file nhạy cảm (sensitive file access).

Kịch bản minh hoạ hành vi tấn công thực tế: attacker cố xoá dấu vết bằng cách append log giả vào audit log:

```bash
kubectl exec -ti nginx-master -- bash
# cat /etc/shadow > /opt/logs/audit.log
```

Đây là hành vi bất thường (đọc file nhạy cảm rồi ghi đè/append vào audit log) — không phải thao tác của admin hợp lệ thông thường, và cần được rule Falco flag lại như một early warning sign của intrusion.

## Behavioral Analytics / Syscall Monitoring

Dù đã áp dụng đầy đủ các biện pháp hardening khác (bảo mật control plane, sandboxing container, mTLS giữa các service, giới hạn network access tới node...), **không có gì đảm bảo tuyệt đối** chống lại threat mới — attacker luôn có thể tìm ra lỗ hổng chưa từng biết. Vì vậy cần chuẩn bị sẵn khả năng **phát hiện sớm** khi container/node đã bị compromise, thay vì chỉ dựa vào phòng ngừa.

Phát hiện sớm (early detection) giúp giảm thiểu thiệt hại: nhận diện nhanh hành vi bất thường cho phép nhanh chóng cô lập/thay thế node hoặc pod bị compromise và vá lỗ hổng đã bị khai thác, trước khi attacker mở rộng được "blast radius".

**Ý tưởng cốt lõi**: khi có hàng trăm ứng dụng chạy trên hàng nghìn pod, số lượng syscall sinh ra là khổng lồ — giám sát thủ công là bất khả thi. Cần công cụ tự động phân tích luồng syscall và lọc ra các sự kiện đáng ngờ, ví dụ: một tiến trình truy cập bash shell trong container, hoặc một chương trình cố đọc `/etc/shadow` (chứa password hash).

**Các công cụ liên quan đến giám sát syscall**:

- **strace**: công cụ trace syscall ở mức cơ bản, dùng để phân tích hành vi ứng dụng bên trong pod trước khi có các công cụ chuyên biệt hơn.
- **Aqua Tracee**: một công cụ mã nguồn mở khác để phân tích syscall/hành vi runtime.
- **Falco** (của Sysdig): công cụ chính được CKS tập trung — xây dựng policy engine trên nền syscall để phát hiện threat theo thời gian thực (xem chi tiết ở phần Falco phía trên).

Việc này gợi nhớ đến cơ chế bảo mật thẻ tín dụng hiện đại: chip + PIN không ngăn được 100% việc thẻ bị đánh cắp, nhưng **thông báo giao dịch tức thời** (instant notification), **khả năng đảo giao dịch** (revert transaction), và **hạn mức giao dịch** (transaction limit) giúp giảm thiểu thiệt hại khi sự cố xảy ra — behavioral analytics trên syscall đóng vai trò tương tự cho hệ thống container: không ngăn 100% breach, nhưng phát hiện và phản ứng đủ nhanh để giới hạn thiệt hại.

## Gotchas

- `readOnlyRootFilesystem: true` **rất dễ làm app crash** nếu app cần ghi file tạm/cache/runtime data (ví dụ Nginx cần ghi `/var/run`, `/var/cache/nginx`) — luôn phải mount `emptyDir` (hoặc PV) vào đúng các path đó, không mount tràn lan ra toàn bộ filesystem vì sẽ làm mất tác dụng của việc enforce read-only.
- Read-only root filesystem **vẫn có hiệu lực dù pod chạy `privileged: true`** — hai cấu hình này độc lập nhau. Nhưng privileged container vẫn nguy hiểm vì có thể sửa `/proc` của host (ví dụ swappiness), ảnh hưởng trực tiếp đến host dù filesystem container là read-only.
- **PodSecurityPolicy đã bị remove khỏi Kubernetes từ v1.25** — nếu gặp câu hỏi CKS về enforce security policy, cân nhắc Pod Security Admission (PSA)/Pod Security Standards hoặc OPA Gatekeeper/Kyverno thay vì PSP (dù tài liệu KodeKloud gốc minh hoạ bằng PSP).
- **Rule file precedence trong Falco**: nếu cùng một rule (`rule:` name trùng) xuất hiện ở nhiều file trong `rules_file`, **file load sau (đứng sau trong danh sách) sẽ override** file load trước. `falco_rules.yaml` (built-in) luôn được liệt kê đầu tiên.
- **Không bao giờ sửa trực tiếp `/etc/falco/falco_rules.yaml`** — đây là file built-in, sẽ bị ghi đè khi package Falco update. Mọi custom rule/override phải để ở `/etc/falco/falco_rules.local.yaml`.
- Falco rule bắt buộc phải có đủ **5 key**: `rule`, `desc`, `condition`, `output`, `priority`. Thiếu key nào rule sẽ không hợp lệ.
- **macro** và **list** trong Falco rule dùng để tái sử dụng logic/dữ liệu (ví dụ macro `container` kiểm tra `container.id != host`, list `linux_shells` liệt kê tên các shell) — không phải là rule độc lập, chỉ là building block được `condition` của rule khác tham chiếu tới.
- Sau khi sửa config hoặc rule, **không cần restart toàn bộ Falco service** — chỉ cần gửi `SIGHUP` (`kill -1 $(cat /var/run/falco.pid)`) để hot-reload.
- Falco tương tác kernel qua 2 cách: **kernel module** (intrusive, một số managed K8s provider cấm) hoặc **eBPF** (ít xâm lấn hơn, thường được ưu tiên). Nếu đề thi nhắc tới managed cluster không cho phép kernel module, nghĩ ngay tới eBPF driver.
- Cài Falco **trực tiếp trên node (host)** thay vì chỉ chạy như pod trong cluster giúp Falco tiếp tục hoạt động độc lập ngay cả khi cluster/pod bị compromise — đây là lý do KodeKloud nhấn mạnh việc cài Falco như systemd service trên node.
- Domain "Monitoring, Logging and Runtime Security" của CKS **không dừng lại ở Falco** — audit logging của Kubernetes API server (audit policy, audit backend) là một phần riêng thuộc domain Cluster Setup and Hardening, không nằm trong phần Falco/runtime security này.
- Behavioral analytics/syscall monitoring là lớp phòng thủ **bổ sung** (defense in depth), không phải thay thế cho các biện pháp hardening khác (RBAC, network policy, mTLS, sandboxing) — mục tiêu là phát hiện nhanh khi các lớp phòng thủ khác đã bị vượt qua, để giảm blast radius chứ không phải để ngăn chặn tuyệt đối.
