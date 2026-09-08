A security context defines privilege and access control settings for a Pod or Container. Security context settings include, but are not limited to:
- Discretionary Access Control: Permission to access an object, like a file, is based on [user ID (UID) and group ID (GID)](https://wiki.archlinux.org/index.php/users_and_groups).
- [Security Enhanced Linux (SELinux)](https://en.wikipedia.org/wiki/Security-Enhanced_Linux): Objects are assigned security labels.
- Running as privileged or unprivileged.
- [Linux Capabilities](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/): Give a process some privileges, but not all the privileges of the root user.
- [AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/): Use program profiles to restrict the capabilities of individual programs.
- [Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/): Filter a process's system calls.
- `allowPrivilegeEscalation`: Controls whether a process can gain more privileges than its parent process. This bool directly controls whether the [`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt) flag gets set on the container process. `allowPrivilegeEscalation` is always true when the container:
    - is run as privileged, or
    - has `CAP_SYS_ADMIN`
- `readOnlyRootFilesystem`: Mounts the container's root filesystem as read-only.

Ex:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false

```