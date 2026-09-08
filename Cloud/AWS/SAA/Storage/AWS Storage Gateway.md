## What is  AWS Storage Gateway?
• AWS is pushing for ”hybrid cloud”
	• Part of your infrastructure is on the cloud
	• Part of your infrastructure is on-premises
• This can be due to
	• Long cloud migrations
	• Security requirements
	• Compliance requirements
	• IT strategy
• S3 is a proprietary storage technology (unlike EFS / NFS), so how do you expose the S3 data on-premises?
**=> AWS Storage Gateway!**

![[Pasted image 20260322014634.png]]
## AWS Storage Gateway
• Bridge between on-premises data and cloud data
**• Use cases:**
• disaster recovery
• backup & restore
• tiered storage
• on-premises cache & low-latency files access
• Types of Storage Gateway:
	**• S3 File Gateway**
	**• Volume Gateway**
	**• Tape Gateway**

![[Pasted image 20260322015106.png]]
### 1. Amazon S3 File Gateway
• Configured S3 buckets are accessible using the NFS and SMB protocol
**• Most recently used data is cached in the file gateway**
• Supports S3 Standard, S3 Standard IA, S3 One Zone A, S3 Intelligent Tiering
**• Transition to S3 Glacier using a Lifecycle Policy**
• Bucket access using IAM roles for each File Gateway
• SMB Protocol has integration with Active Directory (AD) for user authentication
![[Pasted image 20260322014725.png]]
### 2. Volume Gateway
• Block storage using iSCSI protocol backed by S3
• Backed by EBS snapshots which can help restore on-premises volumes!
**• Cached volumes:** low latency access to most recent data
**• Stored volumes:** entire dataset is on premise, scheduled backups to S3

![[Pasted image 20260322014909.png]]

### 3. Tape Gateway
• Some companies have backup processes using physical tapes (!)
• With Tape Gateway, companies use the same processes but, in the cloud
• Virtual Tape Library (VTL) backed by Amazon S3 and Glacier
• Back up data using existing tape-based processes (and iSCSI interface)
• Works with leading backup software vendors

## AWS Transfer Family
• A fully-managed service for file transfers **into and out of Amazon S3 or Amazon EFS using the FTP protocol**
**• Supported Protocols**
	• **AWS Transfer for FTP** (File Transfer Protocol (FTP))
	• **AWS Transfer for FTPS** (File Transfer Protocol over SSL (FTPS))
	• **AWS Transfer for SFTP** (Secure File Transfer Protocol (SFTP))
• Managed infrastructure, Scalable, Reliable, Highly Available (multi-AZ)
• Pay per provisioned endpoint per hour + data transfers in GB
• Store and manage users’ credentials within the service
• Integrate with existing authentication systems (Microsoft Active Directory,
LDAP, Okta, Amazon Cognito, custom)
• Usage: sharing files, public datasets, CRM, ERP, …
![[Pasted image 20260322015541.png]]

## AWS DataSync
• Move large amount of data to and from
	• On-premises / other cloud to AWS (NFS, SMB, HDFS, S3 API…) – needs agent
	• AWS to AWS (different storage services) – no agent needed
• Can synchronize to:
	• Amazon S3 (any storage classes – including Glacier)
	• Amazon EFS
	• Amazon FSx (Windows, Lustre, NetApp, OpenZFS...)
• Replication tasks can be scheduled hourly, daily, weekly
**• File permissions and metadata are preserved (NFS POSIX, SMB…)**
• One agent task can use 10 Gbps, can setup a bandwidth limit

![[Pasted image 20260322102800.png]]