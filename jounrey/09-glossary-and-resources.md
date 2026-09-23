# 09 — Glossary and resource library

Use this with the worked explanations in the chapters. For each term, give your own example and one failure it helps explain. Technical references establish tool behavior; the schedule and exercises are original teaching recommendations, not official hiring requirements.

## Programming

| Term | Plain meaning and example |
|---|---|
| API | Software interface; `GET /events/123` retrieves an event |
| SDK | Library for another platform; Boto3 calls AWS APIs |
| Runtime | Machinery supporting program execution; Go schedules goroutines |
| Type | Category defining valid operations; an account ID is text |
| Struct/dataclass | Named fields grouped into a model; account, Region, ID |
| Interface | Required operations independent of implementation; `get(id)` |
| Composition | Combine small components; handler contains a repository |
| Dependency injection | Pass dependencies into code; test supplies a fake repository |
| Pure function | Computes from inputs without external effects; CIDR membership |
| Side effect | External action; write file or publish message |
| Serialization/schema | Encode data / define its fields and meaning; versioned JSON |
| Unit test | Focused isolated behavior; correct route chosen from fixture |
| Integration test | Cooperating components; service writes to test database |
| End-to-end test | User path across system; submit event and retrieve result |
| Fake/mock | Substitute dependency for a test; two-page AWS response |
| Regression test | Detects recurrence of a known defect |
| Linting/static analysis | Inspect code without running the whole application |
| Coverage | Code exercised by tests; does not prove assertions are useful |
| Refactoring | Change internal structure while preserving behavior |
| Concurrency/parallelism | Overlapping progress / simultaneous execution |
| Goroutine | Go runtime-managed execution; not a dedicated OS thread |
| Channel | Communicates values between goroutines |
| Mutex | Mutual exclusion lock protecting shared state |
| Race | Conflicting unsynchronized accesses; unsafe shared map updates |
| Deadlock | Participants wait indefinitely on each other |
| Semaphore | Admission counter; maximum five concurrent operations |
| Event loop | Coordinates cooperative tasks as operations become ready |
| Generator | Produces values incrementally; streams inventory rows |
| Decorator | Wraps Python function behavior; records elapsed time |
| Garbage collection | Reclaims memory no longer reachable/needed under runtime rules |
| Escape analysis | Compiler reasons about value lifetimes and allocation |
| Profile/benchmark | Attribution of resource use / measurement of a defined workload |

## Operating systems, networks, and data

| Term | Plain meaning and example |
|---|---|
| Process/PID | Executing program / its OS identifier |
| Thread | Execution path within a process sharing much of its memory |
| Kernel | OS core managing CPU, memory, devices, and system calls |
| File descriptor | Handle for an open file/socket; leaks can exhaust limits |
| PID 1 / init | First user-space process / service startup and supervision role |
| systemd unit | Service configuration read by systemd; starting is not health |
| Signal | OS notification to a process; TERM requests graceful shutdown |
| UID/GID / ACL | Numeric user/group identities / additional file-access rules |
| Mount / inode | Expose a filesystem at a path / file metadata and block references |
| RSS / page cache | Resident process memory / reusable cached file data |
| Namespace / cgroup | Process resource view / process-group resource accounting and control |
| WSL 2 | Linux kernel in a lightweight VM integrated with Windows |
| PowerShell object pipeline | Passes structured objects between cmdlets, unlike typical Bash text pipes |
| DNS | Name-to-record lookup; successful resolution does not prove reachability |
| TCP | Reliable ordered byte stream between endpoints |
| TLS | Connection encryption and peer identity verification |
| HTTP | Application request/response protocol with methods and status codes |
| CIDR | Prefix identifying a range; `10.1.0.0/24` contains 256 addresses |
| Longest prefix | Narrowest matching route; `/32` beats `/24` |
| NAT | Address/port translation |
| Stateful/stateless | Tracks connections / evaluates packets without that tracking |
| Latency/throughput | Time per operation / work completed per time |
| Tail latency | Slow end of a latency distribution; p99 is a percentile |
| Backpressure | Slow/reject producers when consumers cannot keep up |
| Timeout/deadline | Waiting duration limit / absolute expiry time |
| Retry/backoff/jitter | Retry operation / increase wait / randomize wait |
| Transaction | Database changes committed together under stated guarantees |
| Atomic operation | Indivisible state transition within a defined scope |
| Unique constraint | Database rejects duplicate key; supports event deduplication |
| Index | Extra lookup structure improving selected queries at write/storage cost |
| Idempotency | Repeated request causes the same intended logical effect |
| Outbox | Outgoing events stored transactionally with business records |
| Visibility timeout | SQS received message temporarily hidden; expiry permits redelivery |
| DLQ | Dead-letter queue for repeatedly failing work |
| Replication/sharding | Copy the same data / divide data across partitions |
| Consistency | Which read/write histories a system permits |
| Consensus/quorum | Agreement protocol / required subset of participants |
| Partition | Communication failure separating participants |
| Cache | Reusable data copy with freshness/invalidation tradeoffs |
| SLI/SLO | User-behavior measurement / target over a time window |
| Error budget | Allowed bad-event share implied by the SLO |
| RPO/RTO | Data-loss window target / restoration-time target |
| Logs/metrics/traces | Events / numerical aggregates / connected timed operations |
| Span | One timed operation within a trace |
| Cardinality | Number of label combinations; event-ID labels grow without bound |
| Invariant | Property that must remain true; one logical result per event |

## Cloud and deployments

| Term | Plain meaning and example |
|---|---|
| IAM/STS | AWS authorization/identity / temporary credential service |
| Principal | Identity making a request; an assumed role session |
| Trust/permissions policy | Who assumes role / what role sessions can do subject to other layers |
| SCP | Organization permission ceiling; grants nothing by itself |
| Region/AZ | AWS geographic deployment area / isolated location within it |
| VPC/subnet/ENI | Virtual network / AZ-specific range / virtual interface |
| TGW attachment | Connection between transit gateway and network |
| Association/propagation | Choose table for incoming TGW traffic / populate routes |
| RAM | Share supported resources across accounts; not a packet route |
| Control/data plane | Configure system / carry application traffic |
| IaC | Versioned desired infrastructure configuration |
| CDK/construct/stack | Code-to-template toolkit / building block / deployment unit |
| Synthesis/bootstrap | Generate template / create CDK deployment support resources |
| Provider/state/drift | API integration / resource mapping / divergence from configuration |
| CI/CD | Continuously validate code / prepare or deploy validated releases |
| OIDC | Federated identity protocol; workflow exchanges identity for AWS role access |
| Artifact | Build output such as container image or executable |
| Rollback | Return to earlier deployment; database recovery can require separate action |
| Container/image | Isolated process environment / package used to start it |
| Pod/Deployment/Service | Kubernetes scheduling unit / replica controller / stable access abstraction |
| Requests/limits | Scheduling reservations / resource ceilings with resource-specific behavior |
| Readiness/liveness | Ready for traffic / restart warranted |
| RBAC | Role-based permissions; Kubernetes RBAC and AWS IAM are separate |
| Reconciliation | Repeatedly converge observed state toward desired state |
| Operator/CRD | Operational controller / definition of custom Kubernetes API type |
| Finalizer | Holds deletion for cleanup; can stall if cleanup fails |

## Official resource library, in study order

Most reading is publicly accessible. Cloud experiments and some optional platforms cost money. Verify current installation prerequisites and record versions; this list does not freeze software versions.

| Resource | Read for | Deliverable |
|---|---|---|
| [Microsoft WSL](https://learn.microsoft.com/en-us/windows/wsl/install) | Linux environment | Versions and shell explanation |
| [Pro Git](https://git-scm.com/book/en/v2) | Basics, branches, remotes | Reviewed commits and conflict exercise |
| [Python tutorial](https://docs.python.org/3/tutorial/) | Functions, collections, modules, errors | Inventory normalizer |
| [Python venv](https://docs.python.org/3/tutorial/venv.html) | Package isolation | Reproducible environment |
| [Python ipaddress](https://docs.python.org/3/library/ipaddress.html) | CIDRs | Route membership tests |
| [Linux ps](https://man7.org/linux/man-pages/man1/ps.1.html), [ss](https://man7.org/linux/man-pages/man8/ss.8.html) | Process/socket inspection | Listener diagnosis |
| [GNU Bash manual](https://www.gnu.org/software/bash/manual/), [Linux man-pages](https://man7.org/linux/man-pages/) | Quoting, scripts, system interfaces | Reproducible shell and process labs |
| [Ubuntu Server docs](https://ubuntu.com/server/docs/), [systemd manuals](https://www.freedesktop.org/software/systemd/man/) | Services, logs, storage, security | VM incident and restore report |
| [Kernel cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) | Container resource control | Host/container limit explanation |
| [PowerShell docs](https://learn.microsoft.com/en-us/powershell/), [WSL networking](https://learn.microsoft.com/en-us/windows/wsl/networking) | Windows and WSL diagnostics | Paired OS investigation |
| [Boto3 guide](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/index.html) | Credentials, pagination, retries | Collector with partial failure states |
| [Go install](https://go.dev/doc/install), [Tour](https://go.dev/tour/), [tutorials](https://go.dev/doc/tutorial/) | Types, errors, packages, concurrency | Go normalizer and service |
| [Go diagnostics](https://go.dev/doc/diagnostics), [race detector](https://go.dev/doc/articles/race_detector) | Measurements and races | Profiled bounded worker pool |
| [Python asyncio](https://docs.python.org/3/library/asyncio.html) | Tasks and cancellation | Endpoint checker |
| [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html) | SQL, joins, transactions | Duplicate-write and rollback tests |
| [Docker](https://docs.docker.com/get-started/) | Images, containers, networks | Local application stack |
| [CLI SSO](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html) | Temporary identity | Correct-account check |
| [IAM evaluation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) | Authorization layers | Allow and deny tests |
| [VPC route priority](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html) | Route selection | Explainable evaluator |
| [Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) | Architecture tradeoffs | Design review |
| [CDK testing](https://docs.aws.amazon.com/cdk/v2/guide/testing.html), [bootstrap](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html), [Go](https://docs.aws.amazon.com/cdk/v2/guide/work-with-cdk-go.html) | Templates and assertions | Python stack, small Go equivalent |
| [Terraform tutorials](https://developer.hashicorp.com/terraform/tutorials), [state](https://developer.hashicorp.com/terraform/language/state) | Plan, drift, import | Separate disposable resource lifecycle |
| [GitHub OIDC to AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) | Temporary deployment identity | Scoped workflow trust |
| [Builders' Library retries](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) | Deadlines, retries, jitter | Failure-injection tests |
| [SQS delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html) | Duplicate delivery | Crash-safe worker demonstration |
| [Google SRE workbook](https://sre.google/workbook/table-of-contents/) | SLOs, monitoring, incidents | Objective, dashboard, review |
| [Kubernetes concepts](https://kubernetes.io/docs/concepts/), [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) | Local cluster | Restart and rollout experiment |
| [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) | Reconciliation | Repeated-reconcile tests |
| [Amazon technical preparation](https://amazon.jobs/content/en-gb/how-we-hire/interview-prep/software-development-topics), [SDE II](https://amazon.jobs/content/en-gb/how-we-hire/sde-ii-interview-prep) | Role-specific interview preparation | Mocks and factual work stories |
| [AWS calculator](https://calculator.aws/), [Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) | Estimate and track cost | Estimate, actual cost, teardown record |

For deeper study, choose weighted graphs, OS internals, consensus, database isolation, or a provider implementation when a project exposes a specific gap. Ask of every new resource: what prerequisite does it assume, what will I implement, and how will I test understanding? Defer it if you cannot answer.
