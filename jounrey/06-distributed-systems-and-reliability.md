# 06 — Distributed systems and reliability: make failure an engineering input

This module starts after you can build and test a single-process HTTP service.
Use local processes or containers for the first labs; cloud deployment is a later experiment.
Use these labs throughout the capstone weeks in [08-weekly-plan-and-mentoring.md](08-weekly-plan-and-mentoring.md). Record a hypothesis, measurement, and conclusion for each lab.
The goal is to explain and demonstrate tradeoffs, not to attach the most services to a diagram.

## The system you will grow

Build a synthetic event processing service: users submit events and later retrieve results.
An event might be `{event_id, customer_id, timestamp, amount}` using invented data.
Start with one process and SQLite or another local database you understand.
Later add a queue, multiple workers, and a durable database only when the lab requires them.

```text
Client → Go HTTP API → durable job record / queue → Python worker → result store
             |                                      |
             +--------------- telemetry ------------+
```

Durable means data survives the failures your design promises to withstand.
A value in process memory survives another function call but not a process crash.
A local file may survive a process crash but not loss of its disk.
A replicated database adds other guarantees and other failure modes.

## Vocabulary with engineering consequences

| Term | Meaning | Why it matters |
|---|---|---|
| Distributed system | Components communicate across a network | A caller cannot instantly know whether a remote operation finished |
| Latency | Time one operation takes | A fast average can hide painfully slow tail requests |
| Throughput | Work completed per unit time | 100 requests/s is different from 100 concurrent requests |
| Concurrency | Work in progress at the same time | Excessive in-flight work consumes memory and connections |
| Availability | The service performs its promised function when needed | A healthy process can still give users failures |
| Consistency | Rules about which writes reads are allowed to observe | Different replicas may disagree temporarily |
| Replication | Keeping data copies on multiple nodes | Availability and recovery improve only within a specified failure model |
| Partition | A communication failure separating nodes | Both sides may be alive but unable to coordinate |
| Sharding | Splitting data across partitions/nodes by a key | A poor key creates a hot partition |
| Backpressure | Slowing or rejecting producers when consumers cannot keep up | Prevents an unlimited queue from exhausting resources |
| Idempotency | Repeating an operation has the same intended effect as one execution | Retried payment/job requests should not double-apply effects |
| Reconciliation | Repeatedly comparing desired and observed state and correcting differences | Makes controllers recover from missed events and restarts |
| SLI | Service-level indicator: a measurement of user-visible behavior | Fraction of valid requests completed successfully |
| SLO | Service-level objective: a target for an SLI over a time window | 99.9% success over a rolling 30 days |
| RPO | Recovery point objective: acceptable data loss measured in time | At most 15 minutes of acknowledged data may be lost |
| RTO | Recovery time objective: target time to restore service | Restore useful service within one hour |

Keep a glossary in your project with your own examples and counterexamples.
If you cannot explain a term without repeating its acronym, schedule a small experiment for it.

## Lab 1 — Measure a single service honestly

Write `POST /jobs`, `GET /jobs/{id}`, and `GET /health`.
Validate types, required fields, maximum payload size, and unsupported input.
Return a request ID in logs and responses so you can connect a report to a specific operation.
Measure elapsed durations with a monotonic clock; wall clocks can jump.

Run a local load generator with a fixed workload and record:

- Requests attempted and completed.
- Successful and failed requests by category.
- p50, p95, and p99 latency.
- CPU, memory, database connection usage, and concurrency.
- Payload sizes and machine/environment details.

p99 means 99% of observed requests were no slower than that value.
It is not “the average of the slowest 1%.”
Do not average percentiles from different servers and label the result a global percentile.
Use mergeable latency histograms or raw observations for the combined distribution.

**Experiment:** raise offered load gradually until latency or errors violate your chosen target.
Record the saturation point and the bottleneck evidence.
**Gate:** produce a repeatable benchmark, and explain why one impressive requests/s number is insufficient.

## Lab 2 — Write a useful SLO before writing an alert

Define the user journey first: “a valid submitted event becomes a retrievable result.”
An API returning 202 Accepted is not success for that entire journey.
Use two indicators: submission success and completion within a defined deadline.
Define excluded traffic explicitly, such as intentionally malformed test requests.
Do not quietly exclude real overload errors just to improve the metric.

Example exercise targets, not universal production requirements:

```text
Submission SLI = accepted valid requests / all valid requests
Submission SLO = 99.9% over 30 days

Completion SLI = accepted jobs completed within 60 seconds / accepted jobs
Completion SLO = 99% over 30 days
```

An error budget is the allowed portion of bad outcomes: `1 - target`.
For 1,000,000 eligible requests and a 99.9% target, that is 1,000 bad requests.
Request-based budgets do not automatically translate into downtime minutes.
Burn rate compares the observed bad-event fraction with the allowed fraction.
At a 0.1% allowed error fraction, 1% observed errors is a 10× burn rate.

Write what happens when the budget is consumed: investigate, slow risky releases, and prioritize repair.
Do not make an arbitrary percentage into a business commitment without discussing user needs.
Read Google's [Implementing SLOs](https://sre.google/workbook/implementing-slos/) and apply it to your own measurements.

**Gate:** calculate the SLI from raw counters and explain every numerator and denominator.

## Lab 3 — Timeouts, cancellation, and retries

A timeout limits how long a caller waits; it does not prove the remote operation was canceled.
The remote service may commit a write after the caller gives up.
A deadline is an absolute completion limit propagated through a chain of operations.
Cancellation tells cooperative work that its caller is no longer interested.

Implement a Go request context with a deadline and pass it to outbound HTTP/database operations.
In Python, use explicit async timeout handling and avoid swallowing cancellation.
Set connection limits and understand connect, TLS handshake, response-header, and overall timeouts.
Use certificate verification; a broken trust chain should be investigated rather than hidden.

Retry only errors your operation can safely retry.
Distinguish a temporary timeout/throttle from invalid input or permanent authorization failure.
Exponential backoff increases wait intervals after repeated failures.
Jitter adds randomness so many clients do not retry at exactly the same instant.
Cap the interval, number of attempts, and total elapsed retry time.
Respect service retry guidance and avoid layering unbounded custom retries over SDK retries.

If five call layers each permit three attempts, worst-case downstream amplification can reach `3^5 = 243`.
Choose an intentional retry boundary and track attempts in metrics.
Read the [Amazon Builders' Library retry article](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

**Failure drill:** local downstream responds slowly, returns 503, then recovers.
Prove that deadlines bound user waiting and recovery does not trigger a retry storm.
**Gate:** show both successful recovery and a bounded terminal failure.

## Lab 4 — Idempotency and the crash between two steps

Add an `Idempotency-Key` to job submission.
Store the key, request payload hash, status, and eventual response durably.
The same key and same payload should retrieve the same job/result.
The same key with a different payload should produce a clear conflict.
Use a database uniqueness constraint or conditional write to prevent concurrent duplicate creation.

Do not implement “read key; if absent, write key” without an atomic guard.
Two concurrent callers can both see the key as absent.
A transaction is a group of database changes that commits together or does not commit.
Use one transaction when the idempotency record and business write share a suitable database.

For an external effect, such as sending an email, a local transaction alone cannot guarantee one delivery.
The process can send the email and crash before recording completion.
Use an external provider's idempotency feature where available, or explicitly document duplicate risk.
“Exactly once” needs a precisely stated boundary and failure model.

**Tests:** 100 simultaneous requests with the same key create one logical job.
Kill a local worker before processing, after committing a result, and before acknowledging the message.
Restart it and verify whether the observable business effect repeats.
**Gate:** explain every crash window and its recovery behavior.

## Lab 5 — Queues are buffers, not infinite capacity

A producer submits work; a consumer processes it.
Acknowledgment confirms that the queue can remove a successfully handled message.
Visibility timeout hides a received SQS message temporarily while the consumer works.
If processing outlasts visibility, another consumer can receive the message.
Renew visibility only within an intentional processing deadline, and acknowledge after durable completion.

SQS standard queues can deliver duplicates; build idempotent consumers.
See [SQS at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html).
An in-memory channel is useful for local concurrency but is not a durable cross-process queue.

Assume arrivals average 100 jobs/s and workers finish 80 jobs/s.
Backlog grows by approximately 20 jobs/s while those rates hold.
Adding a queue stores the deficit; it does not remove it.
Measure age of oldest work, backlog, processing rate, failure rate, and retry count.
Use a bounded worker pool and reject or throttle submissions when your service cannot meet its contract.

A poison message repeatedly fails for a deterministic reason.
Move it to a DLQ after a chosen threshold, record why, repair the cause, then replay deliberately.
Do not blindly redrive the whole DLQ while the same bug remains.

**Failure drill:** one payload triggers a controlled failure while ordinary jobs continue.
**Gate:** prove duplicates do not duplicate results and one bad job cannot stop unrelated work.

## Lab 6 — The transactional outbox

Suppose your API must both save a job and publish a queue message.
Saving first can leave an unpublished job if the process crashes.
Publishing first can create a message for a job the database never saves.
The transactional outbox stores the job and an outgoing-event record in one database transaction.
A separate publisher sends unsent records and marks progress afterward.
Publishing can still repeat after a crash, so the consumer remains idempotent.

Build this locally before selecting a managed cloud implementation.
Give every outgoing event an ID and record delivery attempts.
Measure the delay from committed job to published message.
Keep enough audit evidence to distinguish never published, published, and processed.

**Gate:** inject a crash between every two operations and reconcile all accepted jobs after restart.

## Lab 7 — Replication, consistency, and partitioning

Start by documenting your required invariant: a property that must remain true.
Example: each job ID has one authoritative final result.
Another example: an account balance never becomes negative after accepted withdrawals.
Choose storage behavior based on the invariant, not a memorized database ranking.

Strong read behavior can return the latest completed write under the service's documented guarantees.
Eventually consistent reads may temporarily return older state while replicas converge.
“Eventual” is not automatically a bounded number of milliseconds.
A cache can make an otherwise strongly read database appear stale to the user.

Sharding assigns data to partitions, often by hashing a key.
If most work belongs to one customer, customer-based partitioning can overload one partition.
Measure key distribution before claiming even scaling.
Replication adds copies of the same data; sharding divides the data.
They solve different problems and can coexist.

CAP concerns the tradeoff between specified consistency and availability during a network partition.
It is not a universal instruction to “pick any two” for every operating condition.
Consensus protocols such as Raft coordinate agreement among nodes under stated assumptions.
Study consensus after you can describe retries, persistence, leader failure, and partitions.
Implement a toy replicated log as a learning exercise, never as your production database.

**Gate:** design one system favoring fresh reads and another tolerating stale reads, with user consequences.

## Lab 8 — Observability and incident response

Logs are discrete events; metrics aggregate numerical measurements; traces connect spans of a request.
A span records one timed operation; a trace ties related spans into a request path.
Instrument request ID, job ID, operation, duration, result, and safe error category.
Never log secrets or real customer payloads in your portfolio system.

Use low-cardinality metric labels such as route template and status class.
Cardinality means the number of distinct label combinations.
Using a unique request ID as a metric label can create a time series per request.
Keep unique IDs in logs/traces instead.

Create dashboards for traffic, latency, errors, and saturation.
Write an alert with a clear action and an owner; avoid paging solely because CPU briefly rises.
Correlate elevated queue age with processing throughput and worker errors.
Keep deployment timestamps so a regression can be compared with a release.

Run a 30-minute local incident exercise with one injected fault and one observer.
Write a timeline, impact, hypothesis, evidence, mitigation, contributing conditions, and follow-up action.
A blameless review examines system conditions and decisions rather than shaming the person involved.
**Gate:** someone else diagnoses the seeded failure using your dashboard and runbook.

## Lab 9 — Containers, then Kubernetes only when you can explain its purpose

A container packages an application with its runtime environment; it shares the host kernel.
An image is the packaged template; a running container is an instance of it.
First run the API and worker using a local container workflow with fixed resource limits.
Handle SIGTERM so new work stops and in-flight work finishes or returns safely to the queue.

Then use a disposable local Kubernetes cluster, such as kind, after reading its installation guide.
A Pod is a scheduling unit containing one or more tightly coupled containers.
A Deployment manages replaceable Pod replicas and rollout behavior.
A Service supplies a stable way to reach selected Pods.
A readiness probe controls whether a Pod should receive traffic.
A liveness probe can trigger restart; a slow dependency is not automatically a reason to restart the process.
Requests guide scheduling; limits constrain resource use and can produce throttling or memory termination.
RBAC controls which Kubernetes identities can perform which API operations.
Read the official [Kubernetes concepts](https://kubernetes.io/docs/concepts/).

**Drills:** delete one lab Pod, deploy an unhealthy version, exhaust a bounded queue, then roll back.
**Gate:** explain recovery through controllers and show that a rollout does not lose accepted jobs.
EKS is optional after local competence; its control plane and supporting infrastructure cost money.

## Lab 10 — Optional Go operator

An operator combines a custom resource with a controller that encodes operational knowledge.
A custom resource definition adds an API type; it does not execute automation by itself.
Create `EventWorkerPool` with desired replicas, queue reference, and a maximum processing limit.
Your controller observes it and reconciles a Deployment plus status information.
Repeated reconciliation should converge without creating duplicate objects.
Watch events are useful notifications; reconcile current state instead of assuming every event arrives once.

Test restart, duplicate events, missing managed objects, invalid specifications, and failed API calls.
A finalizer delays deletion until explicit cleanup completes; a broken finalizer can leave an object stuck.
Do not introduce finalizers until your cleanup behavior and recovery runbook are clear.
Follow [Kubernetes operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

**Gate:** explain why a Deployment or a scheduled job was insufficient before defending the operator.

## Exit portfolio and review

- One design document states workload, invariants, failure assumptions, data model, and alternatives.
- Benchmarks identify the machine, load model, percentiles, failures, and bottleneck.
- Duplicate, timeout, and crash tests demonstrate bounded and recoverable behavior.
- A dashboard and incident review show operational competence.
- A restore exercise measures observed RPO/RTO and explains any gap from objectives.
- A capacity model predicts the backlog from a load increase and is compared with measurements.
- A teardown report identifies every retained cloud resource and its cost.

Mentor prompts: “What happens if the request succeeded but the response vanished?”
“Which invariant is protected by your transaction?” “What happens at 10× traffic?”
“What breaks if the queue delivers twice?” “Which metric proves users recovered?”
Your answers must point to tests, measurements, or clearly stated assumptions.
