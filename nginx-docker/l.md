This output changes the situation substantially. You now have enough evidence to identify the likely UAT application VPC:

> `vpc-0b213313d8681a84e` — `CentralInfra-qa-vpc`

This is technically the strongest candidate for Lit Ordering, subject to final network-team approval.

## The six VPCs now make sense

| VPC | Name | Likely purpose |
|---|---|---|
| `vpc-0dd481c82bd81a7ca` | `CentralInfra-Central-vpc` | Central/shared infrastructure |
| `vpc-0196023cb03b20609` | `CentralInfra-dev-vpc` | Development workloads |
| `vpc-0b213313d8681a84e` | `CentralInfra-qa-vpc` | QA/UAT workloads |
| `vpc-0319c899c1d165bf9` | `CentralInfra-Endpoint-vpc` | Central interface endpoints |
| `vpc-00af9ba55a04e6ca4` | `CentralInfra-unclass-vpc` | Unclassified workloads |
| `vpc-00d9df40c81bbc585` | `CentralInfra-prod-vpc` | Production workloads |

Distribution-UAT is a UAT/non-production account. That aligns most closely with:

```text
CentralInfra-qa-vpc
vpc-0b213313d8681a84e
```

## VPCs you can likely eliminate

### Endpoint VPC

```text
vpc-0319c899c1d165bf9
CentralInfra-Endpoint-vpc
```

This is not a general application VPC. Your output shows that its two subnets contain many centralized interface endpoints:

- STS
- SQS
- CloudWatch Monitoring
- DataSync
- Airflow API/environment/operations
- Databricks workspace API
- Databricks secure cluster connectivity
- Grafana
- Snowflake/private endpoint services

That explains why it has only two subnets and many ENIs. Each interface endpoint normally creates one ENI per selected Availability Zone.

Example:

```text
VPC endpoint vpce-0b569b40b99f3f033
Service: STS
    ├── ENI in subnet-0e15356ea8d5b8ea5
    └── ENI in subnet-0fb820406e805e3c6
```

Do not put the Lit Ordering Lambda in this VPC unless Network Services explicitly says the Endpoint VPC also hosts workloads—which would be unusual.

### Central VPC

```text
vpc-0dd481c82bd81a7ca
CentralInfra-Central-vpc
```

Your ENI data shows central infrastructure such as:

- Directory Service ENIs
- Transit Gateway attachment ENIs
- Databricks networking
- Resources owned by central/shared-service accounts

This looks like centralized infrastructure, not the normal UAT application landing zone.

### Development and production VPCs

These are environment-specific:

```text
CentralInfra-dev-vpc
CentralInfra-prod-vpc
```

They should not be selected for Distribution-UAT.

## Evidence that the QA VPC hosts applications

Your ENI data from `vpc-0b213313d8681a84e` includes:

```text
AWS Lambda VPC ENI-qa2-rules-engine-sftp-lambda...
AWS Lambda VPC ENI-sf_metrics-lambda-uat
ELB app/fynsight-mwaa-uat-lb/...
Amazon EKS fynsight-eks-uat
RDSNetworkInterface
ELB app/pi-task-manager-services/...
Directory Service ENI
```

This proves the QA VPC is not merely a networking container. It hosts participant-account application workloads, including:

- Lambda
- EKS
- Application Load Balancers
- RDS/database networking
- Directory Service
- SFTP-related Lambda workloads

That last item is particularly relevant:

```text
qa2-rules-engine-sftp-lambda
```

It strongly suggests another SFTP Lambda already uses the QA VPC—the same general networking shape as Lit Ordering’s TGI SFTP connection.

## The shared-VPC ownership model is now visible

Consider this row:

```text
VPC:
vpc-0b213313d8681a84e

VPC owner:
838540104993

ENI owner:
978219917913

Description:
AWS Lambda VPC ENI-qa2-rules-engine-sftp-lambda...
```

That means:

```text
Network Services account 838540104993
    owns CentralInfra-qa-vpc and its subnets
        ↓ shares subnet through AWS RAM
Participant account 978219917913
    creates a Lambda in that subnet
        ↓
AWS creates a Lambda ENI
        ↓
ENI is owned by participant account 978219917913
        ↓
ENI receives private IP 172.20.114.56
```

Network Services owns the land. The participant owns the workload and its network interface.

## What the account IDs tell you

Your ENI report contains many owner IDs:

```text
123912334908
457090735007
476114131678
978219917913
331225031378
226945379753
...
```

Those are workload-owner accounts using the shared VPCs.

This lets you answer:

```text
Which AWS account consumes this private IP?
Which account created this Lambda ENI?
Which account owns this load balancer?
Which account owns this RDS interface?
```

However, an account ID does not give you the account name. The next useful enhancement is an Organizations account inventory:

```text
account_id → account_name → OU → status
```

Then your correlation report can say:

```text
331225031378
→ Distribution-something/UAT account
→ owns sf_metrics-lambda-uat ENI
→ inside CentralInfra-qa-vpc
```

## Which subnets are potential candidates?

From your previous reports, four QA `/23` subnets have TGW and endpoint routes:

```text
subnet-028ef541ac164b2bd — 172.20.114.0/23
subnet-035e470568c719240 — 172.20.98.0/23
subnet-013b1ab8a2ca02d16 — 172.20.100.0/23
subnet-008c9c8c389564c19 — 172.20.116.0/23
```

The Lit Ordering Lambda needs two approved application subnets in different Availability Zones.

We still should not randomly choose the pair because the `/23` subnets could be paired by application tier, account group or routing policy.

Your evidence suggests these are application-capable:

- `subnet-028ef541ac164b2bd` already hosts an SFTP Lambda.
- `subnet-035e470568c719240` hosts UAT Lambda/EKS/ALB/RDS-related resources.

But those two appear to be in the same physical AZ based on your earlier AZ IDs, so they should not be used together for Lambda high availability. You need one approved subnet from each distinct AZ.

Ask Network Services to identify the intended pair.

## What the endpoint policy means

Many endpoint policies contain:

```json
{
  "Action": "*",
  "Effect": "Allow",
  "Principal": "*",
  "Resource": "*"
}
```

That does not make the endpoint publicly accessible.

It means:

> The endpoint policy itself does not restrict which principals/actions/resources may use the endpoint.

Other controls still apply:

- Endpoint security group
- IAM permissions
- Subnet routing
- Endpoint service acceptance
- Private DNS
- Service-level permissions
- Resource policies

It is a broad endpoint policy and may deserve review, but it is not equivalent to an S3 bucket policy granting public internet access.

## What `PrivateDnsEnabled` means

For an AWS service endpoint such as SQS:

```text
PrivateDnsEnabled = True
```

means a workload resolving:

```text
sqs.us-east-1.amazonaws.com
```

can receive private endpoint IPs instead of using the ordinary public-service path.

When false, the workload generally must use:

- The endpoint-specific DNS name, or
- A separately configured private hosted zone/custom DNS arrangement.

For third-party services such as Databricks or Snowflake, private DNS behavior may be managed through custom PrivateLink DNS configuration.

## Why each endpoint has two ENIs

Example:

```text
vpce-0b569b40b99f3f033 — STS
Subnets:
    subnet-0e15356ea8d5b8ea5
    subnet-0fb820406e805e3c6

ENIs:
    eni-04283e44871e29812
    eni-0969c0ed8615ca9e7
```

AWS creates one endpoint ENI in each selected subnet/AZ:

```text
Client
    ↓ resolves STS private DNS
    ↓
Nearest available endpoint ENI
    ↓
AWS private network
    ↓
STS
```

These ENIs consume private subnet IP addresses.

## Your old empty UAT report now makes perfect sense

From Distribution-UAT, you saw:

```text
0 Lambdas
0 endpoints
0 ENIs
0 security groups
```

From Network Services, you can see participant-owned ENIs across many accounts.

That is because:

```text
Participant account view
    → sees its own workloads

VPC owner view
    → sees network objects and participant ENIs
```

Therefore the UAT report did not prove the VPCs were empty. It proved Distribution-UAT had not yet deployed its own resources there.

## Current migration conclusion

You can now update the Lit Ordering migration plan:

```text
Target VPC candidate:
vpc-0b213313d8681a84e
CentralInfra-qa-vpc

Rejected as normal workload target:
vpc-0319c899c1d165bf9
CentralInfra-Endpoint-vpc

Likely central infrastructure only:
vpc-0dd481c82bd81a7ca
CentralInfra-Central-vpc

Remaining network decision:
Which two QA application subnets are approved for Distribution-UAT?
```

Send the network team:

> Network Services discovery confirms that `vpc-0b213313d8681a84e` is `CentralInfra-qa-vpc` and already hosts participant-owned UAT/QA Lambda, EKS, ALB, RDS and SFTP-related workloads. Is this the approved VPC for Distribution-UAT Lit Ordering? If so, which two shared application subnets in different AZ IDs should the Lambda use? Please also confirm TGW/firewall egress for HTTPS and TGI SFTP TCP 22, the NAT egress IP presented to TGI, and whether Distribution-UAT must create its own security group.

## Run the correlation script next

Run the correlation script against the actual `network_services_discovery` directory:

```powershell
python .\correlate_network_discovery.py
```

Do not run it against the giant pasted-text file. The pasted content is over 200,000 tokens and was truncated while being displayed, but your original CSV files remain complete.

Then filter `network_correlations.csv` by:

```text
vpc_id = vpc-0b213313d8681a84e
```

That produces the complete QA relationship chain. The clearest finding from the data you sent is that `CentralInfra-qa-vpc` is the appropriate technical candidate, while `CentralInfra-Endpoint-vpc` is a centralized endpoint-services VPC rather than the Lit Ordering application destination.
