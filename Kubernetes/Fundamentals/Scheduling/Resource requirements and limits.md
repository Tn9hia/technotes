When you specify a Pod, you can optionally specify how much of each resource a container needs. The most common resources to specify are CPU and memory (RAM); there are others.
**Example**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
      memory: "200Mi"
    requests:
      cpu: "1"
      memory: "100Mi"
  containers:
  - name: pod-resources-demo-ctr-1
    image: nginx
    resources:
      limits:
        cpu: "0.5"
        memory: "100Mi"
      requests:
        cpu: "0.5"
        memory: "50Mi"
  - name: pod-resources-demo-ctr-2
    image: fedora
    command:
    - sleep
    - inf 
```

| Tài nguyên | Khi vượt quá limit                            |
| ---------- | --------------------------------------------- |
| **CPU**    | Không bị giết, chỉ bị giới hạn tốc độ sử dụng |
| **Memory** | **Bị giết (OOMKilled)** ngay lập tức          |
- a `cpu` limit is a hard limit the kernel enforces  
- `memory` limits are enforced reactively

> [!note]
> **Guaranteed** → requests == limits (cả cpu lẫn memory) 
> **Burstable** → requests < limits 
> **BestEffort** → không set gì cả
## What is the difference between resource request and resource limit?
- `requests`: **Tài nguyên tối thiểu** mà container yêu cầu.
- `limits`: **Tài nguyên tối đa** mà container được phép sử dụng.
> [!note]
> By default, there is no limit when create resource. 

**To set default limit for pod**
limit range of cpu
```yaml
apiVersion: v1
kind: LimitRange
metadata:
	name: cpu-resource-constraint
spec:
	limit:
	-	default:
			cpu: 500m ==> limit
		defaultRequest:
			cpu: 500m ==> request
		max:
			cpu: "1" ==> limit
		min:
			cpu: 100m ==> request
		type: Container
```
limit range of ram
```yaml
apiVersion: v1
kind: LimitRange
metadata:
	name: cpu-resource-constraint
spec:
	limit:
	-	default:
			memory: 1Gi ==> limit
		defaultRequest:
			memory: 1Gi ==> request
		max:
			memory: 1Gi ==> limit
		min:
			memory: 500Mi ==> request
		type: Container
```
## Set host limit resource
To set host limit resource, use resource quotas
resource-quotas.yaml
```yaml
apiVerison: v1
kind: ResourceQuota
metadata:
	name: my-resource-quota
spec:
	hard:
		requests.cpu: 4
		requests.memory: 4Gi
		limits.cpu: 10
		limits.memory: 10Gi
```

