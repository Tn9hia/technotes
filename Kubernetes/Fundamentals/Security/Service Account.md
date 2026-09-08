A `service account` is a type of non-human account that, in Kubernetes, provides a distinct identity in a Kubernetes cluster. Application Pods, system components, and entities inside and outside the cluster can use a specific ServiceAccount's credentials to identify as that ServiceAccount. This identity is useful in various situations, including authenticating to the API server or implementing identity-based security policies.

| Description    | ServiceAccount                                                                                                                                    | User or group                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Location       | Kubernetes API (ServiceAccount object)                                                                                                            | External                                                           |
| Access control | Kubernetes RBAC or other [authorization mechanisms](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules) | Kubernetes RBAC or other identity and access management mechanisms |
| Intended use   | Workloads, automation                                                                                                                             | People                                                             |
## How to use service accounts

To use a Kubernetes service account, you do the following:

1. Create a ServiceAccount object using a Kubernetes client like `kubectl` or a manifest that defines the object.
2. Grant permissions to the ServiceAccount object using an authorization mechanism such as [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).
3. Assign the ServiceAccount object to Pods during Pod creation.
	- If you're using the identity from an external service, [retrieve the ServiceAccount token](https://kubernetes.io/docs/concepts/security/service-accounts/#get-a-token) and use it from that service instead.

To provide API credential for pod:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
```


if `automountServiceAccountToken: true` => token auto mount to pod
if `automountServiceAccountToken: false` => token not mount to pod. Need to use `external identity provider` like: volume projected, 


> [!warning]
> Location of token: **/var/run/secrets/kubernetes.io/serviceaccount/token**

