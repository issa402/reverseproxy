# AWS Organization Discovery, Ownership, Tagging, Cost Allocation, and Rubrik Security Cloud Runbook

Date prepared: 2026-09-21

## Purpose

This runbook is for discovering what exists across an AWS Organization, determining who owns it, repairing tagging and cost allocation, confirming the available audit history, and planning AWS workload protection with Rubrik Security Cloud (RSC).

It is intentionally divided into discovery, evidence, remediation, and protection. Do not treat an OU move, a tag, or a backup policy as proof that a workload is safe to retire.

## The five meanings of "owner"

| Term | What it actually means | Where to verify it |
|---|---|---|
| AWS account owner | The AWS account that owns the resource | Resource `OwnerId`, ARN account segment, or resource console |
| RAM owner | The account that owns and shares a resource through AWS RAM | Resource Access Manager > Shared by me / Shared with me |
| Creator/actor | The identity that called the create API | CloudTrail `userIdentity`, session issuer, source identity, and event details |
| Business owner | The team accountable for the application and its data | CMDB/service catalog, account registry, approved tags, manager/application-owner confirmation |
| Technical owner | The team that operates the workload | IaC repository, deployment role, support group, runbook, on-call ownership |

These values may all be different. A CloudFormation role can create an EC2 instance in an application account, using a VPC shared by a networking account, for an application owned by a business team and operated by a platform team.

## Conceptual architecture

```text
AWS Organizations management account
|
+-- Root
|   +-- Security OU
|   |   +-- Security tooling / delegated administrator account
|   |   +-- Log archive account (often, but not always, separate)
|   +-- Infrastructure OU
|   |   +-- Network account (owns TGWs, VPCs, RAM shares)
|   |   +-- Shared services account
|   +-- Workloads OU
|   |   +-- Production accounts
|   |   +-- Development accounts
|   +-- Sandbox OU
|   +-- Legacy OU
|
+-- Organization integrations
    +-- CloudTrail organization trail -> central S3 bucket and/or CloudTrail Lake
    +-- AWS Config -> organization aggregator
    +-- Resource Explorer -> multi-account search index/view
    +-- Security Hub -> security findings
    +-- Billing / Cost Explorer / CUR 2.0 -> spend and allocation
    +-- Rubrik Security Cloud -> assumes scoped cross-account roles for protection
```

An OU is a governance container. It can determine inherited SCPs, tag policies, backup policies, and delegated controls. It does not describe the network path, prove workload ownership, or prove that an account is inactive.

## What each AWS service answers

| Service | Best question it answers | Important limitation |
|---|---|---|
| AWS Organizations | Which accounts exist, their OU path, status, and account tags | It is not a resource inventory |
| AWS Resource Explorer | What current resources can be found across configured accounts/Regions | Multi-account search must be configured; indexes and views determine coverage |
| AWS Config aggregator | What resources exist, their relationships, compliance, and recorded configuration history | Only covers accounts/Regions/resource types where Config recording is enabled |
| CloudTrail Event history | Which management API calls happened during the last 90 days | Regional; management events only; not durable history |
| CloudTrail trail to S3 | Durable API audit records for the events selected by the trail | History begins when logging was enabled and depends on S3 retention |
| CloudTrail Lake | Queryable event store with configured event selectors and retention | Member visibility and retention depend on the event data store configuration |
| Security Hub | Which security controls/findings need attention | It is not a complete resource, ownership, or cost inventory |
| Cost Explorer / CUR 2.0 | Which accounts/services/resources/tags generate cost | Tags must be activated and not every charge is resource-taggable |
| Tag Editor | Which supported resources in the signed-in account and selected Regions have or lack tags | It is not automatically an organization-wide inventory |
| AWS RAM | Which account owns and shares a resource | A participant can use a shared resource without owning it |
| RSC | Which onboarded workloads are discovered, protected, compliant, or recoverable | It is not a replacement for AWS configuration/IaC, account inventory, or ownership governance |

## Missing environment facts that must be discovered

Do not guess these values:

1. AWS Organizations management account name and 12-digit ID.
2. CloudTrail delegated administrator account, if one exists.
3. Organization trail name, home Region, event selectors, logging status, and S3 destination.
4. Owner account for the CloudTrail S3 bucket and its retention, versioning, encryption, and Object Lock configuration.
5. AWS Config delegated administrator and organization aggregator coverage.
6. Resource Explorer delegated administrator, aggregator Region, indexes, and views.
7. Billing/FinOps access role and currently active resource and account cost-allocation tag keys.
8. RSC tenant URL, licensed use cases, currently onboarded AWS accounts/Regions, protection status, and existing SLA Domains.
9. Approved business-owner source of truth and escalation process when ownership is unknown.
10. Required RPO, retention, RTO, replication Region/account, and legal hold requirements for each workload tier.

## Tomorrow: read-only discovery procedure

### 0. Use the right identity

Sign in through IAM Identity Center or assume an approved audit/administrator role. Do not sign in with the AWS account root user for routine administration.

Run this first in CloudShell if permitted:

```bash
aws sts get-caller-identity
```

Record the account ID and ARN. This prevents gathering evidence from the wrong account or role.

### 1. Find the management account and organization structure

Current account: start in the AWS Organizations management account.

Console path:

```text
AWS Console > AWS Organizations > AWS accounts
```

Record for every account:

- Account ID, name, email, status, and OU path.
- Account-level tags.
- Intended purpose and environment.
- Business owner and technical owner, if known.
- Lifecycle state: active, migrate, quarantine, retire candidate, suspended, or closed.
- Evidence date and reviewer.

Then inspect:

```text
AWS Organizations > Settings
AWS Organizations > Services
AWS Organizations > Delegated administrators
AWS Organizations > Policies > Service control policies
AWS Organizations > Policies > Tag policies
AWS Organizations > Policies > Backup policies
```

The account list tells you where an account sits. It does not tell you what it runs or how it routes.

### 2. Determine where CloudTrail history actually lives

Current account: management account first; repeat in the CloudTrail delegated administrator if listed.

Console path:

```text
AWS Console > CloudTrail > Trails
```

For every trail, record:

- Trail name and ARN.
- Home Region and whether it is multi-Region.
- Whether it is an organization trail.
- Whether logging is on and the most recent delivery time.
- S3 bucket and prefix.
- S3 bucket owner account.
- KMS key ARN and owner account.
- Log file validation state.
- CloudWatch Logs destination, if configured.
- Management event selector: read, write, or both.
- Data event selectors and their scope.
- Insights configuration.

Also inspect:

```text
CloudTrail > Lake > Event data stores
```

Record the event data store owner, organization scope, event selectors, retention, and termination protection.

Useful read-only CLI checks:

```bash
aws cloudtrail describe-trails --include-shadow-trails --region us-east-1
aws cloudtrail get-trail-status --name <TRAIL_ARN> --region <HOME_REGION>
aws cloudtrail get-event-selectors --trail-name <TRAIL_ARN> --region <HOME_REGION>
aws cloudtrail get-insight-selectors --trail-name <TRAIL_ARN> --region <HOME_REGION>
aws cloudtrail list-event-data-stores --region <REGION>
```

After finding the bucket, switch to the bucket owner account and inspect:

```text
S3 > Buckets > <trail bucket> > Properties
S3 > Buckets > <trail bucket> > Permissions
S3 > Buckets > <trail bucket> > Management
```

Verify versioning, default encryption, KMS policy, lifecycle, bucket policy, public-access block, and Object Lock. Do not enable or change anything during discovery.

### 3. Understand the 90-day limitation

CloudTrail Event history provides the last 90 days of management events in the selected Region. It is not the complete history of an AWS account and does not include data or network activity.

Use the evidence in this order:

1. Query the organization CloudTrail Lake event data store if its retention includes the date.
2. Query retained organization-trail S3 logs with Athena.
3. Review AWS Config resource history and relationships.
4. Review CloudFormation/StackSets, Terraform state/repositories, deployment roles, and tags.
5. Review current dependencies, network flows, metrics, logs, cost, backup jobs, and support documentation.
6. Ask the accountable business/application teams.

If a resource predates retained CloudTrail and Config history, the exact human creator may be unknowable. Document `unknown` instead of inventing an owner.

### 4. Interpret a CloudTrail creation event correctly

For a resource such as an EC2 instance, search for the relevant API action, for example `RunInstances`. Inspect:

- `eventTime`
- `eventSource`
- `eventName`
- `awsRegion`
- `recipientAccountId`
- `userIdentity.type`
- `userIdentity.arn`
- `userIdentity.sessionContext.sessionIssuer.arn`
- `userIdentity.sessionContext.sourceIdentity`, when present
- `sourceIPAddress`
- `userAgent`
- `requestParameters`
- `responseElements`
- `resources`

Interpretation examples:

- An IAM user may be the direct actor, but not necessarily the current owner.
- An assumed role identifies the deployment/administration path; the session issuer is often more useful than the temporary STS ARN.
- A CloudFormation, StackSets, Control Tower, backup, or service-linked role means automation created the resource.
- `sourceIPAddress` can be an AWS service endpoint, NAT address, pipeline runner, or user network; it is not ownership by itself.

Store creator evidence as `CreatorPrincipal` and `CreatorEvidence`, not as `BusinessOwner`.

### 5. Establish current resource inventory

#### AWS Resource Explorer

Current account: Resource Explorer delegated administrator or management account, depending on your setup.

Console path:

```text
AWS Console > Resource Explorer > Settings
AWS Console > Resource Explorer > Resource search
```

Verify trusted access, delegated administrator, aggregator Region, indexes, and views. A multi-account view can search accounts only after the organization integration and indexes exist.

Useful searches include resource type, Region, account, application tag, and `tag:none` for untagged resources.

#### AWS Config aggregator

Current account: AWS Config delegated administrator/aggregator account.

Console path:

```text
AWS Console > AWS Config > Aggregators
AWS Console > AWS Config > Resources
AWS Console > AWS Config > Advanced queries
```

Verify which accounts, Regions, and resource types are recorded. Use resource relationships and configuration timelines to understand dependencies and historical changes.

#### Per-account Tag Editor

Current account: each member account or a role assumed into it.

Console path:

```text
AWS Console > Resource Groups & Tag Editor > Tag Editor
```

Select Regions and resource types explicitly. Tag Editor can find tagged and untagged supported resources in the signed-in account; it is not automatically organization-wide.

### 6. Build an ownership and migration evidence table

Use one row per workload component, not one row per raw resource only.

Recommended columns:

```text
AccountId
AccountName
OUPath
Region
ResourceArn
ResourceType
ResourceName
Application
Environment
BusinessOwner
TechnicalOwner
CostCenter
DataClassification
AWSResourceOwnerAccount
RAMOwnerAccount
SharedFromOrToAccounts
CreatorPrincipal
CreatorEventTime
CreatorEvidence
IaCStackOrRepo
NetworkDependencies
IdentityDependencies
DataDependencies
BackupStatus
LastObservedActivity
MonthlyCost
MigrationDisposition
DispositionReason
EvidenceConfidence
EvidenceSources
LastValidatedAt
Reviewer
```

Confidence definitions:

- High: authoritative application registry/owner confirmation plus technical evidence.
- Medium: consistent IaC, tags, cost, and activity evidence, but no owner confirmation.
- Low: inferred primarily from names, a creator identity, or account placement.

### 7. Triage Security Hub non-tagging findings

The Security Hub export is a starting queue, not the final truth.

1. Filter to the tagging-related control/finding.
2. De-duplicate by account ID, Region, resource ARN, and control ID.
3. Revalidate that the resource still exists.
4. Confirm that the service/resource supports the required tags.
5. Determine the owning workload before applying ownership tags.
6. Mark AWS-managed, StackSet, service-linked, ephemeral, and manually created resources separately.
7. Produce proposed tags and evidence confidence.
8. Require review for unknown or low-confidence ownership.
9. Apply approved tags in a change window using dry-run/report mode first.
10. Re-evaluate Config/Security Hub compliance after propagation.

Never automatically set `Owner` to the CloudTrail creator. A creator can be a former employee, a CI/CD role, CloudFormation, Control Tower, or another AWS service.

## Recommended tag model

Use stable team and application identifiers rather than a person's name where possible.

| Tag key | Example | Purpose |
|---|---|---|
| `Application` | `globalscape-eft` | Workload grouping |
| `Environment` | `prod` | Environment classification |
| `BusinessOwner` | `managed-file-services` | Accountable business/service team |
| `TechnicalOwner` | `cloud-platform` | Operating team |
| `CostCenter` | `CC1234` | Financial allocation |
| `DataClassification` | `confidential` | Data handling |
| `ManagedBy` | `cloudformation` | IaC/management path |
| `Lifecycle` | `active` | Approved lifecycle state |
| `MigrationDisposition` | `retain`, `migrate`, `retire`, `review` | Temporary migration program field |

Use exact case consistently. Keep allowed values documented. Do not place secrets, credentials, sensitive personal data, or ticket commentary in tags.

## Cost allocation: exact behavior

### Resource tags

Current account: AWS Organizations management/payer account with Billing access.

Console path:

```text
Billing and Cost Management > Cost Organization > Cost Allocation Tags
```

Filter for user-defined tags, select approved keys, and activate the keys. You activate a key such as `CostCenter`, not each value separately. After activation and processing, the values attached to metered resources can be used in Cost Explorer and CUR 2.0.

If the same activated key exists in multiple linked accounts, its values can be analyzed across those accounts. Keep `Linked account` as a separate dimension so you can distinguish the account from the tag value.

Limitations:

- A resource without the activated tag remains unallocated for that key.
- Some charges do not map cleanly to a taggable resource.
- Activation can take up to 24 hours.
- Backfill can be requested for up to 12 months, but it can only use tag values that actually existed on resources during the historical period; it cannot invent past ownership.
- When an account moves to a different organization, its cost-allocation tag activation must be reviewed/reactivated in the new management account.

### AWS Organizations account tags

Tag the AWS accounts themselves with stable values such as business unit, cost center, environment, and lifecycle. Then activate those keys from the `Account Tags` filter in Cost Allocation Tags.

Account tags are useful because they allocate account-level usage, including many costs that do not have a resource tag. They do not replace resource tags when several applications or owners share one account.

### Cost Categories

Use Cost Categories to combine linked accounts, account tags, resource tags, services, and other billing dimensions into business mappings. Include an `Unallocated` category so missing allocation is visible rather than silently ignored.

### Recommended allocation stack

```text
Account tags       -> Who owns/pays for the AWS account as a whole
Resource tags      -> Which application/environment/cost center generated resource-level spend
Cost Categories    -> Business rules, shared-cost groupings, and Unallocated reporting
CUR 2.0/Data Export -> Detailed evidence and reconciliation
Cost Explorer      -> Interactive analysis and management reporting
```

## Tag governance rollout

1. Approve the tag dictionary and allowed values.
2. Inventory current compliance in report-only mode.
3. Remediate existing resources with owner review.
4. Add tag validation to Terraform/CloudFormation/pipelines.
5. Attach a tag policy to one sandbox OU first.
6. Validate key capitalization and allowed values.
7. Use required-tag reporting for missing tags.
8. Use AWS Config `required-tags` only for its supported resource types and understand that it detects rather than universally prevents.
9. Consider narrowly scoped SCPs for create APIs that support `aws:RequestTag`/service tag conditions.
10. Test service roles, CloudFormation, auto scaling, backup, and break-glass paths before expanding an SCP.

Basic tag-policy enforcement can reject incorrect case or values but does not by itself block a resource created with no tags. Required-tag reporting, IaC checks, Config, and selectively tested SCPs fill different parts of that gap.

## Rubrik Security Cloud: mental model

```text
Rubrik Security Cloud SaaS control plane
           |
           | STS AssumeRole into an onboarded AWS account
           v
AWS application account
  +-- EC2 / EBS -> AWS-native snapshots in the source Region
  +-- RDS / Aurora -> snapshot and supported continuous/PITR protection
  +-- S3 -> object protection into the configured backup location
  +-- Optional Exocompute (EKS) -> indexing, file recovery,
      application-consistent operations, S3 processing, and tiering
           |
           +-- optional replication/archival to a separate account or Region
```

RSC is the policy, orchestration, visibility, and recovery control plane. It uses cross-account IAM roles and AWS-native APIs. Depending on the workload and feature, it can also create or use Exocompute resources in AWS.

### What an SLA Domain means

An SLA Domain defines:

- Snapshot frequency: the backup RPO target.
- Retention: how long recovery points remain.
- Replication: where additional copies are stored.
- Archival: longer-term storage location and timing.
- Retention lock: protection against premature deletion when correctly configured.

An SLA Domain does not prove the application's RTO. RTO must be measured by restoring and validating the complete application.

### Assignment precedence to understand

For AWS EC2/EBS, protection can be inherited from the AWS account, assigned by AWS tag rules, or assigned directly to a workload. Direct assignment takes precedence over tag-based assignment, and tag-based assignment takes precedence over account inheritance. `Do Not Protect`/exclusion behavior must be reviewed carefully because it can override expected inheritance.

### Exocompute

Exocompute uses EKS-based compute in an AWS account for operations such as file indexing, file recovery, application-consistent protection, and storage tiering. It requires:

- Appropriate AWS permissions and service quotas.
- A VPC with DNS support.
- Two subnets in different Availability Zones.
- Required outbound HTTPS connectivity or an approved VPC endpoint/PrivateLink design.
- S3, ECR, EKS, EC2, STS, EBS, KMS, and RSC reachability as required by the chosen design.
- KMS grants/sharing when a centralized Exocompute account processes encrypted snapshots from application accounts.

Centralized Exocompute can reduce repeated networking infrastructure by mapping application accounts to a host account, but it creates explicit cross-account KMS, networking, and blast-radius design decisions.

### Immutability and retention lock

Do not say “all Rubrik AWS backups are immutable” without checking configuration. Rubrik documents retention-locked SLA Domains for cloud-native workloads and requires Quorum Authorization for the feature. For cloud-native workloads, Rubrik documents immutability at the archival location rather than the source snapshot, with Instant Archive required for archived snapshot immutability. Verify licensing, workload support, SLA mode, QAuth, archive configuration, and restore behavior in your tenant.

### What RSC does not automatically preserve

Protecting an EC2 volume or database is not the same as preserving the whole application. Maintain separately:

- Terraform/CloudFormation and application deployment artifacts.
- VPCs, subnets, route tables, TGW configuration, security groups, NACLs, endpoints, and DNS.
- IAM roles/policies, permission boundaries, Identity Center assignments, and SCPs.
- KMS keys and key policies needed to decrypt protected data.
- Secrets, certificates, licenses, external integrations, and service accounts.
- Application-specific recovery order and validation steps.

RSC restore testing must therefore validate both the data and the reconstructed AWS/application dependencies.

## RSC implementation sequence

### Phase 1: Discover the current tenant

In RSC:

```text
Settings > Cloud Accounts > AWS
Inventory > Cloud Native > AWS
SLA Domains
Events / Activity / Jobs
```

Record onboarded account IDs, Regions, enabled use cases, role ARNs, discovery state, SLA assignment, compliance, recent failures, replication/archive targets, Exocompute configuration, and KMS dependencies.

Menu names can vary slightly by RSC release and licensed features.

### Phase 2: Classify workloads

For each application, approve:

- Criticality and data classification.
- RPO, RTO, retention, and legal hold.
- AWS account and Regions.
- Workload type: EC2/EBS, RDS/Aurora, S3, or Kubernetes.
- Source and target KMS keys.
- Replication/archival account and Region.
- Restore destination and isolation controls.
- Application owner and restore approver.

### Phase 3: Pilot one non-production workload

1. In RSC, go to Settings > Cloud Accounts > AWS > Add AWS Account.
2. Select only the required protection/compute use cases and Regions.
3. Review SCPs before launching the generated CloudFormation workflow.
4. Deploy the RSC-generated roles using the approved change process.
5. Verify the trust policy, external identity controls, role permissions, and CloudTrail events.
6. Confirm inventory discovery.
7. Create or select a test SLA Domain.
8. Assign it to one test workload.
9. Run an on-demand protection job and verify success.
10. Restore into an isolated test VPC/account or approved test target.
11. Validate boot, file/database integrity, application function, identity, DNS, network, and access control.
12. Record actual recovery duration and compare it to RTO.
13. Clean up only the temporary restore resources approved for deletion.

### Phase 4: Scale safely

Use Rubrik's bulk account/StackSet workflow where supported. Separate StackSets by use case when required. Roll out by pilot OU, then lower-risk accounts, then production. Keep the management account and security/logging accounts under an explicitly reviewed protection design rather than blindly enrolling every account.

### Phase 5: Operate continuously

Daily/weekly checks should cover:

- SLA compliance and missed snapshots.
- Job failures and permission drift.
- KMS access failures.
- Snapshot/service quota exhaustion.
- Exocompute node/network failures.
- Accounts or Regions not discovered.
- Unprotected new workloads and unexpected `Do Not Protect` assignments.
- Replication/archive failures.
- Restore-test age and evidence.
- AWS and Rubrik audit events.
- Cost growth from snapshots, replication, archive, data transfer, and Exocompute.

## Decision rules for legacy-account retirement

Do not retire an account until all of these are true:

1. An accountable owner has approved the disposition.
2. Current resource inventory is complete across all Regions.
3. Resource and RAM ownership are understood.
4. Network, identity, DNS, KMS, data, backup, and external dependencies are documented.
5. Required data and infrastructure definitions have been migrated or retained.
6. The destination workload has passed functional, security, backup, and restore testing.
7. Monitoring shows no required traffic/jobs during the approved observation window.
8. Billing and support owners confirm no remaining required services or commitments.
9. The account is quarantined/restricted for the approved waiting period without breaking a dependency.
10. Backout and recovery procedures have been tested and documented.

Moving an account from Legacy to an unrestricted or quarantine OU changes inherited governance. It does not migrate, archive, or delete the account. The exact effect depends on the SCPs, tag policies, backup policies, and other controls attached to the old and new OU.

## Official references

- [AWS CloudTrail Event history](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html)
- [Create an organization trail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-an-organizational-trail-in-the-console.html)
- [CloudTrail Lake for organizations](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-organizations.html)
- [AWS Config multi-account aggregation](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html)
- [AWS Resource Explorer multi-account search](https://docs.aws.amazon.com/resource-explorer/latest/userguide/manage-service-multi-account.html)
- [Tag Editor resource search](https://docs.aws.amazon.com/tag-editor/latest/userguide/find-resources-to-tag.html)
- [Resource Groups Tagging API GetResources limitations](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/API_GetResources.html)
- [AWS Organizations tag policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html)
- [Tag-policy enforcement behavior](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies-enforcement.html)
- [AWS Config required-tags rule](https://docs.aws.amazon.com/config/latest/developerguide/required-tags.html)
- [User-defined cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/custom-tags.html)
- [Account tags for cost allocation](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/account-tags-cost-allocation.html)
- [Cost allocation tag backfill](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-allocation-backfill.html)
- [RSC Cloud Accounts](https://docs.rubrik.com/en-us/saas/saas/cloud_accounts.html)
- [RSC AWS workloads](https://docs.rubrik.com/en-us/saas/saas/aws_workloads.html)
- [RSC AWS protection architecture](https://docs.rubrik.com/en-us/saas/saas/aws_cnp_ec2.html)
- [RSC SLA Domains](https://docs.rubrik.com/en-us/saas/saas/sla_domains.html)
- [RSC AWS tag-based protection](https://docs.rubrik.com/en-us/saas/saas/aws_ec2_ebs_tag_based_protection.html)
- [RSC Exocompute on AWS](https://docs.rubrik.com/en-us/saas/saas/exocompute_for_aws.html)
- [RSC Exocompute networking](https://docs.rubrik.com/en-us/saas/saas/aws_network_setup_exocompute.html)
- [RSC retention lock for cloud workloads](https://docs.rubrik.com/en-us/saas/saas/rl_sla_cnp.html)
- [RSC AWS bulk account examples](https://docs.rubrik.com/en-us/saas/saas/aws_add_accounts_examples.html)


