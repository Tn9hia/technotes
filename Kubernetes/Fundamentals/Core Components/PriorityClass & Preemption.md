## Priority
**Pods** can have priority. Priority indicates the importance of a Pod relative to other Pods. If a Pod cannot be scheduled, the scheduler tries to preempt (evict) lower priority Pods to make scheduling of the pending Pod possible.

## How to use
- Add one or more `PriorityClasses`.

```shell
kubectl create priorityclass NAME --value=VALUE --global-default=BOOL
```

- Create Pods with `priorityClassName` set to one of the added PriorityClasses. Of course you do not need to create the Pods directly; normally you would add `priorityClassName` to the Pod template of a collection object like a Deployment.
## Preemption Policy

|**Field**|**Value**|**What It Actually Does**|
|---|---|---|
|`preemptionPolicy`|`PreemptLowerPriority`|**Default behavior** - Pod can yeet lower-priority pods to grab resources. Classic survival of the fittest.|
|`preemptionPolicy`|`Never`|Pod waits in queue like a good citizen. Won't kick anyone out, even if it has higher priority. Useful for batch jobs that can chill.|