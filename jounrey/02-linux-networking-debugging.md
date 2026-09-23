# 02 — Linux, networking, and disciplined debugging

Prerequisite: workstation setup in [01](01-start-here.md). Outcome: explain where a request fails using observations from the operating system, network, and application. Run deliberate failure experiments only against your own local lab.

## 1. What runs a program?

A process is an executing program with an identity (PID), memory, open files, and execution state. A thread is an execution path within a process; threads generally share its memory. The kernel manages CPU scheduling, virtual memory, files, and network access. User-space programs request these services through system calls.

Example: Python's `open()` ultimately asks the OS to open a file. The OS checks permissions and returns a file descriptor, an integer identifying an open resource. Sockets use descriptors too. A server leaking descriptors may eventually fail to accept connections even with available CPU.

Virtual memory gives processes address spaces; it does not mean every address occupies physical RAM. RSS is resident physical memory for a process. CPU utilization measures CPU use over time. High latency can occur with low CPU if the process waits for disk, network, locks, or another service.

Practice in Ubuntu:

```bash
ps -eo pid,ppid,comm,%cpu,%mem --sort=-%cpu
free -h
df -h
ss -ltnp
```

Explain `pid`, parent PID, command name, memory availability, filesystem capacity, and listening TCP socket. Permission restrictions can hide another user's process information; absence from one command does not automatically prove a process does not exist. References: [ps](https://man7.org/linux/man-pages/man1/ps.1.html), [ss](https://man7.org/linux/man-pages/man8/ss.8.html).

Exercise: start a lab HTTP server, find its PID, find its listening port, and send it a normal termination signal. Record which command proved each observation. Prefer `kill -TERM <your-lab-PID>` so the program can clean up. An uncatchable kill is a last resort, not normal shutdown design.

## 2. Files and permissions

A path names a file in a filesystem. A directory maps names to filesystem entries. Permissions determine read/write/execute rights for owner, group, and other users.

For a regular file, execute permission allows execution. For a directory, execute permission allows traversing it. A user may be able to read a filename in a directory but lack access to open the target.

Lab: create a small script in a dedicated practice directory. Inspect `ls -l`, run it with `python script.py`, then investigate what changes when you execute it directly with a shebang. A shebang such as `#!/usr/bin/env python3` chooses the interpreter. Explain why `chmod 777` is not a diagnosis for every permission error.

Pass: identify whether failure is caused by directory traversal, file permissions, missing interpreter, wrong working directory, or application logic.

## 3. Follow a web request through layers

```text
URL entered by client
  -> DNS resolves hostname to address
  -> OS route lookup chooses network next hop
  -> TCP connection to destination port
  -> TLS handshake validates encrypted peer identity for HTTPS
  -> HTTP request selects method/path/headers/body
  -> application logic and database work
  -> HTTP response returns through the connection
```

DNS is a naming system; a DNS answer does not prove connectivity. TCP provides an ordered byte stream; it does not prove the remote application will answer quickly. TLS provides encryption and peer authentication; bypassing verification hides identity problems. HTTP defines application requests and responses.

A socket endpoint is an address and port. `127.0.0.1:8000` is your own host on port 8000. A TCP flow is commonly identified using source IP, source port, destination IP, destination port, and protocol. The client usually chooses a temporary source port; the server listens on a known destination port.

Example:

```text
client 10.10.1.15:53124 -> service 10.20.2.8:443
reply  10.20.2.8:443   -> client 10.10.1.15:53124
```

Replies do not normally arrive at the client's port 443. This matters when interpreting stateless filters and return paths.

## 4. Your first network experiment

Create a directory containing only a harmless practice `index.html`. In that directory run:

```bash
python3 -m http.server 8000 --bind 127.0.0.1
```

In another Ubuntu terminal:

```bash
curl -v --max-time 5 http://127.0.0.1:8000/
curl -v --max-time 5 http://127.0.0.1:8000/missing
ss -ltnp
```

Expected: `/` returns successfully; `/missing` returns HTTP 404. Both prove you reached the HTTP server. Stop the server and repeat the first request: connection refusal is a different failure layer.

The Python example server is for local experimentation, not a production service. Do not serve the repository root or a directory containing credentials.

Experiment record:

| Observation | What it supports | What it does not prove |
|---|---|---|
| Connection refused | Connection actively rejected; often no listener | Exact application bug |
| Connection timeout | No successful completion within deadline | Whether routing, filtering, or overload caused it |
| HTTP 404 | HTTP server responded; path not found | Health of every backend |
| HTTP 500 | Application-side error response | Which internal dependency failed |
| TLS certificate error | Peer verification failed | That disabling verification is appropriate |

Next, request `https://127.0.0.1:8000/` from the plaintext server. Predict why it fails. You requested a TLS protocol exchange from a listener expecting plaintext HTTP.

## 5. CIDRs and route selection

IPv4 addresses contain 32 bits, shown as four numbers. CIDR notation says how many leading bits identify a network. `/24` leaves 8 address bits, so it contains 256 addresses; `/32` identifies one address. These are address counts, not AWS assignable-host counts.

Synthetic examples:

| CIDR | Address range | Example match |
|---|---|---|
| `10.20.8.0/21` | `10.20.8.0` through `10.20.15.255` | `10.20.15.103` |
| `10.30.128.0/19` | `10.30.128.0` through `10.30.159.255` | `10.30.130.185` |
| `10.30.130.185/32` | Exactly one address | `10.30.130.185` |
| `0.0.0.0/0` | All IPv4 addresses | Fallback when no more specific route applies |

A route is a destination prefix and a next hop or action. The next hop is where to send the packet next. It need not be the final destination.

For ordinary destination routes, the longest matching prefix wins. If `10.30.128.0/19 -> gateway A` exists, it beats `0.0.0.0/0 -> gateway B` for `10.30.130.185`. If it is absent and no other more-specific prefix matches, the default sends it to B. Equal prefixes, propagated routes, and prefix-list routes can require additional AWS priority rules. [AWS route priority](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html).

Run this Python demonstration:

```python
from ipaddress import ip_address, ip_network

destination = ip_address("10.30.130.185")
routes = [
    (ip_network("0.0.0.0/0"), "gateway-B"),
    (ip_network("10.30.128.0/19"), "gateway-A"),
]
matches = [(network, hop) for network, hop in routes if destination in network]
selected = max(matches, key=lambda item: item[0].prefixlen) if matches else None
print(selected)
```

Change the destination, remove the `/19`, add a `/32`, and write your prediction before each run. This is an intentionally simplified route model, not a full AWS routing simulator. [Python ipaddress](https://docs.python.org/3/library/ipaddress.html).

Checkpoint: given 10 route entries and a destination, identify all matches, select the right one, and explain why another visible route is unused. Never choose a gateway merely because it has a route farther downstream.

## 6. AWS networking terms tied to a packet

A VPC is a logically isolated AWS network. A subnet is a range within it associated with an Availability Zone. An ENI is a virtual network interface. A subnet route table controls where traffic is directed. A security group is a stateful traffic filter associated with supported resources/interfaces. A network ACL is a stateless subnet filter. Stateless means you must separately account for each direction; stateful filtering tracks allowed connections.

A Transit Gateway (TGW) connects networks through attachments. Its associated route table is selected by the attachment through which traffic enters. Propagation populates routes; association determines which table is consulted for ingress traffic. Seeing a correct route in an unrelated TGW table is insufficient.

VPC peering connects VPCs; TGW peering connects TGWs. RAM is a sharing mechanism that allows another account to use a supported resource. Sharing subnets does not create a new VPC. The owner account and workload account can differ.

NAT changes an address, and often a port, for traffic passing through it. An Internet Gateway supports VPC internet connectivity subject to addressing and routes. A VPC endpoint provides supported private service connectivity. Each is a different component; the prefix `vpce-` alone does not tell you the entire application or inspection architecture.

Practice drawing two independent directions:

```text
Forward: client subnet -> its actual selected TGW -> ingress-associated TGW table
         -> next attachment -> destination VPC route -> server filter/listener

Return: server subnet -> its selected route -> ingress-associated TGW table
        -> next attachment -> client subnet -> established connection
```

For every arrow record account, Region, resource ID, destination, selected prefix, and evidence. Do this with synthetic data first, then authorized read-only evidence at work.

## 7. Debugging is hypothesis testing

Use this sequence:

1. State the symptom with timestamp, source, destination, port, and expected behavior.
2. Determine the last successful layer: DNS, TCP, TLS, HTTP, authentication, or business operation.
3. Make one testable hypothesis.
4. Choose the smallest observation that could disprove it.
5. Change one variable only when a change is needed.
6. Capture before/after evidence and decide whether the hypothesis survived.

Example: “The request times out because the worker cannot reach the database.” Test DNS and connection establishment from the worker's actual network context. A test from your laptop does not reproduce the same path. A successful TCP connection still does not verify database credentials.

When a test returns no data, distinguish “nothing happened” from “logging was disabled,” “wrong Region,” “wrong query window,” and “no permission to see it.”

## 8. Databases before distributed databases

Use SQLite locally first, then PostgreSQL for a service project. A primary key identifies a row. An index provides an additional lookup structure, usually trading write/storage overhead for faster selected reads. A transaction groups operations with database guarantees. A constraint rejects invalid states such as duplicate unique identifiers.

Lab: create `jobs(id PRIMARY KEY, status, created_at)` and insert synthetic rows. Compare lookup by ID to scanning by an unindexed status field using the database's query-plan tooling. An index is helpful when it matches the query/selectivity; adding indexes everywhere is not a performance strategy.

Failure exercise: try inserting the same job ID twice. Use the uniqueness constraint to reason about duplicate request handling. Explain what happens if a process commits a row but crashes before replying to the client. The client cannot infer failure of the operation from failure of the response.

Read and complete [PostgreSQL's tutorial](https://www.postgresql.org/docs/current/tutorial.html), especially tables, joins, and transactions. Pass: write a join, explain an index choice, and demonstrate rollback with a tiny test dataset.

## 9. Containers after processes

An image packages application files and execution configuration. A container is a running process environment created from an image with isolation and resource controls. A volume preserves data independently of the writable container layer. A port mapping publishes a container port on the host.

Lab: containerize your HTTP service, bind the published host port to loopback, and inspect logs. Restart the container and compare data stored in a volume versus its disposable layer. Use [Docker's official tutorial](https://docs.docker.com/get-started/).

Pass: explain why `localhost` inside one container refers to that container's network namespace, not an adjacent database container. Demonstrate service-name resolution on the shared container network.

## Final checkpoint

Have a reviewer choose three failures: wrong port, wrong DNS name, missing route in a fixture, malformed input, denied file permissions, stopped service, or duplicate DB insert. Diagnose them with commands and evidence. Write one page containing cause, impact, repair, validation, and prevention. A screenshot of a green status alone is not sufficient evidence.
