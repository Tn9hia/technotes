Make your HTTP (or HTTPS) network service available using a protocol-aware configuration mechanism, that understands web concepts like URIs, hostnames, paths, and more. The Ingress concept lets you map traffic to different backends based on rules you define via the Kubernetes API.
Ingress may provide load balancing, SSL termination and name-based virtual hosting.

> [!note]
> Ingress is frozen. New features are being added to the [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/).

## Grossary
For clarity, this guide defines the following terms:
- **Node**: A worker machine in Kubernetes, part of a cluster.
- **Cluster:** A set of Nodes that run containerized applications managed by Kubernetes. For this example, and in most common Kubernetes deployments, nodes in the cluster are not part of the public internet.
- Edge router: A router that enforces the firewall policy for your cluster. This could be a gateway managed by a cloud provider or a physical piece of hardware.
- Cluster network: A set of links, logical or physical, that facilitate communication within a cluster according to the Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/).
- Service: A Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) that identifies a set of Pods using [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels) selectors. Unless mentioned otherwise, Services are assumed to have virtual IPs only routable within the cluster network.

## What is Ingress?
**Ingress** exposes HTTP and HTTPS routes from outside the cluster to [[Services]] within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

Here is a simple example where an Ingress sends all its traffic to one Service:

![ingress-diagram](https://kubernetes.io/docs/images/ingress.svg)

An Ingress may be configured to give Services externally-reachable URLs, load balance traffic, terminate SSL / TLS, and offer name-based virtual hosting. An Ingress controller is responsible for fulfilling the Ingress, usually with a load balancer, though it may also configure your edge router or additional frontends to help handle the traffic.

An Ingress does not expose arbitrary ports or protocols. Exposing services other than HTTP and HTTPS to the internet typically uses a service of type Service.Type=[[Services#NodePort|NodePort]]  or Service.Type=[[Services#LoadBalancer|LoadBalancer]].

## Prerequisites
You must have an **Ingress controller** to satisfy an Ingress. Only creating an Ingress resource has no effect.

You may need to deploy an Ingress controller such as ingress-nginx. You can choose from a number of Ingress controllers.

Ideally, all Ingress controllers should fit the reference specification. In reality, the various Ingress controllers operate slightly differently.
## Reference
- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.github.io/ingress-nginx/examples/rewrite/