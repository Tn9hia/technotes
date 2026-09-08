### Zone management


```bash
# Tạo zone mới
pdnsutil create-zone nghia.internal

# List tất cả zones
pdnsutil list-all-zones

# Xem toàn bộ records trong zone
pdnsutil list-zone nghia.internal

# Xóa zone
pdnsutil delete-zone nghia.internal

# Check zone có lỗi không
pdnsutil check-zone nghia.internal
```

### Record management


```bash
# Thêm SOA record (bắt buộc)
sudo pdnsutil add-record nghia.internal . SOA "ns.nghia.internal. root.nghia.internal. 1 10800 3600 604800 3600"

# Thêm record
-
pdnsutil add-record nghia.internal mail MX "10 mail.nghia.internal."
pdnsutil add-record nghia.internal www CNAME app.nghia.internal.

# Xóa record
pdnsutil delete-rrset nghia.internal app A

# Sửa record — xóa rồi add lại
pdnsutil delete-rrset nghia.internal app A
pdnsutil add-record nghia.internal app A 192.168.100.99

# Bump serial sau khi sửa (quan trọng nếu có Slave)
pdnsutil increase-serial nghia.internal
```

### Test / verify


```bash
# Query thẳng vào Auth
dig @127.0.0.1 app.nghia.internal A

# Query qua Recursor (end-to-end test)
dig @192.168.100.6 app.nghia.internal A

# Check SOA / serial hiện tại
dig @127.0.0.1 nghia.internal SOA
```

### Lệnh hay dùng khi troubleshoot


```bash
# Xem log realtime
journalctl -fu pdns

# Reload zone không cần restart service
pdns_control reload

# Check backend DB connect được không
pdns_server --guardian=no --daemon=no --loglevel=9 2>&1 | head -20

# Flush toàn bộ cache của recursor
rec_control wipe-cache nghia.internal
rec_control wipe-cache vault.nghia.internal

# Hoặc flush all nếu lười
rec_control reload-zones
```

---

Workflow thông thường mỗi khi thêm/sửa record:

```
add-record / delete-rrset
        ↓
increase-serial
        ↓
dig @127.0.0.1 verify
```