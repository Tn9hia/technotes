## What is Scalability ?
• Scalability means that an application / system can handle greater loads by adapting.
• There are two kinds of scalability:
	• **Vertical Scalability**: increase the size of the instance
	• **Horizontal Scalability** (= elasticity): increase the number of instances / systems for your
application
**• Scalability is linked but different to High Availability**
• Let’s deep dive into the distinction, using a call center as an example

## What is High Availability ?
High availability means running your application / system in at least 2 data centers (== Availability Zones) => survive when a data center loss
- The high availability can be passive (for RDS Multi AZ for example)
- The high availability can be active (for horizontal scaling)