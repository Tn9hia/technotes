

## Cú pháp cơ bản

|Syntax|Mô tả|Example|
|---|---|---|
|`$`|Root node|`$`|
|`@`|Current node (dùng trong filter)|`@.price`|
|`.` hoặc `[]`|Child operator|`$.store.book` hoặc `$['store']['book']`|
|`..`|Recursive descent (tìm ở mọi level)|`$..author`|
|`*`|Wildcard (tất cả elements)|`$.store.*`|
|`[]`|Subscript operator|`$[0]`, `$[-1]`|
|`[,]`|Union operator|`$[0,1]` hoặc `$['name','age']`|
|`[start:end:step]`|Array slice|`$[0:5]`, `$[::2]`|
|`?()`|Filter expression|`$[?(@.price < 10)]`|

---

## 1️⃣ Truy cập cơ bản

### Truy cập object properties

```json
{
  "name": "Nghia",
  "job": "VMware Engineer"
}
```

|JSONPath|Result|
|---|---|
|`$.name`|`"Nghia"`|
|`$['name']`|`"Nghia"`|
|`$.job`|`"VMware Engineer"`|

### Truy cập array elements

```json
{
  "skills": ["Linux", "K8s", "VMware"]
}
```

|JSONPath|Result|
|---|---|
|`$.skills[0]`|`"Linux"`|
|`$.skills[-1]`|`"VMware"` (last element)|
|`$.skills[*]`|All elements|
|`$.skills[0,2]`|`["Linux", "VMware"]`|

---

## 2️⃣ Array Slicing

```json
{
  "numbers": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
}
```

|JSONPath|Result|Mô tả|
|---|---|---|
|`$.numbers[0:3]`|`[0,1,2]`|Từ index 0 đến 2|
|`$.numbers[2:]`|`[2,3,4,5,6,7,8,9]`|Từ index 2 đến hết|
|`$.numbers[:5]`|`[0,1,2,3,4]`|Từ đầu đến index 4|
|`$.numbers[-3:]`|`[7,8,9]`|3 phần tử cuối|
|`$.numbers[::2]`|`[0,2,4,6,8]`|Bước nhảy 2|
|`$.numbers[1::2]`|`[1,3,5,7,9]`|Bắt đầu từ 1, nhảy 2|

---

## 3️⃣ Recursive Descent (..)

Tìm tất cả matching nodes ở mọi level trong JSON tree.

```json
{
  "company": "Viettel IDC",
  "departments": {
    "cloud": {
      "team": "VMware",
      "lead": "Nghia"
    },
    "security": {
      "team": "SecOps",
      "lead": "An"
    }
  }
}
```

|JSONPath|Result|
|---|---|
|`$..team`|`["VMware", "SecOps"]`|
|`$..lead`|`["Nghia", "An"]`|
|`$..departments..lead`|`["Nghia", "An"]`|

---

## 4️⃣ Wildcard (*)

```json
{
  "servers": {
    "web": {"cpu": 4, "ram": 8},
    "db": {"cpu": 8, "ram": 16},
    "cache": {"cpu": 2, "ram": 4}
  }
}
```

|JSONPath|Result|
|---|---|
|`$.servers.*`|All server objects|
|`$.servers.*.cpu`|`[4, 8, 2]`|
|`$.servers.*.ram`|`[8, 16, 4]`|

---

## 5️⃣ Filter Expressions (QUAN TRỌNG!)

### Comparison operators

- `==` equal
- `!=` not equal
- `<` less than
- `<=` less than or equal
- `>` greater than
- `>=` greater than or equal

### Logical operators

- `&&` AND
- `||` OR
- `!` NOT

### Sample data

```json
{
  "vms": [
    {"name": "vm-web-01", "cpu": 4, "ram": 8, "status": "running"},
    {"name": "vm-db-01", "cpu": 8, "ram": 16, "status": "running"},
    {"name": "vm-test-01", "cpu": 2, "ram": 4, "status": "stopped"},
    {"name": "vm-backup-01", "cpu": 4, "ram": 8, "status": "stopped"}
  ]
}
```

### Common filters

|JSONPath|Result|Mô tả|
|---|---|---|
|`$.vms[?(@.cpu > 4)]`|VMs có CPU > 4|Filter bằng số|
|`$.vms[?(@.status == 'running')]`|VMs đang chạy|Filter bằng string|
|`$.vms[?(@.ram >= 8)]`|VMs có RAM >= 8GB|Greater than or equal|
|`$.vms[?(@.cpu == 4 && @.ram == 8)]`|VMs có 4 CPU và 8GB RAM|AND condition|
|`$.vms[?(@.status == 'stopped'||@.cpu < 4)]`|
|`$.vms[?(@.name =~ /web.*/)]`|VMs có tên match regex|Regex (nếu support)|

---

## 6️⃣ Use Cases thực tế trong DevOps/K8s

### Kubernetes JSON output

```json
{
  "items": [
    {
      "metadata": {"name": "nginx-pod", "namespace": "default"},
      "spec": {"containers": [{"name": "nginx", "image": "nginx:1.21"}]},
      "status": {"phase": "Running"}
    },
    {
      "metadata": {"name": "redis-pod", "namespace": "cache"},
      "spec": {"containers": [{"name": "redis", "image": "redis:6"}]},
      "status": {"phase": "Pending"}
    }
  ]
}
```

|Use Case|JSONPath|
|---|---|
|Get tất cả pod names|`$.items[*].metadata.name`|
|Get pods trong namespace default|`$.items[?(@.metadata.namespace == 'default')]`|
|Get running pods|`$.items[?(@.status.phase == 'Running')]`|
|Get container images|`$.items[*].spec.containers[*].image`|
|Get pending pods trong namespace cache|`$.items[?(@.metadata.namespace == 'cache' && @.status.phase == 'Pending')]`|

### VMware vCenter API response

```json
{
  "value": [
    {"vm": "vm-100", "name": "web-server", "power_state": "POWERED_ON", "cpu_count": 4},
    {"vm": "vm-101", "name": "db-server", "power_state": "POWERED_ON", "cpu_count": 8},
    {"vm": "vm-102", "name": "test-server", "power_state": "POWERED_OFF", "cpu_count": 2}
  ]
}
```

|Use Case|JSONPath|
|---|---|
|Get powered on VMs|`$.value[?(@.power_state == 'POWERED_ON')]`|
|Get VM names|`$.value[*].name`|
|Get VMs có >= 4 CPU|`$.value[?(@.cpu_count >= 4)]`|

---

## 7️⃣ Nested Arrays & Objects

```json
{
  "clusters": [
    {
      "name": "prod-cluster",
      "nodes": [
        {"hostname": "node1", "ip": "10.0.1.10", "role": "master"},
        {"hostname": "node2", "ip": "10.0.1.11", "role": "worker"}
      ]
    },
    {
      "name": "dev-cluster",
      "nodes": [
        {"hostname": "node3", "ip": "10.0.2.10", "role": "master"}
      ]
    }
  ]
}
```

|JSONPath|Result|
|---|---|
|`$.clusters[*].nodes[*].hostname`|`["node1", "node2", "node3"]`|
|`$.clusters[0].nodes[?(@.role == 'master')]`|Master nodes trong prod cluster|
|`$..nodes[?(@.role == 'worker')]`|Tất cả worker nodes|
|`$.clusters[*].nodes[*].ip`|Tất cả IPs|

---

## 8️⃣ Tips & Tricks

### 💡 Test JSONPath online

- https://jsonpath.com/
- https://jsonpath.curiousconcept.com/

### 💡 Tools hỗ trợ JSONPath

- **kubectl**: `kubectl get pods -o jsonpath='{.items[*].metadata.name}'`
- **jq**: Alternative mạnh hơn (không phải JSONPath nhưng xịn)
- **Python**: `jsonpath-ng` library
- **Go**: `github.com/oliveagle/jsonpath`

### 💡 Common gotchas

- Single quotes `'` vs double quotes `"`: Phụ thuộc vào tool
- Array index starts at `0`, không phải `1`
- `..` có thể slow nếu JSON lớn
- Không phải tool nào cũng support đầy đủ JSONPath spec

---

## 9️⃣ Quick Reference Commands

### kubectl với JSONPath

```bash
# Get pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Get container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Range (loop)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### curl + jq (alternative)

```bash
# Get specific fields
curl -s api.example.com/vms | jq '.vms[] | select(.status == "running")'

# Filter và format
curl -s api.example.com/vms | jq '.vms[] | {name: .name, cpu: .cpu}'
```

---

## 🔟 Practice Exercises

Thử với JSON này:

```json
{
  "datacenter": "Viettel IDC",
  "racks": [
    {
      "id": "rack-01",
      "servers": [
        {"hostname": "esxi-01", "cpu": 64, "ram": 512, "role": "hypervisor"},
        {"hostname": "esxi-02", "cpu": 64, "ram": 512, "role": "hypervisor"}
      ]
    },
    {
      "id": "rack-02",
      "servers": [
        {"hostname": "storage-01", "cpu": 32, "ram": 256, "role": "storage"},
        {"hostname": "switch-01", "cpu": 8, "ram": 16, "role": "network"}
      ]
    }
  ]
}
```

**Câu hỏi:**

1. Lấy tất cả hostnames?
2. Lấy servers có RAM >= 256GB?
3. Lấy hypervisors?
4. Lấy total CPU của rack-01?

**Đáp án:**

1. `$..servers[*].hostname`
2. `$..servers[?(@.ram >= 256)]`
3. `$..servers[?(@.role == 'hypervisor')]`
4. `$.racks[?(@.id == 'rack-01')].servers[*].cpu` (cộng tay)

---

**Made with 💚 for Nghia @ Viettel IDC**