# 11 — Windows companion: investigate the same systems from PowerShell

Linux is the daily practice environment for the cloud/platform track. Windows remains relevant when the workload is Windows: EC2 Windows servers, Active Directory, SMB shares, FSx for Windows File Server, Windows services, and developer workstations. Learn the same operating-system questions on both platforms while respecting their different internals. This companion is part of the [48-week plan](08-weekly-plan-and-mentoring.md), using a portion of the existing Thursday debugging session roughly every second week.

## Three environments, three perspectives

```text
Windows host: Windows kernel + PowerShell + Windows services
     |
     +-- WSL 2: a Linux kernel in a lightweight VM + Bash + Linux processes
     |
     +-- optional Linux VM: its own boot, init, filesystems, and network
```

`C:\Users\...` is a Windows path; `/home/...` is a Linux path. A PowerShell command runs on Windows unless you explicitly enter WSL or a remote Linux machine. WSL and Windows can communicate, but they do not always have identical IP addresses, DNS settings, firewall behavior, or process lists. WSL's default networking and its optional mirrored networking behave differently; determine the actual mode before inferring where packets go. Keep Linux projects on the Linux filesystem for Linux tooling. [Microsoft WSL interoperability](https://learn.microsoft.com/en-us/windows/dev-environment/wsl-interop), [WSL networking](https://learn.microsoft.com/en-us/windows/wsl/networking).

**PowerShell** passes structured objects between cmdlets. Bash pipelines usually pass text bytes. `Get-Process | Where-Object CPU -gt 1` filters objects with a `CPU` property; `ps | grep` searches displayed text. A command name that sounds similar may answer a different question. Use `Get-Help <cmdlet> -Online`, `Get-Command`, and `Get-Member` to inspect actual behavior. Do not paste Bash syntax into PowerShell and assume it translates.

## Cross-platform investigation card

| Question | Linux | Windows PowerShell | What to explain |
|---|---|---|---|
| Who am I? | `id` | `whoami /all` | User, groups, and effective privileges |
| What OS/kernel? | `uname -a`; `cat /etc/os-release` | `Get-ComputerInfo`; `[Environment]::OSVersion` | Product/version and kernel/build are different facts |
| Where am I? | `pwd`; `ls -la` | `Get-Location`; `Get-ChildItem -Force` | Current directory and hidden items |
| What is running? | `ps -ef`; `top` | `Get-Process` | PID, parent/context, CPU, memory |
| Which service? | `systemctl status <unit>` | `Get-Service -Name <name>` | Installed, running, and healthy are different states |
| Which port listens? | `ss -ltnp` | `Get-NetTCPConnection -State Listen` | Port and owning process; permissions can limit visibility |
| Which route? | `ip route`; `ip route get <IP>` | `Get-NetRoute`; `Find-NetRoute -RemoteIPAddress <IP>` | Selected next hop and interface, where supported |
| Which DNS answer? | `getent hosts <name>`; `dig <name>` | `Resolve-DnsName <name>` | Resolver path and response; cached/stale answers exist |
| Can TCP connect? | `nc -vz <host> <port>` or `curl -v` | `Test-NetConnection <host> -Port <port>` | TCP reachability only, not login or file permissions |
| Which logs? | `journalctl -u <unit>` | `Get-WinEvent -LogName Application -MaxEvents 20` | Filter by time, service/provider, and correct host |
| Disk capacity? | `df -h`; `df -i` | `Get-Volume` | Capacity, filesystem health, and identity |
| Local firewall? | `sudo nft list ruleset` (where used) | `Get-NetFirewallRule` | Rules are one layer; do not change them without a known path |

`Find-NetRoute`, `Get-NetTCPConnection`, and other modules may depend on Windows version and permissions; check `Get-Command` first. On Linux, `nc` may require installing the relevant netcat package in the lab. The table is an investigation map, not a promise of identical output or semantics. [Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection?view=windowsserver2025-ps), [Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-5.1).

## Paired lab W1 — Same local HTTP service, two operating systems

In Ubuntu Bash, use the existing [local server lab](02-linux-networking-debugging.md) on `127.0.0.1:8000`. Record `ps`, `ss`, `curl`, and a stopped-listener failure. Separately, in a Windows PowerShell terminal, start a test HTTP server with a Python installation on Windows **if one exists**:

```powershell
py -m http.server 8001 --bind 127.0.0.1
```

In a second PowerShell terminal:

```powershell
Get-NetTCPConnection -LocalPort 8001
Invoke-WebRequest http://127.0.0.1:8001/ -Method Head
Test-NetConnection 127.0.0.1 -Port 8001
```

Use a directory with harmless sample files; the example server exposes that directory. If Python is not installed on Windows, defer this half until the Windows toolchain lab and use a known local Windows service only with approval. Stop each server normally. Compare which tool proves a listener exists, which proves TCP works, and which proves HTTP replied. Test Windows and WSL separately before attempting cross-environment localhost; WSL networking mode can change that result.

**Pass:** Describe why a successful `Test-NetConnection` to a port does not prove an SMB mount, TLS identity, or authorization. Explain which kernel owns each process and socket.

## Paired lab W2 — Process, service, and event evidence

In a disposable Windows VM, identify one noncritical service with `Get-Service` and the relevant process with `Get-Process`. Read recent Application and System events using `Get-WinEvent` with a narrow time window. Compare a service that is `Running` with an actual functional request. Use [Process Explorer](https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer) or [TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview) to connect a PID to a socket if native cmdlets leave ambiguity.

On Linux, repeat for your systemd lab service with `systemctl`, `journalctl`, `ps`, and `ss`. Write the same five-line incident summary for both: symptom, last working layer, evidence, cause, validation. Do not stop a work service to manufacture a failure.

**Pass:** Distinguish “service started” from “application ready” and explain where you would look for a startup failure on each OS.

## Paired lab W3 — Paths, permissions, and package ownership

Create sample files in dedicated disposable directories. On Linux inspect `id`, `stat`, `namei -l`, and optionally ACLs. On Windows inspect `Get-Acl` and `icacls` for the sample directory. Windows ACLs and Linux mode bits/ACLs have different inheritance and identity models; do not translate one permission string mechanically into the other.

Identify which package or installer provided a tool. On Linux use the distribution package database (`dpkg -S` on Ubuntu for an installed file). On Windows inspect installed application metadata or your approved package manager; do not assume an executable on `PATH` was installed securely. Compare line endings and path separators using a tiny text file created on each side.

**Pass:** Given “access denied,” identify the user token/UID, object permissions, parent path, and attempted operation before proposing a change.

## Paired lab W4 — DNS, routes, and SMB reasoning

Use only a synthetic destination in your report. From a Windows test host run `Resolve-DnsName`, `Find-NetRoute -RemoteIPAddress`, and `Test-NetConnection <approved-test-host> -Port 445` **only when you have an approved reachable test endpoint**. On Linux use `getent`, `ip route get`, and a TCP test from the Linux context. Record source IP, destination IP/name, port, time, network location, and result. Different source hosts can take different AWS routes.

For an SMB file share, a TCP 445 success says the transport can connect. Accessing a share additionally depends on the SMB service, share name, user/domain authentication, share permissions, file permissions, and application credentials. A Windows server can be the SMB client while FSx for Windows is the server; the direction of the initial request matters. Never turn a host's route check into proof of every intermediate TGW/firewall decision.

**Pass:** Give a complete test ladder: name resolution, route selection, TCP 445, authentication, list/read/write as authorized, and actual application workflow. Mark each outcome separately.

## Paired lab W5 — Windows/Linux automation and deployment

Build one tiny report in Bash and a separate PowerShell version: current OS/version, identity, service status, listener, disk space, timestamp, and a JSON output file. Both versions must report partial failure explicitly and avoid collecting secrets. PowerShell has object pipelines and `ConvertTo-Json`; Bash usually combines commands and text/JSON tools. Test missing service, denied information, and an output path containing spaces.

Then run the same Python or Go application on Linux and Windows where supported. Compare environment variables, executable paths, signal/shutdown behavior, file permissions, logging location, and service managers. Keep application logic portable while allowing OS-specific adapters.

**Pass:** Another learner can run both reports from a README and can tell an unavailable measurement from a healthy zero value.

## Windows depth after the five paired labs

If your target job operates Windows fleets, spend additional repeat weeks on Windows service accounts, Active Directory and Kerberos basics, NTFS ACLs and share permissions, SMB, Windows Event Log, update/restart planning, remote administration, and Windows EC2 boot/SSM diagnostics. Use an isolated Windows lab and current Microsoft documentation. If your target role is Linux-heavy SRE or platform engineering, keep the paired labs and prioritize [Linux mastery](10-linux-mastery.md).

Every cross-platform incident report should answer: Which OS/kernel? Which user or role? Which process/service? Which port or file? Which logs? Which network source? Which exact result? Which interpretation remains uncertain? That habit transfers better than memorizing two command lists.

## Official references

- [Microsoft WSL filesystems and interoperability](https://learn.microsoft.com/en-us/windows/wsl/filesystems)
- [Microsoft WSL networking](https://learn.microsoft.com/en-us/windows/wsl/networking)
- [PowerShell documentation](https://learn.microsoft.com/en-us/powershell/)
- [Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection?view=windowsserver2025-ps)
- [Get-WinEvent](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-5.1)
- [Sysinternals Process Explorer](https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer)
- [Sysinternals TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview)
