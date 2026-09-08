## What is IAM
**IAM** = Identity and Access Management, Global service
- **Root account** created by default, shouldn’t be used or shared
- **Users** are people within your organization, and can be grouped
- **Groups** only contain users, not other groups
- Users don’t have to belong to a group, and user can belong to multiple groups
![[Pasted image 20260313101428.png]]

## IAM: Permissions
- **Users or Groups** can be assigned JSON documents called policies
- These policies define the **permissions** of the users
- In AWS you apply **the least privilege principle**: don’t give more permissions than a user needs
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "elasticloadbalancing:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

## IAM Policies Structure

![[Pasted image 20260314014121.png]]
*Consists of*
- Version: policy language version, always include “2012-10-17”
- Id: an identifier for the policy (optional)
- Statement: one or more individual statements (required)
*Statements consists of*
- Sid: an identifier for the statement (optional)
- Effect: whether the statement allows or denies access (Allow, Deny)
- Principal: account/user/role to which this policy applied to
- Action: list of actions this policy allows or denies
- Resource: list of resources to which the actions applied to
- Condition: conditions for when this policy is in effect (optional)

## IAM features
- Password policy
- MFA (Virtual MFA device, Universal 2nd Factor (U2F) Security Key, Hardware Key Fob MFA Device)
- IAM Credentials Report (account-level)
- IAM Access Advisor (user-level)
## How to access AWS
To access AWS, you have three options:
- **AWS Management Console** (protected by password + MFA)
- **AWS Command Line Interface (CLI)**: protected by access keys
- **AWS Software Developer Kit (SDK)** - for code: protected by access keys