## What’s serverless?
• Serverless is a new paradigm in which the developers don’t have to manage servers anymore…
• They just deploy code
• They just deploy… functions !
• Initially... Serverless == FaaS (Function as a Service)
• Serverless was pioneered by AWS Lambda but now also includes anything that’s managed: “databases, messaging, storage, etc.”
• Serverless does not mean there are no servers… it means you just don’t manage / provision / see them

**Example:**
• AWS Lambda
• DynamoDB
• AWS Cognito
• AWS API Gateway
• Amazon S3
• AWS SNS & SQS
• AWS Kinesis Data Firehose
• Aurora Serverless
• Step Functions
• Fargate
## AWS Lambda
[[AWS Lambda Function | Read here]]

## Amazon DynamoDB
[[DynamoDB  | Read here]]

## API Gateway 
[[AWS API Gateway | Read here]]

## AWS Step Functions
• Build serverless visual workflow to orchestrate your Lambda functions
• **Features**: sequence, parallel, conditions, timeouts, error handling, …
• Can integrate with EC2, ECS, On-premises servers, API Gateway, SQS queues, etc…
• Possibility of implementing human approval feature
**• Use cases:** order fulfillment, data processing, web applications, any workflow

![[Pasted image 20260329170810.png]]
### Hai loại workflow chính

| |**Standard**|**Express**|
|---|---|---|
|Duration|Up to 1 year|Up to 5 phút|
|Execution|Exactly-once|At-least-once|
|Audit|Full history|CloudWatch only|
|Price|Per state transition|Per duration + invocation|
|Use case|Long-running business flows|High-volume, short tasks|
## Amazon Cognito
• Give users an identity to interact with our web or mobile application
**• Cognito User Pools:**
	• Sign in functionality for app users
	• Integrate with API Gateway & Application Load Balancer
	
**• Cognito Identity Pools (Federated Identity):**
	• Provide AWS credentials to users so they can access AWS resources directly
	• Integrate with Cognito User Pools as an identity provider
	
**• Cognito vs IAM**: “hundreds of users”, ”mobile users”, “authenticate with SAML”

### Cognito User Pools (CUP)
#### User Features
**• Create a serverless database of user for your web & mobile apps**
• Simple login: Username (or email) / password combination
• Password reset
• Email & Phone Number Verification
• Multi-factor authentication (MFA)
• Federated Identities: users from Facebook, Google, SAML…
• CUP integrates with **API Gateway** and **Application Load Balancer**
#### Integrations
![[Pasted image 20260329171133.png]]


#### Cognito Identity Pools (Federated Identities)
**• Get identities for “users” so they obtain temporary AWS credentials**
• Users source can be Cognito User Pools, 3rd party logins, etc…
• Users can then access AWS services directly or through API Gateway
• The IAM policies applied to the credentials are defined in Cognito
• They can be customized based on the user_id for fine grained control
• **Default IAM roles** for authenticated and guest users

![[Pasted image 20260329172822.png]]


