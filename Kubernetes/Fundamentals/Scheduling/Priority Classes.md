[Pods](https://kubernetes.io/docs/concepts/workloads/pods/) can have _priority_. Priority indicates the importance of a Pod relative to other Pods. If a Pod cannot be scheduled, the scheduler tries to preempt (evict) lower priority Pods to make scheduling of the pending Pod possible.
default value: 0

> [!warning]
> In a cluster where not all users are trusted, a malicious user could create Pods at the highest possible priorities, causing other Pods to be evicted/not get scheduled. An administrator can use ResourceQuota to prevent users from creating pods at high priorities.
> See [limit Priority Class consumption by default](https://kubernetes.io/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default) for details.

> [!note]
> Kubernetes already ships with two PriorityClasses: `system-cluster-critical` and `system-node-critical`. These are common classes and are used to [ensure that critical components are always scheduled first](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/).


## Predemtion Policy
- `preemtLowerPriority` (default): evict pod with lower priority to get space for high priority pod
- `never`: do nothing
## Priority Class
A PriorityClass is a non-namespaced object that defines a mapping from a priority class name to the integer value of the priority. The name is specified in the `name` field of the PriorityClass object's metadata. The value is specified in the required `value` field. The higher the value, the higher the priority. The name of a PriorityClass object must be a valid [DNS subdomain name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names), and it cannot be prefixed with `system-`.

PriorityClass object is from -2147483648 to 1000000000 inclusive

PriorityClass also has two optional fields: `globalDefault` and `description`. The `globalDefault` field indicates that the value of this PriorityClass should be used for Pods without a `priorityClassName`. Only one PriorityClass with `globalDefault` set to true can exist in the system. If there is no PriorityClass with `globalDefault` set, the priority of Pods with no `priorityClassName` is zero.

The `description` field is an arbitrary string. It is meant to tell users of the cluster when they should use this PriorityClass.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

## Non-preempting PriorityClass
When Pods are created, they go to a queue and wait to be scheduled. The scheduler picks a Pod from the queue and tries to schedule it on a Node. If no Node is found that satisfies all the specified requirements of the Pod, preemption logic is triggered for the pending Pod. Let's call the pending Pod P. Preemption logic tries to find a Node where removal of one or more Pods with lower priority than P would enable P to be scheduled on that Node. If such a Node is found, one or more lower priority Pods get evicted from the Node. After the Pods are gone, P can be scheduled on the Node.


priority-class.yml
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
	name: high-priority
value: 1000000000
description: "priority class for critical pod"
preemptionPolicy: never
globalDefault: false
```

pod-definition.yml
```yaml
apiVersion: 2
kind Pod
metadata: 
	containers:
	name: nginx
	labels:
		- name: nginx
		  image: nginx
		  ports:
			- containerPort: 8080
	priorityClassName: high-priority
```