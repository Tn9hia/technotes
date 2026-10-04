# K8s as a Service — Overview (MOC)
Tags: #k8s-provider #moc
Last updated: 2026-10-03

Ghi chú tổng quan (Map of Content) cho series xây dựng Kubernetes-as-a-Service trên **KVM**, dùng **CAPI** (cluster lifecycle) + **Talos** (node OS), với tầng IaaS là **CloudStack** hoặc **OpenStack**. Mỗi mục dưới đây là 1 group kiến thức, link vào root note của group đó.

## Kiến trúc tổng quan

```
            ┌───────────────────────────────┐
            │        Management Plane          │  (CAPI controllers chạy ở đây, GitOps)
            │         [[Management-Plane]]       │
            └──────────────┬────────────────┘
                            │ manages
             ┌──────────────┴───────────────┐
             │         CAPI (core)           │   → [[CAPI]]
             │  Cluster / Machine / ClusterClass
             └──────┬───────────────┬────────┘
       Infrastructure Provider   Bootstrap + Control-Plane Provider
         [[CAPC]] / [[CAPO]]              [[Talos]] (CABPT/CACPPT)
                 │                               │
         ┌───────┴────────┐              ┌───────┴────────┐
         │ CloudStack /    │              │ Node image       │
         │ OpenStack (KVM) │              │ [[Image-Pipeline]] │
         └───────┬────────┘              └────────────────┘
                 │
     ┌───────────┴─────────────────┐
     │      Tenant Cluster(s)        │
     │  - [[Networking]] (CNI/CCM)   │
     │  - [[Storage]] (CSI)          │
     │  - [[Multi-Tenancy]] (model)  │
     └───────────────────────────────┘

Cross-cutting (áp dụng toàn hệ thống, không riêng 1 layer):
  [[Fleet-Ops]]         — vận hành nhiều cluster (upgrade/backup/day-2)
  [[Provider-Security]] — hardening credential + secrets toàn provider
```

## Cluster Lifecycle
- [[CAPI]] — core framework: Cluster/Machine/MachineDeployment/ClusterClass, provider contract.
  - [[capi--clusterclass-topology]] — template hoá flavor cluster qua ClusterClass + patches.
  - [[capi--clusterctl]] — CLI quản lý lifecycle CAPI/provider (init/move/upgrade).

## Infra Provider (IaaS-specific)
- [[CAPC]] — Cluster API Provider CloudStack.
- [[CAPO]] — Cluster API Provider OpenStack.

## Node OS / Bootstrap
- [[Talos]] — Talos Linux: immutable, API-driven, no SSH.
  - [[talos--capi-integration]] — cách CABPT/CACPPT sinh machine config + quản lý PKI qua CAPI.

## Image Pipeline
- [[Image-Pipeline]] — build Talos image (Image Factory) + publish sang CloudStack Template/OpenStack Glance.

## Networking
- [[Networking]] — CNI, Cloud Controller Manager (CCM), LB/Ingress cho tenant cluster.

## Storage
- [[Storage]] — CSI driver cho CloudStack/OpenStack, lưu ý khi chạy trên node Talos.

## Multi-Tenancy
- [[Multi-Tenancy]] — dedicated cluster vs hosted control-plane, ClusterClass flavor, quota IaaS.

## Fleet Ops
- [[Fleet-Ops]] — GitOps quản lý CAPI CR, upgrade theo fleet, MachineHealthCheck/autoscaler, backup/DR.

## Provider Security
- [[Provider-Security]] — hardening credential CAPC/CAPO, secrets Talos, RBAC/network policy per-tenant.

## Management Plane
- [[Management-Plane]] — vận hành chính cluster chạy CAPI controllers: HA, pivot, backup, observability fleet.

---

> Lưu ý: nhiều backlink ở trên có thể còn "dangling" (file chưa tồn tại) tại thời điểm viết — đó là chủ ý, đánh dấu phần sẽ viết tiếp. Mỗi root note đều có banner ⚠️ riêng liệt kê các fact cần tự verify lại trước khi áp dụng vào hệ thống thật.
