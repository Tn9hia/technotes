
Kubernetes _volumes_ provide a way for containers in a pod to access and share data via the filesystem. There are different kinds of volume that you can use for different purposes, such as:

- populating a configuration file based on a ConfigMap or a Secret
- providing some temporary scratch space for a pod
- sharing a filesystem between two different containers in the same pod
- sharing a filesystem between two different pods (even if those Pods run on different nodes)
- durably storing data so that it stays available even if the Pod restarts or is replaced
- passing configuration information to an app running in a container, based on details of the Pod the container is in (for example: telling a  [[Multi Container Pod |sidecar container]] what namespace the Pod is running in)
- providing read-only access to data in a different container image

Data sharing can be between different local processes within a container, or between different containers, or between Pods.

| 🔍 **Criteria**               | 📦 **Volume**                                       | 🗄️ **Persistent Volume (PV)**                  | 📑 **Persistent Volume Claim (PVC)**                    |
| ----------------------------- | --------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------- |
| **Definition**                | Storage attached directly to a Pod                  | A storage resource managed by Kubernetes        | A user’s request for storage from a PV                  |
| **Lifecycle**                 | Tied to the Pod                                     | Independent of any Pod                          | Independent of any Pod                                  |
| **Data persistence**          | Data is lost when the Pod is deleted                | Data remains after the Pod is deleted           | Data remains after the Pod is deleted                   |
| **Creation method**           | Defined directly in the Pod spec                    | Created by an admin or via dynamic provisioning | Created by developers using YAML or referenced in a Pod |
| **Managed by**                | Developer                                           | Cluster Admin or StorageClass                   | Developer                                               |
| **Supported storage types**   | `emptyDir`, `hostPath`, `configMap`, `secret`, etc. | NFS, iSCSI, AWS EBS, GCE PD, Azure Disk, etc.   | Depends on the bound PV                                 |
| **Use case**                  | Temporary storage or internal sharing within a Pod  | Long-term storage, sharable across Pods         | Attaching storage to Pods via PVs                       |
| **How it’s mounted to a Pod** | Defined directly in `volumes` and `volumeMounts`    | Mounted through a PVC                           | Referenced in the Pod spec                              |
| **Reusability**               | No                                                  | Yes, if reclaim policy allows                   | Yes, as long as the PV is available                     |

## 📦 Mounting Volumes to a Pod (Notes)

### 🔹 General Pattern

To mount **any volume** into a Pod, you always need:

1. `volumes` → define the volume source
2. `volumeMounts` → mount it into a container

```yaml
volumes:
- name: my-volume
  <volume-type>: ...

containers:
- name: app
  volumeMounts:
  - name: my-volume
    mountPath: /data
```

---

## 🗂️ emptyDir

**What it is:**  
Temporary storage shared between containers in the same Pod.

**Lifecycle:**  
Created when Pod starts → deleted when Pod dies 💀

**Use cases:**

- Cache
- Temp files
- Sharing data between sidecars

```yaml
volumes:
- name: cache-volume
  emptyDir: {}

volumeMounts:
- name: cache-volume
  mountPath: /cache
```

> ⚠️ Data gone when Pod is deleted. Don’t cry later.

---

## 🖥️ hostPath

**What it is:**  
Mounts a directory/file from the **node’s filesystem**.

**Use cases:**

- Logs
- Node-level access
- Debugging (danger zone)

```yaml
volumes:
- name: host-volume
  hostPath:
    path: /var/log
    type: Directory

volumeMounts:
- name: host-volume
  mountPath: /host-logs
```

> 🚨 Tied to a specific node. Bad for HA. Use sparingly.

---

## 🔐 configMap

**What it is:**  
Injects config files or env-style data into Pods.

**Use cases:**

- App configs
- Feature flags

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config

volumeMounts:
- name: config-volume
  mountPath: /etc/config
```

> 🧠 Read-only by default. Don’t try to write to it.

---

## 🔑 secret

**What it is:**  
Mounts sensitive data (passwords, tokens, certs).

**Use cases:**

- DB credentials    
- TLS certs

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-secret

volumeMounts:
- name: secret-volume
  mountPath: /etc/secret
  readOnly: true
```

> 🔒 Always mark as `readOnly`. Security hygiene 101.

---

## 🗄️ PersistentVolumeClaim (PVC)

**What it is:**  
Mounts **persistent storage** via PV.

**Use cases:**

- Databases
- Stateful apps
- Anything you don’t want to lose

```yaml
volumes:
- name: data-volume
  persistentVolumeClaim:
    claimName: app-pvc

volumeMounts:
- name: data-volume
  mountPath: /data
```

> ✅ Survives Pod restarts. This is “real” storage.

---

## 🧠 Quick Rule of Thumb

|If you need…|Use|
|---|---|
|Temporary storage|`emptyDir`|
|Node filesystem access|`hostPath`|
|App config|`configMap`|
|Secrets|`secret`|
|Long-term data|`PVC`|
