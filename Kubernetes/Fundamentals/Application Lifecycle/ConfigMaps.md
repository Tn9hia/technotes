A ConfigMap is an API object used to store non-confidential data in key-value pairs. [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a [volume](https://kubernetes.io/docs/concepts/storage/volumes/).

A ConfigMap allows you to decouple environment-specific configuration from your [container images](https://kubernetes.io/docs/reference/glossary/?all=true#term-image), so that your applications are easily portable.

> [!Caution]
> ConfigMap does not provide secrecy or encryption. If the data you want to store are confidential, use a Secret rather than a ConfigMap, or use additional (third party) tools to keep your data private.

## Create configmap
### Imperative
```
kubectl create configmap <name> --from-literal=<key>=<value>
```

### Declerative

```yaml
apiVerion: v1
kind: ConfigMap
metadata:
	name: test
data:
	app_color: blue
```

## Attach configmap to pod
**ENV**
```yaml
apiVersion: v1
kind: pod
metadata:
	name: test
specs:
	containers:
	- image: ubuntu
	  name: ubuntu
	  envFrom:
	  - configMapRef:
		name: test
```
**SINGLE ENV**
```yaml
valueFrom:
    configMapKeyRef:
        name: game-demo
        key: ui_properties_file_name
```