## Overview
• Docker is a software development platform to deploy apps
• Apps are packaged in containers that can be run on any OS
• Apps run the same, regardless of where they’re run
	• Any machine
	• No compatibility issues
	• Predictable behavior
	• Less work
	• Easier to maintain and deploy
	• Works with any language, any OS, any technology
• **Use cases**: microservices architecture, lift-and-shift apps from on-premises to the AWS cloud, …

## Registry
• Docker images are stored in Docker Repositories

• Docker Hub (https://hub.docker.com)
	• Public repository
	• Find base images for many technologies or OS (e.g., Ubuntu, MySQL, …)
• Amazon ECR (Amazon Elastic Container Registry)
	• Private repository
	• Public repository (Amazon ECR Public Gallery https://gallery.ecr.aws)

## Docker Containers Management on AWS
 [[Amazon ECS | Amazon Elastic Container Service (Amazon ECS)]]
	• Amazon’s own container platform
[[AWS Fargate | AWS Fargate]]
	• Amazon’s own Serverless container platform
	• Works with ECS and with EKS
[[Amazon ECR | Amazon ECR]]
	• Store container images
[[Amazon EKS | Amazon Elastic Kubernetes Service]]
	• Amazon’s managed Kubernetes (open source)

## AWS App Runner
• Fully managed service that makes it easy to deploy web applications and APIs at scale
• No infrastructure experience required
• Start with your source code or container image
• Automatically builds and deploy the web app
• Automatic scaling, highly available, load balancer, encryption
• VPC access support
• Connect to database, cache, and message queue services

• **Use cases**: web apps, APIs, microservices, rapid production deployments

## AWS App2Container (A2C)
• CLI tool for migrating and modernizing **Java** and .**NET** web apps into Docker Containers
• **Lift-and-shift** your apps running in on-premises bare metal, virtual machines, or in any Cloud to AWS
• Accelerate modernization, no code changes, migrate legacy apps…
• Generates CloudFormation templates (compute, network…)
• Register generated Docker containers to ECR
• Deploy to ECS, EKS, or App Runner
• Supports pre-built CI/CD pipelines

![[Pasted image 20260329021635.png]]