- **On-Demand Instances** – short workload, predictable pricing, pay by second 
- **Reserved** (1 & 3 years)
	- **Reserved Instances** – long workloads
	- **Convertible Reserved Instances** – long workloads with flexible instances
- **Savings Plans (1 & 3 years)** –commitment to an amount of usage, long workload
- **Spot Instances** – short workloads, cheap, can lose instances (less reliable)
- **Dedicated Hosts** – book an entire physical server, control instance placement
- **Dedicated Instances** – no other customers will share your hardware
- **Capacity Reservations** – reserve capacity in a specific AZ for any duration

## EC2 Purchasing Options — Comparison Table

|Purchasing Option|Commitment|Cost vs On-Demand|Reliability|Use Case|Flexibility|
|---|---|---|---|---|---|
|**On-Demand**|None|Baseline (100%)|✅ High|Dev/test, unpredictable spikes, short workloads|Maximum|
|**Reserved Instances**|1 or 3 years|Up to **72% off**|✅ High|Steady-state workloads (DB, app servers)|Low — locked to instance type/region|
|**Convertible Reserved**|1 or 3 years|Up to **66% off**|✅ High|Steady-state but may need instance flexibility|Medium — can change instance family|
|**Savings Plans (Compute)**|1 or 3 years|Up to **66% off**|✅ High|Flexible compute (EC2, Lambda, Fargate)|High — applies across regions & families|
|**Savings Plans (EC2 Instance)**|1 or 3 years|Up to **72% off**|✅ High|Specific instance family in a region|Medium — locked to instance family|
|**Spot Instances**|None|Up to **90% off**|❌ Low — can be interrupted with 2-min notice|Batch jobs, CI/CD workers, stateless workloads|High|
|**Dedicated Hosts**|On-demand or 1/3 years|Most expensive|✅ High|BYOL licensing, compliance/regulatory requirements|Low|
|**Dedicated Instances**|None|~10% premium over On-Demand|✅ High|Hardware isolation without full host control|Medium|
|**Capacity Reservations**|None (any duration)|On-Demand price (no discount)|✅ Guaranteed capacity|Critical workloads needing guaranteed AZ capacity|High|

---

## 🧠 SAA Exam Quick-Fire Decision Tree

```
Need guaranteed capacity in a specific AZ?
└─► Capacity Reservation (combine with Savings Plan for discount)

Compliance / BYOL software license?
└─► Dedicated Host

Hardware isolation, no license concern?
└─► Dedicated Instance

Steady workload, know exact instance type?
└─► Reserved Instance (Standard) → max savings

Steady workload, but might change instance family?
└─► Convertible RI or Compute Savings Plan

Flexible compute across EC2 + Lambda + Fargate?
└─► Compute Savings Plans ← AWS's preferred recommendation now

Fault-tolerant, stateless, batch jobs?
└─► Spot Instances (use Spot Fleet + diversified pools)

Short-term / unpredictable / testing?
└─► On-Demand
```

---

## 💡 Key SAA Gotchas

- **Savings Plans > Reserved Instances** for most new designs — AWS has been pushing this direction. Compute Savings Plans are the most flexible.
- **Spot ≠ unreliable** if designed right — Spot Fleet với `diversified` allocation strategy + checkpointing = production-viable cho batch/data workloads.
- **Capacity Reservation không có discount** — nó chỉ đảm bảo capacity. Muốn cả discount lẫn guaranteed capacity thì **kết hợp Capacity Reservation + Savings Plan**.
- **Dedicated Host vs Dedicated Instance**: Host = bạn own cả con physical server (biết socket, core count) → dùng cho per-socket/per-core BYOL licenses (Oracle, Windows Server). Instance = chỉ isolate hardware, không control placement.
- **Reserved Instances có thể sell lại** trên AWS Marketplace nếu không dùng nữa — Convertible thì không.

## 1. On-Demand instances
Pay for what you use:
• Linux or Windows - **billing per second**, after the first minute
• All other operating systems - **billing per hour**
• Has the **highest** cost but no upfront payment
• No long-term commitment
• Recommended for **short-term** and **un-interrupted workloads**, where
you can't predict how the application will behave
## 2. EC2 Reserved Instances
- Up to 72% discount compared to On-demand
- You reserve a specific instance attributes (Instance Type, Region, Tenancy, OS)
- Reservation Period – 1 year (+discount) or 3 years (+++discount)
- Payment Options – No Upfront (+), Partial Upfront (++), All Upfront (+++)
- Reserved Instance’s Scope – Regional or Zonal (reserve capacity in an AZ)
- Recommended for steady-state usage applications (think database)
- You can buy and sell in the Reserved Instance Marketplace

Convertible Reserved Instance
- Can change the EC2 instance type, instance family, OS, scope and tenancy
- Up to 66% discount 

## 3. EC2 Savings Plans
- Get a discount based on long-term usage (up to 72% - same as RIs)
- Commit to a certain type of usage ($10/hour for 1 or 3 years)
- Usage beyond EC2 Savings Plans is billed at the On-Demand price
- Locked to a specific instance family & AWS region (e.g., M5 in us-east-1)

- Flexible across:
	- Instance Size (e.g., m5.xlarge, m5.2xlarge)
	- OS (e.g., Linux, Windows)
	- Tenancy (Host, Dedicated, Default)
## 4. EC2 Spot Instances
- Can get a discount of up to 90% compared to On-demand
- Instances that you can “**lose**” at any point of time if your max price is less than the current spot price
- The MOST cost-efficient instances in AWS
	- Useful for workloads that are resilient to failure
	- Batch jobs
	- Data analysis
	- Image processing
	- Any distributed workloads
	- Workloads with a flexible start and end time
- **Not suitable for critical jobs or databases**
### EC2 Spot Instance Requests
- Can get a discount of up to 90% compared to On-demand

- Define **max spot price** and get the instance while **current spot price < max**
	- The hourly spot price varies based on offer and capacity
	- If the current spot price > your max price you can choose to stop or terminate your instance with a 2 minutes grace period.
- Other strategy: **Spot Block**
	- “block” spot instance during a specified time frame (1 to 6 hours) without interruptions
	- In rare situations, the instance may be reclaimed
- **Used for batch jobs, data analysis, or workloads that are resilient to failures.**
- **Not great for critical jobs or databases**

### Spot Instance Lifecycle

|              | **One-time**                            | **Persistent**                                          |
| ------------ | --------------------------------------- | ------------------------------------------------------- |
| Bị interrupt | Request → `closed`, instance terminated | Request → `disabled`, tự restart khi capacity available |
| Dùng khi nào | Batch job chạy 1 lần                    | Worker pool, cần luôn có instance chạy                  |

![[Pasted image 20260315120309.png]]
> [!Warning]
> You can only cancel Spot Instance requests that are **open, active, or disabled**.
> *Cancelling a Spot Request does not terminate instances*
> You must first cancel a Spot Request, and then terminate the associated Spot Instances

### Spot Fleets
- Spot Fleets = **set of Spot Instances** + (optional) On-Demand Instances
- The Spot Fleet will try to meet the target capacity with price constraints
	- Define possible launch pools: instance type (m5.large), OS, Availability Zone
	- Can have multiple launch pools, so that the fleet can choose
	- Spot Fleet stops launching instances when reaching capacity or max cost

- Strategies to allocate Spot Instances:
	- **lowestPrice**: from the pool with the lowest price (cost optimization, short workload)
	- **diversified**: distributed across all pools (great for availability, long workloads)
	- **capacityOptimized**: pool with the optimal capacity for the number of instances
	- **priceCapacityOptimized** (recommended): pools with highest capacity available, then selectvthe pool with the lowest price (best choice for most workloads)
*• Spot Fleets allow us to automatically request Spot Instances with the lowest price*

## 5. EC2 Dedicated Hosts
- A physical server with EC2 instance capacity fully dedicated to your use
- Allows you address compliance requirements and use your existing server-bound software licenses (per-socket, per-core, pe—VM software licenses)
- Purchasing Options:
	- On-demand – pay per second for active Dedicated Host
	- Reserved - 1 or 3 years (No Upfront, Partial Upfront, All Upfront)
**The most expensive option**
- Useful for software that have complicated licensing model (**BYOL** – Bring Your
Own License)
- Or for companies that have strong regulatory or compliance needs
## 6. EC2 Dedicated Instances
 - Instances run on hardware that’s dedicated to you
 - May share hardware with other instances in same account
 - No control over instance placement (can move hardware after Stop / Start)

| Characteristic                                        | Dedicated Instances | Dedicated Hosts |
| ----------------------------------------------------- | ------------------- | --------------- |
| Enables the use of dedicated physical servers         | X                   | X               |
| Per instance billing (subject to a $2 per region fee) | X                   |                 |
| Per host billing                                      |                     | X               |
| Visibility of sockets, cores, host ID                 |                     | X               |
| Affinity between a host and instance                  |                     | X               |
| Targeted instance placement                           |                     | X               |
| Automatic instance placement                          | X                   | X               |
| Add capacity using an allocation request              |                     | X               |

## 7. EC2 Capacity Reservations
- Reserve On-Demand instances capacity in a specific AZ for any duration
- You always have access to EC2 capacity when you need it
- **No time commitment** (create/cancel anytime), **no billing discounts**
- Combine with Regional Reserved Instances and Savings Plans to benefit from billing discounts
- You’re charged at On-Demand rate whether you run instances or not
- Suitable for **short-term, uninterrupted workloads** that needs to be in a
specific AZ

