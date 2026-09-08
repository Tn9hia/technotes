# Fortinet Network Debug Cheatsheet 🔧

## 🔍 Basic Info / Show Commands

```txt
admin/v1cc@hcm
```


```bash
# Show tất cả interface + IP
get system interface

# Show 1 interface cụ thể
get system interface physical | grep -A 10 "== [port1]"

# Show routing table
get router info routing-table all

# Show default gateway
get router info routing-table static

# Show ARP table
get system arp

# Show DNS config
get system dns

# Show hostname, firmware, serial
get system status
```

---

## 🌐 IP / Interface Management

```bash
# Set IP cho interface (config mode)
config system interface
  edit port1
    set ip 192.168.1.1 255.255.255.0
    set allowaccess ping https ssh
  next
end

# Bring up/down interface
config system interface
  edit port1
    set status up     # hoặc down
  next
end
```

---

## 📡 Ping / Traceroute / DNS

```bash
# Ping cơ bản
execute ping 8.8.8.8

# Ping với options (source IP, count, size)
execute ping-options source 192.168.1.1
execute ping-options repeat-count 10
execute ping-options data-size 1400
execute ping 8.8.8.8

# Reset ping options về default
execute ping-options repeat-count 5
execute ping-options source auto

# Traceroute
execute traceroute 8.8.8.8

# NSLookup / DNS resolve
execute nslookup google.com
```

---

## 🔥 Debug / Packet Capture (quan trọng nhất)

```bash
# Diagnose network interface stats
diagnose netlink interface list
diagnose netlink interface list port1

# Check interface counter (drop, error,...)
diagnose netlink interface clear port1   # reset counter
diagnose netlink interface list port1    # xem lại

# Packet sniffer — killer command
diagnose sniffer packet port1 'host 8.8.8.8' 4 100
#                              ^filter       ^verbosity ^count
# Verbosity: 1=header, 4=header+data, 6=full hex

# Sniffer any interface
diagnose sniffer packet any 'port 80' 4 50

# Debug flow — trace packet đi qua policy nào
diagnose debug reset
diagnose debug flow filter addr 192.168.1.100
diagnose debug flow show function-name enable
diagnose debug flow trace start 100
diagnose debug enable
# --- chạy traffic, xem log ---
diagnose debug disable
diagnose debug flow trace stop
```

---

## 🛣️ Routing Debug

```bash
# Show full routing table
get router info routing-table all

# Show routing table theo destination
get router info routing-table details 8.8.8.8

# OSPF / BGP neighbor (nếu có dynamic routing)
get router info ospf neighbor
get router info bgp neighbors

# Check policy route
diagnose firewall proute list
```

---

## 📊 Session / Connection Table

```bash
# Show active session
diagnose sys session list

# Filter session theo IP
diagnose sys session filter src 192.168.1.100
diagnose sys session list

# Clear filter
diagnose sys session filter clear

# Session stats
diagnose sys session stat
```

---

## ⚡ Quick Debug Flow — Kiểu "traffic không đi được"

```
1. get router info routing-table all          → có route chưa?
2. execute ping <dst> (source đúng interface) → L3 ok chưa?
3. diagnose sniffer packet any 'host <dst>'   → packet có ra không?
4. diagnose debug flow filter addr <src>      → hit policy nào?
5. diagnose sys session list (filter by src)  → session được tạo chưa?
```

---

> 💡 **Pro tip:** Sau mỗi debug session nhớ `diagnose debug disable` + `diagnose debug reset` — để quên là FortiGate log spam, CPU spike, không ai muốn điều đó lúc 2 giờ sáng đâu.