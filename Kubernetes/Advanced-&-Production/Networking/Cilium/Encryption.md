---
title: Cilium Encryption
tags:
  - cilium
  - encryption
  - wireguard
  - ipsec
  - security
date: 2026-04-30
---

# Cilium Encryption

Cilium cung cấp **transparent node-to-node encryption** — tất cả traffic giữa nodes được mã hóa tự động mà không cần thay đổi applications. Không cần Istio hay sidecar.

```
Node 1                              Node 2
┌─────────────────────────┐        ┌─────────────────────────┐
│  Pod A                  │        │  Pod B                  │
│  10.244.1.5             │        │  10.244.2.8             │
│      │                  │        │      │                  │
│  veth (lxc)             │        │  veth (lxc)             │
│      │                  │        │      │                  │
│  Cilium eBPF            │        │  Cilium eBPF            │
│      │ encrypt          │        │  decrypt │              │
│      ▼                  │        │          ▼              │
│  WireGuard/IPsec        │        │  WireGuard/IPsec        │
│  tunnel interface       │        │  tunnel interface       │
│      │                  │        │          │              │
└──────┼──────────────────┘        └──────────┼─────────────┘
       │                                       │
       └───────────── Network ─────────────────┘
              (encrypted, can't be read
               by anyone on the wire)
```

---

## WireGuard (Recommended)

WireGuard là modern VPN protocol — nhanh hơn IPsec, đơn giản hơn, và built-in Linux kernel 5.6+.

### Enable WireGuard

```bash
# Yêu cầu: Linux kernel >= 5.10 (với WireGuard module)
# Check kernel support
uname -r    # >= 5.6 (WireGuard mainlined), recommend 5.10 LTS

# Enable qua Helm
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard

# Verify
kubectl exec -n kube-system <cilium-pod> -- cilium status | grep Encryption
# Encryption:              WireGuard   [NodeEncryption: Disabled, cilium_wg0 (Pubkey: ...)]
```

### WireGuard Node Encryption

```yaml
# Helm values — encrypt node-to-pod traffic cũng (không chỉ pod-to-pod)
encryption:
  enabled: true
  type: wireguard
  nodeEncryption: true    # mặc định false — chỉ encrypt pod-to-pod traffic
                          # true = encrypt tất cả traffic từ node (kể cả host processes)
```

```bash
# Xem WireGuard interface trên node
kubectl exec -n kube-system <cilium-pod> -- ip link show cilium_wg0
kubectl exec -n kube-system <cilium-pod> -- wg show cilium_wg0

# WireGuard stats (bytes encrypted/decrypted)
kubectl exec -n kube-system <cilium-pod> -- wg show cilium_wg0 transfer

# Public keys của tất cả nodes
kubectl get ciliumnodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.network\.cilium\.io/wg-pub-key}{"\n"}{end}'
```

### WireGuard Key Rotation

```bash
# Keys được tự động rotate khi Cilium agent restart
# Không cần manual key rotation — WireGuard dùng Curve25519

# Force rotate: restart cilium-agent (graceful)
kubectl rollout restart daemonset/cilium -n kube-system
```

---

## IPsec

IPsec là alternative nếu WireGuard không available (older kernels, compliance requirements).

### Enable IPsec

```bash
# Tạo pre-shared key
kubectl create secret generic cilium-ipsec-keys \
  --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128" \
  -n kube-system

# Enable IPsec
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=ipsec

# Verify
kubectl exec -n kube-system <cilium-pod> -- cilium status | grep Encryption
# Encryption:    IPsec
```

### IPsec Key Rotation

```bash
# IPsec cần manual key rotation (khác WireGuard)
# 1. Generate new key
NEW_KEY="4 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"

# 2. Patch secret với new key (giữ old key trong cùng secret để graceful rotation)
kubectl patch secret cilium-ipsec-keys -n kube-system \
  --type='json' \
  -p='[{"op": "replace", "path": "/data/keys", "value": "'$(echo -n "3 rfc4106(gcm(aes)) <old-key> 128\n4 rfc4106(gcm(aes)) <new-key> 128" | base64)'"}]'

# 3. Verify rotation (cilium-agent tự detect và rotate)
kubectl exec -n kube-system <cilium-pod> -- cilium encrypt status
```

---

## WireGuard vs IPsec

| Feature | WireGuard | IPsec |
|---------|-----------|-------|
| Kernel requirement | >= 5.6 (recommend 5.10) | >= 4.9 |
| Performance | Cao (ChaCha20-Poly1305) | Tốt (AES-GCM hardware offload) |
| Key management | Automatic (per-agent) | Manual rotation |
| Cipher | ChaCha20-Poly1305 | AES-128-GCM (rfc4106) |
| Complexity | Đơn giản | Phức tạp hơn |
| Compliance | Ít được audit hơn | FIPS 140-2 compliant variants |
| Debug | `wg show` | `ip xfrm state` |

---

## Verify Encryption đang hoạt động

```bash
# Method 1: Cilium status
kubectl exec -n kube-system <cilium-pod> -- cilium encrypt status

# Method 2: tcpdump trên wire (phải thấy encrypted traffic)
# SSH vào một node, bắt packet giữa 2 nodes
tcpdump -i eth0 -n host <node2-ip>
# WireGuard: thấy UDP port 51871 (Cilium WireGuard port)
# IPsec: thấy ESP packets (protocol 50)
# Không thấy plaintext HTTP/TCP content = encryption đang work

# Method 3: Hubble flow (xem encryption status)
hubble observe --namespace production -o json | \
  jq 'select(.source.pod_name != null) | .traffic_direction'

# Method 4: Check WireGuard handshake
kubectl exec -n kube-system <cilium-pod> -- wg show cilium_wg0
# latest handshake: X seconds/minutes ago — nếu có handshake = peers connected
```

---

## Encryption và Performance

```bash
# WireGuard overhead: ~3-5% CPU cho line-rate 10Gbps
# IPsec với AES-NI hardware: ~1-2% overhead

# Kiểm tra AES-NI support
grep -m1 aes /proc/cpuinfo
# → "aes" trong flags = hardware AES support → IPsec nhanh hơn

# WireGuard throughput test
# Trên node, install iperf3 và test giữa pods trên different nodes
kubectl run iperf-server --image=networkstatic/iperf3 -- iperf3 -s
kubectl run iperf-client --image=networkstatic/iperf3 -- iperf3 -c <server-ip> -t 30 -P 4
```

---

## Gotchas

- **WireGuard và kernel version**: Ubuntu 20.04 LTS kernel 5.4 không có WireGuard mainlined — cần install `wireguard-tools` package hoặc upgrade kernel. Ubuntu 22.04 LTS kernel 5.15 = sẵn sàng.
- **Encryption và Hubble**: WireGuard encrypt trước khi packet ra khỏi node. Hubble eBPF programs chạy trước encryption point → Hubble vẫn thấy plaintext flow data. Đây là feature — observability không bị ảnh hưởng bởi encryption.
- **NodeEncryption và host traffic**: `nodeEncryption: true` encrypt traffic từ node processes (kubelet, system daemons). Có thể ảnh hưởng đến control plane communication nếu kube-apiserver không trên same node. Test kỹ trước khi enable.
- **IPsec và GRO/GSO**: IPsec không tương thích với Generic Receive Offload (GRO) và Generic Segmentation Offload (GSO) trong một số kernel configs → throughput drop. WireGuard tương thích tốt hơn với offloading.
- **Key rotation window**: Khi rotate IPsec keys, có window ngắn (~giây) khi nodes dùng different key versions → packets drop. Giữ old key trong secret trong khi rotate để cilium-agent handle gracefully.
