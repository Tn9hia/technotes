ETCD is a distributed reliable key-value storage that is simple, secure and fast
Run on port 2379 
Default version is 3, some may use version 2

- To store value:
```
./etcdctl set key1 value1
./etcdctl get key1 value1
```
- To interact with etcd
```
kubectl exec -it etcd-node1 -n kube-system -- etcdctl get / --prefix -keys-only 
```
- To Install etcd
	- From source
	- From kubeadm
