## Overview
**• Content Delivery Network (CDN)**
• Improves read performance, content is cached at the edge
• Improves users experience
• Hundreds of Points of Presence globally (edge locations, caches)
**• DDoS protection (because worldwide), integration with Shield, AWS Web Application Firewall** 
## Origin
**• S3 bucket**
	• For distributing files and caching them at the edge
	• For uploading files to S3 through CloudFront
• Secured using Origin Access Control (OAC)
**• VPC Origin**
	• For applications hosted in VPC private subnets
	• Private Application Load Balancer / Network Load Balancer / EC2 Instances
	![[Pasted image 20260318012437.png]]
**• Custom Origin (HTTP)**
	• S3 website (must first enable the bucket as a static S3 website)
	• Any public HTTP backend you want (example: Public ALB)
## CloudFront vs S3 Cross Region Replication
**• CloudFront:**
	• Global Edge network
	• Files are cached for a TTL (maybe a day)
	• Great for static content that must be available everywhere
**• S3 Cross Region Replication:**
	• Must be setup for each region you want replication to happen
	• Files are updated in near real-time
	• Read only
	**• Great for dynamic content that needs to be available at low-latency in few regions**
## CloudFront Geo Restriction
• You can restrict who can access your distribution
	• **Allowlist**: Allow your users to access your content only if they're in one of the countries on a list of approved countries.
	• **Blocklist**: Prevent your users from accessing your content if they're in one of the countries on a list of banned countries.
• The “country” is determined using a 3rd party Geo-IP database
• **Use case**: Copyright Laws to control access to content

## Cache Invalidations
• In case you update the back-end origin, CloudFront doesn’t know about it and will only get the refreshed content after the TTL has expired
• However, you can force an entire or partial cache refresh (thus bypassing the TTL) by performing a **CloudFront Invalidation**
• You can invalidate all files (\*) or a special path (/images/\*)