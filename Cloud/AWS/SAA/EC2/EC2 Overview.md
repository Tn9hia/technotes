## What is EC2?
**EC2** = Elastic Compute Cloud = Infrastructure as a Service

It mainly consists in the capability of :
- Renting virtual machines (EC2)
- Storing data on virtual drives (EBS)
- Distributing load across machines (ELB)
- Scaling the services using an auto-scaling group (ASG)
## EC2 User Data
- Bootstrap script is only run once at the instance first start
- EC2 user data is used to automate boot tasks such as:
	- Installing updates
	- Installing software
	- Downloading common files from the internet
## Instance type
### Name convention
AWS has the following naming convention:
Ex: **m5.2xlarge**
- **m**: instance class
- **5**: generation (AWS improves them over time)
- **2xlarge**: size within the instance class
### Purpose
- **General Purpose**: Great for a diversity of workloads such as web servers or code repositories
- **Compute Optimized**: Great for compute-intensive tasks that require high performance
processors
- **Memory Optimized**: Fast performance for workloads that process large data sets in memory
- **Storage Optimized**: Great for storage-intensive tasks that require high, sequential read and write access to large data sets on local storage
## Elastic IP
- With an Elastic IP address, you can mask the failure of an instance or software by rapidly remapping the address to another instance in your account.
- You can only have 5 Elastic IP in your account (you can ask AWS to increase that).
- Overall, try to avoid using Elastic IP:
	- They often reflect poor architectural decisions
	- Instead, use a random public IP and register a DNS name to it
- If your machine is stopped and then started, **the public IP can change**
## Placement Groups
 When you create a placement group, you specify one of the following strategies for the group:
- **Cluster**—clusters instances into a low-latency group in a single Availability Zone
- **Spread**—spreads instances across underlying hardware (max 7 instances per group per AZ)
- **Partition**—spreads instances across many different partitions (which rely on different sets of racks) within an AZ. Scales to 100s of EC2 instances per group (Hadoop, Cassandra, Kafka)
### Cluster
**Pros**: Great network (10 Gbps bandwidth between instances with Enhanced Networking enabled recommended)
**Cons**: If the AZ fails, all instances fails at the same time
**Use case:**
- Big Data job that needs to complete fast
- Application that needs extremely low latency and high network throughput
### Spread
**Pros**:
- Can span across Availability Zones (AZ)
- Reduced risk is simultaneous failure
- EC2 Instances are on different physical hardware
**Cons**:
- Limited to 7 instances per AZ per placement group
**Use case:**
- Application that needs to maximize high availability
- Critical Applications where each instance must be isolated from failure from each other
### Partition
- Up to 7 partitions per AZ
- Can span across multiple AZs in the same region
- Up to 100s of EC2 instances
- The instances in a partition do not share racks with the instances in the other partitions
- A partition failure can affect many EC2 but won’t affect other partitions EC2 instances get access to the partition information as metadata
- **Use cases**: HDFS, HBase, Cassandra, Kafka
## Elastic Network Interfaces (ENI)
- Logical component in a VPC that represents a **virtual network card**
- You can create ENI independently and attach them on the fly (move them) on EC2 instances for failover
- Bound to a specific availability zone (AZ)
## EC2 Hibernate
Introducing **EC2 Hibernate**:
- The in-memory (RAM) state is preserved
- The instance boot is much faster! (the OS is not stopped / restarted)
- Under the hood: the RAM state is written to a file in the root EBS volume
- The root EBS volume must be encrypted
Use cases:
- Long-running processing
- Saving the RAM state
- Services that take time to initialize
### Good to know
- Supported Instance Families – C3, C4, C5, I3, M3, M4, R3, R4, T2, T3, …
- **Instance RAM Size** – must be less than 150 GB.
- **Instance Size** – not supported for bare metal instances.
- **AMI** – Amazon Linux 2, Linux AMI, Ubuntu, RHEL, CentOS & Windows…
- **Root Volume** – must be EBS, encrypted, not instance store, and large
- Available for **On-Demand, Reserved and Spot Instances**
- An instance can **NOT** be hibernated more than 60 days

## AMI - Amazon Machine Image
AMI are a **customization** of an EC2 instance
- You add your own software, configuration, operating system, monitoring…
- Faster boot / configuration time because all your software is pre-packaged
AMI are built for a **specific region** (and can be copied across regions)
You can launch EC2 instances from:
- **A Public AMI**: AWS provided
- **Your own AMI**: you make and maintain them yourself
- **An AWS Marketplace AMI**: an AMI someone else made (and potentially sells)
### AMI process
![[Pasted image 20260315221617.png]]


1. Start an EC2 instance and customize it
2. Stop the instance (for data integrity)
3. Build an AMI – this will also create EBS snapshots
4. Launch instances from other AMIs


## EBS - Elastic Block Storage
[[EBS | Here for more detail]]
## EFS - Elastic File System
[[EFS| Here for more detail]]

## EC2 Instance Store
- EBS volumes are network drives with good but “limited” performance
- **If you need a high-performance hardware disk, use EC2 Instance Store**
- Better I/O performance
- EC2 Instance Store lose their storage if they’re stopped (ephemeral)
- Good for buffer / cache / scratch data / temporary content
- Risk of data loss if hardware fails
- Backups and Replication are your responsibility

## Comparison for EC2 Storage 
### Details

|Tiêu chí|Instance Store|EBS (gp3/io2)|EFS|
|---|---|---|---|
|**Storage type**|Block (local NVMe/SSD)|Block (network)|File (NFS v4.1)|
|**Persistence**|❌ Ephemeral – mất khi stop/terminate|✅ Persistent|✅ Persistent|
|**Scope**|Gắn cứng với instance|1 AZ (trừ Multi-Attach)|Multi-AZ / Regional|
|**Shared access**|❌ Không|⚠️ Multi-Attach (io1/io2, max 16, cùng AZ, cần cluster FS)|✅ Native – nhiều instance cùng mount|
|**Latency**|🚀 Cực thấp (~μs)|⚡ Thấp (~ms single digit)|🐢 Cao hơn (~ms, network overhead)|
|**Throughput**|🚀 Rất cao (local bus)|⚡ Cao (tùy volume type)|✅ Scale theo số file/connections|
|**Max size**|Cố định theo instance type|64 TiB / volume|Petabyte scale (auto)|
|**Scalability**|❌ Fixed – phụ thuộc instance|⚠️ Manual resize (cần fs resize sau)|✅ Auto-scale|
|**Snapshots / Backup**|❌ Không hỗ trợ native|✅ EBS Snapshot → S3|✅ AWS Backup / EFS Replication|
|**Encryption**|✅ Hỗ trợ|✅ Hỗ trợ (KMS)|✅ Hỗ trợ (KMS)|
|**Pricing model**|💚 Included trong instance cost|💛 Theo GB provisioned + IOPS|💛 Theo GB stored (Standard / IA tier)|
|**Cost (tương đối)**|💚 Free (bundled)|💛 ~$0.08–0.125/GB-month (gp3)|💸 ~$0.30/GB-month (Standard)|
|**OS / Boot volume**|⚠️ Một số instance type hỗ trợ|✅ Thường dùng làm root volume|❌ Không dùng làm boot|
|**Protocol**|Local block|Block (qua AWS network fabric)|NFS v4.1|
|**Multi-AZ**|❌|❌ (1 AZ, trừ io2 Express)|✅ Regional EFS|
|**Use với K8s**|⚠️ Dùng được, nhưng pod phải pin vào node|✅ CSI Driver (aws-ebs-csi-driver)|✅ CSI Driver (aws-efs-csi-driver)|
### Use Cases

|Scenario|Recommended|Lý do|
|---|---|---|
|Database (MySQL, Postgres, Mongo)|**EBS io2 / gp3**|Low latency, persistent, single-attach đủ dùng|
|K8s Persistent Volume (stateful app)|**EBS gp3**|Per-pod volume, CSI driver mature|
|K8s Shared Volume (ReadWriteMany)|**EFS**|Multi-pod, multi-node mount|
|Longhorn storage pool|**EBS gp3** (mỗi node 1 volume)|Longhorn tự replicate, không cần shared|
|Kafka / Elasticsearch data dir|**EBS gp3 / io2**|High throughput, low latency|
|Temporary cache / buffer / scratch|**Instance Store**|Speed ưu tiên, data loss acceptable|
|ML training scratch space|**Instance Store**|NVMe tốc độ cao, dataset temp|
|Shared config / static assets|**EFS**|Multi-instance read, centralized|
|WordPress / CMS media uploads|**EFS**|Multi-instance cùng đọc/ghi file|
|Oracle RAC / clustered DB|**EBS io2 Multi-Attach**|Cần shared block + cluster FS (OCFS2/GFS2)|
|CI/CD build cache|**EBS gp3 hoặc Instance Store**|Tùy persistence requirement|

---

### Risk Matrix

|Storage|Data Loss Risk|Performance Risk|Cost Risk|
|---|---|---|---|
|Instance Store|🔴 HIGH – mất khi stop|🟢 LOW|🟢 LOW|
|EBS gp3|🟢 LOW|🟡 MEDIUM (network-bound)|🟡 MEDIUM|
|EBS io2|🟢 LOW|🟢 LOW|🔴 HIGH (expensive)|
|EFS Standard|🟢 LOW|🔴 HIGH (latency-sensitive workloads)|🔴 HIGH|
|EFS IA (Infrequent Access)|🟢 LOW|🔴 HIGH|🟡 MEDIUM|

---

### Architecture Decision Flowchart

```
Cần lưu data persistent không?
├── KHÔNG → Instance Store (tốc độ max)
└── CÓ
    ├── Nhiều instance/pod cùng mount không?
    │   ├── CÓ → EFS
    │   └── KHÔNG
    │       ├── Cần IOPS cao (>16k), latency cực thấp?
    │       │   ├── CÓ → EBS io2
    │       │   └── KHÔNG → EBS gp3 (default choice)
    │       └── Clustered DB, Oracle RAC?
    │           └── CÓ → EBS io2 Multi-Attach + Cluster FS
```