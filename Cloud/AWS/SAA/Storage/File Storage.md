
## AWS Snowball
• Highly-secure, **portable devices** to collect and process data at the edge, and migrate data into and out of AWS
• Helps migrate up to **Petabytes of data**
=> **AWS Snowball:** offline devices to perform data migrations

## Amazon FSx
**• Launch 3rd party high-performance file systems on AWS**
• Fully managed service
![[Pasted image 20260318014000.png]]

### 1. Amazon FSx for Windows (File Server)
**• FSx for Windows** is a fully managed Windows file system share drive
• Supports SMB protocol & Windows NTFS
• Microsoft Active Directory integration, ACLs, user quotas
**• Can be mounted on Linux EC2 instances**
**• Supports Microsoft's Distributed File System (DFS) Namespaces (group files across multiple FS)**

• Scale up to 10s of GB/s, millions of IOPS, 100s PB of data
• Storage Options:
	• SSD – latency sensitive workloads (databases, media processing, data analytics, …)
	• HDD – broad spectrum of workloads (home directory, CMS, …)
• Can be accessed from your on-premises infrastructure (VPN or Direct Connect)
• Can be configured to be Multi-AZ (high availability)
• Data is backed-up daily to S3
### 2. Amazon FSx for Lustre

• Lustre is a type of parallel distributed file system, for large-scale computing
• The name Lustre is derived from “Linux” and “cluster
• **Machine Learning, High Performance Computing (HPC)**
• Video Processing, Financial Modeling, Electronic Design Automation
• Scales up to 100s GB/s, millions of IOPS, sub-ms latencies
• Storage Options:
	• SSD – low-latency, IOPS intensive workloads, small & random file operations
	• HDD – throughput-intensive workloads, large & sequential file operations
**• Seamless integration with S3**
	• Can “read S3” as a file system (through FSx)
	• Can write the output of the computations back to S3 (through FSx)
**• Can be used from on-premises servers (VPN or Direct Connect)**

### 3. FSx Lustre - File System Deployment Options
**• Scratch File System**
	• Temporary storage
	• Data is not replicated (doesn’t persist if file
	server fails)
	• High burst (6x faster, 200MBps per TiB)
	• **Usage**: short-term processing, optimize costs
**• Persistent File System**
	• Long-term storage
	• Data is replicated within same AZ
	• Replace failed files within minutes
	• **Usage**: long-term processing, sensitive data
### 4. Amazon FSx for NetApp ONTAP
• Managed NetApp ONTAP on AWS
**• File System compatible with NFS, SMB, iSCSI protocol**
• Move workloads running on ONTAP or NAS to AWS
• Works with:
	• Linux
	• Windows
	• MacOS
	• VMware Cloud on AWS
	• Amazon Workspaces & AppStream 2.0
	• Amazon EC2, ECS and EKS
• Storage shrinks or grows automatically
• Snapshots, replication, low-cost, compression and data de-duplication
**• Point-in-time instantaneous cloning (helpful for testing new workloads)**
### 5. Amazon FSx for OpenZFS
• Managed OpenZFS file system on AWS
• File System compatible with **NFS (v3, v4, v4.1, v4.2)**
• Move workloads running on **ZFS** to AWS
• Works with:
	• Linux
	• Windows
	• MacOS
	• VMware Cloud on AWS
	• Amazon Workspaces & AppStream 2.0
	• Amazon EC2, ECS and EKS
• Up to 1,000,000 IOPS with < 0.5ms latency
• Snapshots, compression and low-cost
• Point-in-time instantaneous cloning (helpful for testing new workloads)

---
### Amazon FSx — Giải thích thẳng vào vấn đề

FSx = **"AWS managed file system"** — tức là thay vì tự build NFS/SMB/Lustre server trên EC2, AWS lo hết infra cho mày, mày chỉ cần mount và dùng.

> Nó giống kiểu: EBS là block storage, S3 là object storage — FSx là **managed file system** (có hierarchy, protocol, locking đàng hoàng).

---

### So sánh 4 loại

|FSx Type|Protocol|Best For|Viettel IDC relevance|
|---|---|---|---|
|**Lustre**|Lustre|HPC, ML/AI training, big data|Parallel compute jobs|
|**Windows File Server**|SMB/CIFS|Windows workloads, AD-joined|Legacy enterprise apps|
|**NetApp ONTAP**|NFS, SMB, iSCSI|Hybrid cloud, enterprise storage|Gần nhất với VMware world|
|**OpenZFS**|NFS|Linux workloads, low-latency|General Linux file sharing|

---

### Use case thực tế từng loại

#### 🔥 FSx for Lustre

```
ML Training Job → EC2/EKS cluster → FSx Lustre (sub-ms latency)
                                         ↕ linked
                                      S3 bucket (cold data)
```

- Throughput lên tới hàng trăm GB/s
- Dùng khi cần scratch storage cực nhanh cho HPC/AI
- **Không** dùng cho persistent long-term storage

---

#### 🪟 FSx for Windows File Server

- Basically managed Windows SMB share
- AD integration native
- Use case: lift-and-shift Windows apps sang AWS mà app cần UNC path `\\server\share`
- **Risk**: vendor lock-in khá cao, giá không rẻ

---

#### 🏢 FSx for NetApp ONTAP ← _Nghĩa sẽ thích cái này nhất_

```
On-prem NetApp  ←— SnapMirror replication —→  FSx ONTAP (AWS)
VMware vSphere                                  VMware Cloud on AWS
```

- Support **NFS + SMB + iSCSI** cùng lúc
- Có **SnapMirror** → DR/hybrid cloud với on-prem NetApp
- **Storage efficiency**: dedup, compression, thin provisioning
- Với background VMware của mày: đây là con closest to what you know

---

#### 🐧 FSx for OpenZFS

- ZFS managed, NFS-based
- Copy-on-write snapshots native
- Giá rẻ hơn ONTAP, simpler
- Use case: Linux app cần shared filesystem, dev environments, database storage

---

### TL;DR — Chọn cái nào?

```
Cần gì?
├── Windows app + AD?          → Windows File Server
├── HPC / ML training?         → Lustre
├── VMware / hybrid / iSCSI?   → NetApp ONTAP  ✅ (Nghĩa's pick)
└── Linux workloads, đơn giản? → OpenZFS
```

Với background VMware Cloud engineer ở Viettel IDC, **FSx for NetApp ONTAP** là cái đáng đào sâu nhất — nó bridge được on-prem storage world với AWS, support VMware Cloud on AWS, và protocol coverage rộng nhất. Mấy cái còn lại mostly niche use cases.