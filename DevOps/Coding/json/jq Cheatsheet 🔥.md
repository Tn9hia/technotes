

> **jq** - Command-line JSON processor, powerful AF and must-have cho DevOps/SRE

---

## 0️⃣ Installation & Basics

```bash
# Install
apt install jq        # Debian/Ubuntu
yum install jq        # RHEL/CentOS
brew install jq       # macOS

# Basic usage
echo '{"name":"Nghia"}' | jq '.'
cat file.json | jq '.'
curl -s api.example.com | jq '.'

# Pretty print (default)
jq '.' input.json

# Compact output
jq -c '.' input.json

# Raw output (no quotes for strings)
jq -r '.name' input.json
```

---

## 1️⃣ Basic Selectors

### Sample data

```json
{
  "name": "Nghia",
  "job": "VMware Engineer",
  "company": "Viettel IDC",
  "skills": ["Linux", "K8s", "VMware"],
  "experience": 5
}
```

|Command|Result|Mô tả|
|---|---|---|
|`jq '.'`|Toàn bộ JSON|Identity filter|
|`jq '.name'`|`"Nghia"`|Get property|
|`jq '.job, .company'`|Cả 2 values|Multiple fields|
|`jq '.skills'`|`["Linux", "K8s", "VMware"]`|Get array|
|`jq '.skills[0]'`|`"Linux"`|Array index|
|`jq '.skills[-1]'`|`"VMware"`|Last element|
|`jq '.skills[]'`|3 dòng riêng biệt|Iterate array|

---

## 2️⃣ Array Operations

```json
{
  "servers": [
    {"name": "web-01", "cpu": 4, "ram": 8, "status": "running"},
    {"name": "db-01", "cpu": 8, "ram": 16, "status": "running"},
    {"name": "test-01", "cpu": 2, "ram": 4, "status": "stopped"}
  ]
}
```

### Array Slicing

|Command|Result|
|---|---|
|`jq '.servers[0]'`|First server|
|`jq '.servers[0:2]'`|First 2 servers|
|`jq '.servers[:2]'`|First 2 servers|
|`jq '.servers[-1]'`|Last server|
|`jq '.servers[1:]'`|Từ index 1 đến hết|

### Array Iteration

```bash
# Get all server names
jq '.servers[].name'

# Get all CPUs
jq '.servers[].cpu'

# Get as array (với brackets)
jq '[.servers[].name]'
```

### Array Length

```bash
jq '.servers | length'
jq '.servers | length > 2'  # Boolean check
```

---

## 3️⃣ Filtering & Select

### Basic select

```bash
# Servers có CPU > 4
jq '.servers[] | select(.cpu > 4)'

# Running servers
jq '.servers[] | select(.status == "running")'

# Servers có RAM >= 8
jq '.servers[] | select(.ram >= 8)'

# Not equal
jq '.servers[] | select(.status != "stopped")'
```

### Complex conditions

```bash
# AND condition
jq '.servers[] | select(.cpu > 2 and .ram >= 8)'

# OR condition
jq '.servers[] | select(.status == "stopped" or .cpu < 4)'

# NOT condition
jq '.servers[] | select(.status == "running" | not)'

# Contains
jq '.servers[] | select(.name | contains("web"))'

# Starts with
jq '.servers[] | select(.name | startswith("web"))'

# Ends with
jq '.servers[] | select(.name | endswith("01"))'

# Regex match
jq '.servers[] | select(.name | test("web-.*"))'
```

---

## 4️⃣ Object Construction & Transformation

### Create new objects

```bash
# Simple object
jq '.servers[] | {name: .name, cpu: .cpu}'

# Rename fields
jq '.servers[] | {hostname: .name, cores: .cpu}'

# Add computed field
jq '.servers[] | {name: .name, total_gb: (.ram * 1024)}'

# Nested object
jq '.servers[] | {server: {name: .name, specs: {cpu: .cpu, ram: .ram}}}'
```

### Conditional fields

```bash
# Add field based on condition
jq '.servers[] | . + {tier: (if .cpu >= 8 then "high" else "standard" end)}'

# Multiple conditions
jq '.servers[] | . + {
  tier: (
    if .cpu >= 8 then "high"
    elif .cpu >= 4 then "medium"
    else "low"
    end
  )
}'
```

---

## 5️⃣ Map, Reduce & Advanced Array Operations

### map()

```bash
# Map over array
jq '.servers | map(.name)'

# Map with transformation
jq '.servers | map({name: .name, cpu: .cpu})'

# Map with select
jq '.servers | map(select(.status == "running"))'

# Map to uppercase
jq '.servers | map(.name | ascii_upcase)'
```

### map_values()

```bash
# Transform object values
echo '{"cpu": 4, "ram": 8}' | jq 'map_values(. * 2)'
# Result: {"cpu": 8, "ram": 16}
```

### reduce (tính tổng, aggregate)

```bash
# Sum all CPUs
jq '[.servers[].cpu] | add'
# hoặc
jq '.servers | map(.cpu) | add'

# Count
jq '.servers | length'

# Max CPU
jq '[.servers[].cpu] | max'

# Min CPU
jq '[.servers[].cpu] | min'

# Average
jq '[.servers[].cpu] | add / length'

# Custom reduce
jq 'reduce .servers[].cpu as $cpu (0; . + $cpu)'
```

### group_by()

```bash
# Group by status
jq '.servers | group_by(.status)'

# Group và count
jq '.servers | group_by(.status) | map({status: .[0].status, count: length})'

# Group by CPU tier
jq '.servers | group_by(.cpu >= 4) | map({high_cpu: .[0].cpu >= 4, servers: map(.name)})'
```

### unique & sort

```bash
# Unique values
jq '[.servers[].status] | unique'

# Sort array
jq '.servers | sort_by(.cpu)'

# Reverse sort
jq '.servers | sort_by(.cpu) | reverse'

# Sort by multiple fields
jq '.servers | sort_by(.status, .cpu)'
```

---

## 6️⃣ String Operations

```bash
# Uppercase
jq '.name | ascii_upcase'

# Lowercase
jq '.name | ascii_downcase'

# Split string
jq '.name | split("-")'

# Join array
jq '.skills | join(", ")'

# String interpolation
jq '.servers[] | "\(.name) has \(.cpu) CPUs"'

# String length
jq '.name | length'

# Trim whitespace
jq '.name | ltrimstr(" ") | rtrimstr(" ")'

# Replace
jq '.name | gsub("web"; "application")'

# Contains
jq '.name | contains("web")'

# Test regex
jq '.name | test("^web-")'

# Match regex and extract
jq '.name | match("web-(.*)") | .captures[0].string'
```

---

## 7️⃣ Pipe & Combine Operations

```bash
# Chain operations
jq '.servers | map(select(.status == "running")) | sort_by(.cpu)'

# Multi-stage transformation
jq '.servers 
  | map(select(.cpu >= 4))
  | sort_by(.ram)
  | reverse
  | .[0:2]'

# Alternative operator (fallback)
jq '.name // "unknown"'

# Try-catch
jq '.servers[]? // empty'  # Skip errors
```

---

## 8️⃣ Working with Multiple Files

```bash
# Slurp: read multiple JSON objects into array
jq -s '.' file1.json file2.json

# Combine arrays from multiple files
jq -s 'add' file1.json file2.json

# Compare two files
jq -s '.[0] - .[1]' file1.json file2.json
```

---

## 9️⃣ Real-World DevOps Use Cases

### Kubernetes

```bash
# Get pod names
kubectl get pods -o json | jq '.items[].metadata.name'

# Running pods only
kubectl get pods -o json | jq '.items[] | select(.status.phase == "Running") | .metadata.name'

# Pods with resource limits
kubectl get pods -o json | jq '.items[] | {
  name: .metadata.name,
  cpu: .spec.containers[0].resources.limits.cpu,
  memory: .spec.containers[0].resources.limits.memory
}'

# Count pods by namespace
kubectl get pods -A -o json | jq '.items | group_by(.metadata.namespace) | map({
  namespace: .[0].metadata.namespace,
  count: length
})'

# Pods not ready
kubectl get pods -o json | jq '.items[] | select(.status.phase != "Running") | .metadata.name'

# Container images across all pods
kubectl get pods -o json | jq '[.items[].spec.containers[].image] | unique'
```

### Docker

```bash
# List container IPs
docker inspect $(docker ps -q) | jq '.[].NetworkSettings.IPAddress'

# Container resource usage
docker stats --no-stream --format json | jq '{name: .Name, cpu: .CPUPerc, mem: .MemPerc}'

# Images with size
docker images --format json | jq '{repository: .Repository, tag: .Tag, size: .Size}'
```

### AWS CLI

```bash
# EC2 instances by name
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {
  name: .Tags[]? | select(.Key == "Name") | .Value,
  id: .InstanceId,
  type: .InstanceType,
  state: .State.Name
}'

# Running instances only
aws ec2 describe-instances | jq '.Reservations[].Instances[] | select(.State.Name == "running")'

# S3 buckets
aws s3api list-buckets | jq '.Buckets[].Name'

# Security groups with open ports
aws ec2 describe-security-groups | jq '.SecurityGroups[] | select(
  .IpPermissions[].FromPort == 22 and 
  .IpPermissions[].IpRanges[].CidrIp == "0.0.0.0/0"
)'
```

### VMware (REST API)

```bash
# Powered on VMs
curl -s -k -u user:pass https://vcenter/rest/vcenter/vm | jq '.value[] | select(.power_state == "POWERED_ON")'

# VM with CPU and memory
curl -s -k -u user:pass https://vcenter/rest/vcenter/vm | jq '.value[] | {
  name: .name,
  cpu: .cpu_count,
  memory: .memory_size_MiB
}'

# VMs by cluster
curl -s -k -u user:pass https://vcenter/rest/vcenter/vm | jq 'group_by(.cluster)'
```

### Git & CI/CD

```bash
# Parse package.json
jq '.dependencies' package.json

# Get version
jq -r '.version' package.json

# Update version
jq '.version = "2.0.0"' package.json > tmp.json && mv tmp.json package.json

# Parse GitLab CI artifacts
curl -s "https://gitlab.com/api/v4/projects/ID/jobs" | jq '.[] | {
  id: .id,
  status: .status,
  duration: .duration
}'
```

### Log Processing

```bash
# Parse JSON logs
cat app.log | jq -r 'select(.level == "error") | .message'

# Count errors by type
cat app.log | jq -s 'group_by(.error_type) | map({type: .[0].error_type, count: length})'

# Filter by timestamp
cat app.log | jq 'select(.timestamp > "2024-01-01")'

# Extract specific fields
cat app.log | jq '{time: .timestamp, level: .level, msg: .message}'
```

---

## 🔟 Advanced Techniques

### Variables

```bash
# Assign variable
jq '.servers[] as $s | $s.name'

# Multiple variables
jq '.servers[] | . as $server | .cpu as $cpu | "\($server.name) has \($cpu) cores"'
```

### Functions

```bash
# Define function
jq 'def double: . * 2; .servers[].cpu | double'

# Function with args
jq 'def multiply(n): . * n; .servers[].cpu | multiply(2)'
```

### Recursive descent

```bash
# Find all values for key "name" recursively
jq '.. | .name? // empty'

# All leaf values
jq '.. | scalars'
```

### Walk (transform recursively)

```bash
# Convert all numbers to strings
jq 'walk(if type == "number" then tostring else . end)'
```

### Add/Update/Delete fields

```bash
# Add field
jq '.servers[] | . + {location: "VN"}'

# Update field
jq '.servers[] | .cpu = (.cpu * 2)'

# Delete field
jq '.servers[] | del(.status)'

# Rename field
jq '.servers[] | .cores = .cpu | del(.cpu)'
```

---

## 1️⃣1️⃣ Output Formatting

```bash
# Raw output (no quotes)
jq -r '.name'

# Compact (single line)
jq -c '.'

# Tab-separated values
jq -r '.servers[] | [.name, .cpu, .ram] | @tsv'

# CSV output
jq -r '.servers[] | [.name, .cpu, .ram] | @csv'

# JSON to CSV with headers
jq -r '["Name", "CPU", "RAM"], (.servers[] | [.name, .cpu, .ram]) | @csv'

# URL encode
jq -r '@uri "hello world"'

# Base64 encode
jq -r '@base64 "hello"'

# Base64 decode
echo '"aGVsbG8="' | jq -r '@base64d'

# HTML escape
jq -r '@html "<div>test</div>"'

# Format as JSON
jq '@json'
```

---

## 1️⃣2️⃣ Debugging & Tips

```bash
# Debug mode
jq --debug '.'

# Show errors
jq -e '.name' || echo "Field not found"

# Null handling
jq '.field // "default"'
jq '.field // empty'  # Skip if null

# Type checking
jq 'type'
jq 'if type == "array" then length else 0 end'

# Pretty print with colors (default)
jq '.'

# No color
jq --monochrome-output '.'

# Sort keys
jq -S '.'
```

---

## 1️⃣3️⃣ Common Patterns & One-Liners

```bash
# Find and replace in JSON
jq '(.servers[] | select(.name == "web-01")).status = "stopped"'

# Merge two JSON objects
jq -s '.[0] * .[1]' obj1.json obj2.json

# Deep merge
jq -s '.[0] * .[1]' --arg-merge

# Flatten nested arrays
jq 'flatten'
jq 'flatten(2)'  # Flatten 2 levels

# Get keys
jq 'keys'
jq 'keys_unsorted'

# Check if key exists
jq 'has("name")'

# Get type of value
jq '.servers[0] | type'

# Empty check
jq '.servers | length == 0'

# To/from YAML (requires yq)
yq eval -o=json input.yaml | jq '.'
jq '.' input.json | yq eval -P -
```

---

## 1️⃣4️⃣ jq vs JSONPath Comparison

|Task|JSONPath|jq|
|---|---|---|
|Get field|`$.name`|`.name`|
|Array element|`$.arr[0]`|`.arr[0]`|
|Filter|`$.arr[?(@.cpu > 4)]`|`.arr[] \| select(.cpu > 4)`|
|All elements|`$.arr[*]`|`.arr[]`|
|Recursive|`$..name`|`.. \| .name? // empty`|
|Map|N/A|`.arr \| map(.name)`|
|Power|Limited|Full programming language|

**Verdict**: jq > JSONPath cho complex transformations!

---

## 1️⃣5️⃣ Quick Reference

### Most used commands

```bash
jq '.'                              # Pretty print
jq -r '.field'                      # Raw output
jq '.arr[]'                         # Iterate
jq '.arr[] | select(.x > 5)'       # Filter
jq '.arr | map(.name)'             # Map
jq '[.arr[].cpu] | add'            # Sum
jq '.arr | group_by(.status)'     # Group
jq '.arr | sort_by(.cpu)'         # Sort
jq '{name: .name, cpu: .cpu}'     # Transform
jq -s '.'                          # Slurp files
jq -c '.'                          # Compact
```

### Performance tips

- Use `select()` early để giảm data
- Dùng `limit()` nếu chỉ cần vài kết quả đầu
- `--stream` cho files khủng (GB+)
- Avoid `..` (recursive descent) với large JSON

---

## 1️⃣6️⃣ Practice Exercises

### Data

```json
{
  "datacenter": "Viettel IDC",
  "clusters": [
    {
      "name": "prod",
      "hosts": [
        {"name": "esxi-01", "cpu": 64, "ram": 512, "vms": 45},
        {"name": "esxi-02", "cpu": 64, "ram": 512, "vms": 50},
        {"name": "esxi-03", "cpu": 32, "ram": 256, "vms": 25}
      ]
    },
    {
      "name": "dev",
      "hosts": [
        {"name": "esxi-04", "cpu": 32, "ram": 256, "vms": 20}
      ]
    }
  ]
}
```

### Challenges

1. Total CPUs across all hosts?
2. Host with most VMs?
3. Average VMs per host in prod cluster?
4. Hosts với RAM >= 512GB?
5. Transform thành flat list of host names?

### Solutions

```bash
# 1. Total CPUs
jq '[.clusters[].hosts[].cpu] | add'

# 2. Host with most VMs
jq '.clusters[].hosts | max_by(.vms) | .name'

# 3. Average VMs in prod
jq '.clusters[] | select(.name == "prod") | [.hosts[].vms] | add / length'

# 4. Hosts with RAM >= 512GB
jq '.clusters[].hosts[] | select(.ram >= 512) | .name'

# 5. Flat list of names
jq '[.clusters[].hosts[].name]'
```

---

## Resources

- **Official**: https://jqlang.github.io/jq/
- **Playground**: https://jqplay.org/
- **Cheatsheet**: https://gist.github.com/olih/f7437fb6962fb3ee9fe95bda8d2c8fa4
- **Tutorial**: https://stedolan.github.io/jq/tutorial/

---

**Made with 🔥 for Nghia - jq master in the making!**