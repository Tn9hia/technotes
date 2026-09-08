Expose an application running in your cluster behind a single outward-facing endpoint, even when the workload is split across multiple backends.
In Kubernetes, a Service is a method for **exposing a network application** that is running as **one or more Pods in your cluster.**

## Service Types

| Feature           | Node Port                                                        | ClusterIP                                                      | Loadbalancer                                                                |
| ----------------- | ---------------------------------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Description**   | Exposes the service on each Node's IP at a static port.          | Exposes the service on an internal IP in the cluster.          | Exposes the service externally using a cloud provider's load balancer.      |
| **Use Case**      | For external access with a specific port.                        | For internal cluster communication.                            | For external access with load balancing.                                    |
| **Access**        | Accessible from outside the cluster using `<NodeIP>:<NodePort>`. | Accessible only within the cluster using `<ClusterIP>:<Port>`. | Accessible from outside the cluster via a cloud provider's load balancer.   |
| **Port Range**    | 30000-32767                                                      | Cluster-assigned IP and port                                   | Cloud provider-specific ports                                               |
| **Ease of Setup** | Relatively easy, but requires manual port management.            | Easiest, automatically managed by Kubernetes.                  | Requires cloud provider integration, but provides seamless external access. |

^053bc7

### NodePort
**Term of port**:
- NodePort: Port of k8s node **(port from 30000-32767)**
- Port: Port of service
- TargetPort: Port of Pods
#### Sample
```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service
spec:
	type: NodePort
	port:
	-	targetPort: 80
		port: 80
		nodePort: 30008
```

>[!note] 
 Traffic load random to pod. You can access to any ip of nodes in the cluster with the ip address
### ClusterIP
This default Service type assigns an IP address from a pool of IP addresses that your cluster has reserved for that purpose.
Term of port:
- targetPort: port of back-end service
- port: port of service clusterip

```yaml
apiVersion: v1
kind: Service
metadata:
	name: back-end
spec:
	type: ClusterIP
	-   targetPort:
		port: 80
	selector:
		app: myapp
		type: back-end
```

### LoadBalancer
On cloud providers which support external load balancers, setting the `type` field to `LoadBalancer` provisions a load balancer for your Service. The actual creation of the load balancer happens asynchronously, and information about the provisioned balancer is published in the Service's `.status.loadBalancer`
> [!note]
> Type: loadBalancer only available on supported cloud provider

```yaml
apiVersion: v1
kind: Service
metadata:
	name: my-service
spec:
	selector:
	    app.kubernetes.io/name: MyApp
	ports:
    -   protocol: TCP
	    port: 80
		targetPort: 9376
	clusterIP: 10.0.171.239
	type: LoadBalancer
status:
	loadBalancer:
		ingress:
		-   ip: 192.0.2.127
```