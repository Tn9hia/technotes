---
aliases: []
---
## Command and Argument

| Docker     | Kubernetes |
| ---------- | ---------- |
| ENTRYPOINT | command    |
| CMD        | args       |


```yaml
apiVersion: 2
kind: Pod
metadata: 
	name: ubuntu-sleeper
specs:
	container:
		- name: ubuntu
		  image: ubuntu
		  command: ["sleep"]
		  args: ["10"]
```

## Enviroment Variable
```yaml
apiVersion: 2
kind: Pod
metadata: 
	name: ubuntu-sleeper
specs:
	container:
		- name: ubuntu
		  image: ubuntu
		  command: ["sleep"]
		  args: ["10"]
		  env:
			  - name: app1
			    value: pink
```

There are 3 ways to define enviroment variable
- Plain key value
- ConfigMap
- Secrets