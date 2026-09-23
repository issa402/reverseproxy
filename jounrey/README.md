# Isaac's cloud, software, and systems engineering journey

Prepared September 22, 2026. Start with [01 — Setup and the first 14 days](01-start-here.md). This directory deliberately uses your requested spelling, `jounrey`.

The goal is to become an engineer who can build, explain, debug, measure, and improve a service from source code through its cloud dependencies. Google and Amazon are career targets; readiness depends on the role, demonstrated experience, available positions, and interview performance. This curriculum is a proposed training plan, not a hiring guarantee or a claim that anyone can know every tool.

## Read in this order

| File | What you will do |
|---|---|
| [01 — Start here](01-start-here.md) | Set up Windows/WSL, Git, Python, Go, AWS authentication; finish the first 14 days |
| [02 — Linux, networking, and debugging](02-linux-networking-debugging.md) | Trace a request, inspect processes, understand CIDRs, distinguish network and application failures |
| [03 — Python and Go](03-python-and-go.md) | Build tested programs, manage concurrency, measure performance |
| [04 — Algorithms and interviews](04-algorithms-and-interviews.md) | Practice reasoning, complexity, coding interviews, and behavioral evidence |
| [05 — AWS and infrastructure as code](05-aws-and-infrastructure-as-code.md) | Learn accounts, IAM, networking, storage, CDK, Terraform, and deployment testing |
| [06 — Distributed systems and reliability](06-distributed-systems-and-reliability.md) | Understand queues, failures, consistency, observability, Kubernetes, and controllers |
| [07 — Project ladder and capstone](07-projects-and-capstone.md) | Deliver progressively harder systems with acceptance tests and measurable evidence |
| [08 — Weekly plan and mentoring](08-weekly-plan-and-mentoring.md) | Follow 48 study weeks, track progress, request useful feedback, and prepare applications |
| [09 — Glossary and resource library](09-glossary-and-resources.md) | Look up terminology and choose the next reading with a specific exercise |
| [Templates](templates/) | Record designs, experiments, incidents, learning, and interview stories |

## Choose the role deliberately

These are training emphases, not universal company interview rubrics. Check current job descriptions and confirm the loop with the recruiter.

| Target | Main work to demonstrate | Training emphasis |
|---|---|---|
| Cloud/platform engineer | Reusable deployment systems, account governance, networking, developer tooling | AWS, IaC, automation, Linux, reliability, software testing |
| Systems development engineer | Software that operates and automates infrastructure | Python/Go, Linux, debugging, operations, systems design |
| Site reliability engineer (SRE) | Software and systems work to keep services reliable | Coding, Linux/networking, service objectives, capacity, incident response |
| Software development engineer (SDE/SWE) | Application and service implementation | Algorithms, code design, testing, databases, distributed services |
| Solutions architect | Translate business needs into architectures and explain tradeoffs | Requirements, architecture, security, cost, communication; role-specific coding depth |

Amazon publishes coding and software-design preparation, and its SDE II guidance includes behavioral evidence. Google describes SRE as a combination of software and systems engineering. Neither implies that knowing AWS services or learning Go alone is enough. Sources: [Amazon software topics](https://amazon.jobs/content/en-gb/how-we-hire/interview-prep/software-development-topics), [Amazon SDE II](https://amazon.jobs/content/en-gb/how-we-hire/sde-ii-interview-prep), [Google SRE](https://sre.google/books/).

For your current infrastructure background, start with platform/systems engineering foundations and preserve a strong coding track. Revisit target roles after the first two completed projects. Learn Python first for automation and interview fluency, then Go for services and concurrency. Learn CDK in Python deeply before translating one stack to Go. Learn one Terraform implementation before considering a custom provider. Kubernetes operators and consensus implementations come after simpler service and failure-handling projects.

## How to use the curriculum

Budget assumption: about 10 focused hours per week, around 480 hours over 48 study weeks. This is a planning estimate. Repeat a week when its checkpoint is not met. At 5 hours/week, spread each study week across two calendar weeks. At 15 hours/week, add testing, reviews, and deeper experiments before racing ahead.

Every topic follows the same loop:

1. Predict what will happen and write your reasoning.
2. Build the smallest working example.
3. Test successful, invalid, and failed cases.
4. Change one variable and observe the result.
5. Explain the result without reading the source.
6. Rebuild the essential part several days later.

Reading creates context. Evidence that you can perform the task earns progress. A copied project that you cannot debug has not passed.

## The competence scorecard

Use the same 0–4 scale for Linux, networking, Python, Go, AWS/IAM, IaC, databases, distributed systems, debugging, algorithms, and communication.

| Score | Demonstration |
|---|---|
| 0 | Cannot yet explain the concept |
| 1 | Can explain with notes and follow a tutorial |
| 2 | Can complete a small task independently and test it |
| 3 | Can diagnose an unfamiliar failure and justify tradeoffs |
| 4 | Can teach, review others' designs, and operate it under realistic constraints |

Target consistent 2s before advanced projects and several 3s before using a project as strong interview evidence. These are internal learning standards, not Amazon or Google cutoffs.

## Your portfolio should show judgment

Each substantial project needs a README, reproducible setup, tests, diagram, design decisions, performance experiment, failure experiment, runbook, and known limitations. Preserve actual measurements with hardware/runtime/configuration details. Never invent scale, savings, uptime, or production impact.

Use synthetic accounts, IPs, people, and datasets in public examples. Your work experience can inspire problems such as ownership discovery and routing diagnosis; company inventories and credentials stay in their approved environment.

The console is useful for inspecting what deployed, checking logs, and understanding failures. IaC becomes the reviewed source of desired configuration. Your goal is to understand the relationship between both.

## Start today

Open [01-start-here.md](01-start-here.md), complete Day 1, and commit a baseline self-assessment using the [learning log](templates/learning-log.md). Bring the commands, outputs, code, and your explanation to a mentoring session. We can then review what you actually understand and choose the next exercise.
