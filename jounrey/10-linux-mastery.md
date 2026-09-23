# 10 — Linux mastery: from shell to production diagnosis

Linux is the primary operating-system track for this journey. Use it while learning Python, Go, networking, AWS, containers, and distributed systems. The objective is to explain *what the machine is doing*, find evidence when it fails, and make changes you can validate and reverse. No single list of commands makes someone a master; the gates below require unfamiliar failures and independent explanations.

Start with [01 — setup](01-start-here.md) and [02 — initial Linux/networking labs](02-linux-networking-debugging.md). Use [11 — Windows companion](11-windows-companion.md) for the Windows side of the same ideas. Follow the Linux practice column in [08 — weekly plan](08-weekly-plan-and-mentoring.md) throughout the year.

## Your lab environments and their limits

| Environment | Use it for | What it may not show faithfully |
|---|---|---|
| Ubuntu in WSL 2 | Daily shell, files, code, Git, processes, basic networking, containers where supported | Full machine boot, hardware drivers, some networking and kernel features; host/VPN behavior can differ |
| Disposable Linux VM | Boot and services, disks, firewall, network namespaces, users, security, performance, recovery | Cloud-specific networking and metadata |
| Temporary sandbox EC2 Linux instance | IAM role/instance profile, VPC path, user data, SSM, CloudWatch, boot and deployment diagnostics | Full control of the hypervisor or physical hardware |

Do the daily work in WSL. Add a disposable VM when the lab needs a real boot sequence or system changes. Add a cloud instance only in an approved sandbox after [05 — AWS/IaC](05-aws-and-infrastructure-as-code.md) covers account identity, cost, and teardown. Record distribution, kernel, architecture, init system, privileges, and environment in each report. Commands and output vary by distribution and version.

Before changing a VM, snapshot it or make it reproducible from a script. Run one fault at a time. Never practice partitioning, firewall changes, `chmod -R`, `chown -R`, package removal, or forced process termination on a work machine. Use read-only inspection first. Some tracing tools require elevated privileges and can expose process arguments or data; use them only on your own disposable lab.

## The map of one Linux service

```text
firmware/virtual machine starts
  -> bootloader loads kernel
  -> kernel initializes CPU, memory, drivers, mounts initial root
  -> PID 1 starts system services (often systemd)
  -> service process opens files and a listening socket
  -> client DNS lookup -> route -> TCP -> optional TLS -> application
  -> application calls database/queue/storage
  -> kernel schedules CPU and handles disk/network I/O
  -> logs, metrics, and exit status tell you what happened
```

A **kernel** is the privileged part of the OS that manages hardware and provides system calls. A **system call** is a program's request to the kernel, such as opening a file. **PID 1** is the first user-space process and supervises startup/shutdown in a typical Linux system. A **service** is a long-running program managed by an init system. A **socket** is an OS endpoint for communication. **I/O** is input/output, usually disk or network. **Exit status** is the result code a process returns to its parent; zero conventionally means success. Point to the precise layer before proposing a fix.

## Level 1 — Shell and files: weeks 1–4, then daily

**Learn:** absolute versus relative paths; working directory; `PATH`; shell expansion; single and double quotes; variables; pipes; redirection; exit codes; `stdin`/`stdout`/`stderr`; hidden files; symbolic links; globbing; text encodings; line endings; executable scripts. Bash expands command text before a program receives arguments. A path with spaces must remain one argument, so quote it.

```bash
pwd
printf '%s\n' "$PWD" "$HOME"
command -v python3
type ls
printf 'hello\n' | wc -l
printf 'sample\n' > ./sample.txt
ls -l ./sample.txt
file ./sample.txt
stat ./sample.txt
```

`command -v` shows which executable the shell will find through `PATH`. `type` distinguishes built-ins, aliases, and executables. `|` sends one command's standard output into the next command's standard input. `>` writes output to a file and replaces its contents; practice only in your lab directory. `stat` shows metadata such as owner, size, and timestamps. A **symbolic link** is a path that points to another path; compare `ls -l` with `readlink` and predict what happens if its target disappears.

**Lab L1:** Make a small directory tree with a filename containing spaces, a symlink, a UTF-8 text file, and an executable script. Use `find` to list files; `grep` or `rg` to search their contents; a pipeline to count matching lines. Repeat while your working directory is elsewhere. Explain why `./script.sh`, `bash script.sh`, and `python3 script.sh` behave differently.

**Lab L2:** Write a Bash script that takes a directory argument, rejects missing or non-directory inputs, reports counts, writes a report to a caller-specified output path, and preserves errors. Test an empty tree, a path with spaces, and an unreadable file. Avoid assumptions that `set -e` catches every error; deliberately inspect command status and test your failure branches. Read the [GNU Bash manual](https://www.gnu.org/software/bash/manual/).

**Pass:** Recreate the file report from memory and explain each quote, redirection, and exit code. A reviewer gives you a broken script and you identify exactly which argument was expanded incorrectly.

## Level 2 — Identity, permissions, and software: weeks 3–8

**Learn:** users, groups, UID/GID, owner/group/other mode bits, directory traversal, `umask`, ACLs, `sudo`, package repositories, package versions, environment variables, and process credentials. A UID is the numeric user identity the kernel checks. `sudo` is controlled privilege escalation, not a standard prefix for every command. An ACL can grant access beyond the simple owner/group/other bits.

```bash
id
umask
ls -ld .
namei -l ./sample.txt
sudo -l
apt-cache policy curl
```

`namei -l` examines each path component; access may fail on a parent directory even when the file itself appears readable. `sudo -l` lists your permitted elevated commands and may prompt for a password. Package installation and updates belong in your own VM or approved learning environment.

**Lab L3:** Create two unprivileged users in a disposable VM, a shared directory, and one file. Predict who can list, traverse, read, edit, or execute. Test mode bits and an ACL, then restore the original permissions. Record before/after `stat`, `id`, and a test result. Do not use `chmod 777` as a generic solution.

**Lab L4:** Install a tiny package in the VM, identify the repository and version, locate the executable, read its manual, and explain what a security update would change. Record the upgrade/rollback or VM restore plan.

**Pass:** Diagnose a planted `Permission denied` error by checking identity, every directory in the path, ACLs, mount options, and the operation attempted. Explain why changing ownership of the whole tree would be excessive.

## Level 3 — Processes, services, logs, and boot: weeks 9–16

**Learn:** parent/child process; PID; foreground/background jobs; threads; signals (`TERM`, `INT`, `KILL`); zombie versus orphan; file descriptors; process environment; service dependencies; boot target; unit file; restart policy; journal; clocks/time zones. A **file descriptor** is a process-local integer representing an open file, pipe, or socket. A **unit file** tells systemd how a service should start and be supervised. `SIGTERM` requests graceful shutdown; `SIGKILL` ends execution without application cleanup.

```bash
ps -eo pid,ppid,stat,comm,%cpu,%mem --sort=-%cpu
cat /proc/self/status
ls /proc/self/fd
ss -ltnp
systemctl is-system-running
systemctl list-units --type=service --state=running
journalctl -b -p warning --no-pager
```

`/proc` is a kernel-provided view, not ordinary disk data. `systemctl`/`journalctl` require systemd and are best practiced in a VM; in WSL first check whether systemd is enabled. A filtered journal can still be incomplete due to privileges, log retention, or a service writing elsewhere. `-b` selects the current boot. [proc manual](https://man7.org/linux/man-pages/man5/proc.5.html); [systemd journal manual](https://www.freedesktop.org/software/systemd/man/255/journalctl.html).

**Lab L5:** Run your Python or Go HTTP server as a normal process. Record its PID, parent, listening socket, open descriptors, memory, and start time. Send a normal termination signal and show a clean exit. Add a graceful-shutdown handler; compare in-flight work with and without it.

**Lab L6 (VM):** Package the same service as a systemd unit with a dedicated unprivileged user. Practice `systemctl status`, `journalctl -u`, a bad executable path, an unavailable port, and a restart loop. Change one condition at a time. Find which evidence distinguishes “never started,” “started then crashed,” and “running but unhealthy.” Restore a known-good unit before finishing.

**Lab L7 (VM):** Reboot a disposable VM and produce a boot timeline: kernel, systemd, your unit, first healthy request. Explain the difference between a service being enabled at boot, active now, and actually answering requests. Check time with `date -Is` and `timedatectl`; a wrong clock can break certificate validation and mislead log correlation.

**Pass:** Diagnose an unfamiliar failing service within 20 minutes using status, journal, process, socket, and a real request. Explain the cause and one way to prevent recurrence.

## Level 4 — Networking, DNS, TLS, and packet evidence: weeks 9–24

**Learn:** interface, address, prefix, loopback, default gateway, route lookup, ARP/neighbor, resolver, DNS record, TCP handshake, ephemeral port, TLS certificate chain and hostname check, HTTP status, firewall, NAT, and return path. The local OS route lookup is one decision; AWS VPC and TGW tables make additional decisions after the packet leaves the host.

```bash
ip -br addr
ip route
ip route get 1.1.1.1
getent hosts example.com
ss -tan
curl -v --max-time 5 http://127.0.0.1:8000/
```

`ip route get` calculates a route; it does not send a packet to that address. `getent hosts` follows the host's name-service configuration, which may differ from a direct DNS query. `ss` reports socket state at a point in time. `curl -v` can show connection and protocol stages; it does not prove every application operation succeeded. Use [Ubuntu's networking guide](https://ubuntu.com/server/docs/how-to/networking/) and [WSL networking guide](https://learn.microsoft.com/en-us/windows/wsl/networking).

**Lab L8:** Put a local server on `127.0.0.1:8000`. Cause five distinct failures: stopped listener, wrong port, wrong URL path, server that responds 500, and HTTPS sent to an HTTP-only server. For each, record predicted layer, command, actual output, and the next discriminating test. Make one fix only after identifying the layer.

**Lab L9 (VM):** Create two isolated network contexts (VMs or disposable namespaces if supported). Trace a request and reply with `ip route`, `ss`, and a capture limited to your lab interfaces. Compare DNS success with TCP failure, and TCP success with HTTP failure. Read packet capture timestamps and the 5-tuple without assuming the capture proves application authentication. Do not capture work traffic.

**Lab L10:** Draw the path from your service process through its Linux socket and route, then through synthetic VPC/TGW tables, to a destination. Draw the reply separately. Apply longest-prefix matching at each table and the *ingress attachment's associated* TGW table. Finish with what an OS test can prove and what requires firewall/server evidence. Use [Project 2](07-projects-and-capstone.md).

**Pass:** Given a timeout or TLS error, produce at least two plausible causes, one test that separates them, and a result-limited conclusion. Do not label a domain allowlist, certificate, or route as the cause without the relevant evidence.

## Level 5 — Storage, memory, CPU, and performance: weeks 17–36

**Learn:** block device, partition, filesystem, mount, inode, page cache, RSS, swap, CPU scheduling, run queue, I/O wait, latency percentiles, throughput, and contention. A block device exposes addressable blocks; a filesystem gives files/directories meaning. A mount makes a filesystem visible at a path. An inode stores file metadata and references data blocks. Free bytes and free inodes are separate limits.

```bash
lsblk -f
findmnt
df -h
df -i
du -sh .
free -h
uptime
vmstat 1 5
```

`df` reports filesystem capacity, while `du` estimates space consumed by files reachable in a directory. They can differ due to deleted open files, reserved space, or mounts. `free` reports memory categories; large cache can be reclaimed, so “low free” alone is not an out-of-memory diagnosis. `uptime` load average counts runnable and certain blocked tasks; it is not CPU percentage. Record the number of CPUs before interpreting load.

**Lab L11 (VM):** Attach a small disposable data disk or use a small test filesystem image. Format and mount only the disk you created, then verify the exact device and mount before any write. Compare `df`, `du`, `lsblk`, and `findmnt`. Fill a dedicated small lab filesystem with test data, observe the error, remove only your test files, and restore from a known backup. Never use the host OS disk for this fault.

**Lab L12:** Run the Go service and Python worker under a measured request load. Record CPU, RSS, thread count, file descriptors, request latency, and queue depth. Change one variable (for example, worker count) and repeat. Explain whether the bottleneck is CPU, memory, disk, lock, network, or downstream service. Use Go `pprof` after you have a baseline; advanced `perf` may need VM/kernel permissions. [Kernel perf security](https://docs.kernel.org/admin-guide/perf-security.html).

**Pass:** A reviewer plants a full-filesystem, high-CPU, or slow-downstream fault. You identify the limit with measured evidence, state a recovery action and rollback, and show a regression test or alert.

## Level 6 — Security, containers, and cloud operations: weeks 21–48

**Learn:** least privilege, SSH key authentication, host firewall, OS patching, secrets exposure via arguments/environment/logs, systemd service identity, namespaces, cgroups, image layers, container runtime, EC2 instance identity, user data, and SSM. A **namespace** gives a process a particular view of resources such as network or PIDs. A **cgroup** organizes processes and controls or measures resource use. A container combines isolation and packaging, but the container shares the host kernel. Read the [kernel cgroup v2 guide](https://docs.kernel.org/admin-guide/cgroup-v2.html) after the basic container lab.

```bash
id
ls -l /proc/self/ns
cat /proc/self/cgroup
systemctl status ssh --no-pager
```

The `ssh` unit name may differ and the service may not be installed; inspect available units before using it. Avoid placing passwords, tokens, or private keys in commands, screenshots, or public reports. An IAM role attached to an EC2 instance and a Linux user account answer different authorization questions.

**Lab L13:** Run a containerized service and compare host and container views of PID, hostname, IP, filesystem, and cgroup. Set a small memory limit on your own disposable container; observe an out-of-memory outcome and logs. Explain why `localhost` in a container may not reach a service in another container.

**Lab L14 (cloud sandbox):** Deploy a disposable Linux EC2 workload through IaC. Record AMI, architecture, subnet, security group, instance profile, bootstrap/user-data result, SSM availability, OS service status, CloudWatch/log path, and cost. Inject a bad environment variable, use host evidence plus AWS evidence to diagnose it, correct the IaC, redeploy, and tear down after verification. Do not assume “instance running” means “application healthy.”

**Lab L15:** Write an operating runbook for the capstone: how to identify the right instance/container, read only the relevant logs, check health and dependency reachability, observe saturation, restart safely, restore data, and confirm accepted jobs were processed once at the side-effect boundary. Include a failure you have actually reproduced.

**Pass:** You can operate the capstone from source code to Linux process to AWS resource and explain each boundary. You can reproduce, diagnose, fix, and prevent three unfamiliar faults without altering unrelated infrastructure.

## Advanced depth after the core gates

These are focused electives for a role that needs deeper Linux internals. Take them after the related core lab; do not postpone your capstone until you have studied every kernel subsystem.

| Topic | Mechanism to explain | Evidence-producing task |
|---|---|---|
| System calls and errors | User code crosses into the kernel; ENOENT is a missing path, EACCES is denied access | In your disposable VM, run a tiny file-opening program under strace with file-call filtering; compare a missing file with a denied file. Trace only your own process and remove the trace artifact after review. [strace](https://strace.io/) |
| Memory and OOM | Virtual address, resident pages, page cache, cgroup limit, OOM kill are distinct | Run a bounded test container with a small memory limit; collect exit status, memory metric, and kernel/container evidence. Explain why a CPU graph alone would miss it. |
| Filesystems | Block device, filesystem, inode, mount, journal, and open-deleted file affect recovery | On a disposable data disk, compare lsblk, findmnt, df, du, and a restore. Write down the exact device before formatting. [Ubuntu storage](https://ubuntu.com/server/docs/how-to/storage/) |
| Isolation | PID, mount, user, and network namespaces change a process's view; cgroups constrain resources | Compare namespace and cgroup views inside and outside a local container. Explain what remains shared with the host kernel. |
| Network state | A listening socket, SYN exchange, established connection, TLS handshake, and HTTP reply are separate observations | Capture only your loopback lab request; label the TCP states and show where the deliberate wrong-protocol request fails. |
| Scheduling and profiling | CPU saturation, runnable queue, blocked I/O, lock contention, and downstream wait have different signatures | Hold input load constant while changing one resource limit; record vmstat, per-process CPU/RSS, latency, and a language-level profile. |
| Boot and service dependencies | Kernel starts PID 1; unit ordering and readiness determine when a service can answer | Reboot a VM with a broken unit dependency, diagnose from the journal, repair the unit, and validate the first healthy request. |
| Host security | OS identity, SSH, file permissions, firewall, service privileges, and AWS IAM operate at different layers | Give a service only the file and port access it needs; show a denied extra operation and explain which layer denied it. |
| Kernel observability | perf and eBPF can expose deeper kernel behavior but require suitable permissions and a focused question | After an ordinary profile leaves a specific unanswered question, formulate what to measure, which tool could measure it, and its overhead/permission limits. Read [kernel perf security](https://docs.kernel.org/admin-guide/perf-security.html) and [BPF docs](https://docs.kernel.org/bpf/). |

### Three Linux portfolio challenges

1. **Host evidence collector:** In Python or Go, gather OS version, uptime, mounts, CPU/memory, listening ports, and chosen service status into a versioned JSON report. Redact secrets and mark unavailable data explicitly. Test malformed command output and limited permissions. Compare the result with manual commands.
2. **Broken-service tournament:** Create five reproducible faults in a disposable VM: missing executable, wrong working directory, denied file, occupied port, and failing downstream dependency. Give a reviewer the machine with one unknown fault. Require a timestamped diagnosis, smallest repair, validation request, and rollback description.
3. **Cloud-host runbook:** Deploy the capstone to a sandbox host, then document source commit, image/AMI, boot, unit/container, identity, socket, network path, logs/metrics, saturation, restore, and teardown. Run a timed recovery drill and publish only synthetic or sanitized evidence.

For each challenge, maintain a clean setup script or IaC, an expected-output fixture, and a design review. A command transcript alone is not a passing result; a second person must reproduce and explain your conclusion.
## The recurring Linux exercise in every project

For each weekly coding/cloud task, answer these five questions in your [learning log](templates/learning-log.md):

1. **Identity:** Which Linux user, process/PID, container, and AWS identity (if any) performed the action?
2. **Location:** Which directory, mount, network namespace, and machine/VM was it in?
3. **Dependency:** Which file, socket, DNS answer, database, queue, or external service did it depend on?
4. **Failure:** Which log, exit status, metric, or packet observation would distinguish the likely causes?
5. **Recovery:** What state must survive restart, and how will you prove it did?

By week 48, the portfolio should contain at least four Linux incident reports, one reproducible VM or container setup, one measured performance experiment, one restore drill, and one cloud host runbook. Each report should include the wrong first hypothesis and what changed your mind. That evidence is more valuable than a long command list.

## Reference ladder

Read the relevant section immediately before its lab; do not try to read every manual cover to cover.

| Need | Primary reference |
|---|---|
| Shell syntax and quoting | [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/) |
| System calls, processes, files, sockets | [Linux man-pages](https://man7.org/linux/man-pages/) and [proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) |
| Ubuntu server administration | [Ubuntu Server documentation](https://ubuntu.com/server/docs/) |
| Services and logs | [systemd manuals](https://www.freedesktop.org/software/systemd/man/) |
| Kernel resource control | [Linux kernel cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) |
| WSL differences | [Microsoft WSL networking](https://learn.microsoft.com/en-us/windows/wsl/networking) and [troubleshooting](https://learn.microsoft.com/en-us/windows/wsl/troubleshooting) |

Your mastery test is simple to state and hard to fake: on a machine you did not configure, identify its environment, reproduce a problem, collect only the evidence needed, explain the mechanism, make a minimal repair, verify it, and document how to recognize recurrence.
