# 01 — Workstation setup and the first 14 days

Use this file first. You do not need Kubernetes, multiple AWS accounts, or a paid course to complete these two weeks. The commands below are instructions for you; writing this curriculum did not install tools or provision AWS resources.

## 1. Separate the environments in your head

| Environment | How you open it | What belongs there |
|---|---|---|
| Windows PowerShell | Windows Terminal > PowerShell | Install/check WSL and Windows applications |
| Ubuntu Bash | Windows Terminal > Ubuntu | Linux labs, Git, Python, Go, container commands |
| AWS Console | Browser through your approved login | Inspect the dedicated learning account |
| AWS CLI | Ubuntu terminal after configuration | Call AWS APIs using the chosen profile |

A shell reads commands. PowerShell and Bash have different syntax. A terminal is the application displaying that shell. A runtime executes a program, such as Python; a compiler such as Go's compiler translates source into an executable.

Choose one Linux environment for the labs. Avoid mixing a Windows Python virtual environment with Ubuntu's interpreter. Keep Linux project files in Ubuntu's home filesystem and open them using VS Code's WSL integration.

## 2. Install and verify Linux

In Windows PowerShell:

```powershell
wsl --status
wsl --list --verbose
```

If WSL is not installed, follow [Microsoft's WSL install guide](https://learn.microsoft.com/en-us/windows/wsl/install). Its basic install command is `wsl --install`, run from an administrator terminal; restart when requested. On a managed workstation, use the organization's approved installation process.

Open Ubuntu. In Ubuntu Bash:

```bash
pwd
whoami
uname -a
cat /etc/os-release
```

`pwd` identifies your current directory. `whoami` identifies your Linux user. `uname` identifies the kernel. `/etc/os-release` identifies the Linux distribution. Write all four meanings in your learning log.

Install the small set of Ubuntu packages used by the initial labs:

```bash
sudo apt update
sudo apt install git python3 python3-venv python3-pip curl jq dnsutils iproute2
```

`sudo` runs a command with elevated privileges. `apt` manages distribution packages. `jq` reads JSON. `dnsutils` supplies DNS tools; `iproute2` supplies network inspection tools. If the approved environment already has these, verify them instead of reinstalling.

## 3. Clone your learning materials and configure Git

In Ubuntu Bash:

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/issa402/reverseproxy.git
cd reverseproxy
git status
git switch -c learning/week-01
```

If you already have this clone in Ubuntu, use it instead of cloning over it. `git status` shows changes. A branch records a line of work. A commit records a snapshot and explanation. A remote is the server location. A push uploads commits; it is different from saving a file.

Configure the author name and email you want associated with this clone using `git config user.name` and `git config user.email`. Use your GitHub-provided private email if desired. Do not copy another person's identity.

Read [Pro Git: Git Basics and Branching](https://git-scm.com/book/en/v2), then practice the loop:

```bash
git status
git diff
git add path/to/your-specific-file
git diff --cached
git commit -m "docs: record week one learning evidence"
```

Replace the example path with a file you actually created. The staging area is the set of changes selected for the next commit. Review it before committing. Do not use `git add .` as a substitute for understanding what changed.

## 4. Python workspace

In Ubuntu Bash from the clone:

```bash
mkdir -p jounrey/practice/python
cd jounrey/practice/python
python3 --version
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pytest boto3 ruff
python -m pip freeze
```

A virtual environment isolates this project's packages. `python -m pip` runs the package installer belonging to that interpreter. Record the actual versions in your lab notes; create a requirements file when you have a reproducible project. See [Python virtual environments](https://docs.python.org/3/tutorial/venv.html).

Create `test_smoke.py` in your editor:

```python
def test_environment():
    assert 2 + 2 == 4
```

Run `python -m pytest -q`. You should see one passing test. This proves the test runner executes, not that you understand Python. Deliberately change the expected value to 5, observe a failure, and restore it.

## 5. Go workspace

Install a currently supported Go release using the official [Go installation instructions](https://go.dev/doc/install) for your environment. Record `go version`; avoid using an old distribution package without checking compatibility.

In Ubuntu Bash from the repository root:

```bash
mkdir -p jounrey/practice/go
cd jounrey/practice/go
go mod init example.com/journey/practice
go version
go env GOOS GOARCH
```

A module is a collection of Go packages with a dependency manifest called `go.mod`. `GOOS` and `GOARCH` identify the target operating system and processor architecture. `example.com` is a placeholder module name for local practice.

Create `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("learning environment ready")
}
```

Run `go run .`, then `go build -o journey-demo .`. Explain why the latter creates an executable. Keep generated binaries out of your commits.

## 6. Add tools only when the lesson needs them

| Tool | Install when | Verification | Purpose |
|---|---|---|---|
| VS Code with WSL integration | Now | Open the Ubuntu project folder | Edit and debug using the Linux toolchain |
| Docker Desktop/Engine | Container lab | `docker version`, `docker compose version` | Run isolated application processes and dependencies |
| AWS CLI v2 | First AWS lab | `aws --version` | Call AWS service APIs |
| Node.js supported by CDK | First CDK lab | `node --version` | Runs the CDK CLI even when constructs are Python or Go |
| AWS CDK v2 CLI | First CDK lab | `cdk --version` | Synthesize and deploy CloudFormation templates |
| Terraform or OpenTofu | State/module lab | `terraform version` or `tofu version` | Plan desired infrastructure changes |
| kubectl and kind | Local Kubernetes lab | `kubectl version --client`, `kind version` | Interact with and create a local Kubernetes cluster |

Use official links in [09](09-glossary-and-resources.md). Record versions per project. The curriculum does not require installing every tool on Day 1.

## 7. AWS account preparation, when you reach cloud labs

Use an approved sandbox or a personal learning account. Production credentials are unnecessary. If your organization has IAM Identity Center, ask for the learning-account permission set and configure it:

```bash
aws configure sso --profile journey-sandbox
aws sso login --profile journey-sandbox
aws sts get-caller-identity --profile journey-sandbox
```

Follow [AWS CLI Identity Center configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html) for the start URL, SSO Region, account, and permission set. Identity Center is not automatically present in every personal account. If absent, establish an appropriate temporary-credential role workflow for that account before cloud deployment.

STS is the Security Token Service. Its identity command tells you whose credentials are active. A profile is a named CLI configuration. It is not a separate AWS account; it selects credentials and settings.

Save these nonsecret values in a local lab configuration:

```text
LAB_ACCOUNT_ID = your learning account's 12-digit ID
LAB_REGION = the single Region chosen for the lab
PROFILE = journey-sandbox
OWNER_TAG = your chosen learning identifier
```

Write an account-ID check into deployment scripts before allowing changes. Prefer temporary credentials. Do not paste access keys into source files or disable TLS certificate verification to fix corporate certificate problems; configure the approved CA bundle instead.

## 8. Money and cleanup

Start with local-only labs. Before each AWS deployment:

1. List every resource and Region in the proposed design.
2. Estimate compute hours, storage, requests, data transfer, and logging using the AWS calculator.
3. Choose your own maximum spend for that experiment.
4. Create actual and forecast notifications in Billing > Budgets.
5. Write the teardown command and list any retained resources before deployment.
6. After teardown, verify remaining snapshots, volumes, logs, buckets, addresses, endpoints, NAT gateways, and load balancers.

Budgets provide notifications and optional actions; an alert is not a universal immediate spending cap. EKS control planes, NAT gateways, load balancers, public IPv4 addresses, logs, and idle storage can continue costing money when your application does no work. Check current regional pricing rather than assuming the free tier covers a lab. [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html).

## First 14 days: approximately 60–90 minutes per study day

| Day | Task | Evidence to commit | Pass condition |
|---|---|---|---|
| 1 | Set up shell/Git; rate your baseline | Learning log and actual tool versions | Explain terminal, shell, process, file, branch, commit |
| 2 | Python lists/dictionaries/functions | Function counting synthetic resources by type | Correct on empty input and repeated types |
| 3 | JSON/CSV parsing | Five-row inventory plus parser | Missing values and malformed rows reported separately |
| 4 | Tests | At least five behavior tests | Explain why each would detect a real defect |
| 5 | Linux files/permissions/processes | Annotated commands from module 02 | Find and stop only your lab server |
| 6 | HTTP experiment | Successful and failed `curl` outputs | Explain refused versus HTTP 404 |
| 7 | Review | Five-minute explanation and fixed tests | Rebuild counting function without looking |
| 8 | CIDR membership | Python `ipaddress` exercise | Identify narrowest matching route for five addresses |
| 9 | Parser improvements | Explicit error model and return values | Bad input never appears as successful empty inventory |
| 10 | Go basics | Struct and slice representing resources | Format code and explain error return |
| 11 | Go tests | Table-driven count tests | Same input/output behavior as Python |
| 12 | Debug a planted bug | Regression test and root-cause note | Test fails before and passes after fix |
| 13 | Git review | Small branch with meaningful commits | Explain staged versus unstaged changes |
| 14 | Checkpoint | Demo and self-assessment | Complete the exercise below without a tutorial |

The 14 days are study sessions, not a demand to skip rest days. Take an extra session wherever needed.

## Day 14 independent exercise

Given synthetic rows `{account, region, type, id, owner}`, produce a report containing counts by account/type, missing owner IDs, duplicate resource identities, and malformed rows. Identify a resource by account + Region + type + ID, not name alone.

Tests must include two accounts with the same display name, missing owner, duplicate input, empty input, and invalid JSON. Sort report output for reproducibility. Explain which operations are proportional to input size. Do not connect it to AWS yet.

If you struggle, return to functions/dictionaries/errors before adding concurrency. Bring one failing example and your attempted explanation to the next mentoring session.
