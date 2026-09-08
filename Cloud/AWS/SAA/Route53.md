• A highly available, scalable, fully managed and Authoritative DNS
• Authoritative = the customer (you) can update the DNS records
• Route 53 is also a Domain Registrar
• Ability to check the health of your resources
• The only AWS service which provides 100% availability SLA

## Route 53 – Hosted Zones
• A container for records that define how to route traffic to a domain and
its subdomains
• **Public Hosted Zones** – contains records that specify how to route traffic on the Internet (public domain names)
application1.mypublicdomain.com
• **Private Hosted Zones** – contain records that specify how you route traffic within one or more VPCs (private domain names) application1.company.internal
• You pay $0.50 per month per hosted zone

## CNAME vs Alias
CNAME:
	• Points a hostname to any other hostname. (app.mydomain.com => blabla.anything.com)
	**• ONLY FOR NON ROOT DOMAIN (aka. something.mydomain.com)**
Alias:
	• Points a hostname to an AWS Resource (app.mydomain.com => blabla.amazonaws.com)
	**• Works for ROOT DOMAIN and NON ROOT DOMAIN (aka mydomain.com)**
	• Free of charge
	• Native health check

## Alias Records Targets
• Elastic Load Balancers
• CloudFront Distributions
• API Gateway
• Elastic Beanstalk environments
• S3 Websites
• VPC Interface Endpoints
• Global Accelerator accelerator
• Route 53 record in the same hosted zone
**You cannot set an ALIAS record for an EC2 DNS name**
## Routing Policies
• Define how Route 53 responds to DNS queries
• Route 53 Supports the following Routing Policies
	• **Simple**:  If multiple values are returned, a random one is chosen by the client
		• Can be associated with Health Checks
	• **Weighted**: Control the % of the requests that go to each specific resource
		• Assign a weight of 0 to a record to stop sending traffic to a resource
		• If all records have weight of 0, then all records will be returned equally
	**• Failover**: route to primary and failover to secondary when primary fail.
	**• Latency based**: Redirect to the resource that has the least latency close to us
		• Latency is based on traffic between users and AWS Regions
		• Can be associated with Health Checks (has a failover capability)
	**• Geolocation**: This routing is based on user location (Should create a “**Default**” record)
	**• Multi-Value Answer**: Use when routing traffic to multiple resources (*Multi-Value is not a substitute for having an ELB*)
	• **Geoproximity** (using Route 53 **Traffic Flow** feature): Route traffic to your resources based on the geographic location of users and resources => Ability to **shift more traffic to resources based** on the defined **bias**
	**• IP-based Routing**: Routing is based on clients’ IP addresses => **You provide a list of CIDRs for your clients** and the corresponding endpoints/locations (user-IP-to-endpoint mappings)
## Health Checks
• HTTP Health Checks are only for public resources
• Health Check => Automated DNS Failover
### Calculated Health Checks
• Combine the results of multiple Health Checks into a single Health Check
• You can use **OR, AND, or NOT**
• Can monitor up to 256 Child Health Checks
• Specify how many of the health checks need to pass to make the parent pass
• **Usage**: perform maintenance to your website without causing all health checks to fail
![[Pasted image 20260316233214.png]]
### Monitor an Endpoint
**• About 15 global health checkers will check the endpoint health**
	• Healthy/Unhealthy Threshold – 3 (default)
	• Interval – 30 sec (can set to 10 sec – higher cost)
	• Supported protocol: HTTP, HTTPS and TCP
	• If > 18% of health checkers report the endpoint is healthy, Route 53 considers it Healthy. Otherwise, it’s Unhealthy
	• Ability to choose which locations you want Route 53 to use
• Health Checks pass only when the endpoint responds with the 2xx and 3xx status codes
• Health Checks can be setup to **pass / fail based** on the **text** in the first **5120** bytes of the response
• Configure you router/firewall to allow incoming requests from Route 53 Health Checkers

### Private Hosted Zones
• Route 53 health checkers are outside the VPC => They can’t access **private** endpoints
• You can create a **CloudWatch Metric** and associate a **CloudWatch Alarm**, then create a **Health Check** that checks the alarm itself


## Route 53 Resolver Endpoints 

**Core problem nó giải quyết:**

Mặc định, DNS resolution trong AWS VPC dùng Route 53 Resolver (169.254.169.253). Còn on-prem dùng DNS server riêng. Hai thằng này không nói chuyện được với nhau → private hostname của bên nào bên đó tự resolve, không cross được.

Resolver Endpoints là cái bridge để fix điều đó.

**Hai loại Endpoints:**

- **Inbound Endpoint** — On-prem DNS forward query _vào_ AWS. VD: `db.internal.company.com` → forward đến AWS để resolve private hosted zone.
- **Outbound Endpoint** — AWS VPC forward query _ra_ on-prem. VD: `legacy.corp.local` → forward đến DNS server on-prem.

Kết hợp cả hai → full bidirectional hybrid DNS resolution.**Chi tiết kỹ thuật cần nhớ:**

- Mỗi endpoint cần **ít nhất 2 ENI** ở 2 AZ khác nhau (HA by design, không optional)
- ENI sẽ có IP riêng trong VPC subnet → on-prem forward DNS đến IP này
- **Resolver Rules** (Forward Rule) define: domain nào → forward đến IP nào
- Rules có thể share qua **RAM (Resource Access Manager)** → dùng chung cho nhiều account trong Organization

**Inbound Endpoint — use case:**

```
On-prem DNS server
  └─ forward zone "aws.internal" → <Inbound EP IP>:53
       └─ Route 53 Resolver resolves Private Hosted Zone
            └─ trả kết quả về on-prem
```

**Outbound Endpoint — use case:**

```
EC2 trong VPC query "db.corp.local"
  └─ Resolver check Forwarding Rules
       └─ match "corp.local" → forward → On-prem DNS 10.0.0.53
            └─ on-prem resolve và trả về
```

**Security notes (đừng bỏ qua):**

- ENI của Resolver Endpoint phải gắn Security Group → **restrict port 53 UDP/TCP** chỉ từ dải IP on-prem, không để 0.0.0.0/0
- Traffic đi qua Direct Connect hoặc VPN — không có nghĩa là mày trust everything. DNS poisoning vẫn là attack vector
- Nếu dùng DNSSEC, R53 Resolver hỗ trợ validation — nên bật nếu on-prem domain sensitive

**Pricing reality check:**

- ~$0.125/endpoint/hour × 2 endpoints × 2 ENI mỗi cái = không rẻ nếu mày deploy nhiều VPC
- Consider dùng **centralized DNS VPC** rồi share rules qua RAM thay vì deploy endpoint per VPC