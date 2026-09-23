# Python and Go: become someone who can build and debug

Use Python first for your inventory/reporting work and algorithm practice. Add Go after you can build a tested Python tool without following a tutorial line by line. This is a teaching sequence, not a claim that either company prefers one language for every role.

Your target is practical fluency: explain the program, change its behavior, test failures, and investigate why it is slow. Knowing advanced vocabulary without those abilities is not mastery.

## 1. How to work through this module

1. Read the named reference section for at most 30 minutes.
2. Close it and implement the small exercise from a blank file.
3. Write tests before adding complicated behavior.
4. Compare your implementation against the reference.
5. Explain one bug and one tradeoff in your learning journal.
6. Rebuild the exercise a week later without copying your first version.

Spend roughly half your language practice writing code, a quarter testing/debugging, and a quarter reading/explaining. Adjust the balance when you discover a weakness.

Use synthetic AWS inventories in public projects. Replace employer account IDs, names, addresses, credentials, and business data with invented fixtures.

## 2. Core words, translated

| Word | Meaning | Example |
|---|---|---|
| Variable | A name associated with a value | `account_id = "111122223333"` |
| Type | A category that determines valid operations | An account ID is a string, not a number to add |
| Function | A named operation with inputs and an output | `normalize_tags(resource)` returns cleaned tags |
| Collection | Several values stored together | A list of snapshots or a map from ID to resource |
| Mutation | Changing an existing object | Appending a row to a list |
| Side effect | An observable action outside the returned value | Writing a file or calling AWS |
| Pure function | Computes an output without external side effects | Determining whether an IP belongs to a CIDR |
| Exception/error | A failure the program must represent or handle | Invalid JSON or an API permission denial |
| Module/package | A unit of reusable code | Separate inventory parsing from AWS calls |
| Dependency | Something your program needs | A Python library or a database |
| Interface | A contract describing supported operations | An inventory reader exposes `list_resources()` |
| Serialization | Turning structured data into a portable representation | A resource object becomes JSON |
| Idempotency | Repeating an operation has the same intended effect | Processing the same event ID twice creates one report row |
| Concurrency | Multiple tasks make progress over overlapping time | Several requests wait for network responses |
| Parallelism | Work executes simultaneously | Two CPU cores process different inputs |
| Backpressure | Slowing producers when consumers cannot keep up | Stop accepting jobs when a bounded queue fills |

## 3. Python foundation: data before AWS

Read the official [Python tutorial](https://docs.python.org/3/tutorial/), focusing on control flow, data structures, modules, exceptions, classes, and file handling. Use its examples to learn syntax; the labs below are your independent work.

Start with these building blocks:

- `str`: keep IDs as text, including leading zeroes.
- `list`: preserve an ordered sequence of records.
- `dict`: map a resource ID to its metadata.
- `set`: track IDs already seen.
- `tuple`: represent a small fixed grouping such as account and Region.
- `None`: represent absence explicitly; it does not mean zero or an empty string.
- `with`: ensure a file or other resource is cleaned up when work finishes.
- `try/except`: handle a specific failure without hiding unrelated errors.

### Lab P1: inventory normalizer

Create a tool that reads a synthetic CSV with `account_id,region,resource_id,tags` and writes normalized JSON.

The `tags` cell contains JSON such as `{"Application":"Billing"}`. Use Python's CSV parser; splitting a line on commas breaks quoted JSON cells.

Requirements:

1. Keep account IDs as strings.
2. Reject missing IDs with a row number and explanation.
3. Parse tags as an object; reject arrays or malformed JSON.
4. Preserve the original record in the output for evidence.
5. Produce deterministic output: the same input produces the same ordering.
6. Return a nonzero exit status when any row fails validation.

Test cases: empty file, header-only file, quoted commas, Unicode names, duplicated resource IDs, malformed tags, missing Region, and a leading-zero account ID.

Pass: every valid row survives unchanged except documented normalization; every invalid row has a visible reason; reruns do not accumulate duplicate output.

Hint: split `read_rows`, `validate_row`, `normalize_row`, and `write_report`. Only the first and last need filesystem access.

### Lab P2: ownership evidence classifier

Input: normalized resources with optional `BusinessOwner`, stack name, creator role, and recent metrics.

Output: `owner_candidate`, `confidence`, `evidence`, and `needs_review`.

Do not invent an owner from a name. A deployment role is evidence of creation, not proof of business accountability. A resource with no observed metrics is `activity_unknown` unless your observation coverage is known.

Test cases: conflicting owner tags, AWS service role creator, missing metrics, inherited stack owner, and two teams claiming the same resource.

Pass: ambiguous cases remain ambiguous; every recommendation can be traced to a specific input field.

Hint: write explicit rules as data or small functions before considering elaborate classes.

## 4. Python engineering: turn scripts into maintained software

Learn type hints, `dataclasses`, logging, configuration, tests, and packaging. Type hints describe intended values for tools and readers; they do not automatically validate untrusted runtime inputs.

Separate your program into these responsibilities:

```text
input adapter -> validated model -> business rules -> output adapter
CSV or AWS       Resource          ownership         JSON/report
```

An adapter translates between your application and an external format or service. This lets your business logic run against local fixtures rather than requiring live AWS for every test.

Use the `pytest` setup from [01-start-here.md](01-start-here.md) for your project tests. The standard-library [unittest](https://docs.python.org/3/library/unittest.html) runner is also useful when you need to avoid third-party dependencies:

```bash
python -m unittest discover -s tests -v
```

### Lab P3: read-only AWS collector

Add an AWS adapter to P1 using Boto3. Use the normal credential provider chain with an approved profile or temporary session. Check the caller account before collecting data.

Learn these API realities:

- Pagination means one response may contain only part of the inventory.
- Throttling means the service limits request rate; retrying faster makes it worse.
- Backoff means waiting longer after retryable failures.
- Jitter means varying retry waits so clients do not all retry together.
- An access-denied result means incomplete visibility, not zero resources.
- Region scope means inventory in one Region does not prove absence elsewhere.

Use the official [Boto3 paginator guide](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/paginators.html) and [retry guide](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html).

Tests must simulate two pages, denied access, an empty successful response, a retryable failure, and a permanent failure. Inspect final coverage in the output: which accounts, Regions, services, and time windows actually succeeded?

Pass: your report differentiates `complete`, `partial`, and `failed`; it never prints credentials; running tests makes no real AWS calls.

## 5. Python concurrency: learn why before adding it

I/O-bound work spends time waiting for disks or networks. CPU-bound work spends time computing. A thousand waiting API calls and a thousand image transformations need different designs.

`asyncio` coordinates cooperative tasks around an event loop: tasks yield when awaiting compatible asynchronous operations. Writing `async def` does not make a blocking library asynchronous. Boto3's normal calls are synchronous; choose bounded threads or a deliberately selected asynchronous integration instead of assuming `await` changes that.

Read the official [asyncio guide](https://docs.python.org/3/library/asyncio.html), then learn task cancellation, timeouts, semaphores, and queues. A semaphore limits how many operations may enter a section simultaneously.

### Lab P4: bounded endpoint checker

Start with a local test HTTP server and 30 synthetic endpoints. Some reply immediately, some wait, some return errors.

Implement a sequential baseline, then a bounded concurrent version. Record total duration, request count, failures, peak outstanding work, and memory use.

Acceptance cases:

- At most five requests run simultaneously.
- One slow endpoint cannot block the report indefinitely.
- Canceling the run stops outstanding work and closes resources.
- A failing endpoint does not disappear from the report.
- The concurrent output matches the sequential output except measured timings.

Hint: control concurrency explicitly and pass deadlines through the call chain. Do not use public websites as your load-test target.

### Lab P5: advanced Python only after P4

Build a generator that streams large reports without loading all rows. A generator yields one value at a time; it is useful when the full dataset would consume too much memory.

Write a decorator that records function duration. A decorator wraps behavior around a function; preserve its metadata and avoid hiding failures.

Use `tracemalloc` to compare streaming and full-file loading. Explain reference cycles and garbage collection at a practical level before studying interpreter internals. Do not treat implementation details or one interpreter build's threading behavior as universal Python guarantees.

Pass: demonstrate measured memory savings with the same input and correct output; explain where memory is still retained.

## 6. Go foundation: rebuild a familiar problem

Use [A Tour of Go](https://go.dev/tour/) and the official [Go tutorials](https://go.dev/doc/tutorial/). Complete basics, methods/interfaces, and concurrency in that order.

Start with variables, structs, slices, maps, loops, functions, and explicit errors. A struct groups named fields; a slice is a view over an underlying array; a map stores key/value associations.

Learn `go.mod`: it identifies a module and its dependencies. Learn packages before creating many folders. Use `gofmt` so formatting stops being a decision you debate.

```bash
go mod init example.com/inventorylab
go fmt ./...
go test ./...
go vet ./...
```

### Lab G1: rewrite the normalizer

Implement P1 in Go using `encoding/csv` and `encoding/json`. Keep the same input fixtures and expected output contract so the Python version becomes a comparison tool.

Explain every error path. Include the row number when returning invalid-input errors. Do not replace every error with `panic`; ordinary invalid input is expected behavior to handle.

Pass: Python and Go agree on all fixtures; your tests cover malformed input and duplicate handling; you can explain why a missing map key differs from a present zero value.

### Lab G2: small HTTP inventory service

Build `GET /healthz`, `GET /resources`, and `GET /resources/{id}` using `net/http`. Start with an in-memory repository, then place storage behind a small interface.

An interface specifies behavior. Your handler can use a repository that supports resource lookup whether the underlying implementation is a test fixture, file, or database.

Test cases: valid ID, unknown ID, unsupported method, oversized input if you add writes, malformed JSON, and storage failure. Check status codes and response bodies.

Pass: requests have timeouts, errors have a consistent response shape, and tests run using a local test server without cloud credentials.

## 7. Go concurrency and correctness

A goroutine is a function executing under the Go runtime's concurrency scheduling. It is not one dedicated operating-system thread. A channel communicates values between goroutines. A mutex protects shared state while one goroutine accesses it.

Neither channels nor mutexes are automatically the best answer. Use a mutex for straightforward shared-state protection; use channels when communication and ownership transfer simplify the design.

`context.Context` carries cancellation/deadlines across API boundaries. Cancellation only helps when the code and libraries observe it.

### Lab G3: worker pool

Build a checker with four worker goroutines, a bounded job channel, and a result collector. A worker pool limits how many operations are active instead of creating unbounded work.

Test: zero jobs, fewer jobs than workers, 1,000 jobs, cancellation mid-run, one worker error, a full queue, and a result consumer that stops early.

Pass: every accepted job has one final outcome; cancellation terminates cleanly; no data race is found in your exercised paths; goroutine count returns near the idle baseline.

```bash
go test -race ./...
go test -count=20 ./...
```

The [Go race detector](https://go.dev/doc/articles/race_detector) observes executed code paths; a clean run does not prove all possible schedules safe. A race is unsynchronized conflicting memory access. A deadlock is a wait cycle in which nobody can progress. They are different defects.

Hint: write down who closes each channel and what happens when the receiver exits. The sender that owns production generally controls closure; arbitrary workers should not all close the same channel.

## 8. Performance: measurements before folklore

Profiling measures where a program spends resources. Benchmarking measures a defined operation under defined conditions. Tracing helps inspect the sequence and timing of execution.

Follow the official [Go diagnostics guide](https://go.dev/doc/diagnostics). Learn CPU profiles, heap profiles, allocation counts, and goroutine/blocking inspection.

### Lab G4: explain a real bottleneck

Generate a repeatable 100,000-row input. Benchmark parsing, normalization, and output independently. Add one deliberately expensive operation, identify it in a profile, then remove or improve it.

Record: machine, toolchain, input size, concurrency, sample count, elapsed time, allocations, and correctness results. Report uncertainty; a single run is weak evidence.

Pass: your claimed improvement survives repeated runs and preserves output. Explain whether the bottleneck is CPU, allocation, locking, disk, or network waiting.

A pointer refers to a value's location; it does not guarantee the value is allocated on the heap. Escape analysis is the compiler's reasoning about where a value must live. Learn it after profiling exposes allocations worth investigating.

## 9. Bridge project: mini durable task processor

Build a Go HTTP producer and a Python worker connected through a local persistent store or queue. Start with one process and one machine before adding network distribution.

Each job has an ID, payload, state, attempt count, and timestamps. Define states such as `pending`, `running`, `succeeded`, and `failed`.

Milestone A: accept a job, process it, and report its result.

Milestone B: crash a worker mid-job and recover without losing the job.

Milestone C: deliver the same job twice and demonstrate idempotent side effects.

Milestone D: implement retry limits and a dead-letter collection for jobs needing investigation.

Milestone E: add metrics for queue age, completion rate, retries, and failures.

Do not call it production-ready or exactly-once because a happy-path demo worked. Write the precise guarantees and failure cases. A duplicate message can still cause a duplicate external action unless that action has its own idempotency strategy.

## 10. Practice-session menu

Choose from these activities within the ten-hour weekly schedule in [08-weekly-plan-and-mentoring.md](08-weekly-plan-and-mentoring.md); they do not add extra required hours.

- Monday: one language concept and a 30-line exercise.
- Tuesday: implement a small project requirement with tests.
- Wednesday: break it deliberately and diagnose it.
- Thursday: read an unfamiliar standard-library API and use it.
- Friday: explain the week's code aloud and update the README.
- Weekend: one focused integration session; finish an existing milestone before starting another tutorial.

Ask your mentor: “Review this function and these tests. Tell me the largest correctness risk, ask me to explain it, then give one hint before showing a solution.”

Keep a bug journal with symptom, hypothesis, evidence, root cause, fix, and regression test. This develops engineering judgment more effectively than collecting syntax notes alone.

## 11. Advancement gate

Advance when you can demonstrate all of these without copying a tutorial:

1. Parse and validate real-looking data with visible failures.
2. Design small interfaces between core logic and external services.
3. Test pagination, timeouts, partial failure, and duplicates.
4. Bound concurrency and stop work on cancellation.
5. Profile one bottleneck and defend your optimization.
6. Explain your code to another engineer and incorporate review feedback.
7. Package setup instructions so another person can run it cleanly.

You may still consult documentation. Professional fluency means finding and applying reliable information, not memorizing every API.
