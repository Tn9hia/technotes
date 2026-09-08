Assign pod to spectify node
**Labeling nodes**
```shell
kubectl label nodes <your-node-name> disktype=ssd
```
**Check labels of nodes**
```shell
kubectl get nodes --show-labels
```
## Schedule a Pod using required node affinity
This manifest describes a Pod that has a `requiredDuringSchedulingIgnoredDuringExecution` node affinity,`disktype: ssd`. This means that the pod will get scheduled only on a node that has a `disktype=ssd` label.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```
## Schedule a Pod using preferred node affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

## Type of Pod affinity and anti-affinity
1. `requiredDuringSchedulingIgnoredDuringExecution`
- **Bắt buộc khi lên lịch (`requiredDuringScheduling`)**:
    - Pod **chỉ có thể được lên lịch** trên các node thỏa mãn điều kiện affinity.
    - Nếu không có node nào phù hợp, **Pod sẽ không thể chạy**.
- **Bỏ qua khi đang chạy (`IgnoredDuringExecution`)**:
    - Sau khi Pod đã được lên lịch, nếu node thay đổi và không còn phù hợp nữa, Pod vẫn tiếp tục chạy (không bị di chuyển hay xóa).
2. `preferredDuringSchedulingIgnoredDuringExecution`
- **Ưu tiên khi lên lịch (`preferredDuringScheduling`)**:
    - Pod **cố gắng** chạy trên các node phù hợp nhất, nhưng **không bắt buộc**.
    - Nếu không có node nào phù hợp, Kubernetes vẫn có thể lên lịch Pod trên các node khác.
- **Bỏ qua khi đang chạy (`IgnoredDuringExecution`)**:
    - Giống với `requiredDuringSchedulingIgnoredDuringExecution`, nếu node thay đổi sau khi Pod được lên lịch, Pod vẫn tiếp tục chạy.
3. `requiredDuringSchedulingRequiredDuringExecution`
 - **Bắt buộc khi lên lịch (`requiredDuringScheduling`)**:
    - Pod **chỉ có thể được lên lịch** trên các node thỏa mãn điều kiện affinity.
    - Nếu không có node nào phù hợp, **Pod sẽ không thể chạy**.
- **Cũng bắt buộc khi Pod đang chạy.(`requiredDuringExecution`)**
	- Nếu node thay đổi và không còn thỏa mãn điều kiện **Pod sẽ bị evict (bị xóa và tạo lại trên node khác phù hợp)**.

## Compare

|                   Loại Affinity                   |                                Khi Lên Lịch (Scheduling)                                 |                       Khi Đang Chạy (Execution)                       |
| :-----------------------------------------------: | :--------------------------------------------------------------------------------------: | :-------------------------------------------------------------------: |
| `requiredDuringSchedulingIgnoredDuringExecution`  |           **Bắt buộc**, nếu không có node phù hợp thì Pod **không chạy được**            |            **Bỏ qua**, nếu node thay đổi thì Pod vẫn chạy             |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Ưu tiên**, Pod **cố gắng** chạy trên node phù hợp nhưng vẫn có thể chạy trên node khác |            **Bỏ qua**, nếu node thay đổi thì Pod vẫn chạy             |
| `requiredDuringSchedulingRequiredDuringExecution` |           **Bắt buộc**, nếu không có node phù hợp thì Pod **không chạy được**            | **Cũng bắt buộc**, nếu node thay đổi thì Pod **sẽ bị xóa và tạo lại** |





