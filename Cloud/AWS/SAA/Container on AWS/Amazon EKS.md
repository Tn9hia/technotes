**• Amazon EKS = Amazon Elastic Kubernetes Service**
• It is a way to launch **managed Kubernetes clusters on AWS**
• Kubernetes is an open-source system for automatic deployment, scaling and management of containerized (usually Docker) application
• It’s an alternative to ECS, similar goal but different API
• EKS supports **EC2** if you want to deploy worker nodes or **Fargate** to deploy
serverless containers

• **Use case**: if your company is already using Kubernetes on-premises or in another cloud, and wants to migrate to AWS using Kubernetes
• **Kubernetes is cloud-agnostic** (can be used in any cloud – Azure, GCP…)
• For multiple regions, deploy one EKS cluster per region
• Collect logs and metrics using **CloudWatch Container Insights**

![[Pasted image 20260329021058.png]]

## Node Types

**• Managed Node Groups**
	• Creates and manages Nodes (EC2 instances) for you
	• Nodes are part of an ASG managed by EKS
	• Supports On-Demand or Spot Instances
**• Self-Managed Nodes**
	• Nodes created by you and registered to the EKS cluster and managed by an ASG
	• You can use prebuilt AMI - Amazon EKS Optimized AMI
	• Supports On-Demand or Spot Instances
**• AWS Fargate**
	• No maintenance required; no nodes managed
## Data Volumes
• Need to specify **StorageClass** manifest on your EKS cluster
• Leverages a **Container Storage Interface (CSI)** compliant driver
• Support for…
	• Amazon EBS
	• Amazon EFS (works with Fargate)
	• Amazon FSx for Lustre
	• Amazon FSx for NetApp ONTAP
