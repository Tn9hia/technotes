## Amazon S3 Use cases
• Backup and storage
• Disaster Recovery
• Archive
• Hybrid Cloud storage
• Application hosting
• Media hosting
• Data lakes & big data analytics
• Software delivery
• Static website

## Bucket
• Amazon S3 allows people to store **objects (files)** in **“buckets” (directories)**
• Buckets must have a **globally unique name (across all regions all accounts)**
• Buckets are defined at the region level
• S3 looks like a global service but buckets are created in a region
• Naming convention
• No uppercase, No underscore
• 3-63 characters long
• Not an IP
• Must start with lowercase letter or number
• Must NOT start with the prefix xn--
• Must NOT end with the suffix -s3alias

### Object
• Objects (files) have a **Key**
• The **key** is the FULL path:
	• s3://my-bucket/**my_file.txt**
	• s3://my-bucket/**my_folder1/another_folder/my_file.txt**
• The key is composed of *prefix* + **object name**
	• s3://my-bucket/*my_folder1/another_folder*/**my_file.txt**
• Object values are the content of the body:
	• Max. Object Size is **5TB (5000GB)**
	• If uploading more than 5GB, must use “**multi-part upload**”
• Metadata (list of text key / value pairs – system or user metadata)
• Tags (Unicode key / value pair – up to 10) – useful for security / lifecycle
• Version ID (if versioning is enabled)

## Security
**• User-Based**
	• IAM Policies – which API calls should be allowed for a specific user from IAM
**• Resource-Based**
	• Bucket Policies – bucket wide rules from the S3 console - allows cross account
	• Object Access Control List (ACL) – finer grain (can be disabled)
	• Bucket Access Control List (ACL) – less common (can be disabled)
• **Note**: an IAM principal can access an S3 object if
	• The user IAM permissions **ALLOW** it **OR** the resource policy **ALLOWS** it
	• AND there’s no explicit **DENY**
• **Encryption**: encrypt objects in Amazon S3 using encryption keys

## S3 Bucket Policies
• JSON based policies
	• Resources: buckets and objects
	• Effect: Allow / Deny
	• Actions: Set of API to Allow or Deny
	• Principal: The account or user to apply the policy to
• Use S3 bucket for policy to:
	• Grant public access to the bucket
	• Force objects to be encrypted at upload
	• Grant access to another account (Cross Account)


```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-prod-bucket",
        "arn:aws:s3:::my-prod-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowSpecificRoleReadWrite",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/app-backend-role"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-prod-bucket/*"
    },
    {
      "Sid": "AllowCrossAccountReadOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:root"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-prod-bucket/shared/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "DenyDeleteWithoutMFA",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "s3:DeleteBucket",
      "Resource": "arn:aws:s3:::my-prod-bucket",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

**Breakdown từng Statement:**

|Sid|Mục đích|Risk nếu thiếu|
|---|---|---|
|`DenyNonHTTPS`|Block mọi request qua HTTP|Data in transit bị sniff — **High**|
|`AllowSpecificRoleReadWrite`|Chỉ cho phép IAM Role của app|Least privilege — nếu dùng `*` là xong phim|
|`AllowCrossAccountReadOnly`|Cross-account access nhưng chỉ trong Org|Không giới hạn Org → account lạ vào được|
|`DenyDeleteWithoutMFA`|Bảo vệ bucket khỏi bị xóa nhầm|Không có → junior xóa bucket production lúc 2am|
## Static Website Hosting
• S3 can host static websites and have them accessible on the Internet
• The website URL will be (depending on the region)
• http://bucket-name.s3-website-aws-region.amazonaws.com
OR
• http://bucket-name.s3-website.aws-region.amazonaws.com
• If you get a 403 Forbidden error, make sure the bucket policy allows public reads!

## Versioning
• You can version your files in Amazon S3
• It is enabled at the bucket level
• Same key overwrite will change the “version”: 1, 2, 3….
• It is best practice to version your buckets
	• Protect against unintended deletes (ability to restore a version)
	• Easy roll back to previous version
• **Notes**:
	• Any file that is not versioned prior to enabling versioning will have version “null”
	**• Suspending versioning does not delete the previous versions**

## Replication (CRR & SRR)
• Must enable Versioning in source and destination buckets
• Cross-Region Replication (CRR)
• Same-Region Replication (SRR)
• Buckets can be in different AWS accounts
• Copying is asynchronous
• Must give proper IAM permissions to S3
**• Use cases:**
	• CRR – compliance, lower latency access, replication across accounts
	• SRR – log aggregation, live replication between production and test accounts
• After you enable Replication, only new objects are replicated
• Optionally, you can replicate existing objects using **S3 Batch Replication**
	• Replicates existing objects and objects that failed replication
• For DELETE operations
	• **Can replicate delete markers** from source to target (optional setting)
	• Deletions with a version ID are not replicated (to avoid malicious deletes)
**• There is no “chaining” of replication**
	• If bucket 1 has replication into bucket 2, which has replication into bucket 3
	• Then objects created in bucket 1 are not replicated to bucket 3

## S3 Durability and Availability
• **Durability**:
• High durability (99.999999999%, 11 9’s) of objects across multiple AZ
• If you store 10,000,000 objects with Amazon S3, you can on average expect to incur a loss of a single object once every 10,000 years
• Same for all storage classes
**• Availability:**
• Measures how readily available a service is
• Varies depending on storage class
• Example: S3 standard has 99.99% availability = not available 53 minutes a year

## S3 Express One Zone
• High performance, **single Availability Zone** storage class
• Objects stored in a **Directory Bucket (bucket in a single AZ)**
• Handle 100,000s requests per second with single-digit millisecond latency
• Up to 10x better performance than S3 Standard (50% lower costs)
• High Durability (99.999999999%) and Availability (99.95%)
• Co-locate your storage and compute resources in the same AZ (reduces latency)
• Use cases: latency-sensitive apps, data-intensive apps, AI & ML training, financial modeling, media processing, HPC…
• Best integrated with SageMaker Model Training, Athena, EMR, Glue…

## S3 Storage Class

### 1. Amazon S3 Standard - General Purpose
• 99.99% Availability
• Used for frequently accessed data
• Low latency and high throughput
• Sustain 2 concurrent facility failures
**• Use Cases**: Big Data analytics, mobile & gaming applications, content distribution…
### 2. Amazon S3 Standard-Infrequent Access (IA)
• For data that is **less frequently accessed**, but requires rapid **access when needed**
• Lower cost than S3 Standard
**• Amazon S3 Standard-Infrequent Access (S3 Standard-IA)**
	• 99.9% Availability
	• Use cases: Disaster Recovery, backups
**• Amazon S3 One Zone-Infrequent Access (S3 One Zone-IA)**
	• High durability (99.999999999%) in a single AZ; data lost when AZ is destroyed
	• 99.5% Availability
• **Use Cases**: Storing secondary backup copies of on-premises data, or data you can recreate
### 3. Amazon S3 One Zone-Infrequent Access
• Low-cost object storage meant for archiving / backup
• Pricing: price for storage + object retrieval cost

**• Amazon S3 Glacier Instant Retrieval**
	• Millisecond retrieval, great for data accessed once a quarter
	• Minimum storage duration of 90 days
**• Amazon S3 Glacier Flexible Retrieval (formerly Amazon S3 Glacier):**
	• Expedited (1 to 5 minutes), Standard (3 to 5 hours), Bulk (5 to 12 hours) – free
	• Minimum storage duration of 90 days
**• Amazon S3 Glacier Deep Archive – for long term storage:**
	• Standard (12 hours), Bulk (48 hours)
	• Minimum storage duration of 180 days
### 4. Amazon S3 Glacier Instant Retrieval

### 5. Amazon S3 Glacier Flexible Retrieval

### 6. Amazon S3 Glacier Deep Archive

### 7. Amazon S3 Intelligent Tiering
• Small monthly monitoring and auto-tiering fee
• Moves objects automatically between Access Tiers based on usage
• There are no retrieval charges in S3 Intelligent-Tiering

*• Frequent Access tier (automatic)*: default tier
• *Infrequent Access tier (automatic)*: objects not accessed for 30 days
*• Archive Instant Access tier (automatic):* objects not accessed for 90 days
*• Archive Access tier (optional):* configurable from 90 days to 700+ days
*• Deep Archive Access tier (optional):* config. from 180 days to 700+ days
**Note**: Can move between classes manually or using S3 Lifecycle configurations

### Comparison
![[Pasted image 20260317121722.png]]

![[Pasted image 20260317121736.png]]

## Moving between Storage Classes
• You can transition objects between storage classes
• For infrequently accessed object, move them to **Standard IA**
• For archive objects that you don’t need fast access to, move them to **Glacier or Glacier Deep Archive**
• Moving objects can be automated using a **Lifecycle Rules**
![[Pasted image 20260317122231.png]]

## Lifecycle Rules
• **Transition Actions** – configure objects to transition to another storage class
	• Move objects to Standard IA class 60 days after creation
	• Move to Glacier for archiving after 6 months
**• Expiration actions** – configure objects to expire (delete) after some time
• Access log files can be set to delete after a 365 days
**• Can be used to delete old versions of files (if versioning is enabled)**
• Can be used to delete incomplete Multi-Part uploads
• Rules can be created for a certain prefix (example: s3://mybucket/mp3/\*)
• Rules can be created for certain objects Tags (example: Department: Finance)

## Storage Class Analysis
• Help you decide when to transition objects to the right storage class
• Recommendations for **Standard and Standard IA**
• Does NOT work for One-Zone IA or Glacier
• Report is updated daily
• 24 to 48 hours to start seeing data analysis
• Good first step to put together Lifecycle Rules (or improve them)!

## Requester Pays
• In general, bucket owners pay for all Amazon S3 **storage** and **data transfer** costs associated with their bucket
• With Requester Pays buckets, the requester instead of the bucket owner pays the cost of the request and the data download from the bucket
• Helpful when you want to share large datasets with other accounts
• The requester must be authenticated in AWS (cannot be anonymous)
**=> Pay to access content**

![[Pasted image 20260317125104.png]]

## S3 Event Notifications
• S3:ObjectCreated, S3:ObjectRemoved, S3:ObjectRestore, S3:Replication…
• Object name filtering possible (\*.jpg)
• Use case: generate thumbnails of images uploaded to S3
**• Can create as many “S3 events” as desired**
• S3 event notifications typically deliver events in seconds but can sometimes take a minute or longer

=> Event can be pushed to SQS, SNS, Lambda Functions, Amazon EventBridge
### Amazon EventBridge
**• Advanced filtering** options with JSON rules (metadata, object size, name...)
**• Multiple Destinations** – ex Step Functions, Kinesis Streams / Firehose…
**• EventBridge Capabilities** – Archive, Replay Events, Reliable delivery

## Performance
### Baseline 
• Amazon S3 automatically scales to high request rates, **latency 100-200 ms**
• Your application can achieve at least **3,500 PUT/COPY/POST/DELETE or 5,500 GET/HEAD requests per second per prefix in a bucket.**
• There are no limits to the number of prefixes in a bucket.
	• Example (object path => prefix):
	• bucket/folder1/sub1/file => /folder1/sub1/
	• bucket/folder1/sub2/file => /folder1/sub2/
	• bucket/1/file => /1/
	• bucket/2/file => /2/
**• If you spread reads across all four prefixes evenly, you can achieve 22,000 requests per second for GET and HEAD**

### Upload
**• Multi-Part upload:**
• recommended for files > 100MB, must use for files > 5GB
• Can help parallelize uploads (speed up transfers)

**• S3 Transfer Acceleration**
• Increase transfer speed by transferring file to an AWS edge location which will forward the data to the S3 bucket in the target region
• Compatible with multi-part upload
![[Pasted image 20260317125843.png]]

### Download - S3 Byte-Range Fetches
• Parallelize GETs by requesting specific byte ranges
• Better resilience in case of failures
=> *Can be used to retrieve only partial data (for example the head of a file)*
=> *Can be used to speed up downloads*

## Batch Operations
• Perform bulk operations on existing S3 objects with a single request, example:
	• Modify object metadata & properties
	• Copy objects between S3 buckets
	• Encrypt un-encrypted objects
	• Modify ACLs, tags
	• Restore objects from S3 Glacier
	• Invoke Lambda function to perform custom action on each object
• A job consists of a list of objects, the action to perform, and optional parameters
• S3 Batch Operations manages retries, tracks progress, sends completion notifications, generate reports …
**• You can use S3 Inventory to get object list and use Athena to query and filter your objects**

![[Pasted image 20260317130130.png]]

## Storage Lens

• Understand, analyze, and optimize storage across entire AWS Organization
• Discover anomalies, identify cost efficiencies, and apply data protection best practices across entire AWS Organization (30 days usage & activity metrics)
• Aggregate data for Organization, specific accounts, regions, buckets, or prefixes
• Default dashboard or create your own dashboards
• Can be configured to export metrics daily to an S3 bucket (CSV, Parquet)
![[Pasted image 20260317130239.png]]
### Default Dashboard
• Visualize summarized insights and trends for both free and advanced metrics
• Default dashboard shows Multi-Region and Multi-Account data
• Preconfigured by Amazon S3
• Can’t be deleted, but can be disabled

### Metrics
**1. Summary Metrics**
	• General insights about your S3 storage
	• StorageBytes, ObjectCount…
	• **Use cases**: identify the fastest-growing (or not used) buckets and prefixes
**2• Cost-Optimization Metrics**
	• Provide insights to manage and optimize your storage costs
	• NonCurrentVersionStorageBytes, IncompleteMultipartUploadStorageBytes…
	• **Use cases**: identify buckets with incomplete multipart uploaded older than 7 days, Identify which objects could be transitioned to lower-cost storage class
**3• Data-Protection Metrics**
	• Provide insights for data protection features
	• VersioningEnabledBucketCount, MFADeleteEnabledBucketCount, SSEKMSEnabledBucketCount, CrossRegionReplicationRuleCount…
	• **Use cases**: identify buckets that aren’t following data-protection best practices
**4• Access-management Metrics**
	• Provide insights for S3 Object Ownership
	• ObjectOwnershipBucketOwnerEnforcedBucketCount…
	• **Use cases**: identify which Object Ownership settings your buckets use
**5• Event Metrics**
	• Provide insights for S3 Event Notifications
	• EventNotificationEnabledBucketCount (identify which buckets have S3 Event Notifications configured)
**6• Performance Metrics**
	• Provide insights for S3 Transfer Acceleration
	• TransferAccelerationEnabledBucketCount (identify which buckets have S3 Transfer Acceleration enabled)
**7• Activity Metrics**
	• Provide insights about how your storage is requested
	• AllRequests, GetRequests, PutRequests, ListRequests, BytesDownloaded…
**8• Detailed Status Code Metrics**
	• Provide insights for HTTP status codes
	• 200OKStatusCount, 403ForbiddenErrorCount, 404NotFoundErrorCount…
### Free vs. Paid

**• Free Metrics**
	• Automatically available for all customers
	• Contains around 28 usage metrics
	• Data is available for queries for 14 days
**• Advanced Metrics and Recommendations**
	• Additional *paid metrics* and features
	• *Advanced Metrics* – Activity, Advanced Cost Optimization, Advanced Data Protection, Status Code
	• *CloudWatch Publishing* – Access metrics in CloudWatch without additional charges
	• *Prefix Aggregation* – Collect metrics at the prefix level
	• Data is available for queries for *15 months*