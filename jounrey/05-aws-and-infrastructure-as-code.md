# 05 — AWS and infrastructure as code: prove every resource and every route

This module turns your AWS investigations into repeatable engineering work.
Finish the local Python/Go and networking foundations before deploying its cloud labs.
Use this material during weeks 21–28 of [08-weekly-plan-and-mentoring.md](08-weekly-plan-and-mentoring.md), with earlier preparation and repeats whenever a gate needs more practice.
All account IDs, address ranges, and resource names in these labs are fictional examples.
Use an explicitly approved sandbox or your own learning account, never an employer production account.

## What you will produce

- A diagram and inventory explaining who owns each account, resource, and network.
- A route evaluator that proves which destination CIDR wins.
- A Python CDK application with infrastructure tests and a documented teardown.
- A small Go CDK equivalent showing that you understand the generated infrastructure.
- A Terraform exercise explaining remote state, imports, drift, and replacement.
- A CI deployment workflow using temporary AWS credentials.
- A recovery report proving that preserved data can actually be restored.

## First, learn the words by connecting them to objects

| Term | Plain meaning | Example |
|---|---|---|
| Account | A boundary for AWS resource ownership, permissions, and billing | Sandbox account owns an S3 bucket |
| Organization | A collection of centrally governed AWS accounts | Management account governs several member accounts |
| OU | Organizational unit: a group of accounts to which policies can apply | `Learning` OU contains two lab accounts |
| Region | A geographic AWS service deployment area | `us-east-1` |
| Availability Zone / AZ | An isolated infrastructure location inside a Region | Two application replicas use different AZs |
| ARN | Amazon Resource Name: a resource identifier with service and scope | An IAM role ARN identifies the role you assume |
| IAM principal | An identity making a request | A human role session or application role |
| IAM role | Permissions that an authorized identity can assume temporarily | A deployment workflow assumes `LabDeployRole` |
| VPC | Your logically isolated AWS network | `10.80.0.0/16` contains the lab subnets |
| Subnet | An address range inside a VPC, in one AZ | `10.80.1.0/24` |
| ENI | Elastic network interface: a virtual network card | EC2 receives its private address on an ENI |
| CIDR | A prefix describing an address range | `/24` contains 256 IPv4 addresses |
| Route table | Rules mapping destination prefixes to next hops | `0.0.0.0/0 → NAT gateway` |
| Security group | Stateful traffic permission rules attached to supported resources | Permit TCP 8080 from the test client |
| NACL | Network ACL: stateless subnet-level packet rules | Return traffic needs its own matching allowance |
| IaC | Infrastructure as code: versioned configuration that provisions resources | A Python CDK program declares a queue |
| Drift | Deployed configuration diverging from managed configuration | Someone changes a security group through the console |
| Control plane | Interfaces that configure resources | `CreateVpc` API call |
| Data plane | Interfaces carrying application traffic or data | HTTP request through the VPC |

The AWS console is useful for investigation and verification.
Learn to read it accurately, then make repeatable changes through reviewed code.
Console navigation is not a substitute for understanding what an API call changes.

## Lab 0 — Build your account and cost boundary

**Goal:** know exactly where a command will run and what can continue billing afterward.

1. Pick one sandbox account and one Region; record both in `lab-environment.md`.
2. Use IAM Identity Center or another approved temporary role session.
3. Run `aws sts get-caller-identity --profile journey-sandbox` before deployment.
4. Compare its account and role with your written expected values.
5. Record installed CLI, Python, Go, CDK, and Terraform versions.
6. Create a budget notification with an amount you can actually afford.
7. Verify the notification destination and understand that a budget is not a hard spending cap.
8. Set tags such as `Project=journey`, `Owner=<your-team>`, and `ExpiresOn=<date>`.
9. Decide which resources are disposable and which must be retained before writing a destroy command.

Record estimated cost as a formula: unit price × expected units × expected duration.
Use the current AWS Pricing Calculator rather than copying an old dollar estimate.
Include NAT gateway hours/data, interface endpoints, public IPv4 addresses, logs, EBS,
load balancers, EKS control plane, and retained snapshots when those resources are present.
Budgets and billing data can be delayed; finish each lab by checking the resource inventory.

**Gate:** explain your account ID, assumed role, Region, cost ceiling, and teardown scope aloud.

## Lab 1 — Make an ownership inventory that does not invent owners

Create a CSV with these columns:

```text
account_id,account_name,region,resource_id,resource_type,resource_owner_account,
network_owner_account,application,technical_owner,business_owner,creator_principal,
creator_event_time,evidence,confidence,activity_window,migration_decision
```

The resource-owning account, network-sharing account, original creator, and current
business owner can all differ. An automation role that created a server is not its business owner.
An OU named `Legacy` does not mean the resources are unused.

1. In Organizations, identify the management account, OU placement, and account tags.
2. In a member account, inspect EC2, Lambda, S3, RDS, and CloudFormation resources.
3. Record a resource's tags and any CloudFormation stack relationship.
4. If a subnet is shared, inspect its owner ID and RAM share.
5. Inspect a retained CloudTrail creation event, if one exists.
6. Distinguish `userIdentity.arn`, the assumed-role session issuer, and an AWS service identity.
7. Record what you cannot establish as `unknown`, with the next evidence source to check.

CloudTrail Event history is limited to recent management events; historical trails/Lake
must already retain older events to establish older creation evidence.
CloudWatch metrics describe measured activity, not business importance.
A stopped server with a unique disk can be important even with no recent CPU activity.

**Deliverable:** ten resources grouped into applications, with evidence for every grouping.
**Gate:** another person can distinguish a fact from your inference without asking you.

## Lab 2 — Prove routing locally before buying networking resources

Use this fictional route table:

```text
Client address:       10.80.1.25
Target file service:  10.90.130.185

10.80.0.0/16       → local
10.90.0.0/21       → private-transit
10.90.128.0/19     → private-transit
0.0.0.0/0         → egress-transit
```

`10.90.128.0/19` covers `10.90.128.0` through `10.90.159.255`.
It includes `10.90.130.185`; `10.90.0.0/21` does not.
The `/19` wins over `/0` because the more specific matching prefix wins.
If you remove the `/19`, the target follows `/0`; an attachment existing elsewhere does not change this.
AWS documents this as [longest prefix match](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html).

Write a Python function using `ipaddress.ip_network()` and `ipaddress.ip_address()`.
Return all matching routes sorted by prefix length, then identify the winner.
Do not implement same-prefix AWS tie-breaking by guessing; flag ambiguous equal-prefix inputs.

Required tests:

- Destination inside the `/19` chooses private transit.
- Removing `/19` chooses default egress.
- Destination inside the local prefix chooses local.
- `/32` is a route for one address and beats broader matching routes.
- Invalid input fails clearly.
- No route returns `unreachable`, not a fabricated default.

**Gate:** solve ten unfamiliar CIDR examples correctly and explain why each winning route wins.

## Lab 3 — Deploy the smallest useful private network

Start with one VPC and two isolated subnets; do not add NAT, TGW, or EKS yet.
An isolated subnet has no intended internet route; it can still have routes to private resources.
Create test connectivity only when you have a deliberate management access path.
Session Manager may require internet egress or suitable VPC endpoints plus IAM and agent configuration.
Endpoint charges can exceed the cost of a short-lived tiny instance, so compare designs first.

When deploying two test instances, use a simple HTTP service on TCP 8080 and synthetic data.
The client security group permits the required outbound request.
The server security group permits inbound TCP 8080 from the client group.
Do not open SSH or RDP to the internet just to make this lab work.

```text
LAB account / one Region
  VPC 10.80.0.0/16
    client subnet → client ENI → client SG
    server subnet → server ENI → server SG → HTTP service :8080
```

Read the instance's subnet association to find the actual route table.
Then inspect the destination, winning route, ENIs, security groups, and NACLs.
Test DNS separately from TCP and TCP separately from application responses.
An ICMP ping failure does not establish that TCP 8080 or SMB 445 is blocked.

**Failure drill:** remove only the lab server's TCP 8080 allowance through the lab IaC.
Predict the result, run the test, restore the rule, and record evidence.
**Gate:** distinguish DNS failure, timeout, connection refusal, and HTTP error.

## Lab 4 — Draw cross-account networking before deploying it

Use a paper design or local route simulator first; TGW labs incur attachment and processing charges.

```text
Application account A
  source subnet route → TGW owned by Network account N
                          ingress attachment association
                            → associated TGW route table
                              → destination attachment
                                  Shared subnet owned by N
                                    Service owned by account B
```

An attachment connects a VPC or another network to a TGW.
An association selects which TGW route table evaluates traffic entering on that attachment.
Propagation supplies learned routes to selected TGW route tables; it does not mean every table receives them.
TGW peering connects two TGWs; VPC peering connects VPCs. Neither term means IAM trust or RAM sharing.
RAM makes a supported resource available to another account; it does not itself create a packet route.

Trace both directions independently, using destination addresses appropriate to each direction.
For SMB, the client initiates `client:ephemeral-port → server:445`.
The response is `server:445 → client:ephemeral-port`.
Stateful security groups allow response traffic for an allowed tracked connection;
stateless NACLs and intermediate inspection still need a valid response path.

**Gate:** explain why a valid return route does not prove the client's forward route is valid.

## Lab 5 — IAM as request evaluation, not a pile of attached policies

For each action write: principal, action, resource, and request conditions.
Example: worker role, `s3:GetObject`, only the input bucket prefix, approved account context.
Authentication establishes identity; authorization determines whether that identity may act.
A trust policy controls who can assume a role; role permissions control what its sessions can do.
SCPs constrain member-account permissions; they do not grant permissions by themselves.
Permissions boundaries are also ceilings, not grants.
Explicit deny and cross-account/resource-policy rules matter; study the full
[IAM policy evaluation model](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html).

1. Create a lab reader role restricted to one synthetic bucket prefix.
2. Prove reading an approved object succeeds.
3. Prove reading another prefix fails.
4. Prove writing fails.
5. Record the role session ARN and denial evidence without publishing credentials.
6. Add a cross-account exercise only after explaining both trust and access authorization.

**Gate:** diagnose an access denial with a written request tuple and the relevant policy layers.

## Lab 6 — Python CDK: own the infrastructure your program generates

CDK code synthesizes a CloudFormation template; CloudFormation performs the deployment.
A construct is a reusable infrastructure building block; a stack is a deployment unit.
An environment is an account/Region pair.
Bootstrapping creates deployment support resources such as asset storage and roles;
it is an AWS change with permissions and lifecycle implications.
Read [CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html) before executing it.

In an empty project directory, initialize `cdk init app --language python`.
Activate the generated virtual environment and install its declared dependencies.
First build a private S3 bucket, SQS queue, DLQ, and a narrowly permitted worker role.
A DLQ, or dead-letter queue, holds messages that repeatedly fail processing for later investigation.
Set explicit log retention, encryption choices, tags, and removal policies.
Avoid creating a VPC unless the workload actually needs private network resources.

For each iteration:

1. Write an assertion for the intended generated resource property.
2. Run tests; inspect the failure before implementing the property.
3. Run `cdk synth` and read the resulting template.
4. Run `cdk diff --profile journey-sandbox` and explain replacements and permission changes.
5. Deploy the specific lab stack after verifying the account/Region.
6. Validate behavior with a synthetic integration test.
7. Run a second diff; explain any unexpected change.

Assertions must check public access settings, queue encryption, retention choices, and IAM scope.
Template snapshot tests alone can approve an unsafe change if you blindly update the baseline.
Use [CDK fine-grained assertions](https://docs.aws.amazon.com/cdk/v2/guide/testing.html) for meaningful properties.

**Gate:** explain every generated resource and destroy/retain decision in your stack.

## Lab 7 — Go CDK and Terraform comparison

Rebuild only the bucket-and-queue component in Go CDK, in a separate lab stack.
Use Go composition and small factory functions instead of copying Python inheritance patterns.
Learn why `jsii.String`, optional pointers, and generated interfaces appear in CDK Go.
Compare synthesized templates and costs, not just line counts.
Follow the official [CDK Go guide](https://docs.aws.amazon.com/cdk/v2/guide/work-with-cdk-go.html).

Next build a separate disposable resource with Terraform; never give CDK and Terraform ownership of the same object.
Terraform state maps configuration addresses to actual resource IDs.
The backend stores that state; a provider translates resource operations into service API calls.
State can contain sensitive data and needs access control, versioning, and supported locking.
Do not commit state or credential files. Read [Terraform state](https://developer.hashicorp.com/terraform/language/state).

Practice `fmt`, `validate`, `plan`, reviewed `apply`, and reviewed `destroy`.
Change one lab property outside Terraform, then inspect a new plan to understand drift.
Import one separately created disposable resource into a single configuration address.
Explain that import does not automatically give you a good resource design or complete configuration.
Demonstrate a replacement in a plan without applying it until you understand data consequences.

**Gate:** explain configuration, state, live resource, drift, import, and replacement using one concrete object.

## Lab 8 — Storage preservation and deployment delivery

An EBS volume is a block device; an EBS snapshot is a recovery point for block data.
S3 stores objects; an ordinary S3 bucket is not where you manually place native EBS snapshots.
An AMI combines launch metadata with referenced disk snapshots; copying snapshots alone does not copy all launch configuration.
For a sandbox migration, copy synthetic snapshots into a destination account and verify destination ownership.
Restore a volume, inspect its contents, and compare hashes before declaring preservation successful.
Keep source-to-destination IDs and metadata in a migration report.
For encrypted copies, verify source and destination KMS permissions and key availability.

Build CI stages: language tests → IaC assertions → synth/plan → review → deployment → integration test.
CI means continuous integration: automated validation of code changes.
CD means continuous delivery/deployment: preparing or releasing validated changes.
Use a narrowly trusted OIDC role for hosted CI instead of repository-stored access keys.
OIDC lets AWS validate the workflow's short-lived identity token under specified trust conditions.

**Gate:** reproduce the lab from a clean checkout and document how to restore data after failure.

## Exit evidence and teardown

- Diagram includes account, Region, VPC, subnet, owner, route-table association, and next hop.
- Route tests demonstrate `/32`, `/19`, and default behavior.
- IAM tests demonstrate both allowed and denied operations.
- CI produces test evidence and a reviewable infrastructure change.
- Restore test validates content, not merely a `completed` snapshot status.
- Teardown targets only your named lab stacks; review retained buckets, snapshots, logs, ECR assets, and bootstrap resources separately.
- Your final report explains remaining charges and retained resources.

Use [AWS Pricing Calculator](https://calculator.aws/) before every paid topology change.
Use [AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) to ask what your design trades off.
Your mentor review question: “Show me one claim you initially got wrong and the evidence that corrected it.”
