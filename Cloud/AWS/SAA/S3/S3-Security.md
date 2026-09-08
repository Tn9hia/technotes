## Amazon S3 – Object Encryption
• You can encrypt objects in S3 buckets using one of 4 methods

**• Server-Side Encryption (SSE)**
	• **Server-Side Encryption with Amazon S3-Managed Keys (SSE-S3)** – *Enabled by Default*
		• Encrypts S3 objects using keys handled, managed, and owned by AWS
	• **Server-Side Encryption with KMS Keys stored in AWS KMS (SSE-KMS)**
		• Leverage AWS Key Management Service (AWS KMS) to manage encryption keys
	• **Server-Side Encryption with Customer-Provided Keys (SSE-C)**
		• When you want to manage your own encryption keys
**• Client-Side Encryption**
• It’s important to understand which ones are for which situation for the exam

### 1. Server-Side Encryption with Amazon S3-Managed Keys (SSE-S3)
• Encryption using keys handled, managed, and owned by AWS
• Object is encrypted server-side
• Encryption type is **AES-256**
• Must set header "**x-amz-server-side-encryption": "AES256"**
**• Enabled by default for new buckets & new objects**
![[Pasted image 20260318003405.png]]
### 2. Server-Side Encryption with KMS Keys stored in AWS KMS (SSE-KMS)
• Encryption using keys handled and managed by **AWS KMS (Key Management Service)**
• KMS advantages: user control + audit key usage using CloudTrail
• Object is encrypted server side
• Must set header **"x-amz-server-side-encryption": "aws:kms"**

**Limitation:**
• If you use SSE-KMS, you may be impacted by the KMS limits
• When you upload, it calls the **GenerateDataKey KMS API**
• When you download, it calls the **Decrypt KMS API**
• Count towards the **KMS quota per second** (5500, 10000, 30000 req/s based on region)
• You can request a quota increase using the **Service Quotas Console**

![[Pasted image 20260318003419.png]]
### 3. Server-Side Encryption with Customer-Provided Keys (SSE-C)
• Server-Side Encryption using keys fully managed by the customer outside of AWS
• Amazon S3 does **NOT** store the encryption key you provide
**• HTTPS must be used**
• Encryption key must provided in HTTP headers, for every HTTP request made

![[Pasted image 20260318003736.png]]

### 4. Client-Side Encryption
• Use client libraries such as **Amazon S3 Client-Side Encryption Library**
• Clients must encrypt data themselves before sending to Amazon S3
• Clients must decrypt data themselves when retrieving from Amazon S3
• Customer fully manages the keys and encryption cycle

![[Pasted image 20260318003827.png]]

### 5. Encryption in transit (SSL/TLS)
• Encryption in flight is also called SSL/TLS
• Amazon S3 exposes two endpoints:
• HTTP Endpoint – non encrypted
• HTTPS Endpoint – encryption in flight
**• HTTPS is recommended**
**• HTTPS is mandatory for SSE-C**
• Most clients would use the HTTPS endpoint by default
## Force Encryption in Transit aws:SecureTransport
![[Pasted image 20260318003958.png]]

## Default Encryption vs. Bucket Policies
**• SSE-S3 encryption is automatically applied to new objects stored in S3 bucket**
• Optionally, you can “force encryption” using a bucket policy and refuse any API call to PUT an S3 object without encryption headers (SSE-KMS or SSE-C)
![[Pasted image 20260318004126.png]]
**Note**: Bucket Policies are evaluated before “Default Encryption”
## Amazon S3 – CORS
• If a client makes a cross-origin request on our S3 bucket, we need to enable the correct CORS headers
**• It’s a popular exam question**
• You can allow for a specific origin or for * (all origins)
• Any request made to S3, from any account, authorized or denied, will be logged into another S3 bucket
• That data can be analyzed using data analysis tools…
• The target logging bucket must be in the same AWS region
## MFA Delete
• **MFA (Multi-Factor Authentication)** – force users to generate a code on a device (usually a mobile phone or hardware) before doing important operations on S3
• MFA will be required to:
	• **Permanently delete** an object version
	• **Suspend Versioning** on the bucket
• MFA won’t be required to:
	• Enable Versioning
	• List deleted versions
• To use MFA Delete, **Versioning must be enabled** on the bucket
**• Only the bucket owner (root account) can enable/disable MFA Delete**

## S3 Access Logs
• Any request made to S3, from any account, authorized or denied, will be logged into another S3 bucket
• That data can be analyzed using data analysis tools…
• The target logging bucket must be in the same AWS region

## S3 Access Logs: Warning
• Do not set your logging bucket to be the monitored bucket
• It will create a logging loop, and your **bucket will grow exponentially**

## Pre-Signed URLs
Amazon S3 – Pre-Signed URLs
• Generate pre-signed URLs using the **S3 Console, AWS CLI or SDK**
**• URL Expiration**
	• S3 Console – 1 min up to 720 mins (12 hours)
	• AWS CLI – configure expiration with --expires-in parameter in seconds (default 3600 secs, max. 604800 secs ~ 168 hours)
• Users given a pre-signed URL inherit the permissions of the user
that generated the URL for GET / 

## S3 Glacier Vault Lock
• Adopt a WORM (Write Once Read Many) model
• Create a Vault Lock Policy
• Lock the policy for future edits (can no longer be changed or deleted)
• Helpful for compliance and data retention

## S3 Object Lock (versioning must be enabled)
• Adopt a WORM (Write Once Read Many) model
• Block an object version deletion for a specified amount of time
**• Retention mode - Compliance:**
	• Object versions can't be overwritten or deleted by any user, including the root user
	• Objects retention modes can't be changed, and retention periods can't be shortened
**• Retention mode - Governance:**
	• Most users can't overwrite or delete an object version or alter its lock settings
	• Some users have special permissions to change the retention or delete the object
**• Retention Period:** protect the object for a fixed period, it can be extended
**• Legal Hold:**
	• protect the object indefinitely, independent from retention period
	• can be freely placed and removed using the s3:PutObjectLegalHold IAM permission
## Access Points

![[Pasted image 20260318010221.png]]

• Access Points simplify security management for S3 Buckets
• Each Access Point has:
	• its own DNS name (Internet Origin or VPC Origin)
	• an access point policy (similar to bucket policy) – manage security at scale
### VPC Origin
• We can define the access point to be accessible only from within the VPC
• You must create a VPC Endpoint to access the Access Point (Gateway or Interface Endpoint)
• The VPC Endpoint Policy must allow access to the target bucket and Access Point

![[Pasted image 20260318010428.png]]

## Object Lambda
• Use AWS Lambda Functions to change the object before it is retrieved by the caller application
• Only one S3 bucket is needed, on top of which we create **S3 Access Point** and **S3 Object Lambda Access Points.**
• Use Cases:
	• Redacting personally identifiable information for analytics or non-production environments.
	• Converting across data formats, such as converting XML to JSON.
	• Resizing and watermarking images on the fly using caller-specific details, such as the user who requested the object.

![[Pasted image 20260318010708.png]]