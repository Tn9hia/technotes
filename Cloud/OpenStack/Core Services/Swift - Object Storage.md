---
tags:
  - openstack
  - swift
  - object-storage
---

# Swift — Object Storage Service

Swift là **distributed object storage** của OpenStack — tương đương Amazon S3. Lưu trữ unstructured data: images, backups, static files, logs.

## Kiến trúc

```
Client
  │
swift-proxy (load balanced)
  │
  ├── Account Server (user account metadata)
  ├── Container Server (như S3 bucket)
  └── Object Server (actual data)
          │
     Storage nodes (với replication)
```

### Replication

Swift mặc định dùng **3 replicas** (có thể configure). Data được phân tán theo **Consistent Hashing Ring**.

```
Object → Hash → Ring → 3 Storage nodes
```

## Concepts

| Concept | Tương đương S3 | Mô tả |
|---------|---------------|-------|
| Account | AWS Account | Top-level namespace |
| Container | S3 Bucket | Nhóm objects |
| Object | S3 Object | File/data |

## CLI Usage

```bash
# List containers
openstack object store account show
openstack container list

# Tạo container
openstack container create my-bucket

# Upload object
openstack object create my-bucket local-file.txt

# Upload với object name tùy chọn
openstack object create my-bucket local-file.txt --name path/to/remote-file.txt

# Download
openstack object save my-bucket remote-file.txt

# List objects
openstack object list my-bucket

# Delete
openstack object delete my-bucket remote-file.txt

# Generate temp URL (pre-signed URL)
openstack object store account set --property Temp-URL-Key=mysecret
swift tempurl GET 3600 /v1/AUTH_project/my-bucket/file.txt mysecret
```

## S3 API Compatibility

Swift hỗ trợ **S3 API** qua middleware:

```ini
# proxy-server.conf
[pipeline:main]
pipeline = ... s3api ... proxy-server

[filter:s3api]
use = egg:swift#s3api
```

Khi đó có thể dùng AWS CLI với Swift:

```bash
aws --endpoint-url http://swift-proxy:8080 s3 ls
aws --endpoint-url http://swift-proxy:8080 s3 cp file.txt s3://my-bucket/
```

## Cấu hình Storage Policy

```ini
# swift.conf
[storage-policy:0]
name = Policy-0
default = yes
aliases = gold

[storage-policy:1]
name = silver
aliases = 3x-replicated
```

## Erasure Coding

Thay vì replication 3x, dùng **Erasure Coding** tiết kiệm storage hơn (trade-off: CPU cao hơn, latency cao hơn):

```ini
[storage-policy:2]
name = ec42
policy_type = erasure_coding
ec_type = liberasurecode_rs_vand
ec_num_data_fragments = 4
ec_num_parity_fragments = 2
```

## Swift vs Ceph

| Tiêu chí | Swift | Ceph |
|---------|-------|------|
| Protocol | HTTP/REST | RBD, RADOS, S3, CephFS |
| Block storage | ❌ | ✅ |
| Object storage | ✅ | ✅ (RadosGW) |
| File storage | ❌ | ✅ (CephFS) |
| Complexity | Thấp hơn | Cao hơn |
| Performance | OK | Tốt hơn |
| Ecosystem | OpenStack only | Đa dạng |

> [!tip] Dùng Ceph cho production
> Nếu đã deploy Ceph cho Cinder/Nova, dùng **Ceph RadosGW** (RADOS Gateway) thay Swift — một hệ thống thay vì hai.

---
*Xem thêm: [[Cinder - Block Storage]] | [[Cloud/AWS/SAA/Storage/Storage Overview]] | [[Ceph Integration]] | [[OpenStack]]*
