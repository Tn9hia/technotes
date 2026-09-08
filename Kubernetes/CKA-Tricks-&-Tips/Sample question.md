
**Question**
Cluster có 3 node:
- `worker1`
- `worker2`
- `worker3`
Yêu cầu:
1. Thêm label `disk=ssd` vào node `worker2`.
2. Tạo một Pod tên `busybox-aff` chạy image `busybox`, command: `sleep 3600`.
3. Pod phải **chỉ được phép schedule lên node có label `disk=ssd`**.
4. Pod phải nằm trong namespace `prod`.
**Output yêu cầu:**  
→ Toàn bộ các lệnh bạn sẽ dùng (kubectl + YAML nếu cần).  
→ Nếu dùng YAML thì chỉ cần Pod manifest, không cần extra file.
**Answer**
```shel
k get nodes --show-labels
k label nodes worker2 disk=ssd
k get ns | grep prod
k create ns prod
k run busybox-aff --image busybox --dry-run=client -n prod -o yaml --command sleep 3600 > busybox-aff.yaml 
vi busybox-aff.yaml 
k create -f busybox-aff.yaml
k get pod -n prod |grep busybox-aff
```

Pod manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox-aff
  name: busybox-aff
  namespace: prod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
  containers:
  - command:
    - sleep
    - "3600"
    image: busybox
    name: busybox-aff
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```