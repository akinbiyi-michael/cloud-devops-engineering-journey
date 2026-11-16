# Cloud Infrastructure & DevOps Engineering Journey

**Michael Moyosore Akinbiyi** â€” Lagos, Nigeria
BEng Agricultural & Bio-resources Engineering, Federal University of Technology Minna
Graduate Member, Nigerian Society of Engineers

Working through a structured cloud infrastructure and DevOps programme and documenting every phase
as I go â€” notes, labs, evidence, and the things that broke along the way.

This repository is the record of the learning. The projects below are the work.

---

## Showcase projects

| Project | What it does | Stack |
|---|---|---|
| **linux-baseline-audit** | Audits a Linux host against a security and health baseline and emits a Markdown report. Checks chosen and justified by me. | Bash, Linux |
| **fleet-health-app** _(planned)_ | Collects baseline reports from hosts and serves them over HTTP. Built on AWS, containerised, deployed by a pipeline, defined in Terraform. | Python, AWS, Docker, GitHub Actions, Terraform |
| **fleet-health-platform** _(planned)_ | The same system on Kubernetes: GitOps-deployed, monitored, with runbooks and developer self-service. | Kubernetes, Helm, Argo CD, Prometheus, Grafana |

_Links added as each repository goes public._

---

## Progress

| # | Phase | Status |
|---|---|---|
| 01 | [Computer & IT Fundamentals](01-it-fundamentals/) | in progress |
| 02 | [Linux System Administration](02-linux/) | not started |
| 03 | [Git & Version Control](03-git-github/) | not started |
| 04 | [Scripting & Automation](04-scripting/) | not started |
| 05 | [Cloud Computing & AWS](05-aws/) | not started |
| 06 | [Docker & Containerisation](06-docker/) | not started |
| 07 | [CI/CD & Software Delivery](07-cicd/) | not started |
| 08 | [Infrastructure as Code](08-infrastructure-as-code/) | not started |
| 09 | [Kubernetes](09-kubernetes/) | not started |
| 10 | [Monitoring & Observability](10-monitoring/) | not started |
| 11 | [Platform Engineering](11-platform-engineering/) | not started |

---

## How this is organised

One folder per phase, each with its own README explaining what it covers and what it produced.
Notes are named for their subject rather than a lesson number; hands-on work is prefixed `lab-`;
screenshots live in each phase's `evidence/` folder.

## Environment

Work is done on an Intel Core i7-4600U laptop â€” 2 physical cores, 4 threads, 12 GB RAM â€” running
Windows 11 Pro with Ubuntu under WSL2.

That hardware is a real constraint and it shaped the approach: multi-VM labs are not viable on two
cores, so Linux work runs under WSL2, and Kubernetes work targets a managed cloud cluster rather
than a local multi-node one. The reasoning is written up in
[01-it-fundamentals/workstation-inventory.md](01-it-fundamentals/workstation-inventory.md) â€”
measuring the machine first, then deciding what it could support.

## Contact

- **Email:** akinbiyimichael007@gmail.com
- **LinkedIn:** _add your profile URL_
