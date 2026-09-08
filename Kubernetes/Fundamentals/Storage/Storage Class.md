A **StorageClass** provides a way for administrators to describe the _classes_ of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent.

The Kubernetes concept of a storage class is similar to “profiles” in some other storage system designs.

Each StorageClass contains the fields `provisioner`, `parameters`, and `reclaimPolicy`, which are used when a PersistentVolume belonging to the class needs to be dynamically provisioned to satisfy a PersistentVolumeClaim (PVC).

## Provisioning
To enable dynamic storage provisioning based on storage class, the cluster administrator needs to enable the `DefaultStorageClass` [[Admission control]] on the API server.

### Static
A cluster administrator creates a number of PVs. They carry the details of the real storage, which is available for use by cluster users. They exist in the Kubernetes API and are available for consumption.

### Dynamic
When none of the static PVs the administrator created match a user's PersistentVolumeClaim, the cluster may try to dynamically provision a volume specially for the PVC. This provisioning is based on StorageClasses: the PVC must request a *storage class* and the administrator must have created and configured that class for dynamic provisioning to occur.

## Reclaim policy
PersistentVolumes that are dynamically created by a StorageClass will have the reclaim policy specified in the `reclaimPolicy` field of the class, which can be either `Delete` or `Retain`. If no `reclaimPolicy` is specified when a StorageClass object is created, it will default to `Delete`.

| Reclaim Policy              | Khi PVC bị xóa  | Số phận dữ liệu   | PV sau đó    | Dùng khi nào                    | Nguy cơ thường gặp           |
| --------------------------- | --------------- | ----------------- | ------------ | ------------------------------- | ---------------------------- |
| **Delete** (default)        | PV bị xóa theo  | ❌ Mất sạch        | ❌ Không còn  | Dev / Test / workload stateless | Xóa nhầm PVC → bay luôn data |
| **Retain**                  | PV giữ nguyên   | ✅ Còn data        | ⏸ Released   | DB, data quan trọng, cần backup | Quên cleanup → rác storage   |
| **Recycle** ⚠️ (deprecated) | Xóa data cơ bản | ❌ (không an toàn) | ♻️ Available | ❌ Không nên dùng                | Không xóa sạch, đã bị loại   |
`Retain` → **phải xử lý thủ công** nếu muốn dùng lại PV:
- Xóa `claimRef`
- Gán lại PVC mới
- Hoặc xoá PV + backend disk

## Volume biding mode
| volumeBindingMode          | Khi nào tạo PV        | Phù hợp với             | Ưu điểm               | Nhược điểm                | Khi nên dùng            |
| -------------------------- | --------------------- | ----------------------- | --------------------- | ------------------------- | ----------------------- |
| **Immediate** (default)    | Ngay khi tạo PVC      | Cluster nhỏ, 1 zone     | Tạo nhanh, đơn giản   | Dễ lệch zone/node         | Dev, test, lab          |
| **WaitForFirstConsumer** ⭐ | Khi Pod được schedule | Multi-node / multi-zone | PV tạo đúng node/zone | Chờ Pod nên chậm hơn chút | Production, StatefulSet |

## Access Mode
| Access Mode                 | Ý nghĩa                | Nhiều Pod cùng lúc? | Backend phổ biến             | Dùng cho                   |
| --------------------------- | ---------------------- | ------------------- | ---------------------------- | -------------------------- |
| **ReadWriteOnce (RWO)**     | 1 node được read/write | ❌ (1 node)          | EBS, vSphere block, Local PV | DB, app stateful           |
| **ReadOnlyMany (ROX)**      | Nhiều node đọc         | ✅ (read-only)       | NFS, CephFS                  | Data share, static content |
| **ReadWriteMany (RWX)**     | Nhiều node read/write  | ✅                   | NFS, CephFS, GlusterFS       | Shared storage, web app    |
| **ReadWriteOncePod (RWOP)** | 1 pod duy nhất         | ❌ (1 pod)           | CSI mới                      | DB cần isolation           |
**Note**: 
- AccessMode **không phải bạn muốn là được**
- Nó **phụ thuộc backend storage**
- PVC khai báo RWX nhưng backend không support → PVC **pending mãi**

| Backend     | RWO | RWX       | Ghi chú                |
| ----------- | --- | --------- | ---------------------- |
| AWS EBS     | ✅   | ❌         | Block storage          |
| vSphere CNS | ✅   | ❌         | Block                  |
| Ceph RBD    | ✅   | ❌         | Block                  |
| CephFS      | ❌   | ✅         | File system            |
| NFS         | ❌   | ✅         | Đơn giản nhưng latency |
| Longhorn    | ✅   | ⚠️ (beta) | Tuỳ config             |
