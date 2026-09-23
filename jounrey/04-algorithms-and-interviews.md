# Algorithms, problem solving, and interviews

This track builds reasoning you can use both in production and during interviews. A coding puzzle score does not measure all engineering ability, and a cloud certification does not replace coding ability. Train them as complementary skills.

Use Python as your first interview language if it currently lets you express correct ideas faster. Use Go if the role and your fluency make it the better choice. Ask the recruiter which languages and tools are supported for your actual interview.

## 1. Choose the job family before optimizing preparation

| Target | Evidence you should build | Preparation emphasis |
|---|---|---|
| Software engineer / SDE | Tested services, APIs, data modeling, coding decisions | Algorithms, software design, system design appropriate to level |
| Platform / infrastructure software engineer | Automation, control planes, developer tooling, reliable services | Strong coding plus Linux, networking, distributed systems, operational tradeoffs |
| SRE | Reliability improvements, incident reasoning, automation, recovery tests | Coding plus systems troubleshooting, observability, capacity and reliability |
| Cloud support / infrastructure operations | Structured diagnosis, customer communication, network/IAM/service knowledge | Troubleshooting depth and role-specific scripting |
| Solutions architect | Requirements discovery, architecture tradeoffs, migrations, communication | Customer scenarios, cloud breadth and depth, cost and reliability |

Titles vary. A “cloud engineer” opening can be mostly operations at one employer and mostly software development at another. Read duties, minimum qualifications, and interview guidance for each specific requisition.

Amazon's official [software development interview topics](https://amazon.jobs/content/en/how-we-hire/interview-prep/software-development-topics) describe preparation across coding and computer-science topics. Its [interviewing page](https://www.amazon.jobs/en/landing_pages/interviewing-at-amazon) explicitly notes that processes differ by role. Treat recruiter instructions as the authority for your loop.

Google's published [technical interview advice](https://blog.google/company-news/inside-google/life-at-google/google-engineer-shares-her-technical-interview-tips/) is useful preparation guidance, not a promise of a fixed modern interview format. Obtain current candidate materials from your recruiter.

## 2. What an algorithm actually is

An algorithm is a repeatable procedure for solving a problem. “Find the first route whose destination matches” is an algorithm, but it can be incorrect when route selection requires the most specific matching prefix.

A data structure is a way to organize information so required operations are convenient. A dictionary provides fast key lookup; a queue preserves arrival order; a graph describes relationships.

An invariant is a statement that remains true during an algorithm. In a sliding window with no duplicate characters, the invariant is that every character inside the current window is unique.

A proof of correctness explains why the algorithm produces the required answer for every valid input. Start with plain language: what is true before an iteration, what the iteration preserves, and what is true when it stops?

## 3. Complexity without mystery

Let `n` represent input size. Big-O describes how work or extra memory grows as input grows; it is not an exact stopwatch prediction.

| Complexity | Example | At 1,000 items, roughly |
|---|---|---|
| O(1) | Access one array index | Constant number of operations |
| O(log n) | Binary search over sorted IDs | About 10 comparisons |
| O(n) | Read every resource once | 1,000 inspections |
| O(n log n) | Comparison sorting | About 10,000 comparison-scale steps |
| O(n²) | Compare every resource with every other | About 1,000,000 pair-scale steps |

These counts illustrate growth, not precise runtime. Hash-table lookup is commonly expected O(1), but implementation, collisions, and adversarial inputs matter. Sorting may use different extra memory depending on the language and algorithm.

Practice distinguishing input storage from auxiliary space: a function receiving a million-row list may use O(1) additional space while the caller still stores that large list.

## 4. Your first diagnostic

Try these untimed, without a solution video:

1. Count resource kinds in a list of inventory rows.
2. Remove duplicate IDs while preserving first-seen order.
3. Find the largest and second-largest distinct values.
4. Check whether parentheses are balanced.
5. Explain the runtime of two nested loops versus two separate loops.

If you struggle with syntax, revisit the language module. If syntax is fine but choosing a structure is difficult, begin the stages below. There is no reason to start with hard dynamic-programming problems.

## 5. Stage A: arrays, strings, dictionaries, sets

An array/list stores a sequence. A dictionary/map relates keys to values. A set answers whether a value is present without storing an associated payload.

Exercises:

1. Find duplicate snapshot IDs.
2. Count requests per hostname.
3. Determine whether two strings contain identical character counts.
4. Find two numbers whose sum equals a target.
5. Merge two account inventories and report added, removed, and changed resources.

Cloud connection: inventory comparison often becomes clearer when you build a map keyed by `(account, region, resource_id)` rather than repeatedly scanning all rows.

Tests: empty list, repeated IDs, all-equal values, negative numbers, no answer, multiple valid answers, and case-sensitive names.

Hint ladder for two-sum: first compare pairs; then ask what information would let you avoid revisiting earlier values; then consider storing what you have seen.

Pass: solve a new variation, explain why the chosen key is unique, and compare the brute-force and improved complexity.

## 6. Stage B: two pointers, windows, and prefix sums

Two pointers track positions in a sequence. They may move toward each other or in the same direction. This is useful when ordering lets you discard impossible candidates.

A sliding window tracks a contiguous portion of input while updating its state as boundaries move. A prefix sum stores cumulative totals so range totals become quick subtraction.

Exercises:

1. Merge two sorted event timelines.
2. Find the longest substring without repeated characters.
3. Count failures in each fixed-size window of observations.
4. Find the sum between two indexes using prefix sums.
5. Find the shortest interval reaching a target under explicitly nonnegative values.

The nonnegative condition matters: when negatives are allowed, expanding a window can reduce its sum, breaking a simple window argument.

Tests: window larger than input, zero-length input, equal values, repeated boundary values, and inputs violating your assumptions.

Pass: state what each pointer means and why it never needs to move backward. If that argument fails, reconsider the algorithm.

## 7. Stage C: stacks, queues, linked lists

A stack removes the most recently added item first: think nested function calls. A queue removes the earliest added item first: think jobs awaiting a worker.

A linked list connects nodes through references. It supports pointer manipulation exercises, although an array or deque is often more practical in ordinary application code.

Exercises:

1. Validate nested brackets.
2. Simplify a path containing `.` and `..` with clearly stated rules.
3. Reverse a linked list.
4. Detect a linked-list cycle.
5. Build a bounded queue and define what happens when it fills.

Tests: one node, empty structure, self-cycle, unmatched closing bracket, deeply nested input, queue overflow.

Pass: draw the state before and after a pointer update. Explain why saving the next reference is necessary before overwriting a link.

## 8. Stage D: binary search and intervals

Binary search repeatedly eliminates half a search space. Its real prerequisite is a monotonic condition: once a condition becomes true, it stays true in the relevant ordering.

Intervals represent ranges such as maintenance windows or IP address spans. Overlap problems often become easier after sorting by start position.

Exercises:

1. Find an ID in a sorted list.
2. Find the first version where a simulated health check fails.
3. Merge overlapping maintenance windows.
4. Determine whether a new booking overlaps an existing one.
5. Implement IPv4 longest-prefix matching against synthetic routes.

For the route exercise, `/24` is more specific than `/16`; select the most specific matching route, not the first row. Keep AWS-specific route priority exceptions outside this simplified exercise and state that boundary.

Tests: target absent, first/last element, one element, touching interval boundaries, duplicate starts, default route only, and no matching route.

Pass: explain whether your intervals are closed or half-open and why your binary search always shrinks the search space.

## 9. Stage E: trees, heaps, and recursion

A tree has parent/child relationships without cycles in the usual rooted-tree model. Recursion solves a problem by calling the same procedure on smaller subproblems. A heap efficiently retrieves an extreme value without keeping the entire structure fully sorted.

Exercises:

1. Traverse an OU hierarchy and print the full path of each account.
2. Find a tree's maximum depth.
3. Validate the ordering rules of a binary search tree.
4. Find the top `k` most frequent resource types.
5. Merge `k` sorted event streams.

Tests: empty tree, highly unbalanced tree, repeated values under an explicit policy, `k=0`, `k` larger than input, and equal priorities.

Hint: for top `k`, consider whether you need to sort everything or only retain the current best `k` candidates.

Pass: explain recursion depth and when an iterative implementation avoids stack-limit problems. Do not assume every tree is balanced.

## 10. Stage F: graphs and dependencies

A graph contains vertices and edges. A vertex can represent an application, subnet, or account; an edge can represent “depends on” or “connects to.” Directed edges have a direction.

Breadth-first search (BFS) explores layer by layer. Depth-first search (DFS) follows a branch before backtracking. A topological ordering respects dependency direction in a directed acyclic graph; cycles prevent such an ordering.

Exercises:

1. Find connected components in an invented account connectivity map.
2. Find the fewest hops between two nodes in an unweighted graph.
3. Detect a cycle in resource dependencies.
4. Compute a valid deployment order.
5. Reverse dependencies to identify resources affected by a proposed deletion.

Cloud connection: graph reachability is an abstraction, not proof that TCP connectivity works. Routes, security rules, listeners, and credentials still determine whether a real application request succeeds.

Tests: isolated nodes, cycle, disconnected graph, duplicate edges, self-edge, and multiple valid orderings.

Pass: explain why BFS finds a shortest path for equal-weight edges and why that does not automatically apply to weighted edges.

## 11. Stage G: greedy reasoning and dynamic programming

A greedy algorithm makes a locally attractive choice. You must justify why those choices produce a globally correct result; intuition alone is insufficient.

Dynamic programming reuses answers to overlapping subproblems. A state describes a subproblem, a recurrence relates states, and a base case starts the computation.

Exercises:

1. Select the maximum number of non-overlapping tasks.
2. Count ways to climb steps under a stated move rule.
3. Find minimum coins for a target and demonstrate why a simple greedy choice can fail.
4. Find a maximum-value selection under a small capacity constraint.
5. Compare recursive, memoized, and table-based solutions to one problem.

Tests: zero target, impossible target, duplicate choices, negative values if supported, and very small cases you can enumerate manually.

Hint ladder: define a tiny subproblem; identify which prior answers determine it; write those dependencies before writing loops.

Pass: derive the state and recurrence aloud and explain time/space from the number of states and work per state.

## 12. A repeatable problem-solving session

Use this 45-minute practice format after you know the relevant foundations. It is a training format, not a claim about a company's interview duration.

1. Minutes 0–5: restate the problem and clarify constraints.
2. Minutes 5–10: make examples and describe a correct simple approach.
3. Minutes 10–15: identify the bottleneck and choose a structure.
4. Minutes 15–30: implement while explaining invariants and tradeoffs.
5. Minutes 30–40: manually test and correct defects.
6. Minutes 40–45: discuss complexity and a changed requirement.

When stuck, ask for one hint after a serious attempt. Record what the hint changed in your reasoning. Copying the final solution and moving on creates recognition, not independent problem solving.

## 13. Practice menu and review

Use these sessions within the time budget in [08-weekly-plan-and-mentoring.md](08-weekly-plan-and-mentoring.md), adjusting the mix to the current week's goals.

- Two sessions: learn one pattern using easy exercises.
- Two sessions: solve unfamiliar moderate variations.
- One session: re-solve a previous failure without notes.
- One session: explain a project design or diagnose a system failure.
- Every second week: one mock interview with feedback.

Review difficult problems after roughly two days, one week, and one month. These are convenient practice intervals, not a guaranteed memory formula.

Track `problem`, `pattern`, `first approach`, `missed assumption`, `hint needed`, `test missed`, and `revisit result`. Count independent reasoning progress rather than total problems viewed.

## 14. System-design interview practice

System design asks how components work together under requirements and failures. It does not reward naming the largest number of AWS services.

Use this sequence: clarify users and operations; estimate traffic/data; define APIs and data; draw a simple working design; identify the bottleneck; add failure handling; explain security, cost, and observability.

Example: design an inventory collection service. Clarify whether it scans ten accounts nightly or thousands every hour. Decide how pagination progress survives a worker crash. Explain rate limits, partial permissions, duplicate jobs, stale data, and who can read reports.

Practice increasing scope:

1. One-process file-backed inventory tool.
2. HTTP API with a database and authentication.
3. Queue plus workers across multiple accounts.
4. Multi-tenant service with isolation, quotas, and recovery.

Read selected chapters of the official [Google SRE books](https://sre.google/books/) alongside your projects: service objectives, monitoring, overload, incident handling, and distributed-system failure. Treat them as design reasoning resources, not proof that every Google team operates identically.

## 15. Behavioral evidence from your current work

Use Situation, Task, Action, Result (STAR) to structure a story. Add what you learned and what you would change.

Potential stories: discovering an unexpected default route, preserving data before account retirement, separating creator identity from business ownership, or replacing a manual process with tested automation.

For each story record your actual contribution, alternatives considered, evidence gathered, collaboration, measurable outcome, and remaining limitations. Do not claim a migration or remediation succeeded unless you have validation evidence.

Practice explaining the same story in 90 seconds and five minutes. Use synthetic identifiers in public portfolio material and preserve confidentiality when describing employer incidents.

## 16. Readiness gates

You are ready to begin targeted applications when you can demonstrate a meaningful portion of the actual job requirements; you do not need to finish all computer science first.

For coding: solve unfamiliar foundational problems, explain complexity, test boundaries, and respond to a changed requirement without restarting from a memorized answer.

For engineering: build and debug a tested service, handle partial failure, read logs and metrics, and explain one design tradeoff using measurements.

For communication: distinguish observed facts from assumptions, discuss mistakes constructively, and give concise answers before adding detail.

For interviews: complete several realistic mocks, classify recurring weaknesses, and revisit them. No curriculum or problem count guarantees a particular hiring outcome.

Bring your mentor a solution, tests, and reasoning: “Ask me one follow-up that exposes a weakness. Let me revise before showing another approach.” That makes preparation an active skill rather than a collection of answers.
