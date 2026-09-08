## Backup configuration resource
For resource created by declearative:
- Using github or backup configuration file
For resource created by imperative:
- Using tools like Veloro to backup all resource
> [!node]
> It is recommend to use declearative way to create resource

## Backup ETCD
- Use **etcdctl** to create a snapshot

```shell
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

## Restore ETCD
- Stop kube-apiserver

```shell
etcdutl --data-dir <data-dir-location> snapshot restore snapshot.db
```

- Restore snapshot to a new directory
- Update manifest of etcd to a new location