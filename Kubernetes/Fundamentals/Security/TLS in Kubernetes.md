Three main components that use ssl certificate to authenticate
-  Kube-API
-  ETCD Server
-  Kubelet
## Renew ssl cert
### Renew the certificate 
*With kubeadm*
Check cert
```
kubeadm certs check-expiration
```

Renew all cert
```
kubeadm certs renew all
```

Renew single cert
```
sudo kubeadm certs renew apiserver
sudo kubeadm certs renew apiserver-kubelet-client
```

*Update the kubeconfig file*
```
sudo kubeadm init phase kubeconfig admin
sudo kubeadm init phase kubeconfig controller-manager
sudo kubeadm init phase kubeconfig scheduler
```
if you have many users or your own kubeconfig file
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

*Restart the static pod*
```
sudo systemctl restart kubelet
```
kubelet will automatically restart the static pod define in `/etc/kubernetes/manifests`

## Certificate API Server
The **Kubernetes CertificateSigningRequest (CSR) API**—often called the **Certificates API**—exists to help manage the creation and approval of TLS certificates _inside_ the Kubernetes cluster.
Example flow
- A node or user submits a CSR:
```
kubectl create -f my-csr.yaml
```
- An admin approves it:
```
kubectl certificate approve my-csr
```
- Kubernetes signs and returns the certificate:
```
kubectl get csr my-csr -o jsonpath='{.status.certificate}' | base64 -d > signed.crt
```