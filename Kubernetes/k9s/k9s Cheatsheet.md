## k9s là gì, dùng sao cho đỡ bỡ ngỡ

Từ Lens (GUI) chuyển qua k9s (TUI) thì khác biệt lớn nhất là **không có chuột** — mọi thứ đi bằng bàn phím. Tư duy chung:

1. Gõ `:<resource>` để **nhảy view** (giống việc click vào mục bên sidebar của Lens)
2. Dùng mũi tên / `j k` để **di chuyển** trong list (giống scroll)
3. Gõ 1 ký tự để **hành động** trên item đang chọn (`d` describe, `l` logs, `e` edit...) — giống menu chuột phải của Lens
4. `Esc` để **lùi lại** 1 bước, `:q` hoặc `Ctrl-c` để **thoát** hẳn

> [!note]
> Mặc định alias `kubectl` → `k`. Trong k9s bạn không cần gõ `kubectl` gì cả, mọi thứ là hotkey.

---

## Cài đặt & khởi động nhanh

```shell
brew install k9s          # macOS
k9s                        # mở với context/namespace hiện tại trong kubeconfig

k9s --context my-context   # mở thẳng 1 context
k9s -n kube-system          # mở thẳng 1 namespace
k9s --readonly              # mở chế độ chỉ xem (an toàn, không sợ bấm nhầm xoá)
```

---

## Điều hướng cơ bản (Navigation)

| Phím | Chức năng |
|---|---|
| `:<resource>` | Nhảy đến view resource, VD `:po`, `:deploy`, `:svc`, `:ns` |
| `:<resource> <tên>` | Nhảy thẳng và filter theo tên |
| `/` | Filter theo tên (fuzzy search) trong view hiện tại |
| `Esc` | Thoát khỏi filter / lùi về view trước |
| `Ctrl-a` | Hiện toàn bộ alias/resource types (lúc quên gõ gì) |
| `j` / `k` hoặc `↓` `↑` | Di chuyển lên xuống |
| `g` / `G` | Nhảy lên đầu / xuống cuối list |
| `Ctrl-d` / `Ctrl-u` | Cuộn xuống / lên nửa trang |
| `Enter` | Vào sâu hơn (VD: từ Deployment → ReplicaSet → Pod) |
| `Esc` nhiều lần | Quay dần lại các view cha |
| `?` | Xem help, danh sách hotkey đầy đủ của view hiện tại |
| `:q` hoặc `Ctrl-c` | Thoát k9s |

---

## Alias resource hay dùng (gõ sau `:`)

| Alias | Resource |
|---|---|
| `po` | Pods |
| `deploy` | Deployments |
| `rs` | ReplicaSets |
| `sts` | StatefulSets |
| `ds` | DaemonSets |
| `svc` | Services |
| `ing` | Ingress |
| `cm` | ConfigMaps |
| `secrets` | Secrets |
| `ns` | Namespaces |
| `no` | Nodes |
| `pv` / `pvc` | PersistentVolume / Claim |
| `sa` | ServiceAccounts |
| `events` | Events (rất hữu ích để debug) |
| `ctx` | Contexts (đổi cluster) |
| `cj` / `jobs` | CronJobs / Jobs |
| `aliases` hoặc `:alias` | Xem full danh sách alias có sẵn |

---

## Hotkey hành động trên resource (chung, áp dụng ở nhiều view)

| Phím | Hành động |
|---|---|
| `d` | Describe (giống `kubectl describe`) |
| `y` | Xem YAML |
| `e` | Edit YAML trực tiếp (mở editor, mặc định `$EDITOR`) |
| `l` | Logs (vào view log của pod/container) |
| `s` | Shell vào container (giống `kubectl exec -it`) |
| `Ctrl-d` | Delete resource đang chọn (có confirm) |
| `Ctrl-k` | Kill (xoá không cần confirm — cẩn thận!) |
| `p` | Xem Logs trước đó (previous container logs, hữu ích khi pod crash loop) |
| `Shift-f` | Port-forward |
| `Ctrl-r` | Refresh view |

> [!tip]
> `Ctrl-d` xoá có hỏi xác nhận, `Ctrl-k` xoá ngay không hỏi — nhớ phân biệt kẻo xoá nhầm prod 😅

---

## Riêng trong view Pods

| Phím | Hành động |
|---|---|
| `l` | Logs (nếu nhiều container sẽ hỏi chọn container) |
| `s` | Shell vào container |
| `Shift-r` | Toggle hiển thị theo node/container resource |
| `c` | Vào view Containers của pod |
| `Ctrl-k` | Kill pod |
| `Shift-f` | Port-forward pod |
| `o` | Xem Pod trên Node nào (sort theo node) |

---

## Riêng trong view Logs

| Phím | Hành động |
|---|---|
| `Ctrl-s` | Toggle wrap text |
| `Ctrl-f` / `Ctrl-b` | Scroll trang trước/sau |
| `e` | Toggle hiện timestamps |
| `s` | Toggle auto-scroll (tail -f) |
| `/` | Search trong log (giống `grep`) |
| `c` | Clear log buffer đang xem |
| `0`-`9` | Chọn container khi pod có nhiều container |
| `Ctrl-c` hoặc `Esc` | Thoát log view |

---

## Sort & Filter (thay cho việc gõ `kubectl get ... | grep`)

| Phím / Cú pháp | Chức năng |
|---|---|
| `Shift-o` | Sort theo cột đang hover (bấm lại để đảo thứ tự) |
| `/<text>` | Filter theo tên (fuzzy) |
| `/-l app=web` | Filter theo **label selector** |
| `/-i` | Filter **inverse** (loại trừ) |
| `/-f`  hoặc filter theo fuzzy field khác | Filter theo field cụ thể (namespace, status...) |

> [!tip]
> Gõ `/` rồi `app=foo` không ăn label đâu, phải thêm cờ `-l` kiểu `/-l app=foo` mới filter theo label thật sự.

---

## Namespace & Context (thay vì phải nhớ `kubectl config use-context`)

| Phím | Chức năng |
|---|---|
| `:ns` | Mở list namespace, Enter để switch |
| `0` (trong view ns) | Chọn `All namespaces` |
| `:ctx` | Mở list context (cluster), Enter để switch |
| `Shift-i` | Toggle theo dõi tất cả namespace cho resource hiện tại |

---

## Những view "chẩn đoán cluster" hay dùng khi debug

| Lệnh `:` | Dùng khi nào |
|---|---|
| `:events` | Xem Events toàn cluster — chỗ đầu tiên nên nhìn khi có gì sai |
| `:pulses` | Dashboard tổng quan sức khoẻ cluster |
| `:xray <resource>` | Xem cây quan hệ resource dạng tree (VD `:xray deploy`) |
| `:popeye` | Chạy Popeye linter — soi config "có vấn đề" trong cluster (cần cài plugin) |
| `:dp` | Deployments shortcut |
| `:node` → `d` | Describe node để xem resource pressure, taints |
| `:top pod` / `:top node` | Xem usage CPU/Memory (cần metrics-server) |

---

## Benchmark & port-forward nhanh

```shell
# Trong view pod/service, bấm Shift-f để port-forward,
# k9s sẽ hỏi local port <-> remote port, không cần gõ tay kubectl port-forward
```

| Phím | Chức năng |
|---|---|
| `Shift-f` | Port-forward resource đang chọn |
| `Ctrl-b` | Xem danh sách port-forward đang active (view riêng) |

---

## Plugin hay dùng

File định nghĩa: `~/.config/k9s/plugins.yaml`. Mỗi plugin = 1 phím tắt + 1 command chạy lên resource đang chọn, dùng biến sẵn có của k9s (`$NAME`, `$NAMESPACE`, `$CONTAINER`, `$RESOURCE_NAME`, `$RESOURCE_KIND`...).

| Plugin | Việc nó làm | Cần cài thêm |
|---|---|---|
| **stern** | Tail log của nhiều pod cùng lúc (match theo tên/label), màu mè dễ đọc hơn log view mặc định | `brew install stern` |
| **popeye** | Lint toàn cluster, chỉ ra config "có mùi" (resource limit thiếu, probe thiếu, RBAC quá rộng...) | `brew install derailed/popeye/popeye` |
| **kubectl-neat** | Xem YAML "sạch", bỏ hết field rác k8s tự sinh (status, managedFields...) | `krew install neat` |
| **ketall (get-all)** | Liệt kê toàn bộ resource trong namespace, kể cả loại "mồ côi" ít ai nhớ tới | `krew install get-all` |
| **kubectl debug (ephemeral container)** | Gắn 1 container debug (busybox/netshoot) vào pod đang lỗi mà không cần sửa pod spec | built-in `kubectl` ≥ 1.18 |
| **node-shell** | SSH-like shell thẳng vào node qua 1 pod privileged, khỏi cần SSH key | `krew install node-shell` |

Cấu hình mẫu cho `~/.config/k9s/plugins.yaml`:

```yaml
plugins:
  # Tail nhiều pod cùng lúc, log màu, dễ đọc hơn log view gốc
  stern:
    shortCut: Ctrl-L
    confirm: false
    description: "Logs <Stern>"
    scopes:
      - po
    command: sh
    background: false
    args:
      - -c
      - "stern $NAME -n $NAMESPACE --color always | less -R"

  # Scan toàn cluster tìm config có vấn đề
  popeye:
    shortCut: Shift-P
    confirm: false
    description: "Popeye scan"
    scopes:
      - all
    command: sh
    background: false
    args:
      - -c
      - "popeye -n $NAMESPACE --out standard | less"

  # Xem YAML sạch, bỏ field rác
  neat:
    shortCut: Shift-N
    confirm: false
    description: "YAML (neat)"
    scopes:
      - all
    command: sh
    background: false
    args:
      - -c
      - "kubectl get $RESOURCE_NAME.$RESOURCE_KIND -n $NAMESPACE -o yaml | kubectl neat | less"

  # Gắn container debug vào pod đang lỗi (không sửa pod spec)
  debug:
    shortCut: Shift-D
    confirm: true
    description: "Debug container"
    scopes:
      - pods
    command: kubectl
    background: false
    args:
      - debug
      - -it
      - --image=busybox
      - --target=$NAME
      - -n
      - $NAMESPACE
      - $NAME

  # Shell thẳng vào node, không cần SSH key
  node-shell:
    shortCut: Shift-S
    confirm: true
    description: "Node shell"
    scopes:
      - no
    command: kubectl
    background: false
    args:
      - node-shell
      - $NAME
```

> [!tip]
> Sau khi thêm/sửa `plugins.yaml`, không cần restart k9s — mở `?` ra sẽ thấy phím tắt mới xuất hiện ngay trong help của view tương ứng (nếu `scopes` khớp).

> [!note]
> Có repo community gom sẵn rất nhiều plugin, chỉ cần copy block cần dùng vào `plugins.yaml`: tìm "k9s plugins" trên GitHub (`derailed/k9s` wiki cũng có list).

---

## Tuỳ biến nhẹ cho đỡ bỡ ngỡ

- File config: `~/.config/k9s/config.yaml`
- Đổi theme/skin: `~/.config/k9s/skins/<ten-skin>.yaml`, rồi set trong config
- Bật chế độ an toàn khi mới làm quen:

```shell
k9s --readonly        # tắt hết action xoá/sửa, chỉ xem
```

- Trong config.yaml có thể set `confirmLookForward` hoặc giảm `refreshRate` để UI đỡ giật nếu cluster chậm.

---

## So sánh nhanh với Lens (để đỡ mất phương hướng)

| Lens (GUI) | k9s (TUI) |
|---|---|
| Click sidebar chọn resource | Gõ `:<resource>` |
| Search box trên cùng | `/` để filter |
| Click phải → View Logs | Chọn pod, bấm `l` |
| Click phải → Exec Shell | Chọn pod, bấm `s` |
| Click phải → Edit | Bấm `e` |
| Click phải → Delete | `Ctrl-d` |
| Tab đổi cluster context | `:ctx` |
| Resource graph / relations | `:xray <resource>` |
| Terminal riêng mở port-forward | `Shift-f` |

---

## Tips tổng hợp

- Gõ sai alias? Cứ gõ `:` rồi start gõ tên, k9s có autocomplete fuzzy — không cần nhớ chính xác 100%.
- `?` ở bất kỳ view nào để xem **đúng** hotkey hợp lệ cho context hiện tại (hotkey có thể khác nhau tuỳ resource).
- Muốn an toàn tuyệt đối lúc mới học: luôn mở `k9s --readonly` trước, quen tay rồi mới bật quyền sửa/xoá.
- `Ctrl-a` khi lạc hướng là cứu cánh — hiện full danh sách resource/alias đang hỗ trợ trên cluster đó (bao gồm cả CRD).
