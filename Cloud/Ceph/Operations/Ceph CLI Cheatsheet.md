---
tags:
  - ceph
  - operations
  - cli
  - cheatsheet
---

# Ceph CLI Cheatsheet

**Tra cứu nhanh** các lệnh `ceph`, `rbd`, `radosgw-admin`, `cephadm` dùng hằng ngày khi vận hành cluster. Đây là trang được mở nhiều nhất trong thao tác thực tế — nên ưu tiên thuộc lòng nhóm "cluster health" và "orchestrator" trước, phần còn lại tra khi cần.

> [!tip] So với VMware vSAN
> vSAN gần như mọi thao tác đi qua `esxcli vsan` hoặc PowerCLI/vCenter UI — một layer duy nhất. Ceph có **nhiều CLI theo từng interface**: `ceph` (cluster/RADOS), `rbd` (block), `radosgw-admin` (object), `ceph fs`/`ceph-fuse` (file), và `cephadm`/`ceph orch` (deployment/lifecycle). Quen dần sẽ thấy chúng khá nhất quán về cú pháp (`<noun> <verb>` hoặc `<verb> <noun>`).

## Cluster Health & Trạng thái tổng quan

```bash
ceph -s                          # hoặc: ceph status — tổng quan nhanh
ceph health                      # HEALTH_OK / HEALTH_WARN / HEALTH_ERR
ceph health detail               # chi tiết từng cảnh báo/lỗi
ceph -w                          # theo dõi realtime (watch), Ctrl+C để thoát
ceph df                          # dung lượng cluster + theo pool
ceph df detail                   # kèm object count, dirty, compression stats
ceph versions                    # version từng loại daemon đang chạy
ceph features                    # feature flags của client/daemon đang kết nối
ceph fsid                        # lấy fsid của cluster
ceph time-sync-status            # kiểm tra clock skew giữa các MON
```

| Trạng thái | Ý nghĩa |
|---|---|
| `HEALTH_OK` | Cluster khỏe, không có cảnh báo |
| `HEALTH_WARN` | Có vấn đề cần chú ý nhưng chưa mất dữ liệu/downtime (VD: scrub trễ, gần đầy dung lượng) |
| `HEALTH_ERR` | Vấn đề nghiêm trọng, có nguy cơ mất dữ liệu hoặc dịch vụ gián đoạn (VD: PG inconsistent, PG down) |

## OSD

```bash
ceph osd tree                    # cây topology: root → host → OSD, kèm trạng thái up/down, weight
ceph osd df                      # dung lượng dùng theo từng OSD, %USE, PGs
ceph osd df tree                 # kết hợp cả hai trên
ceph osd stat                    # tổng số OSD, bao nhiêu up/in
ceph osd perf                    # commit_latency, apply_latency từng OSD (phát hiện disk chậm)
ceph osd dump                    # dump toàn bộ osdmap (dài, dùng khi debug sâu)
ceph osd find <id>               # OSD đang nằm trên host nào, IP nào
ceph osd metadata <id>           # thông tin phần cứng, kernel, rotational hay ssd

# Bật/tắt các "flag" toàn cluster (xem chi tiết ở Ceph Day 2 Operations)
ceph osd set noout
ceph osd unset noout
ceph osd set noscrub
ceph osd set nodeep-scrub
ceph osd unset noscrub
ceph osd unset nodeep-scrub

# Reweight (giảm tải OSD gần đầy mà không đổi CRUSH weight vĩnh viễn)
ceph osd reweight <id> 0.8
ceph osd reweight-by-utilization         # tự động, dùng thận trọng

# Ra/vào cluster
ceph osd out <id>
ceph osd in <id>
ceph osd down <id>               # đánh dấu down (hiếm khi cần thủ công)
```

| Lệnh | Dùng khi |
|---|---|
| `ceph osd perf` | Nghi ngờ 1 OSD/disk cụ thể làm chậm cả cluster (latency cao bất thường) |
| `ceph osd df` | Kiểm tra phân bố dung lượng có lệch (skew) giữa các OSD không |
| `ceph osd tree` | Xác nhận OSD nào down, thuộc host/rack nào theo CRUSH |

## Placement Groups (PG)

```bash
ceph pg stat                     # tổng quan số PG theo trạng thái (active+clean, degraded...)
ceph pg dump                     # dump toàn bộ PG (rất dài, nên dump ra file)
ceph pg dump_stuck                # PG bị "stuck" (inactive/unclean/stale) quá lâu
ceph pg ls                        # danh sách PG, có thể lọc theo state: ceph pg ls degraded
ceph pg <pgid> query              # chi tiết 1 PG cụ thể — dùng khi debug PG lỗi
ceph pg <pgid> repair             # yêu cầu sửa PG inconsistent (xem cảnh báo ở Ceph Troubleshooting)
ceph pg <pgid> scrub              # ép scrub 1 PG cụ thể ngay
ceph pg <pgid> deep-scrub         # ép deep-scrub 1 PG cụ thể ngay
ceph pg map <pgid>                # PG này map vào những OSD nào (theo CRUSH hiện tại)
```

## Pool

```bash
ceph osd pool ls                        # danh sách pool
ceph osd pool ls detail                 # kèm size, min_size, pg_num, crush_rule, flags
ceph osd pool create <name> <pg_num>    # tạo pool replicated (mặc định)
ceph osd pool create <name> <pg_num> erasure <profile>   # tạo pool erasure-coded

ceph osd pool set <name> size 3         # số bản sao (replicated)
ceph osd pool set <name> min_size 2     # số bản sao tối thiểu để pool còn writable
ceph osd pool set <name> pg_num 128
ceph osd pool set <name> pg_autoscale_mode on|off|warn

ceph osd pool get <name> size           # đọc lại 1 giá trị cụ thể
ceph osd pool get <name> all            # đọc toàn bộ config của pool

ceph osd pool rename <old> <new>
ceph osd pool delete <name> <name> --yes-i-really-really-mean-it   # xóa pool (phải gõ tên 2 lần + flag)
ceph osd pool application enable <name> rbd|cephfs|rgw             # gán application tag (bắt buộc khi tạo pool mới thủ công)
```

> [!warning] Lesson learned: quên `ceph osd pool application enable`
> Pool tạo thủ công (không qua `rbd pool init` hoặc CephFS/RGW tự động) mà chưa gán application tag sẽ khiến `ceph health` báo `POOL_APP_NOT_ENABLED` — vô hại nhưng gây nhiễu HEALTH_WARN liên tục, và một số client/tool có thể từ chối ghi vào pool "chưa xác định mục đích". Luôn gán tag ngay sau khi tạo pool thủ công.

## Orchestrator (cephadm)

```bash
ceph orch ls                            # danh sách service (osd, mon, mgr, rgw...) và số lượng instance
ceph orch ps                            # danh sách từng daemon, node, trạng thái, version
ceph orch ps --daemon_type osd          # lọc theo loại daemon
ceph orch host ls                       # danh sách host trong cluster, labels
ceph orch host add <hostname> <ip> --labels _admin,mon
ceph orch host rm <hostname>
ceph orch host maintenance enter <hostname>
ceph orch host maintenance exit <hostname>

ceph orch apply mon --placement="3 host1,host2,host3"   # đặt số lượng/placement cho 1 service
ceph orch apply osd --all-available-devices              # tự nhận toàn bộ disk trống làm OSD
ceph orch daemon add osd <host>:<device>                  # thêm OSD thủ công 1 disk cụ thể
ceph orch daemon rm osd.<id> --force                      # xóa daemon (cẩn thận, xem Scaling)
ceph orch daemon restart <daemon-name>                    # restart 1 daemon cụ thể
ceph orch upgrade start --image <image> --ceph-version <ver>
ceph orch upgrade status

cephadm ls                              # daemon chạy trên host hiện tại (chạy trực tiếp trên host, không qua ceph CLI)
cephadm logs --name <daemon-name>       # xem log daemon (đọc từ journald)
cephadm shell                           # mở shell có sẵn ceph CLI + keyring, kể cả khi chưa cài package ceph-common
```

> [!tip] So với VMware vSAN
> `ceph orch` tương đương tầng "lifecycle management" mà vCenter làm ẩn cho bạn (deploy/scale/upgrade ESXi). Với cephadm, daemon chạy dưới dạng **container** trên từng host — không còn khái niệm cài package `ceph-osd` trực tiếp như thời `ceph-deploy` cũ. `ceph orch ps` là lệnh tương đương "xem toàn bộ VM hệ thống đang chạy ở đâu" mà bạn sẽ dùng liên tục.

## Auth / cephx

```bash
ceph auth ls                                    # toàn bộ user/keyring trong cluster
ceph auth get client.admin                      # xem 1 keyring cụ thể
ceph auth get-or-create client.myapp mon 'allow r' osd 'allow rwx pool=rbd-pool'
ceph auth caps client.myapp mon 'allow r' osd 'allow rwx pool=rbd-pool'   # đổi cap cho user có sẵn
ceph auth rm client.myapp                       # xóa user
ceph auth print-key client.admin                # in riêng phần key (dùng để copy vào config client)
```

## Config

```bash
ceph config dump                                 # toàn bộ config override đang áp dụng (centralized config)
ceph config get osd.0 osd_max_backfills          # đọc giá trị hiệu lực cho 1 daemon cụ thể
ceph config set osd osd_max_backfills 4          # set cho toàn bộ nhóm "osd"
ceph config set osd.3 osd_max_backfills 1        # set riêng cho 1 daemon
ceph config rm osd osd_max_backfills             # xóa override, quay lại default
ceph config show osd.0                           # config đầy đủ (default + override) daemon đang chạy thực tế
ceph config help osd_max_backfills                # mô tả + kiểu dữ liệu + default của 1 tham số
```

## RBD (Block Storage)

```bash
rbd ls <pool>                            # danh sách image trong pool
rbd info <pool>/<image>                  # size, object size, features, parent (nếu là clone)
rbd du <pool>/<image>                    # dung lượng thật đang dùng (provisioned vs actual)
rbd status <pool>/<image>                # ai đang watch/lock image này (client đang mount)
rbd snap ls <pool>/<image>               # danh sách snapshot
rbd snap create <pool>/<image>@<snap>
rbd snap rm <pool>/<image>@<snap>
rbd snap purge <pool>/<image>            # xóa toàn bộ snapshot của image
rbd resize --size <size-in-M/G/T> <pool>/<image>
rbd rm <pool>/<image>                    # xóa image (không rollback được)
rbd map <pool>/<image>                   # map image thành /dev/rbdX trên node hiện tại
rbd showmapped                           # danh sách image đã map trên node này
```

## RGW (Object Storage)

```bash
radosgw-admin user list
radosgw-admin user info --uid=<uid>
radosgw-admin user create --uid=<uid> --display-name="<name>"
radosgw-admin bucket list
radosgw-admin bucket stats --bucket=<bucket-name>
radosgw-admin bucket limit check                 # kiểm tra bucket gần chạm giới hạn shard
radosgw-admin usage show --uid=<uid>             # thống kê usage theo user
radosgw-admin period get                          # config multi-site hiện tại (nếu có)
```

## CephFS

```bash
ceph fs status                           # tổng quan file system: rank, MDS active/standby
ceph fs volume ls                        # danh sách volume (từ Ceph 16+ khuyến nghị quản lý qua volume)
ceph fs ls                               # danh sách file system (kiểu cũ hơn)
ceph fs get <name>                        # chi tiết cấu hình 1 file system
ceph mds stat                             # trạng thái các MDS daemon
```

## Crash Reports

```bash
ceph crash ls                            # danh sách crash report daemon đã ghi nhận
ceph crash info <crash-id>               # chi tiết 1 crash (stack trace)
ceph crash archive <crash-id>            # đánh dấu đã xem/xử lý (không xóa dữ liệu)
ceph crash archive-all                   # archive toàn bộ crash đang pending
ceph crash stat                          # đếm số crash chưa archive
```

## Performance Testing

```bash
# rados bench — test trực tiếp tầng RADOS, không qua RBD/CephFS/RGW
rados bench -p <pool> 30 write --no-cleanup
rados bench -p <pool> 30 seq
rados bench -p <pool> 30 rand
rados -p <pool> cleanup                  # dọn object test còn sót lại sau bench write

# rbd bench — test qua tầng block, gần với workload thật của VM hơn
rbd bench --io-type write --io-size 4K --io-threads 16 --io-total 1G <pool>/<image>
rbd bench --io-type read <pool>/<image>
```

> [!warning] Lesson learned: quên `rados -p <pool> cleanup` sau khi bench
> `rados bench ... write --no-cleanup` để lại hàng nghìn object benchmark trong pool production nếu chạy nhầm pool hoặc quên dọn — âm thầm ăn dung lượng và làm sai lệch số liệu capacity planning sau này. Luôn bench trên pool test riêng, và luôn chạy `rados -p <pool> cleanup` ngay sau khi xong việc.

## Bảng tổng hợp nhanh theo mục đích

| Muốn biết... | Lệnh |
|---|---|
| Cluster có khỏe không | `ceph -s` |
| Vì sao WARN/ERR | `ceph health detail` |
| Daemon nào đang chạy ở đâu | `ceph orch ps` |
| Disk nào chậm | `ceph osd perf` |
| Pool nào gần đầy | `ceph df detail` |
| PG nào có vấn đề | `ceph pg ls` rồi `ceph pg <id> query` |
| Ai đang mount RBD image | `rbd status <pool>/<image>` |
| Log của 1 daemon | `cephadm logs --name <daemon>` hoặc `journalctl -u ceph-<fsid>@<daemon>` |
| Daemon nào vừa crash | `ceph crash ls` |

---
*Xem thêm: [[Ceph Day 2 Operations]] | [[Ceph Troubleshooting]] | [[Ceph|Ceph]]*
