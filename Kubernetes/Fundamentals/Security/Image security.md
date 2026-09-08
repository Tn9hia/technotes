In the yaml file:
image: docker.io (Registry) /library (User/Account) /nginx (Image/Repository)

To use private registry, you must input *full path of container image* => create *secret* => Referring to `imagePullSecrets` on a Pod
Ex:

#### Creating a Secret with a Docker config
```shell
kubectl create secret docker-registry <name> \
  --docker-server=<docker-registry-server> \
  --docker-username=<docker-user> \
  --docker-password=<docker-password> \
  --docker-email=<docker-email>
```

#### Referring to `imagePullSecrets` on a Pod 
```shell
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```