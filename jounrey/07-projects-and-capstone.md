# 07 — Projects that demonstrate engineering ability

These are build specifications, not claims that software has already been implemented. Deliver one working slice at a time. Project timings in [08](08-weekly-plan-and-mentoring.md) assume roughly 10 hours/week and may need repetition.

## Project 1: inventory evidence engine

**Problem:** a team has inconsistent resource exports and cannot distinguish missing ownership from failed collection. Build a Python report generator that preserves provenance: where each fact came from and when it was observed.

Start entirely locally using synthetic JSON/CSV. Suggested fields:

```json
{
  "account": "111111111111",
  "region": "us-east-1",
  "type": "volume",
  "id": "vol-example-a",
  "owner_team": null,
  "collection_status": "success",
  "observed_at": "2026-09-22T12:00:00Z",
  "relationships": [{"type": "attached_to", "id": "i-example-a"}]
}
```

The IDs above are fixture labels, not actual AWS API IDs. A nullable owner means unknown. A collection failure is a separate state; it must not become an empty successful inventory.

Build in order:

1. Parse and validate fields. Keep account IDs as strings; leading zeroes matter.
2. Deduplicate by account, Region, type, and ID.
3. Report counts and explicit malformed rows.
4. Group connected resources using relationships; explain unresolved references.
5. Add ownership candidates with evidence and confidence, keeping creator separate from current owner.
6. Add a report comparing two observations: added, removed, changed, and collection-unknown.
7. Add a read-only Boto3 adapter for one service in your learning account.
8. Implement pagination, bounded retries, partial failures, and timestamps.

An adapter translates an external interface into your internal model. Keeping it separate lets tests supply fixtures without calling AWS. See [Boto3 pagination](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/paginators.html).

Acceptance tests: empty export; malformed JSON; duplicate records; two Regions; same name on different resources; AccessDenied for one collector; multiple pages; missing timestamp; deleted attachment; unknown owner; invalid account format. Never classify inactivity or retirement solely from names or absence of a metric.

Evidence: sample sanitized report, schema, tests, one diagram, limits, and a design note explaining why `unknown` is a real result. Stretch: use a graph traversal to group an instance, its volumes, and security groups without claiming every shared security group is one application.

## Project 2: explainable route evaluator

**Problem:** people follow a plausible route in the console rather than the route a destination actually selects. Build a pure Python or Go library that takes route-table fixtures and explains its choice.

Inputs: source resource, destination IP, VPC route entries, TGW attachment-to-table associations, and attachment targets. Start IPv4 only. Explicitly reject unsupported route types rather than silently simulating them incorrectly.

Build in order:

1. CIDR membership and longest-prefix selection.
2. Exact explanation of candidates, winner, and fallback.
3. Traverse a VPC-to-TGW-to-VPC path.
4. Select the TGW table by ingress attachment association.
5. Detect repeated traversal states and report a possible loop.
6. Evaluate the return direction separately.
7. Report missing data as unresolved.

Acceptance fixture:

```text
Destination: 10.30.130.185
Source routes:
  0.0.0.0/0       -> TGW-B
  10.30.0.0/21   -> TGW-A

Expected: TGW-B, because 10.30.130.185 is outside 10.30.0.0/21.
Adding 10.30.128.0/19 -> TGW-A changes the selected next hop to A.
```

Test missing associations, blackhole routes, no match, overlapping prefixes, cycles, IPv6 rejected explicitly, and two route tables containing the same destination with different targets. Explain that this tool predicts configured forwarding; it does not establish live packet delivery or firewall/application acceptance. [AWS route priority](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html).

Stretch: a read-only collector exports the required AWS data. Preserve account owner and Region on every node. Add live flow-log evidence as a distinct observation, never as a replacement for the topology model.

## Project 3: recoverable migration planner

**Problem:** copying a resource is not enough if metadata, encryption dependencies, and validation are missing. Build a planner first; use only disposable learning snapshots for the later execution exercise.

Input: explicit allowlist of source IDs and intended destination account/Region. Output: deterministic plan listing identity checks, share/copy steps, tags/metadata, polling, validation, and recorded destination IDs.

A state machine describes valid stages. Example:

```text
DISCOVERED -> VALIDATED -> COPY_REQUESTED -> COPYING -> VERIFIED
                              |                |
                              +-> UNKNOWN      +-> FAILED
```

`UNKNOWN` means an API request may have succeeded even though the client lost its response. The program must reconcile known state before retrying an operation that could create another copy.

Acceptance: wrong source account aborts; unapproved ID omitted; missing KMS permission reported; pending copy never called successful; interruption resumes from a journal; verification checks ownership/size/encryption/state; no source deletion exists in the executor. A metadata match alone is not a full restore test.

Stretch: restore a destination volume in an isolated lab and verify known file hashes. A hash is a fingerprint used to detect changed bytes; it does not independently establish application correctness. Record cleanup and any retained backup policy separately.

This is an advanced learning exercise; service behavior varies between EBS snapshots, AWS Backup recovery points, AMIs, and S3 objects. Do not implement one generic copy call for all four. Use the official service-specific guides from module 05.

## Project 4: durable event-processing service

**Business requirement:** accept synthetic resource-change events, process them into an inventory summary, and allow clients to query status. This connects your inventory work with service design.

Start small. Do not provision EKS, RDS, NAT gateways, and multiple accounts merely to draw a large diagram.

### Event contract

```json
{
  "event_id": "demo-000001",
  "schema_version": 1,
  "resource_key": "lab/us-east-1/instance/demo-a",
  "observed_at": "2026-09-22T12:00:00Z",
  "payload": {"state": "running"}
}
```

A contract defines accepted fields, types, meanings, size limits, and error behavior. Decide whether an identical event ID with different contents is rejected. Decide what to do with older observations arriving after newer ones. Store both event identity and observation time.

### Stage A: one process and one database

Implement a Go HTTP API with `POST /events`, `GET /events/{id}`, and `GET /healthz`. Use a local SQLite database initially or PostgreSQL if you have finished its module. Record accepted events durably before reporting acceptance. Start with synchronous processing to understand the path.

Define responses: accepted event, duplicate same payload, conflicting payload, malformed JSON, unsupported schema, oversized body, unknown ID, database unavailable. HTTP 202 means accepted for processing, not completed; 200 can represent a completed status query.

Gate: restarting the service preserves accepted events. Tests prove client timeout does not automatically imply the event was rejected. Input validation limits resource consumption.

### Stage B: Go ingress and Python worker

```text
client -> Go API -> durable event/outbox records
                         |
                         v
                  publisher -> queue -> Python worker -> result table
                                                   |
                                      acknowledge only after durable work
```

An outbox is a database table of messages to publish, written in the same transaction as the accepted event. This avoids accepting a database row and losing the corresponding queue message between separate writes. A publisher can send twice after a crash, so the consumer still needs duplicate handling.

Implement a local adapter for learning, then SQS for the AWS stage. Local behavior is not proof of SQS delivery semantics. Standard SQS can deliver more than once; design accordingly. [AWS SQS delivery behavior](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html).

Worker requirements:

- Validate event version before processing.
- Make durable side effects idempotent: repeated delivery has the same intended effect.
- Use a transaction/unique key or appropriate conditional operation, not just an in-memory set.
- Bound concurrent work and set deadlines.
- Retry transient failures with a limit and randomized backoff.
- Quarantine permanently invalid work in a dead-letter path.
- Stop accepting new work on shutdown and finish or safely release in-flight work.

`asyncio` is useful only where work can yield while waiting. Boto3 calls are synchronous. If using them from an async worker, isolate them with bounded threads or choose a compatible async client and document the choice. A synchronous bounded worker is a valid initial design.

### Stage C: instrument and measure

Add structured logs with event/request IDs; counters for accepted/completed/failed events; histograms for request/processing latency; queue backlog and age; database errors; worker concurrency. Metrics labels must remain bounded: do not put every event ID into a metric label.

Proposed laboratory targets, not performance promises:

| Requirement | Experiment |
|---|---|
| No missing accepted events after recovery | Compare durable accepted IDs against completed/pending/dead-letter IDs |
| Duplicate delivery does not duplicate effects | Deliver each selected event several times, including concurrent deliveries |
| Bounded memory under overload | Send faster than processing capacity and observe limits/backpressure |
| Explain latency under load | Measure p50/p95/p99 at several sustained rates after warmup |
| Predictable shutdown | Stop a worker during processing and reconcile the event afterward |

Choose an initial modest load such as 10 events/second on your own machine, then increase gradually. Record event size, duration, tool, workers, hardware, database, CPU, memory, failures, and percentiles. Compare configurations using the same workload. Do not label a laptop test “internet scale.”

### Stage D: deploy using CDK in Python

A first AWS deployment can use API Gateway plus a Go Lambda ingress, SQS, a Python Lambda worker, and DynamoDB. This reduces always-on infrastructure, though it still incurs usage charges. It changes the service runtime; document those differences from the local HTTP process.

| Resource | Why present | Required design choice |
|---|---|---|
| API Gateway | Managed HTTP entry | Authentication, request size, throttling |
| Go ingress Lambda | Validates and accepts requests | Timeout, concurrency, logging, durable acceptance model |
| SQS queue | Buffers work | Visibility timeout, retention, redrive |
| Dead-letter queue | Holds failed work for inspection | Alarm, replay policy, duplicate handling |
| Python worker Lambda | Processes batches | Batch size, partial failure handling, idempotency |
| DynamoDB table | Durable event/result records | Partition key, conditional writes, consistency needs |
| CloudWatch logs/alarms | Operational evidence | Retention and actionable alarms |
| IAM roles | Scoped service access | Individual permissions per component |

You must resolve the database/queue dual-write issue when translating the local outbox. Options include an appropriate change-stream publisher or an acceptance contract based directly on successful queue submission. A queue-first design and a database-first outbox are different designs; draw and test the one you choose.

CDK gates: assertions on encryption, public exposure, policies, log retention, and queue configuration; inspect `cdk synth` and `cdk diff`; deploy only to the lab account; run integration tests; remove resources and inspect retention leftovers. Keep estimates and actual spend in the project report.

### Stage E: containers and Kubernetes, after Stage D is understood

Run the Go service and Python worker in containers with a local Kubernetes cluster. Learn Deployment, Service, ConfigMap, Secret handling, probes, limits, and graceful shutdown. Add worker scaling based on backlog only after explaining why more workers may overload the database.

EKS is an optional timed AWS exercise after local Kubernetes passes. Provision networking, node/compute capacity, identity, registry access, logging, and cleanup deliberately. Explain the operational cost of running a cluster compared with the Lambda version.

### Failure drill matrix

| Failure | Expected behavior | Evidence |
|---|---|---|
| Duplicate event | One logical effect | Durable uniqueness and result assertion |
| Worker crash before commit | Work eventually retried | Delivery and result timeline |
| Worker crash after commit before acknowledgment | Retry does not repeat effect | Repeated delivery with same final state |
| Database unavailable | Bounded retries; visible backlog | Queue age, errors, no false success |
| Poison message | Eventually quarantined | Dead-letter event and diagnostic reason |
| Slow dependency | Deadline enforced | Measured duration and cancellation |
| Traffic burst | Bounded queue/concurrency behavior | CPU/memory/backlog graphs |
| Old observation arrives last | Explicit version/time policy | Final state consistent with contract |
| Deployment regression | Rollback procedure exercised | Previous artifact and recovered health |

### Capstone completion

Deliver source, tests, IaC, architecture diagram, contracts, security assumptions, benchmark report, restore/replay test, cost record, incident writeup, and a five-minute demonstration. A reviewer must be able to reproduce the local version from the README.

## Optional advanced projects

Choose one, not all at once:

1. **Kubernetes controller in Go:** introduce a custom `ReportJob` resource. Reconcile desired state into Jobs, report status, and handle retries/deletion. Test repeated reconciliation and controller restart. Explain why a controller is a continuous convergence loop, not a one-shot script.
2. **Terraform provider in Go:** manage a small local API resource. Implement create/read/update/delete/import and tests. Destroy an object outside Terraform, run refresh/plan, and explain drift. Do this after using modules and state successfully.
3. **Replicated key-value store:** begin with a durable single-node store. Then study Raft and implement a bounded learning prototype with explicit failure assumptions. Never present a classroom consensus implementation as production-safe.

The signal is your explanation of correctness, recovery, and limits. The number of services in the diagram is not the goal.
