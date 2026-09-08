## Requirement

- 2 VM
- Controlplane must have
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlasyisy
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

```bash
sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
### Install container runtime
#### Installing containerd
Source: https://github.com/containerd/containerd/blob/main/docs/getting-started.md

Download containerd
```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.23/containerd-1.7.23-linux-amd64.tar.gz
```
 
```bash
tar Cxzvf /usr/local containerd-1.7.23-linux-amd64.tar.gz

wget -P /usr/lib/systemd/system/ https://raw.githubusercontent.com/containerd/containerd/main/containerd.service 

systemctl daemon-reload
systemctl enable --now containerd
```

#### Installing runc
Source: https://github.com/opencontainers/runc/releases

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.15/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### Installing CNI plugins
Source: https://github.com/containernetworking/plugins/releases

```bash
mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz

tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.0.tgz
```

#### Configuring the `systemd` cgroup driver 
```bash
mkdir /etc/containerd/ 
touch /etc/containerd/config.toml
sudo containerd config default > /etc/containerd/config.toml
```

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

### Install kubeadm, kubelet và kubectl


### Create a k8s cluster
#### Create Control Plane
```bash
sudo kubeadm init --pod-network-cidr=100.64.0.0/16 --service-cidr=10.96.0.0/12
```
To start using your cluster, you need to run the following as a regular user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


Alternatively, if you are the root user, you can run:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Setup kubeconfig
```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc
source ~/.bashrc
```

### Install CNI
#### Calico
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml
```

Monitor install status
```
watch kubectl get tigerastatus
```


Monitor traffic

```
kubectl port-forward -n calico-system service/whisker 8081:8081
```

#### Cilium 
```bash
cilium install \
  --version 1.18.4 \
  --helm-set kubeProxyReplacement=true \
  --helm-set ipam.operator.clusterPoolIPv4PodCIDRList="{100.64.0.0/16}" \
  --helm-set ipam.operator.clusterPoolIPv4MaskSize=24

```

## Additional component
### Gateway API
- Install Gateway API CRDs
```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

- Verify CRDs
```bash
kubectl get crd | grep gateway
```
### ETCD Client
```shell
ETCD_VER=v3.5.12    # thay bằng version của bạn

wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xvf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo cp etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo cp etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/

```


### Vertical Pod Autoscalling
```shell
git clone ;https://github.com/kubernetes/autoscaler.git

# Enable VPA
./autoscaler/vertical-pod-autoscaler/hack/vpa-up.sh
```
### Storage
#### Longhorn
- Create folder for longhorn
```bash
sudo mkdir -p /var/lib/longhorn
sudo chmod 700 /var/lib/longhorn

# Tạo partition
sudo parted /dev/sdb mklabel gpt
# Format XFS
sudo apt install xfsprogs
sudo parted /dev/sdb mkpart primary xfs 0% 100%
sudo mkfs.xfs -f /dev/sdb1
# Tạo mount point
sudo mount /dev/sdb1 /var/lib/longhorn
df -h | grep longhorn
```

- Persistent mount point after reboot
```bash
# Get UUID 
sudo blkid /dev/sdb1

# Add vào /etc/fstab
echo "UUID=xxxx-xxxx /var/lib/longhorn-storage xfs defaults,noatime 0 0" | sudo tee -a /etc/fstab

# Test fstab
sudo umount /var/lib/longhorn 
sudo mount -a df -h | grep longhorn
```
- Install necessary package for Longhorn (Ubuntu 24.04)
```bash
sudo apt update
sudo apt install -y \
  open-iscsi \
  nfs-common \
  cryptsetup \
  util-linux \
  dmsetup
```
- Enable and start iSCSI service
```bash
sudo systemctl enable --now iscsid
systemctl status iscsid
```
- Load kernel modules (ngay & sau reboot)
```bash
sudo modprobe iscsi_tcp
sudo modprobe dm_crypt
sudo modprobe nfs

lsmod | egrep 'iscsi|dm_crypt' # Check module is loaded

# Auto load after reboot
echo -e "iscsi_tcp\ndm_crypt" | sudo tee /etc/modules-load.d/longhorn.conf 
echo nfs | sudo tee /etc/modules-load.d/nfs.conf
```
- Verify
```bash
iscsiadm -m node
cryptsetup --version
```
- Disable multipathd.service
```bash
sudo systemctl stop multipathd
sudo systemctl disable multipathd
sudo systemctl mask multipathd

sudo systemctl stop multipathd.socket
sudo systemctl disable multipathd.socket
sudo systemctl mask multipathd.socket

systemctl status multipathd
```
- Longhorn preflight check
```shell
curl -sSfL -o longhornctl https://github.com/longhorn/cli/releases/download/v1.10.1/longhornctl-linux-amd64

chmod +x longhornctl
./longhornctl check preflight --kubeconfig=<kube-config-file>
```
#### OpenEBS
#### Ceph


### Ingress
#### Traefik
```bash
# Create traefik with helm and enable necessary commponent
# Create gatewayclass => gateway => httproute/grpcroute
```
####  Envoy

### Monitor
#### Grafana/Prometheus stack
#### Grafana Loki

#### Metric Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- Change metrics server deployment
```shell
kubectl -n kube-system edit deploy metric-servers
```
- Change the args tag
```yaml
    spec:
      containers:
      - args:
        - --secure-port=4443
        - --cert-dir=/tmp
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-insecure-tls=true
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
```
### Cert Manager
### Backup
### Private Registry

### Backup 
#### Velory