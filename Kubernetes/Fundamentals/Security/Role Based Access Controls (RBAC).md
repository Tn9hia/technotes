## Overview
**Role-based access control (RBAC)** is a method of regulating access to computer or network resources based on the roles of individual users within your organization.

The RBAC API declares four kinds of Kubernetes object: _Role_, _ClusterRole_, _RoleBinding_ and _ClusterRoleBinding_.
- **Role**: sets permissions (`get`, `list`, `create`,...) on (`pods`, `services`,...) within a particular *namespace*
- **ClusterRole**: by contrast, is a non-namespaced resource.
- **RoleBinding**: set `Role` or `ClusterRole` within a particular *namespace
- **ClusterRoleBinding**: set `ClusterRole` within cluster

| **Role / ClusterRole** | **Assign RoleBinding** | **Assign ClusterRoleBinding** |
| ---------------------- | ---------------------- | ----------------------------- |
| **Role**               | ✅ Limit in namespace   | ❌ Cannot use                  |
| **ClusterRole**        | ✅ Limit in namespace   | ✅ Cluster permission          |

**Role verbs**
- `verb` options: get, list, watch, create, update, patch, delete, '\*\'

## Setup
To enable RBAC, start the [API server](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver) with the `--authorization-config` flag set to a file that includes the `RBAC` authorizer; for example:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthorizationConfiguration
authorizers:
  ...
  - type: RBAC
  ...
```

Or, start the [API server](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver) with the `--authorization-mode` flag set to a comma-separated list that includes `RBAC`; for example:
```shell
kube-apiserver --authorization-mode=...,RBAC --other-options --more-options
```



## Non-namespaced resource
| Resource                         | API Group                    | Description                                         |
| -------------------------------- | ---------------------------- | --------------------------------------------------- |
| nodes                            | core                         | Node trong cluster                                  |
| namespaces                       | core                         | Namespace của cluster                               |
| persistentvolumes (pv)           | core                         | Volume không bị ràng buộc namespace                 |
| componentstatuses                | core                         | Trạng thái các thành phần control plane             |
| endpointslices                   | discovery.k8s.io             | Endpoint phân mảnh, hỗ trợ load balancing tốt hơn   |
| clusterroles                     | rbac.authorization.k8s.io    | Định nghĩa quyền toàn cluster                       |
| clusterrolebindings              | rbac.authorization.k8s.io    | Gán ClusterRole cho user/SAs                        |
| certificatesigningrequests       | certificates.k8s.io          | Yêu cầu cấp TLS cert cho kubelet, mTLS...           |
| tokenreviews                     | authentication.k8s.io        | Kiểm tra token                                      |
| subjectaccessreviews             | authorization.k8s.io         | Kiểm tra quyền truy cập người khác                  |
| selfsubjectaccessreviews         | authorization.k8s.io         | Kiểm tra quyền truy cập của bản thân                |
| selfsubjectrulesreviews          | authorization.k8s.io         | Lấy danh sách rule của user hiện tại                |
| storageclasses                   | storage.k8s.io               | Class cho provisioner cấp PV                        |
| volumeattachments                | storage.k8s.io               | Gắn PV vào node                                     |
| csinodes                         | storage.k8s.io               | CSI node driver info                                |
| csidrivers                       | storage.k8s.io               | CSI driver metadata                                 |
| csistoragecapacities             | storage.k8s.io               | Thông tin dung lượng khả dụng của CSI theo topology |
| customresourcedefinitions (crds) | apiextensions.k8s.io         | Định nghĩa resource tuỳ chỉnh (CRD)                 |
| apiservices                      | apiregistration.k8s.io       | Mở rộng API server thông qua aggregation layer      |
| flowschemas                      | flowcontrol.apiserver.k8s.io | Cấu hình xử lý request ưu tiên                      |
| prioritylevelconfigurations      | flowcontrol.apiserver.k8s.io | Cấu hình mức độ ưu tiên cho request                 |
| runtimeclasses                   | node.k8s.io                  | Định nghĩa loại container runtime                   |
| validatingwebhookconfigurations  | admissionregistration.k8s.io | Webhook validate resource khi apply                 |
| mutatingwebhookconfigurations    | admissionregistration.k8s.io | Webhook thay đổi resource khi apply                 |

>[!note]
>Resource need to clarify --namespace => **namespaced**
>Resource does not need --namespace => **Cluster-level** (non-namespaced).

## CKA pro tips
### Check permission
```bash
# Xem mình có làm được gì không
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i '*' '*'  # check god mode

# Check cho user/SA khác (impersonate)
kubectl auth can-i create pods --as dev-user
kubectl auth can-i get secrets --as system:serviceaccount:default:my-sa
```
### Create Role/RoleBiding
```bash
# Tạo role imperative (ko cần yaml)
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods

# Cluster role luôn
kubectl create clusterrole deployment-manager \
  --verb=* \
  --resource=deployments

# Binding ngay lập tức
kubectl create rolebinding dev-binding \
  --role=pod-reader \
  --user=dev-user

# ClusterRoleBinding cho SA
kubectl create clusterrolebinding admin-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:my-sa
```
### Resource in cluster level
```
kubectl api-resources --namespaced=false
```
### Verbs

### Flow create Role/RoleBiding
```ascii
User/SA ──► RoleBinding ──► Role ──► Resources (namespace) (rules) (pods, etc)

User/SA ──► ClusterRoleBinding ──► ClusterRole ──► All Resources (cluster-wide) (rules) (nodes, PV, etc)
```
