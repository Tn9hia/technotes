**Horizontal auto scaling**: Adding more pod to the system
**Vertical auto scaling**: Increase the size of pod (RAM/CPU)

**Manual**: adding node, scale pod, increase size of pod
**Automation**: 
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
> [!note]
>Use Metrics server to monitor resouce
## Horizontal Pod Autoscaler
*Imperative*
```bash
kubectl autoscale deployment my-app --cpu-percentage50 --min=1 --max=10
```

*Declerative*
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: example-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: example-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## Vertical Pod Autoscaler (VPA)
> [!note]
> VPA does not come with build-in deployment => you have to deploy VPA (from github)

VPA contains several components:
- VPA recommender: Collect metrics of pod
- VPA updater: Collect information from VPA recommender and remove pod if need
- VPA Admision Controller: collect information and update deployment

| Mode       | Description                                                                                                                                                                               |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Auto`     | Currently `Recreate`. This might change to in-place updates in the future.                                                                                                                |
| `Recreate` | The VPA assigns resource requests on pod creation as well as updates them on existing pods by evicting them when the requested resources differ significantly from the new recommendation |
| `Initial`  | The VPA only assigns resource requests on pod creation and never changes them later.                                                                                                      |
| `Off`      | The VPA does not automatically change the resource requirements of the pods. The recommendations are calculated and can be inspected in the VPA object.                                   |

```yaml
apiversion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler metadata:
name: my-app-vpa
spec:
	targetRef:
		apiVersion: apps/v1
		kind: Deployment
		name: my-app
	updatePolicy:
		updateMode: "Auto"
	resourcePolicy:
		containerPolicies:
		- containerName: "my-app" minAlLowed:
		   cpu: "250m"
		   maxAlLowed :
		   cpu: "2"
		   controlledResources: ["cpu"]

```

## In-place resize of pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my=app
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
    containers:
    - name: my-ap
      image: nginx
      resizePolicy:
      - resourceName: cpu
        restartPolicy: NotRequired
      - resourceName: memory
        restartPolicy: RestartContainer
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

### Limitation
- only CPU and RAM can be changed
- pod QoS cannot be changed
- init and emphemeral container cannot be change
- resource request and limit cannot be remove
- cannot scale down