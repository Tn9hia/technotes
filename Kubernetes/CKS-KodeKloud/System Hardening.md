---
title: System Hardening
tags:
  - kubernetes
  - security
  - cks
  - system-hardening
date: 2026-08-18
---

# System Hardening

## Nguyên tắc Least Privilege (Principle of Least Privilege)

Nguyên tắc cốt lõi: mỗi role/user/process/service chỉ được cấp đúng quyền cần thiết để thực hiện công việc của nó — không hơn. Áp dụng vào Kubernetes/Linux, các biện pháp cụ thể gồm:

| Biện pháp | Mô tả |
|---|---|
| Limit Node Access | Giới hạn quyền user trên node để tránh sửa đổi trái phép |
| RBAC | Định nghĩa quyền truy cập chính xác cho user/service trong cluster |
| Remove Obsolete Packages | Gỡ bỏ phần mềm không còn cần thiết |
| Restrict Network Access | Giới hạn giao tiếp mạng giữa các component để giảm attack surface |
| Restrict Kernel Modules | Chỉ load các kernel module cần thiết, chặn các module không cần |
| Fix Open Ports | Xác định và đóng các port không cần thiết |

Nguyên tắc này áp dụng cả cho user lẫn system component: chỉ cài phần mềm cần thiết trên host, không để service thừa expose node, không load kernel module không dùng, và luôn rà soát/đóng open port.

## Privilege Escalation, Sudo và Linux Capabilities

### Privilege Escalation qua sudo

Không nên login trực tiếp bằng root (kể cả qua SSH — xem phần SSH Hardening bên dưới); thay vào đó dùng `sudo` để escalate quyền tạm thời, có audit trail và giữ nguyên shell của user thường (không chuyển hẳn sang root shell).

```bash
# Không có sudo -> lỗi permission
apt install nginx
E: Could not open lock file /var/lib/dpkg/lock-frontend - open (13: Permission denied)
E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), are you root?

# Có sudo -> yêu cầu password của chính user đó
sudo apt install nginx
[sudo] password for michael:
```

Quyền sudo được cấu hình trong `/etc/sudoers` (chỉ sửa bằng `visudo`, không sửa trực tiếp bằng editor thường để tránh lỗi cú pháp làm hỏng sudo toàn hệ thống):

```bash
cat /etc/sudoers
# User privilege specification
root    ALL=(ALL:ALL) ALL
# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
# Allow mark to run any command
mark    ALL=(ALL:ALL) ALL
# Allow Sarah to reboot the system
sarah localhost=/usr/bin/shutdown -r now
#include /etc/sudoers.d
```

Cấu trúc mỗi dòng: `<user hoặc %group> <host>=(<run-as user>) <command(s)>`. Có thể giới hạn user chỉ được chạy một lệnh cụ thể (như dòng của `sarah` chỉ được `shutdown -r now`), thay vì cấp `ALL`.

### Linux Capabilities

Trước kernel 2.2, process chỉ có hai loại: **privileged** (chạy bởi root, bỏ qua hầu hết permission check) và **unprivileged**. Từ kernel 2.2 trở đi, quyền "siêu người dùng" được chia nhỏ thành các **capability** riêng lẻ, cho phép cấp đúng từng quyền cụ thể ngay cả cho process chạy dưới root.

Một số capability tiêu biểu:

- `CAP_CHOWN` — đổi ownership của file
- `CAP_NET_ADMIN` — sửa cấu hình network interface, routing table, bind vào địa chỉ cụ thể
- `CAP_SYS_BOOT` — reboot hệ thống
- `CAP_SYS_TIME` — set/sửa system clock

Kiểm tra capability mà một binary yêu cầu bằng `getcap`:

```bash
getcap /usr/bin/ping
/usr/bin/ping = cap_net_raw+ep
```

Kiểm tra capability của một process đang chạy bằng `getpcaps <PID>`:

```bash
ps -ef | grep /usr/sbin/sshd | grep -v grep
# root     779     1  0 03:55 ?  00:00:00 /usr/sbin/sshd -D
getpcaps 779
```

Ngoài `getpcaps`, có thể xem trực tiếp bitmask capability của một process qua `/proc/<pid>/status` — cách này dùng được cả bên trong container (ví dụ PID 1 là process chính của container):

```bash
kubectl exec myapp -- cat /proc/1/status | grep Cap
# CapPrm: 0000000000000400
# CapEff: 0000000000000400
# CapBnd: 0000000000000400
```

Ba dòng `CapPrm`/`CapEff`/`CapBnd` là bitmask hex biểu diễn tập capability (Permitted/Effective/Bounding). Giải mã bitmask này thành tên capability dễ đọc bằng `capsh --decode`:

```bash
capsh --decode=0000000000000400
# 0x0000000000000400=cap_net_bind_service
```

**Container và capability mặc định**: dù chạy container với user root (UID 0), container vẫn bị giới hạn — vì container runtime (Docker) chỉ khởi động container với một tập con capability mặc định (14 capability), không phải toàn bộ quyền root thật. Đây là lý do đổi system date bên trong container thất bại dù đang chạy as root:

```bash
docker run -it --rm --security-opt seccomp=unconfined docker/whalesay /bin/sh
# date -s '19 APR 2012 22:00:00'
date: cannot set date: Operation not permitted
```

Danh sách 14 capability mặc định của Docker (Go source):

```go
func DefaultCapabilities() []string {
    return []string{
        "CAP_CHOWN",
        "CAP_DAC_OVERRIDE",
        "CAP_FOWNER",
        "CAP_MKNOD",
        "CAP_NET_RAW",
        "CAP_SETGID",
        "CAP_SETUID",
        "CAP_SETFCAP",
        "CAP_SETPCAP",
        "CAP_NET_BIND_SERVICE",
        "CAP_SYS_CHROOT",
        "CAP_KILL",
        "CAP_AUDIT_WRITE",
    }
}
```

**Thêm/bớt capability cho pod** qua `securityContext.capabilities`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu
      command: ["sh", "-c", "sleep 1h"]
      securityContext:
        capabilities:
          add: ["SYS_TIME"]     # thêm CAP_SYS_TIME -> cho phép đổi giờ hệ thống
          drop: ["CHOWN"]       # bỏ CAP_CHOWN -> lệnh chown sẽ không hoạt động bên trong container
```

Lưu ý khi khai báo trong YAML, tên capability bỏ tiền tố `CAP_` (ví dụ `SYS_TIME` thay vì `CAP_SYS_TIME`).

## Seccomp — Giới hạn Syscalls

### Syscall là gì

Kernel Linux là lớp trung gian giữa hardware và process; process chạy trong user space không được truy cập hardware trực tiếp mà phải gọi **syscall** để yêu cầu kernel thực hiện (mở file, đọc/ghi, tạo process...). Có hơn 435 syscall trên Linux. Ngay cả một lệnh đơn giản như `touch` cũng gọi hàng chục syscall (`execve`, `open`, `close`, `read`, `mmap`, `brk`...).

Trace syscall của một lệnh bằng `strace`:

```bash
which strace
strace touch /tmp/error.log
# execve("/usr/bin/touch", ["touch", "/tmp/error.log"], 0x7ffce8f874f8 /* 23 vars */) = 0
```

Trace một process đang chạy theo PID:

```bash
pidof etcd
strace -p 3596
# Ctrl+C để detach
```

Xem summary số lần gọi từng syscall bằng cờ `-c`:

```bash
strace -c touch /tmp/error.log
```

Attack surface càng lớn khi càng nhiều syscall được phép — ví dụ **Dirty COW (CVE-2016-5195)** khai thác syscall `ptrace` để ghi vào file read-only, dẫn đến privilege escalation và container escape.

### Cơ chế Seccomp

**Seccomp** (Secure Computing, có từ kernel 2.6.12) cho phép sandbox một ứng dụng bằng cách lọc (filter) syscall nào được phép gọi. Kiểm tra kernel có hỗ trợ Seccomp:

```bash
grep -i seccomp /boot/config-$(uname -r)
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP_FILTER=y
CONFIG_SECCOMP=y
```

Ba mode của Seccomp:

- **Mode 0 — Disabled**: không lọc gì cả.
- **Mode 1 — Strict**: chỉ cho phép đúng 4 syscall: `read`, `write`, `exit`, `sigreturn`.
- **Mode 2 — Filter**: cho phép một tập syscall tuỳ chỉnh theo profile (dùng bởi Docker mặc định).

Có thể xem field `Seccomp` trong `/proc/<pid>/status` (giá trị `2` = filtered).

Docker tự động áp default Seccomp profile (JSON, whitelist ~60 syscall, chặn khoảng 60 syscall nguy hiểm như liên quan tới đổi giờ hệ thống, mount filesystem, load kernel module — kể cả `ptrace` bị chặn để phòng Dirty COW):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    {
      "names": ["arch_prctl", "brk", "capget", "capset", "mkdir", "close", "execve", "...", "clone"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

**Whitelist profile** (an toàn hơn — chỉ cho phép syscall liệt kê, mặc định deny hết):

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    { "names": ["<syscall-1>", "<syscall-2>", "<syscall-3>"], "action": "SCMP_ACT_ALLOW" }
  ]
}
```

**Blacklist profile** (dễ làm hơn nhưng kém an toàn hơn — mặc định allow hết, chỉ deny syscall liệt kê; dễ bỏ sót syscall nguy hiểm):

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86", "SCMP_ARCH_X32"],
  "syscalls": [
    { "names": ["<syscall-1>", "<syscall-2>", "<syscall-3>"], "action": "SCMP_ACT_ERRNO" }
  ]
}
```

Chạy container với custom profile qua Docker:

```bash
docker run -it --rm --security-opt seccomp=/root/custom.json docker/whalesay /bin/sh
/ # mkdir test
mkdir: can't create directory 'test': Operation not permitted
```

Tắt hẳn Seccomp (không khuyến khích):

```bash
docker run -it --rm --security-opt seccomp=unconfined docker/whalesay /bin/sh
```

### Seccomp trong Kubernetes

Khác với Docker, **Kubernetes mặc định KHÔNG bật Seccomp** cho pod (Seccomp: disabled) — chỉ có khoảng 21 syscall bị chặn bởi các cơ chế khác (so với 64 syscall bị chặn nếu dùng default Docker Seccomp profile). Kiểm tra bằng tool `amicontained`:

```bash
docker run r.j3ss.co/amicontained amicontained          # chạy trực tiếp bằng Docker -> Seccomp: filtering, 64 syscalls blocked
kubectl run amicontained --image=r.j3ss.co/amicontained amicontained -- amicontained
kubectl logs amicontained                                 # -> Seccomp: disabled, chỉ 21 syscalls blocked
```

**Bật Seccomp = RuntimeDefault** (dùng lại default profile của container runtime):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amicontained
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: amicontained
      image: r.j3ss.co/amicontained
      args: ["amicontained"]
      securityContext:
        allowPrivilegeEscalation: false
```

`allowPrivilegeEscalation: false` ngăn process con giành thêm quyền vượt quá quyền của process cha (nên luôn set cùng với Seccomp/AppArmor để hardening đầy đủ).

**Unconfined** (không giới hạn gì — tương đương hành vi mặc định của Kubernetes nếu không set field này):

```yaml
spec:
  securityContext:
    seccompProfile:
      type: Unconfined
```

**Custom profile từ file trên node** (`type: Localhost`) — path là **path tương đối** so với thư mục Seccomp profile mặc định của kubelet, `/var/lib/kubelet/seccomp/`. Ví dụ file đặt tại `/var/lib/kubelet/seccomp/profiles/audit.json` thì khai báo `localhostProfile: profiles/audit.json`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-audit
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["bash", "-c", "echo 'I just made some syscalls' && sleep 100"]
      securityContext:
        allowPrivilegeEscalation: false
```

Profile chỉ log syscall (không chặn) để phân tích ứng dụng cần syscall nào, hữu ích khi xây whitelist:

```json
{ "defaultAction": "SCMP_ACT_LOG" }
```

Syscall được log vào `/var/log/syslog` trên node:

```bash
grep syscall /var/log/syslog
```

Profile deny tuyệt đối mọi syscall (kể cả syscall thiết yếu) sẽ khiến pod không chạy được:

```json
{ "defaultAction": "SCMP_ACT_ERRNO" }
```

```bash
kubectl apply -f test-violation.yaml
kubectl get pods
# NAME             READY   STATUS               RESTARTS   AGE
# test-violation   0/1     ContainerCannotRun   0          2m2s
```

Quy trình thực tế để xây custom profile: phân tích syscall ứng dụng cần dùng (qua audit log `SCMP_ACT_LOG` hoặc tool như Tracee), rồi viết whitelist chỉ chứa các syscall đó, đặt file vào `/var/lib/kubelet/seccomp/profiles/`, reference bằng `localhostProfile: profiles/<file>.json`.

### AquaSec Tracee — theo dõi syscall bằng eBPF

**Tracee** (Aqua Security) là công cụ mã nguồn mở dùng **eBPF** để trace syscall của container tại runtime mà không cần sửa kernel hay load thêm module — chạy chương trình trực tiếp trong kernel space, overhead thấp.

Yêu cầu khi chạy Tracee dưới dạng Docker container:
- Bind-mount `/tmp/tracee` (nơi Tracee lưu eBPF program đã compile, để persist giữa các lần chạy).
- Bind-mount `/lib/modules` và `/usr/src` (đọc-only) — chứa kernel headers cần để compile eBPF program.
- Chạy với `--privileged` vì Tracee cần quyền mở rộng để trace syscall.

Trace syscall của một lệnh cụ thể (ví dụ `ls`):

```bash
docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace comm=ls
```

Trace tất cả process mới trên host:

```bash
sudo docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace pid=new
```

Trace syscall của container mới khởi tạo (hữu ích để xây whitelist Seccomp cho một image):

```bash
sudo docker run --name tracee --rm --privileged --pid=host \
  -v /lib/modules/:/lib/modules:ro \
  -v /usr/src:/usr/src:ro \
  -v /tmp/tracee:/tmp/tracee \
  aquasec/tracee:0.4.0 --trace container=new
```

### Security Profiles Operator — quản lý Seccomp profile theo kiểu Kubernetes-native

Viết/deploy từng file JSON thủ công lên `/var/lib/kubelet/seccomp/` của mỗi node không scale tốt khi cluster có nhiều node và nhiều profile — phải tự đảm bảo file có mặt đúng trên đúng node. **Security Profiles Operator** (dự án `kubernetes-sigs/security-profiles-operator`, tên cũ là seccomp-operator) giải quyết vấn đề này bằng cách quản lý Seccomp (và SELinux) profile như **Custom Resource** chuẩn của Kubernetes — operator tự động phân phối file profile lên đúng node và dọn dẹp khi không còn dùng.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/security-profiles-operator/deploy/operator.yaml
```

Định nghĩa profile bằng CRD `SeccompProfile` thay vì tự tay quản lý file JSON trên từng node:

```yaml
apiVersion: security-profiles-operator.x-k8s.io/v1beta1
kind: SeccompProfile
metadata:
  name: web-app-profile
  namespace: production
spec:
  defaultAction: SCMP_ACT_ERRNO
  syscalls:
    - action: SCMP_ACT_ALLOW
      names:
        - accept4
        - bind
        - listen
        - read
        - write
```

CRD `ProfileBinding` tự động gắn một `SeccompProfile` vào mọi pod dùng một image cụ thể — không cần sửa từng pod spec để trỏ `localhostProfile` thủ công:

```yaml
apiVersion: security-profiles-operator.x-k8s.io/v1alpha1
kind: ProfileBinding
metadata:
  name: bind-web-profile
  namespace: production
spec:
  profileRef:
    kind: SeccompProfile
    name: web-app-profile
  image: myapp:*    # áp dụng cho mọi pod dùng image này
```

Lợi ích so với quản lý file thủ công: operator tự sync profile lên đúng node có pod matching, tự cập nhật khi CRD đổi, và cho phép audit/versioning profile qua `kubectl` như mọi resource Kubernetes khác.

## AppArmor

### Vì sao cần AppArmor bên cạnh Seccomp

Seccomp chỉ giới hạn **syscall nào** được gọi, không kiểm soát **tài nguyên nào** (file/directory cụ thể, network, capability...) được truy cập. Ví dụ, Seccomp có thể chặn hẳn syscall `mkdir` — nhưng nếu muốn cho phép tạo thư mục ở một số nơi và cấm ở nơi khác thì Seccomp không làm được; đây là lúc dùng **AppArmor** — một Linux Security Module (LSM) kiểm soát truy cập tài nguyên ở mức resource (file, path, network, capability), cài đặt và bật sẵn mặc định trên hầu hết distro.

So sánh nhanh hai cơ chế:

| | Seccomp | AppArmor |
|---|---|---|
| **Kiểm soát** | Syscall | File, capability, network |
| **Độ chi tiết (granularity)** | Mức syscall | Mức path/capability |
| **Độ phức tạp profile** | JSON | Cú pháp profile riêng của AppArmor |
| **Mặc định trong K8s** | `RuntimeDefault` | `docker-default` (nếu container runtime có sẵn) |

### Kiểm tra AppArmor trên node

```bash
systemctl status apparmor                          # AppArmor service có chạy không
cat /sys/module/apparmor/parameters/enabled         # phải trả về "Y"
cat /sys/kernel/security/apparmor/profiles          # liệt kê toàn bộ profile đã load
```

`aa-status` cho cái nhìn tổng quan: profile nào loaded, đang ở mode nào (enforce/complain/unconfined), process nào đang bị confine bởi profile nào:

```bash
aa-status
apparmor module is loaded.
12 profiles are loaded.
12 profiles are in enforce mode.
    /sbin/dhclient
    ...
    docker-default
0 profiles are in complain mode.
11 processes have profiles defined.
```

### Ba mode của AppArmor profile

- **Enforce**: rule được áp dụng chặt chẽ, hành vi vi phạm bị chặn.
- **Complain**: hành vi vi phạm rule vẫn được cho phép chạy, nhưng bị log lại dưới dạng warning (dùng để "tập luyện"/thu thập log trước khi chuyển sang enforce).
- **Unconfined**: không áp rule nào, không log.

### Viết profile AppArmor thủ công

Profile là file text thuần định nghĩa các resource mà app được phép truy cập. Ví dụ deny toàn bộ ghi (write) lên filesystem:

```
profile apparmor-deny-write flags=(attach_disconnected) {
    file,
    # Deny all file writes.
    deny /** w,
}
```

Ví dụ chặn remount root filesystem thành read-only:

```
profile apparmor-deny-remount-root flags=(attach_disconnected) {
  # Deny remounting the root filesystem as read-only.
  deny mount options=(ro, remount) -> /,
}
```

AppArmor profile cũng có thể deny trực tiếp từng **capability** cụ thể (bổ sung cho việc kiểm soát capability qua `securityContext.capabilities` — đây là kiểm soát ở tầng LSM, độc lập với cơ chế capability của kernel):

```
profile apparmor-deny-dangerous-caps flags=(attach_disconnected) {
  #include <abstractions/base>

  # Cho phép bind port đặc quyền nhưng chặn các capability nguy hiểm khác
  capability net_bind_service,
  deny capability sys_admin,
  deny capability sys_ptrace,
}
```

### Sinh profile tự động bằng `aa-genprof`

Thay vì viết tay, dùng bộ công cụ `apparmor-utils` để tự sinh profile bằng cách quan sát hành vi thực tế của ứng dụng:

```bash
apt-get install -y apparmor-utils
aa-genprof /root/add_data.sh
```

Quy trình: `aa-genprof` đặt app vào **complain mode**, sau đó bạn chạy lại app ở terminal khác để tạo ra các sự kiện AppArmor (file access, exec...), rồi quay lại `aa-genprof`, nhấn `s` (Scan) để scan log hệ thống. Với mỗi sự kiện, chọn hành động:

- `(I)nherit` — process con kế thừa profile của process cha.
- `(C)hild` — tạo sub-profile riêng cho process con.
- `(P)rofile` — tạo profile riêng độc lập.
- `(A)llow` — cho phép truy cập.
- `(D)eny` — từ chối truy cập (chọn khi resource không thực sự cần thiết, ví dụ đọc `/proc/filesystems`).
- `(F)inish` — kết thúc, sau đó `(S)ave` để lưu và chuyển profile sang **enforce mode**.

Profile sinh ra được lưu tại `/etc/apparmor.d/`, ví dụ:

```
# Last Modified: Mon Mar 22 11:21:42 2021
#include <tunables/global>

/root/add_data.sh {
    #include <abstractions/base>
    #include <abstractions/bash>
    #include <abstractions/consoles>

    deny owner /proc/filesystems r,
    /root/add_data.sh r,
    /usr/bin/bash ix,
    /usr/bin/date mrix,
    /usr/bin/mkdir mrix,
    /usr/bin/tee mrix,
    owner /opt/app/ rw,
    owner /opt/app/data/ w,
    owner /opt/app/data/create.log w,
}
```

Xác nhận enforce mode bằng `aa-status`, và test bằng cách cho app ghi ra ngoài path đã cấp quyền — sẽ nhận `Permission denied`.

Để **load** một profile đã viết sẵn, dùng AppArmor parser (không trả output nghĩa là load thành công). Để **disable** một profile, chạy parser với cờ `-r` và tạo symlink tới profile trong thư mục `/etc/apparmor.d/disable/`.

Ví dụ load/reload một profile kèm ghi cache để lần load sau nhanh hơn:

```bash
apparmor_parser -r -W /etc/apparmor.d/k8s-myapp
# -r : reload nếu profile đã tồn tại (thay vì báo lỗi "profile already exists")
# -W : ghi cache (write-cache) sau khi parse, giúp lần load sau nhanh hơn
aa-status | grep k8s-myapp   # xác nhận profile đã loaded
```

### Áp dụng AppArmor profile cho Pod trong Kubernetes

AppArmor được hỗ trợ từ Kubernetes v1.4 (beta tới v1.20). Yêu cầu trên **mỗi node** pod có thể chạy:

- AppArmor kernel module đã enable.
- Profile mong muốn đã được load sẵn trên node đó (kiểm tra bằng `aa-status`).
- Container runtime (Docker/CRI-O/containerd) hỗ trợ AppArmor.

Pod ví dụ (Ubuntu sleeper, không cần ghi file nên áp profile deny-write):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
    - name: hello
      image: ubuntu
      command: ["sh", "-c", "echo 'Sleeping for an hour!' && sleep 1h"]
```

**Cách cũ (annotation, thời AppArmor còn beta)** — annotation key có dạng `container.apparmor.security.beta.kubernetes.io/<container-name>`, value dạng `localhost/<profile-name>`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  annotations:
    container.apparmor.security.beta.kubernetes.io/ubuntu-sleeper: localhost/apparmor-deny-write
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu
      command: ["sh", "-c", "echo 'Sleeping for an hour!' && sleep 1h"]
```

**Cách mới (`securityContext.appArmorProfile`)**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: apparmor-deny-write
  containers:
    - name: ubuntu-sleeper
      image: ubuntu
      command: ["sh", "-c", "echo 'Sleeping for an hour!' && sleep 1h"]
```

Test rule hoạt động:

```bash
kubectl create -f ubuntu-sleeper.yaml
kubectl logs ubuntu-sleeper
kubectl exec -ti ubuntu-sleeper -- touch /tmp/test
# touch: cannot touch '/tmp/test': Permission denied
# command terminated with exit code 1
```

## User Namespace (Kubernetes 1.30+)

### Vấn đề: root trong container vẫn có thể là root trên host

Ngay cả khi đã giới hạn capability và Seccomp/AppArmor, nếu container chạy với `runAsUser: 0` thì UID 0 bên trong container **map trực tiếp** tới UID 0 (root) trên host theo mặc định — nếu container escape xảy ra (ví dụ qua lỗ hổng kernel), attacker có ngay quyền root thật trên node. **User namespace** giải quyết việc này bằng cách remap UID/GID: container vẫn thấy mình chạy UID 0 (root) bên trong, nhưng UID đó thực chất được ánh xạ sang một UID không có đặc quyền (thường là UID cao) trên host.

### Bật User Namespace cho Pod

Tính năng ở trạng thái beta từ Kubernetes 1.30, cần feature gate `UserNamespacesSupport=true` trên cluster và container runtime hỗ trợ (containerd ≥ 2.0, CRI-O). Bật bằng field `hostUsers: false` trong pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: userns-demo
spec:
  hostUsers: false      # false = pod chạy trong user namespace riêng (remap UID/GID)
  containers:
    - name: app
      image: myapp:v1
      securityContext:
        runAsUser: 0     # vẫn là "root" bên trong container...
```

Khi `hostUsers: false`, root (UID 0) bên trong container không còn tương đương root trên host — container escape (nếu xảy ra) chỉ cho attacker quyền của một UID không có đặc quyền trên node, thu hẹp đáng kể mức độ nghiêm trọng của lỗ hổng. `hostUsers: true` (hoặc bỏ trống field này) giữ hành vi mặc định trước đây — pod dùng chung host user namespace, UID 0 trong container = UID 0 thật trên host.

### Verify: UID bên trong container khác UID trên host

```bash
# Bên trong container — vẫn thấy mình là root
kubectl exec userns-demo -- id
# uid=0(root) gid=0(root) groups=0(root)

# Trên host node — process thực chạy dưới một UID cao, không có đặc quyền
ps aux | grep myapp
# user      65536  ...   myapp
```

## Host/OS Hardening

### Xác định và đóng open port

Khi một process start, nó bind vào một port để nhận traffic. Càng nhiều port mở không cần thiết, attack surface càng lớn. Dùng `netstat` để liệt kê port đang LISTEN:

```bash
netstat -an | grep -w LISTEN
```

Ví dụ output trên control-plane node cho thấy các port quen thuộc của Kubernetes: `2379`/`2380` (etcd), `6443` (kube-apiserver), `10250` (kubelet), `10257`/`10259` (controller-manager/scheduler), `22` (SSH), `53` (DNS)...

Tra cứu mục đích của một port qua `/etc/services`:

```bash
cat /etc/services | grep ssh
# ssh    22/tcp    # SSH Remote Login Protocol
```

Quy trình: liệt kê port đang mở → xác định port nào thực sự cần thiết cho workload → đóng/chặn phần còn lại (qua service, firewall, hoặc gỡ package tương ứng). Trước khi cài phần mềm mới, luôn kiểm tra tài liệu chính thức xem nó cần mở port nào (ví dụ tài liệu kubeadm liệt kê rõ các port Kubernetes cần).

### Hai cách tiếp cận giới hạn truy cập mạng

1. **Network-wide security**: firewall/appliance chuyên dụng ở tầng mạng (Cisco ASA, Juniper, Fortinet...) kiểm soát traffic toàn mạng.
2. **Server-level security**: host-based firewall trên từng máy — `iptables`, `firewalld`, hoặc `UFW` trên Linux; firewall built-in trên Windows Server.

### UFW (Uncomplicated Firewall)

UFW là front-end đơn giản hoá cho `iptables`/Netfilter (bộ lọc gói tin trong kernel Linux).

Kiểm tra port đang listen trước khi cấu hình:

```bash
netstat -an | grep -w LISTEN
# 22/tcp open, 80/tcp open, 8080/tcp open (muốn chặn 8080)
```

Cài đặt và kiểm tra trạng thái:

```bash
apt-get update
apt-get install ufw
ufw status
# Status: inactive
```

Thiết lập default policy (allow toàn bộ outbound, deny toàn bộ inbound):

```bash
ufw default allow outgoing
ufw default deny incoming
```

Thêm rule allow cụ thể theo IP/port/protocol (ví dụ: chỉ cho jump server `172.16.238.5` SSH vào; port 80 mở cho jump server và mạng nội bộ `172.16.100.0/28`):

```bash
ufw allow from 172.16.238.5 to any port 22 proto tcp
ufw allow from 172.16.238.5 to any port 80 proto tcp
ufw allow from 172.16.100.0/28 to any port 80 proto tcp
```

Deny tường minh một port cụ thể (dù default policy đã deny, deny tường minh giúp rõ ràng ý định — ví dụ port 8080 đang listen nhưng không được phép truy cập từ ngoài):

```bash
ufw deny 8080
```

Bật firewall (cảnh báo có thể làm rớt kết nối SSH hiện tại nếu rule sai — luôn giữ một session đang mở để không bị khoá ngoài):

```bash
ufw enable
# Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
ufw status
```

Xoá rule theo nội dung hoặc theo số thứ tự (line number) trong `ufw status`:

```bash
ufw delete deny 8080
# hoặc
ufw delete <line-number>
```

### SSH Hardening

Kết nối SSH cơ bản (nếu không chỉ định user, dùng username local hiện tại):

```bash
ssh node01
# hoặc
ssh user@node01
```

**Key-based authentication** thay vì password:

```bash
ssh-keygen -t rsa
# tạo cặp khoá tại ~/.ssh/id_rsa (private) và ~/.ssh/id_rsa.pub (public)
# có thể đặt passphrase để tăng bảo mật cho private key

ssh-copy-id mark@node01
# copy public key vào ~/.ssh/authorized_keys trên server đích
```

**Hardening `/etc/ssh/sshd_config`** (sửa xong nhớ `systemctl restart sshd`, và giữ session hiện tại mở để test trước khi đóng, tránh tự khoá mình ngoài server):

```bash
vi /etc/ssh/sshd_config
```

```
PermitRootLogin no          # tắt đăng nhập root trực tiếp qua SSH -> bắt buộc dùng user thường + sudo
PasswordAuthentication no   # tắt đăng nhập bằng password -> chỉ cho phép key-based auth
```

```bash
systemctl restart sshd
```

### Gỡ bỏ package/service không cần thiết

Hệ thống thường tích luỹ phần mềm cài sẵn từ image/snapshot template mà không thực sự cần — mỗi package/service thừa là một phần attack surface không cần thiết (ví dụ Apache vô tình có mặt trên node Kubernetes).

Liệt kê toàn bộ service đang chạy qua `systemd`:

```bash
systemctl list-units --type service
```

Kiểm tra một service cụ thể:

```bash
systemctl status apache2
```

Dừng, disable, rồi gỡ package không cần:

```bash
systemctl stop apache2
systemctl disable apache2
apt remove apache2
```

Trước khi purge, luôn xác nhận package không phải dependency của service khác đang cần thiết. Tham khảo thêm CIS Benchmarks (section 2, "Distribution Independent Linux") cho best practice quản lý service.

## Restrict Kernel Modules

Kernel Linux có kiến trúc modular — module có thể được load động (thủ công qua `modprobe`/`insmod`, hoặc tự động bởi kernel khi cần, ví dụ khi có hardware mới hoặc khi một **process không có privilege** tạo network socket khiến kernel tự load module giao thức mạng tương ứng). Đây chính là điểm attacker có thể lợi dụng để kích hoạt module không mong muốn.

Load thủ công một module (ví dụ PC speaker):

```bash
modprobe pcspkr
```

Liệt kê toàn bộ module đang active:

```bash
lsmod
```

### Blacklist module để chặn tự động load

Ví dụ: blacklist module `sctp` (ít khi cần trong Kubernetes cluster) và `dccp` bằng cách thêm file cấu hình trong `/etc/modprobe.d/` (tên file bất kỳ, đuôi `.conf`):

```bash
cat /etc/modprobe.d/blacklist.conf
blacklist sctp
blacklist dccp
```

Sau khi sửa file, **phải reboot** để thay đổi có hiệu lực, rồi xác nhận module không còn active:

```bash
shutdown -r now
lsmod | grep dccp
```

Tham khảo thêm CIS Benchmarks for Kubernetes, section 3.4, về kernel module security.

## Minimize IAM Roles (Cloud Provider)

Áp dụng nguyên tắc least privilege lên IAM (Identity and Access Management) của cloud provider (AWS/GCP/Azure) — root account (tài khoản tạo khi đăng ký) có toàn quyền, nên chỉ dùng root để tạo IAM user rồi cất kỹ, không dùng cho tác vụ hàng ngày.

- Tạo **IAM user** riêng cho từng người, cấp policy đúng nhu cầu tối thiểu (ví dụ chỉ cấp quyền tạo EC2 + truy cập S3 cho developer thay vì full admin).
- Gom user có nhu cầu quyền giống nhau vào **IAM group** (ví dụ "Developer Group"), gắn policy vào group thay vì từng user riêng lẻ để dễ quản lý.
- Với **AWS resource** (không phải user) cần quyền — ví dụ EC2 instance cần truy cập S3 — không thể gắn policy trực tiếp vào resource; phải tạo **IAM role** (ví dụ "S3 Access Role") rồi gán role đó cho EC2 instance. Tránh nhúng access key/secret key trực tiếp vào ứng dụng — kém an toàn hơn IAM role.
- Định kỳ **audit** lại IAM policy, gỡ quyền không dùng tới. Các cloud có sẵn tool hỗ trợ: AWS Trusted Advisor, GCP Security Command Center, Azure Advisor.

IAM chi tiết không phải trọng tâm chính của đề thi CKS nhưng nắm khái niệm least privilege ở tầng cloud vẫn hữu ích.

## Gotchas

- **Seccomp không kiểm soát filesystem/network/capability** — nó chỉ lọc syscall. Muốn giới hạn truy cập file/thư mục cụ thể thì cần AppArmor (hoặc SELinux), không phải Seccomp.
- **Kubernetes KHÔNG bật Seccomp mặc định** (`Seccomp: disabled`), khác với Docker (tự bật default profile filtering ~64 syscall). Phải set `securityContext.seccompProfile.type: RuntimeDefault` (hoặc `Localhost`) tường minh trong pod spec để bật.
- `localhostProfile` trong `seccompProfile` là **đường dẫn tương đối** so với `/var/lib/kubelet/seccomp/` trên node (ví dụ `profiles/audit.json` → file thật nằm ở `/var/lib/kubelet/seccomp/profiles/audit.json`). Đây là bẫy hay gặp trong bài thi — dễ nhầm là path tuyệt đối.
- Whitelist profile (`defaultAction: SCMP_ACT_ALLOW` cho từng syscall cụ thể + default deny) an toàn hơn blacklist (`defaultAction: SCMP_ACT_ALLOW` toàn cục + deny một số syscall) vì blacklist dễ bỏ sót syscall nguy hiểm chưa biết.
- Profile Seccomp deny toàn bộ syscall (`defaultAction: SCMP_ACT_ERRNO` không kèm allow list) khiến container ở trạng thái `ContainerCannotRun` — vì ngay cả syscall khởi động container cũng bị chặn.
- Luôn set `allowPrivilegeEscalation: false` cùng với Seccomp/AppArmor để ngăn container giành thêm quyền so với process cha (một field hay bị quên trong bài lab/thi).
- **AppArmor profile phải được load sẵn trên đúng node** nơi pod sẽ schedule tới, trước khi tạo pod — Kubernetes không tự phân phối AppArmor profile lên node giùm bạn. Nếu profile chưa load, pod sẽ lỗi hoặc không enforce đúng như kỳ vọng.
- Cú pháp annotation AppArmor cũ (`container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/<profile>`) vẫn có thể gặp trong đề thi phiên bản cũ hơn — phân biệt với cách mới dùng `securityContext.appArmorProfile.type/localhostProfile`.
- Giá trị `type: Localhost` cần đi kèm `localhostProfile: <profile-name>`; `type: RuntimeDefault` dùng default profile của container runtime; `type: Unconfined` tắt hẳn AppArmor cho container đó.
- Ba mode AppArmor dễ nhầm: **complain** vẫn cho hành vi vi phạm chạy (chỉ log), không phải "chặn nhẹ" — chỉ **enforce** mới thực sự chặn.
- Container dù chạy **as root (UID 0)** vẫn bị giới hạn bởi (a) tập capability mặc định hạn chế của container runtime (chỉ 14/nhiều chục capability) và (b) Seccomp/AppArmor nếu có — "root trong container" không đồng nghĩa "root trên host".
- Khi khai báo capability trong `securityContext.capabilities.add`/`drop`, bỏ tiền tố `CAP_` (ví dụ `SYS_TIME`, không phải `CAP_SYS_TIME`).
- Sau khi sửa `/etc/modprobe.d/*.conf` để blacklist kernel module, **phải reboot** thì thay đổi mới có hiệu lực — quên bước này là lỗi thường gặp.
- Sau khi sửa `PermitRootLogin no` / `PasswordAuthentication no` trong `sshd_config`, luôn giữ một session SSH đang mở để test trước khi đóng — nếu key-based login lỗi mà đã mất session cũ, có thể bị khoá ngoài server hoàn toàn.
- Luôn sửa `/etc/sudoers` bằng `visudo`, không sửa trực tiếp bằng editor thường — `visudo` kiểm tra cú pháp trước khi lưu, tránh làm hỏng sudo cho toàn hệ thống.
- UFW: default policy áp dụng trước, rule allow/deny cụ thể chỉ bổ sung/làm rõ — deny tường minh một port vẫn hữu ích dù default đã là deny, để tài liệu hoá rõ ràng ý định và tránh nhầm lẫn khi audit rule sau này.
- `netstat -an | grep LISTEN` là câu lệnh nền tảng lặp lại xuyên suốt domain System Hardening (open ports, UFW, general host hardening) — thuộc nằm lòng cho bài thi.
- Custom Seccomp profile phải khai báo đúng `architectures` khớp với kiến trúc CPU của node — profile viết cho `SCMP_ARCH_X86_64` sẽ không hoạt động trên node ARM (Raspberry Pi, AWS Graviton...), cần dùng `SCMP_ARCH_AARCH64` tương ứng.
- Seccomp có hai action hay bị nhầm khi block syscall: `SCMP_ACT_KILL` giết process ngay khi gọi syscall bị chặn (mạnh tay, dùng cho production sau khi đã test kỹ), còn `SCMP_ACT_ERRNO` chỉ trả về mã lỗi (ví dụ `EPERM`) cho process tự xử lý — nên dùng `ERRNO` trong giai đoạn debug/xây whitelist trước khi chuyển sang `KILL`.
- Sửa nội dung AppArmor profile **không tự động áp dụng** cho container đang chạy — chỉ có hiệu lực khi container được tạo lại/restart; cần rolling restart workload sau khi cập nhật profile.
