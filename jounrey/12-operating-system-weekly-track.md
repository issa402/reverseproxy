# 12 — Operating-system practice every study week

Use this alongside [08 — the 48-week plan](08-weekly-plan-and-mentoring.md). The **Linux action** belongs in the existing Thursday debugging session (90 minutes), with any unfinished evidence carried into Saturday's integration time. It is not an extra weekly time commitment. On even-numbered weeks, spend roughly 20–30 minutes of that same session on the **Windows comparison** when you have a Windows lab. If you do not, write the PowerShell equivalent and its expected evidence without pretending it was run. Keep all faults in your own disposable environments.

Each line produces one small artifact in the [learning log](templates/learning-log.md): prediction, exact environment, command/test, output, explanation, and one remaining uncertainty. On weeks 4, 12, 20, 28, 36, 44, and 48, use an unfamiliar fault as a gate. Repeat the week if you cannot explain the result.

| Week | Linux action inside the main project | Windows comparison on even weeks |
|---|---|---|
| 1 | Locate shell, user, kernel, distro, working directory, and chosen interpreter. | — |
| 2 | Run tests from a second directory; fix a broken relative path by identifying the working directory. | Find current location, executable, and OS build in PowerShell. |
| 3 | Inspect file owner, group, mode, symlink target, and each parent directory. | — |
| 4 | Diagnose a planted permission error; explain `stdin`, `stdout`, `stderr`, and exit code. | Compare `Get-Acl` with Linux mode bits; explain why they differ. |
| 5 | Find the inventory program's PID, parent, memory, and open files during a run. | — |
| 6 | Break a shell pipeline with malformed input; make errors visible in the report. | Compare PowerShell object pipelines with Bash text pipelines. |
| 7 | Run the collector under a least-privileged test user in a disposable VM. | — |
| 8 | Record package version and provenance; rebuild on a fresh VM/WSL environment. | Locate a Windows executable and distinguish installed package from `PATH`. |
| 9 | Use `ip route get` for five synthetic destinations and explain selected prefixes. | — |
| 10 | Compare `getent hosts`, DNS query, socket state, and HTTP response. | Use `Resolve-DnsName` and `Get-NetTCPConnection` on a local service. |
| 11 | Map source socket, ephemeral port, route, firewall layer, and reply path. | — |
| 12 | Diagnose an unfamiliar wrong-port/DNS/TLS failure before inspecting fixture answer keys. | Diagnose the same local symptom with Windows tools. |
| 13 | Run Go service in foreground; record signals and graceful shutdown behavior. | — |
| 14 | Read `/proc/<PID>/status` and descriptors for your Go service; tie numbers to workload. | Compare `Get-Process` and a process/socket viewer. |
| 15 | Increase concurrency; observe threads, CPU, open sockets, and cancellation. | — |
| 16 | Measure baseline before changing worker count; label CPU time versus wall time. | Inspect CPU and memory for a local Windows process. |
| 17 | Locate database files, mounts, and space usage; distinguish `df` from `du`. | — |
| 18 | Crash after DB commit but before client reply; explain what survived. | Compare Windows service/application event logs after a local failure. |
| 19 | Inspect host versus container PID, IP, mount, and localhost. | — |
| 20 | Reboot disposable VM; prove service startup, readiness, and restored data. | Compare Windows service `Running` with application readiness. |
| 21 | Determine Linux user versus AWS caller identity for one lab action. | — |
| 22 | On a sandbox host, compare OS route lookup with VPC/TGW path evidence. | Test only an approved Windows test host; record actual source and port. |
| 23 | Identify a test filesystem recovery point and restore into a new location. | — |
| 24 | Explain how CDK-deployed user data becomes a Linux process/service. | Inspect Windows EC2 bootstrap and service model in documentation or lab. |
| 25 | Inspect a least-privileged unit user, permissions, and needed file paths. | — |
| 26 | Run a small Go binary and record architecture, executable type, and environment. | Run its Windows build if supported; compare executable and paths. |
| 27 | Detect drift between IaC expectation and host state without changing production. | — |
| 28 | Restore disposable host from code/backup; prove behavior matches acceptance tests. | Draft an equivalent Windows rollback and log evidence. |
| 29 | Follow one job through publisher process, socket, queue, and worker process. | — |
| 30 | Send a termination request during work; verify no accepted job is silently lost. | Compare Windows process/service shutdown behavior in a local lab. |
| 31 | Filter service journal logs by unit/time; correlate request ID and UTC time. | — |
| 32 | Observe load, RSS, backlog, latency, and one measured bottleneck. | Measure process CPU/memory and correlate event timing. |
| 33 | Record Linux AMI, boot status, user-data output, service, and listener. | — |
| 34 | Review deploy identity versus instance identity versus Linux service user. | Compare Windows service account/token with AWS IAM role concept. |
| 35 | Diagnose three timed local host/service faults with before/after evidence. | — |
| 36 | Reproduce one capstone incident, restore service/data, and check accepted work. | Explain what Windows event or service evidence would differ. |
| 37 | In Kubernetes, map pod/container process to node kernel and network namespace. | — |
| 38 | Observe restart, readiness, memory limit, and logs after a fault. | Compare a Windows service restart policy conceptually. |
| 39 | Trace controller reconcile process and the underlying API/socket request. | — |
| 40 | Explain what container limits use from cgroup v2 and what the node still shares. | Identify which parts are Linux-specific. |
| 41 | In a design, include host failure, clock, storage, and network assumptions. | — |
| 42 | Build a capacity estimate using measured CPU, memory, file descriptors, and I/O. | Compare a Windows process counter to one Linux measurement. |
| 43 | Explain system calls and memory allocation in one code path under interview pressure. | — |
| 44 | Diagnose an unfamiliar slow or unreachable service in 20 minutes. | Diagnose an analogous Windows service with events and socket evidence. |
| 45 | Draft one truthful story about an OS diagnosis, including the first wrong hypothesis. | — |
| 46 | Give a reviewer a clean Linux VM/container setup and one planted fault. | Provide the Windows investigation card for the same fault. |
| 47 | Map three target roles to their required Linux, Windows, and debugging depth. | — |
| 48 | Perform an unseen incident: identify, isolate, repair, verify, and document. | Explain how you would adapt the procedure to a Windows host. |

## Review gates

- **Week 4:** You can navigate, quote, inspect permissions, and explain a command's exit status.
- **Week 12:** You can isolate a network failure by layer and draw the actual return path.
- **Week 20:** You can operate and restore a service after a reboot.
- **Week 28:** You can connect host observations to IaC and cloud network evidence.
- **Week 36:** You can measure, alert on, and recover an application failure without losing accepted work.
- **Week 44:** You can troubleshoot a new Linux incident and explain the Windows equivalent.
- **Week 48:** You can lead an unfamiliar end-to-end diagnostic with evidence, minimal repair, validation, and a clear handoff.

Read the relevant sections of [10 — Linux mastery](10-linux-mastery.md) just before each exercise. Use [11 — Windows companion](11-windows-companion.md) for paired labs. Keep the output of work systems out of a public portfolio; recreate the lesson with synthetic data.
