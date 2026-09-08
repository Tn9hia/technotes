---
tags:
  - cloudstack
  - operations
  - cli
  - api
---

# CLI & API — CloudMonkey

## CloudMonkey (`cmk`) — CLI chính thức

```bash
pip install cloudmonkey
# hoặc tải binary release từ Apache CloudStack

cmk set url https://cloudstack.company.local:8080/client/api
cmk set apikey <api-key>
cmk set secretkey <secret-key>

# Interactive mode
cmk
> list virtualmachines state=Running
> deploy virtualmachine serviceofferingid=... templateid=... zoneid=... networkids=...
```

> [!tip] So với PowerCLI/govc
> `cmk` gần giống `govc` (CLI thuần cho vSphere API) hơn là PowerCLI — mọi lệnh ánh xạ **1-1 với 1 API call**, không có logic client-side phức tạp như PowerCLI cmdlet. Muốn biết `cmk` đang gọi API nào, thêm `-d`/`--debug` hoặc xem tài liệu API tương ứng tên lệnh (VD: `cmk list virtualmachines` → API `listVirtualMachines`).

## REST API — cơ chế ký request (quan trọng khi tự viết script/tool)

CloudStack API dùng **HMAC-SHA1 signature** (không phải Bearer token như nhiều API hiện đại):

```bash
#!/bin/bash
API_URL="https://cloudstack.company.local:8080/client/api"
API_KEY="your-api-key"
SECRET_KEY="your-secret-key"

# 1. Build query string, sort params theo alphabet, lowercase, URL-encode
PARAMS="apikey=${API_KEY}&command=listVirtualMachines&response=json"
SORTED=$(echo "$PARAMS" | tr '&' '\n' | sort | tr '\n' '&' | sed 's/&$//')

# 2. Tạo signature: HMAC-SHA1 rồi base64, input là chuỗi lowercase
SIG=$(echo -n "$(echo "$SORTED" | tr 'A-Z' 'a-z')" | \
  openssl dgst -sha1 -hmac "$SECRET_KEY" -binary | base64)

# 3. Gọi API với signature đã URL-encode
curl -s "${API_URL}?${SORTED}&signature=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$SIG")"
```

> [!warning] Lesson learned: lỗi ký signature là nguyên nhân #1 khi tự viết integration
> Các lỗi thường gặp: (1) quên **sort tham số theo alphabet** trước khi ký, (2) quên **lowercase toàn bộ chuỗi** trước khi tính HMAC, (3) URL-encode sai thứ tự (phải ký trước, encode sau, rồi mới encode `signature` khi gắn vào query cuối cùng). CloudStack trả lỗi `531 - Unable to verify user credentials` chung chung cho **mọi lỗi ký sai**, không phân biệt rõ nguyên nhân — khi gặp lỗi này, kiểm tra lại đúng 3 điểm trên trước khi nghi ngờ API key sai.

## Async Job — luôn phải poll kết quả

```bash
# API tạo VM trả về ngay jobid, chưa chắc đã tạo xong
cmk deploy virtualmachine ... 
# → {"jobid": "abc-123", ...}

# Phải query để biết kết quả thật
cmk query asyncjobresult jobid=abc-123
# jobstatus: 0 = đang chạy, 1 = thành công, 2 = thất bại
```

> [!warning] Đừng coi "API trả về 200 OK" là "thao tác thành công"
> Vì gần như mọi thao tác thay đổi state (deploy, snapshot, migrate...) đều async, **HTTP 200 chỉ nghĩa là job đã được nhận**, không phải đã hoàn thành. Script tự động hóa **bắt buộc** phải poll `queryAsyncJobResult` tới khi `jobstatus != 0`, và kiểm tra `jobresultcode` để biết thành công hay lỗi — bỏ qua bước này là nguyên nhân phổ biến khiến pipeline "tưởng thành công" trong khi VM thực ra deploy fail.

## Terraform Provider

```hcl
terraform {
  required_providers {
    cloudstack = {
      source = "cloudstack/cloudstack"
    }
  }
}
provider "cloudstack" {
  api_url    = var.cs_api_url
  api_key    = var.cs_api_key
  secret_key = var.cs_secret_key
}
```

## Ansible

```yaml
- name: Deploy VM
  ngine_io.cloudstack.cs_instance:
    name: web-01
    template: ubuntu-22.04
    service_offering: Standard-4C8G
    zone: zone-a
    api_url: "{{ cs_api_url }}"
    api_key: "{{ cs_api_key }}"
    api_secret: "{{ cs_secret_key }}"
```

## Các lệnh `cmk` hay dùng hằng ngày

```bash
cmk list virtualmachines state=Running listall=true
cmk list events startdate=2026-09-01               # audit trail
cmk list alerts                                     # cảnh báo hệ thống
cmk list capacity                                    # capacity theo zone/pod/cluster
cmk list asyncjobs listall=true                      # job đang chạy
cmk list systemvms
cmk list routers listall=true
cmk list hosts listall=true state=Up
```

---
*Xem thêm: [[CloudStack Management Server]] | [[RBAC & Roles (CloudStack)]] | [[CloudStack Troubleshooting]] | [[Cloudstack|CloudStack]]*
