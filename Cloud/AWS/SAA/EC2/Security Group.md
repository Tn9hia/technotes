## Overview
- Security Groups are the fundamental of network security in AWS
- They **control how traffic is allowed into or out of our EC2 Instances**
- Security groups only contain **allow** rules
- Security groups rules can reference by IP or by security group
- Security groups are acting as a “**firewall**” on EC2 instances
- They regulate:
	- Access to Ports
	- Authorised IP ranges – IPv4 and IPv6
	- Control of inbound network (from other to the instance)
	- Control of outbound network (from the instance to other)
![[Pasted image 20260315104946.png]]

> [!note]
> Image Security group as NSX-T firewall in VMware
### Good to know
- Can be **attached to multiple instances**
- Locked down to a region / VPC combination
- Does live “**outside**” the EC2 – if traffic is blocked the EC2 instance won’t see it
- **It’s good to maintain one separate security group for SSH access**
- If your application is not accessible (time out), then it’s a security group issue
- If your application gives a “connection refused“ error, then it’s an application error or it’s not launched
- All inbound traffic is **blocked** by default
- All outbound traffic is **authorised** by default

> [!warning]
> - EC2 can have multiple Security Group
> 
> - Rules from all Security Groups are merged using a union approach, meaning the most permissive rule ultimately applies.
