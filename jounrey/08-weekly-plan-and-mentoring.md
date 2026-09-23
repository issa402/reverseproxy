# 08 — Forty-eight study weeks and a mentoring routine

This is a proposed sequence at approximately 10 hours/week. Dates are intentionally absent so the plan survives vacations and difficult topics. Advance on evidence. Weeks containing large projects may repeat.

## A practical weekly schedule

| Session | Time | Work |
|---|---|---|
| Monday | 90 minutes | Read one focused section; explain it; implement a tiny example |
| Tuesday | 90 minutes | Project implementation and behavior tests |
| Wednesday | 90 minutes | Algorithms with reasoning and edge cases |
| Thursday | 90 minutes | Linux operating-system diagnosis tied to the current project; on paired weeks compare Windows |
| Saturday | 3 hours | Integrate the project, measure, write evidence |
| Sunday | 1 hour | Recall from memory, review, plan next week |

Keep Friday free or use it for recovery. If a session is missed, resume the next prerequisite; do not compensate by dropping tests and explanations. Use [10 — Linux mastery](10-linux-mastery.md) for depth, [11 — Windows companion](11-windows-companion.md) for Windows workloads, and [12 — the week-by-week OS track](12-operating-system-weekly-track.md) for the exact Thursday exercise. This work fits the existing ten-hour schedule.

## Ordered weekly backlog

| Week | Focus | Concrete deliverable | Checkpoint question |
|---|---|---|---|
| 1 | Setup, Python basics, Git | First-half tasks in module 01 | Can you explain what ran and where? |
| 2 | Parsing, testing, initial Go | Day-14 independent report | What happens with corrupt input? |
| 3 | Linux processes/files | Debugging lab notes | Is the service running and listening? |
| 4 | DNS/TCP/TLS/HTTP | Layer-by-layer request explanation | Which observation isolates the failed layer? |
| 5 | Python data model | Inventory engine schema/parser | What uniquely identifies a resource? |
| 6 | Exceptions and tests | Invalid/partial-data tests | Is unknown distinguishable from zero? |
| 7 | API adapters and pagination | Fixture-backed collector adapter | What happens on page two failure? |
| 8 | Inventory integration | Project 1 demo and retrospective | Can another person reproduce the report? |
| 9 | CIDR and route rules | Route selector with tests | Why does this prefix win? |
| 10 | Graphs and topology | TGW association fixtures | Which table is consulted on arrival? |
| 11 | Network evidence | Bidirectional route report | What remains unproven without live tests? |
| 12 | Checkpoint and interview baseline | Project 2 demo plus one coding mock | Can you diagnose an unfamiliar fixture? |
| 13 | Go types/errors/interfaces | Small Go service | Why is this interface needed? |
| 14 | HTTP validation/context | Timeout and malformed-request tests | What cancels downstream work? |
| 15 | Goroutines/synchronization | Bounded worker pool | Can concurrency exceed its limit? |
| 16 | Profiling and benchmarks | Before/after measurement | What is the measured bottleneck? |
| 17 | SQL schema/indexes | Event/result tables and queries | Why this index and key? |
| 18 | Transactions/idempotency | Concurrent duplicate tests | Can two workers both apply the effect? |
| 19 | Containers | Reproducible local API/database stack | Where does persistent data live? |
| 20 | Service checkpoint | Capstone Stage A and restore exercise | What survives a restart? |
| 21 | AWS identity/accounts | Lab access and policy evaluation notes | Who is calling; what authorizes it? |
| 22 | AWS networking | Small timed VPC experiment or fixtures | How does a packet leave and return? |
| 23 | AWS storage/backups | Disposable-data restore evidence | Does the recovery point actually restore? |
| 24 | CDK Python basics | Synthesized small stack with assertions | What CloudFormation will change? |
| 25 | CDK constructs/testing | Reusable construct and negative tests | Which unsafe input is rejected? |
| 26 | Go CDK translation | Equivalent small stack comparison | What differs in language, not AWS behavior? |
| 27 | Terraform/OpenTofu | Separate equivalent lab and state exercise | How is real infrastructure linked to state? |
| 28 | IaC checkpoint | Drift, import, replacement, cleanup report | What would a mistaken replacement destroy? |
| 29 | Durable queue/outbox | Capstone Stage B publisher | What if enqueue succeeds and acknowledgment is lost? |
| 30 | Python worker | Duplicate/retry/shutdown tests | Is retry safe after a partial success? |
| 31 | Monitoring and objectives | Logs, metrics, SLI/SLO proposal | What user failure triggers an alert? |
| 32 | Load and backpressure | Stage C benchmark | Does backlog grow faster than it drains? |
| 33 | CDK AWS service deployment | Stage D in sandbox | Which resources create recurring costs? |
| 34 | CI and deployment identity | Tests and scoped OIDC design | Can an untrusted branch assume the deploy role? |
| 35 | Failure drills | Three timed incidents | What evidence disproved your first hypothesis? |
| 36 | AWS capstone review | Cost, recovery, design, demo | Why did you choose this compute model? |
| 37 | Kubernetes local basics | Deployment and Service | How do desired and actual replicas differ? |
| 38 | Probes/resources/shutdown | Failure and capacity experiments | Can a bad readiness probe worsen an incident? |
| 39 | Controller concepts | Reconciliation prototype/spec | Is the second reconcile safe? |
| 40 | Optional operator or provider | One tested extension | What is its state and failure model? |
| 41 | Distributed design | Two written system designs | What fails during a partition? |
| 42 | Performance and database tradeoffs | Capacity model and query tuning | Which assumption dominates the estimate? |
| 43 | Coding interviews | Two reviewed mocks and corrections | Can you reason while implementing? |
| 44 | Systems/operations interviews | Two unfamiliar troubleshooting mocks | What is your first discriminating test? |
| 45 | Behavioral evidence | Six factual story drafts | What did you personally decide and learn? |
| 46 | Portfolio polish | Reproducible READMEs and two demos | Can a reviewer validate your claims? |
| 47 | Role-specific preparation | Three current job requirement maps | Where is evidence still weak? |
| 48 | Final assessment/applications | Interview plan and next quarter goals | What level and role fit the demonstrated work? |

Algorithms continue weekly from the beginning; weeks 43–44 add focused mocks rather than introducing interviews for the first time. Once basics are comfortable, spend 15 minutes of the Sunday review explaining a design tradeoff aloud.

## Checkpoints every four weeks

Ask a reviewer to score code correctness, testing, explanation, diagnosis, and judgment from 0–4 using the README scale. Record evidence for every score. Do not average away a severe weakness such as unhandled duplicate side effects just because documentation is polished.

If below 2 in the current prerequisite, repeat with a smaller exercise. If at 2, advance and keep a maintenance exercise. If at 3, add an unfamiliar failure or review another design. A high score from one memorized demo needs independent confirmation.

## How to use me as a mentor

Paste this with each session:

```text
Current module/week:
What I think the concept means:
What I built:
Input and expected result:
Actual result and relevant error:
What I already tried:
My current hypothesis:
What I want: explanation / hint / review / mock interview
```

For learning, request hints before a full solution. After receiving code, close it and reconstruct a smaller version. Explain each dependency, error path, and test. Use AI to challenge reasoning, generate adversarial inputs, and review tradeoffs; use independent implementation and recall to establish understanding.

Mentor prompts:

- “Ask me five questions about this module, one at a time. Require an example and correct my reasoning.”
- “Review this code for correctness, failure handling, simplicity, and test gaps. Give the three most important changes first.”
- “Give me a synthetic route-table puzzle and wait for my next-hop prediction.”
- “Inject one bug into this local fixture. Give no diagnosis until I explain my evidence.”
- “Run a 40-minute coding mock. Evaluate clarification, reasoning, implementation, complexity, and tests.”
- “Act as an application owner. Challenge the retention/RPO assumptions in my design.”
- “Ask why each AWS resource exists and whether a simpler design satisfies the requirement.”

A mentoring session ends with one corrected mental model, one tested change, and the next exercise. It does not need ten new tools.

## When you get stuck

After about 20–30 focused minutes without progress, write the smallest failing input and the exact unexpected behavior. Read the relevant primary documentation, not ten unrelated tutorials. Ask for a targeted hint. If you cannot state the expected behavior, clarify the problem before changing implementation.

For a concept you forget repeatedly, create a retrieval question. Example: “Why can a duplicate queue delivery repeat a side effect even when message processing succeeded?” Answer from memory after one day, one week, and one month. The intervals are a suggested practice routine, not a scientific guarantee.

## Role selection and job search

Review current openings monthly once you have two solid projects. Keep a matrix with role title, team, location, minimum qualifications, preferred qualifications, interview topics confirmed by recruiter, and your evidence. Do not assume Google requires Go or that an Amazon software role is primarily an AWS service trivia test.

For platform/SysDE roles, emphasize tested automation, Linux/network diagnosis, IaC, and incidents. For SWE/SDE roles, strengthen coding, algorithms, API/data modeling, and service implementation. For SRE, practice coding alongside operating-system, networking, reliability, and capacity problems. For architect roles, add requirements discovery and stakeholder communication.

Read [Amazon's official technical topics](https://amazon.jobs/content/en-gb/how-we-hire/interview-prep/software-development-topics), [SDE II preparation](https://amazon.jobs/content/en-gb/how-we-hire/sde-ii-interview-prep), and [Google's SRE books](https://sre.google/books/). These inform preparation; actual loops depend on role and level.

## Turning real work into honest interview evidence

Use the [story template](templates/interview-story.md). Explain situation, responsibility, actions, results, and reflection. Make your personal decisions explicit, acknowledge collaborators, and provide measured outcomes only when available.

Good claim: “Built a report that distinguishes failed inventory collection from an empty account and tested pagination and permission failures.” Weak claim: “Automated all cloud governance.”

Good claim: “Traced forward and return routes and documented which ingress association selected the TGW table.” Weak claim: “Fixed networking” when only diagnosis was completed.

Use internal workplace examples only at an appropriate level of confidentiality. Public project fixtures must be synthetic. Keep a factual distinction among proposed, implemented, validated, and deployed work.

## Certificates and courses

A certification can organize AWS study and meet a specific job requirement. It is optional within this curriculum and does not replace code, debugging, or restore demonstrations. Choose a current certification only after comparing the exam guide with your actual knowledge gaps and budget. Avoid collecting several exams before finishing a project.

## Final readiness demonstration

1. Implement a fresh small coding problem with edge-case tests and explain complexity.
2. Diagnose a planted service/network failure using evidence.
3. Design a service with requirements, storage, failures, security, costs, and measured recovery assumptions.
4. Review an IaC change and identify a replacement, permission, or networking risk.
5. Demonstrate a project restart/restore and reconcile all accepted work.
6. Explain a disagreement or mistake using a factual work story.

The remaining weak points become the next quarter's plan. Begin applications when the role requirements fit your evidence; no need to wait until every advanced topic in this directory is complete.
