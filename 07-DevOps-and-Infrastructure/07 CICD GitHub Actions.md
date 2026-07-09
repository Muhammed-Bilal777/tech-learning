# CI/CD with GitHub Actions: Beginner-to-Expert Engineering Guide

> **Scope:** This guide teaches CI/CD from first principles through production-grade GitHub Actions engineering — every workflow YAML construct (triggers, jobs, steps, matrices, reusable/composite workflows), the complete expression/context system, secrets/OIDC security, caching, artifacts, environments, self-hosted runners, and advanced patterns (concurrency, conditional execution, dynamic matrices, monorepo strategies). It is a near-exhaustive reference for writing GitHub Actions `.yml` files.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [1. Fundamentals](#1-fundamentals-beginner-level)
3. [2. Core Concepts](#2-core-concepts-intermediate-level)
4. [3. Advanced Concepts](#3-advanced-concepts-senior-level)
5. [4. Real-World System Design Usage](#4-real-world-system-design-usage)
6. [5. Interview Preparation](#5-interview-preparation)
7. [6. Hands-On Thinking](#6-hands-on-thinking)
8. [7. Deep Dive](#7-deep-dive-optional-but-important)
9. [Production Checklists](#production-checklists)
10. [Learning Roadmap](#learning-roadmap)
11. [Self-Review Completion Loop](#self-review-completion-loop)
12. [Official References](#official-references)

---

## Executive Summary

**CI/CD** (Continuous Integration / Continuous Delivery-Deployment) is the practice of automatically building, testing, and shipping code on every change. **GitHub Actions** is GitHub's built-in CI/CD platform: you define **workflows** as YAML files in `.github/workflows/`, and GitHub runs them on **runners** (virtual machines) in response to **events** (a push, a pull request, a schedule, etc.).

The core idea:

```mermaid
flowchart LR
    Event["Event (push, PR, schedule, manual...)"] --> Workflow["Workflow (.github/workflows/*.yml)"]
    Workflow --> Job1["Job A (runs on a fresh runner VM)"]
    Workflow --> Job2["Job B (parallel by default)"]
    Job1 --> Step1["Step: checkout"]
    Job1 --> Step2["Step: run tests"]
    Job1 --> Step3["Step: build & deploy"]
    Job2 --> Result["Pass/Fail reported back to the PR/commit"]
```

> [!TIP]
> Learn GitHub Actions as a strict hierarchy: **Workflow → Jobs → Steps**. A workflow reacts to events. Jobs run in parallel on separate fresh VMs (sharing nothing unless you explicitly pass artifacts). Steps run sequentially within a job, sharing the same filesystem and environment. Almost every "why doesn't this work" question resolves to understanding *which* of these levels state lives at.

---

# 1. Fundamentals (Beginner Level)

## 1.1 What Is CI/CD?

| Term | Meaning |
|---|---|
| Continuous Integration (CI) | Automatically build and test every code change so bugs are caught early |
| Continuous Delivery (CD) | Automatically prepare every validated change for release (deployable, but a human approves the actual deploy) |
| Continuous Deployment (CD) | Automatically deploy every validated change to production with no manual gate |

```mermaid
flowchart LR
    Commit["Developer pushes code"] --> CI["CI: build + test automatically"]
    CI -->|"pass"| Delivery["CD (Delivery): package, stage, await approval"]
    Delivery -->|"approved"| Deploy["Deploy to production"]
    CI -->|"fail"| Block["Block the merge, notify developer"]
```

## 1.2 Why GitHub Actions Exists

| Problem | GitHub Actions' Answer |
|---|---|
| Manual build/test/deploy is slow and error-prone | Automated workflows triggered by repo events |
| Separate CI tools (Jenkins, CircleCI) require external setup/integration | Built directly into GitHub — no external service to wire up |
| Reusing automation logic across projects is hard | Reusable actions from the Marketplace + reusable/composite workflows |
| Needing different OSes/versions to test against | Matrix builds run the same job across many configurations |
| Secrets management for deploys | Encrypted secrets + OIDC for keyless cloud authentication |

## 1.3 The Anatomy of a Workflow File

```yaml
name: CI                          # workflow's display name

on: [push]                        # WHAT triggers this workflow

jobs:                             # one or more jobs
  build:                          # job ID
    runs-on: ubuntu-latest        # which runner (VM) to use
    steps:                        # sequential steps within the job
      - uses: actions/checkout@v4 # a reusable action
      - name: Run a command
        run: echo "Hello, Actions!"   # a shell command
```

```mermaid
flowchart TB
    File[".github/workflows/ci.yml"] --> Name["name: (optional display label)"]
    File --> On["on: (triggering events)"]
    File --> Jobs["jobs:"]
    Jobs --> Job["build: (a job)"]
    Job --> RunsOn["runs-on: (runner)"]
    Job --> Steps["steps: (ordered list)"]
    Steps --> Uses["uses: (call an action)"]
    Steps --> Run["run: (shell command)"]
```

> [!IMPORTANT]
> Workflow files **must** live in `.github/workflows/` at the root of the repository, and use `.yml` or `.yaml`. GitHub reads them from the default branch for most triggers. A single repo can have many workflow files, each reacting to different events.

## 1.4 `uses` vs. `run`: The Two Kinds of Steps

```yaml
steps:
  # 1. uses — invoke a prebuilt, reusable Action
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:                          # inputs to the action
      node-version: 20

  # 2. run — execute a shell command directly on the runner
  - name: Install dependencies
    run: npm ci
  - name: Run tests
    run: npm test
```

| Step Type | Purpose |
|---|---|
| `uses:` | Calls a reusable **action** (from the Marketplace, a repo, or a Docker image) with `with:` inputs |
| `run:` | Runs one or more shell commands directly (bash on Linux/macOS, PowerShell on Windows by default) |

## 1.5 A Complete Beginner Workflow

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm test
```

## 1.6 Real-World Analogy

Think of a workflow like a factory assembly line that automatically kicks off whenever raw materials arrive.

The delivery truck arriving (an **event** — a push) triggers the line. The line has stations (**jobs**), some running in parallel on separate benches (separate VMs), each with an ordered sequence of tasks (**steps**). Each bench starts completely empty and clean every single time (a fresh runner) — if station A builds something station B needs, you must explicitly ship it between benches (**artifacts**), because they don't share a workspace.

```text
Delivery truck arriving = an event (push/PR/schedule)
The assembly line        = the workflow
Work benches (parallel)  = jobs, each on a fresh isolated VM
Ordered tasks at a bench = steps (sequential, shared filesystem)
Shipping between benches = artifacts / job outputs (nothing is shared implicitly)
```

## 1.7 Core Vocabulary

| Term | Meaning |
|---|---|
| Workflow | An automated process defined in a YAML file, triggered by events |
| Event | Something that triggers a workflow (push, pull_request, schedule, etc.) |
| Job | A set of steps that run on the same runner; jobs run in parallel by default |
| Step | An individual task: either a `run` command or a `uses` action |
| Action | A reusable unit of code, invoked via `uses` |
| Runner | The server/VM that executes a job (GitHub-hosted or self-hosted) |
| Artifact | Files uploaded from a job to share between jobs or download later |
| Secret | An encrypted variable (API keys, tokens) available to workflows |

## 1.8 Runners

```yaml
jobs:
  linux-job:
    runs-on: ubuntu-latest        # GitHub-hosted Linux VM

  windows-job:
    runs-on: windows-latest       # GitHub-hosted Windows VM

  mac-job:
    runs-on: macos-latest         # GitHub-hosted macOS VM

  custom-job:
    runs-on: [self-hosted, linux, gpu]   # your own machine, matched by labels
```

## 1.9 Viewing Results

Every workflow run appears in the repository's **Actions** tab, showing each job's status (queued, in-progress, success, failure), logs per step, and — for PRs — status checks that can block merging.

---

# 2. Core Concepts (Intermediate Level)

## 2.1 Every Way to Trigger a Workflow (`on:`)

```yaml
on:
  # Single event
  push:

  # Multiple events (array form)
  # on: [push, pull_request]

  # Filtered push
  push:
    branches: [main, 'release/**']    # glob patterns supported
    branches-ignore: [experimental]    # OR use ignore (not both)
    tags: ['v*']
    paths: ['src/**', '!**/*.md']       # only run if these paths changed
    paths-ignore: ['docs/**']

  pull_request:
    types: [opened, synchronize, reopened, labeled]   # specific PR activity types

  # Scheduled (cron)
  schedule:
    - cron: '0 2 * * *'                  # daily at 02:00 UTC

  # Manual trigger with inputs
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        type: choice
        options: [staging, production]

  # Triggered by another workflow completing
  workflow_run:
    workflows: ["CI"]
    types: [completed]

  # Called by another workflow (reusable)
  workflow_call:

  # Repository events
  release:
    types: [published]
  issues:
    types: [opened]
  # ...and many more: fork, watch, deployment, etc.
```

| Trigger | Fires When |
|---|---|
| `push` | Commits pushed to the repo |
| `pull_request` | PR opened/updated/etc. (runs against a merge commit) |
| `pull_request_target` | Like `pull_request` but runs in the **base** repo's context (secrets available — security-sensitive!) |
| `schedule` | On a cron schedule (UTC) |
| `workflow_dispatch` | Manually from the UI/API, optionally with inputs |
| `workflow_call` | Invoked by another workflow (makes it reusable) |
| `workflow_run` | After another workflow completes |
| `repository_dispatch` | External webhook/API-triggered |

> [!WARNING]
> `pull_request_target` runs in the context of the **base repository** and therefore **has access to secrets**, even for PRs from forks. Combined with checking out untrusted PR code, this is a well-known privilege-escalation vector. Never check out and execute untrusted PR code in a `pull_request_target` workflow with secrets present.

## 2.2 Path and Branch Filtering Patterns

```yaml
on:
  push:
    branches:
      - main                # exact
      - 'release/**'         # matches release/, release/1.0, release/1.0/hotfix
      - 'feature/*'          # single segment only
    paths:
      - 'src/**'
      - '!src/**/*.test.js'  # negation: exclude test files
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'   # semantic version tags
```

## 2.3 Jobs: Dependencies, Parallelism, and Ordering

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: [lint, test]           # waits for BOTH lint and test to succeed
    runs-on: ubuntu-latest
    steps: [...]
```

```mermaid
flowchart TB
    Lint["lint (parallel)"] --> Deploy["deploy (needs: lint, test)"]
    Test["test (parallel)"] --> Deploy
    Deploy --> Done["Workflow complete"]
```

> [!IMPORTANT]
> By default, **all jobs run in parallel**. Use `needs:` to create dependencies and sequence them into a DAG (directed acyclic graph). A job with `needs:` only starts after all listed jobs succeed (by default — see conditional `needs` in §3.x). Each job still runs on its own fresh runner with no shared filesystem.

## 2.4 Job Outputs (Passing Data Between Jobs)

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.value }}
    steps:
      - id: get-version
        run: echo "value=1.2.3" >> "$GITHUB_OUTPUT"

  build:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building version ${{ needs.setup.outputs.version }}"
```

Since jobs share no filesystem, `outputs` (small string values) and **artifacts** (files) are the two mechanisms for passing data between them.

## 2.5 The Matrix Strategy (Every Form)

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false            # don't cancel other combos when one fails
      max-parallel: 3             # limit concurrent matrix jobs
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        # This creates 3 x 3 = 9 jobs

        include:                  # add extra combinations or extra vars to existing ones
          - os: ubuntu-latest
            node: 20
            coverage: true        # adds a variable to this specific combo

        exclude:                  # remove specific combinations
          - os: macos-latest
            node: 18
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

```mermaid
flowchart TB
    Matrix["matrix: os x node"] --> Combo1["ubuntu / 18"]
    Matrix --> Combo2["ubuntu / 20"]
    Matrix --> Combo3["windows / 22"]
    Matrix --> More["... all combinations run in PARALLEL"]
```

| Matrix Feature | Purpose |
|---|---|
| Base axes (`os`, `node`) | Cartesian product of all values |
| `include` | Add new combos, or add variables to existing combos |
| `exclude` | Remove specific combos from the product |
| `fail-fast: false` | Let all combos finish even if one fails (default is `true`) |
| `max-parallel` | Cap how many run at once |

## 2.6 Environment Variables and Contexts

```yaml
env:                              # workflow-level (all jobs)
  NODE_ENV: production

jobs:
  build:
    env:                          # job-level
      LOG_LEVEL: debug
    runs-on: ubuntu-latest
    steps:
      - name: Step with its own env
        env:                      # step-level (narrowest scope)
          API_URL: https://api.example.com
        run: echo "$NODE_ENV $LOG_LEVEL $API_URL"
```

```yaml
# Setting an env var for LATER steps (dynamic)
- run: echo "BUILD_ID=$(date +%s)" >> "$GITHUB_ENV"
- run: echo "Build was $BUILD_ID"   # available here
```

| Scope | Applies To |
|---|---|
| `env:` at workflow root | Every job |
| `env:` at job level | Every step in that job |
| `env:` at step level | That single step only |
| `$GITHUB_ENV` file | Sets a var for all *subsequent* steps in the job |

## 2.7 Expressions and Contexts

```yaml
steps:
  - if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    run: echo "Push to main"

  - run: echo "Repo: ${{ github.repository }}, Actor: ${{ github.actor }}"
  - run: echo "SHA: ${{ github.sha }}, Run #: ${{ github.run_number }}"
```

| Context | Contains |
|---|---|
| `github.*` | Event/repo metadata (`github.sha`, `github.ref`, `github.actor`, `github.event`) |
| `env.*` | Environment variables |
| `secrets.*` | Encrypted secrets |
| `vars.*` | Configuration variables (non-secret) |
| `matrix.*` | Current matrix combination values |
| `needs.*` | Outputs from dependency jobs |
| `steps.*` | Outputs from earlier steps (by `id`) |
| `job.*`, `runner.*` | Job status and runner info |
| `inputs.*` | Inputs to `workflow_dispatch`/`workflow_call` |

```yaml
# Functions available in expressions
${{ contains(github.event.head_commit.message, '[skip ci]') }}
${{ startsWith(github.ref, 'refs/tags/') }}
${{ format('{0}-{1}', matrix.os, matrix.node) }}
${{ fromJSON(steps.set.outputs.json) }}
${{ hashFiles('**/package-lock.json') }}
${{ success() }} ${{ failure() }} ${{ always() }} ${{ cancelled() }}
```

## 2.8 Conditional Execution (`if:`)

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'    # job-level condition
    runs-on: ubuntu-latest
    steps:
      - name: Only on failure of a previous step
        if: failure()
        run: echo "Something failed"

      - name: Always run (cleanup)
        if: always()
        run: echo "Runs even if prior steps failed"

      - name: Only if a specific step succeeded
        if: steps.build.outcome == 'success'
        run: echo "Build passed"
```

| Status Function | Runs When |
|---|---|
| `success()` | (Default) all previous steps succeeded |
| `failure()` | A previous step failed |
| `always()` | Always, regardless of outcome |
| `cancelled()` | The workflow was cancelled |

## 2.9 Secrets and Variables

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Use a secret
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: ./deploy.sh
      - name: Use a config variable (non-secret)
        run: echo "Region: ${{ vars.AWS_REGION }}"
```

| Type | Purpose | Visibility |
|---|---|---|
| Secrets | Sensitive values (tokens, keys) | Encrypted, masked in logs |
| Variables (`vars`) | Non-sensitive config | Plain text, visible |
| `GITHUB_TOKEN` | Auto-provided per-run token for GitHub API | Scoped, auto-expires |

> [!WARNING]
> Secrets are automatically masked in logs, but a secret piped through a transformation (e.g., base64-decoded, or passed to a tool that prints it) can leak in plaintext. Also, secrets are **not** passed to workflows triggered by `pull_request` from forks (a deliberate security measure).

## 2.10 Caching and Artifacts

```yaml
# CACHING — speed up repeated runs by reusing dependencies
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# ARTIFACTS — pass files between jobs or download later
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/

# In another job:
- uses: actions/download-artifact@v4
  with:
    name: build-output
```

| Mechanism | Purpose | Lifetime |
|---|---|---|
| Cache | Speed up builds by reusing dependencies/build outputs | Evicted by LRU/age, keyed by hash |
| Artifact | Share build outputs between jobs / provide downloadable results | Retained for a configurable period (default ~90 days) |

> [!TIP]
> Caches and artifacts solve different problems. **Cache** is an *optimization* — a cache miss just means slower, not broken. **Artifacts** are *data transfer* — a downstream job that needs an artifact will break if it's missing. Never use caching as a way to pass required data between jobs.

---

# 3. Advanced Concepts (Senior Level)

## 3.1 Reusable Workflows (`workflow_call`)

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy-token:
        required: true
    outputs:
      url:
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - id: deploy
        run: echo "url=https://${{ inputs.environment }}.example.com" >> "$GITHUB_OUTPUT"
```

```yaml
# Calling workflow
jobs:
  call-deploy:
    uses: ./.github/workflows/reusable-deploy.yml   # or org/repo/.github/workflows/x.yml@ref
    with:
      environment: production
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

```mermaid
flowchart TB
    Caller["Caller Workflow"] -->|"uses: reusable.yml + inputs/secrets"| Reusable["Reusable Workflow (on: workflow_call)"]
    Reusable --> Jobs["Runs its jobs"]
    Jobs -->|"outputs"| Caller
```

## 3.2 Composite Actions

```yaml
# .github/actions/setup-project/action.yml
name: Setup Project
description: Checkout, install Node, install deps
inputs:
  node-version:
    default: '20'
runs:
  using: composite               # composite = bundles multiple steps
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash                 # REQUIRED for run steps in composite actions
```

```yaml
# Using it
steps:
  - uses: ./.github/actions/setup-project
    with:
      node-version: 22
```

| Reuse Mechanism | Scope | Best For |
|---|---|---|
| Composite Action | Bundles **steps** into one reusable step | Repeated step sequences (setup, common tooling) |
| Reusable Workflow | Bundles **jobs** into one reusable workflow | Repeated multi-job pipelines (standard deploy) |

## 3.3 Concurrency Control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true       # cancel older runs of the same group when a new one starts
```

```mermaid
flowchart LR
    Run1["Run 1 (in progress)"] -->|"new push arrives"| Cancel["Run 1 CANCELLED"]
    Run2["Run 2 starts (same concurrency group)"]
    Cancel --> Run2
```

> [!TIP]
> Concurrency is essential for two patterns: (1) **cancel superseded CI runs** — when you push twice quickly, cancel the outdated first run to save minutes; (2) **serialize deployments** — with `cancel-in-progress: false`, ensure only one deploy to production happens at a time, queuing others.

## 3.4 Environments and Deployment Protection

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com   # shown in the deployment UI
    steps:
      - run: ./deploy.sh
```

```mermaid
flowchart LR
    Job["deploy-prod job"] --> Env["environment: production"]
    Env --> Gate{"Protection rules"}
    Gate -->|"required reviewers"| Approval["Wait for manual approval"]
    Gate -->|"wait timer"| Delay["Enforced delay"]
    Gate -->|"branch restriction"| Branch["Only from allowed branches"]
    Approval --> Deploy["Proceed with deployment"]
```

Environments provide deployment protection rules (required reviewers, wait timers, branch restrictions) and environment-scoped secrets — the gate between "validated" and "in production."

## 3.5 OIDC: Keyless Cloud Authentication

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write            # REQUIRED to request an OIDC token
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1
          # NO long-lived AWS keys needed!
```

```mermaid
sequenceDiagram
    participant Workflow as GitHub Actions Job
    participant GitHub as GitHub OIDC Provider
    participant Cloud as AWS/Azure/GCP

    Workflow->>GitHub: request a short-lived OIDC token
    GitHub-->>Workflow: signed JWT (identity claims: repo, branch, etc.)
    Workflow->>Cloud: assume role, presenting the JWT
    Cloud->>Cloud: verify JWT against configured trust policy
    Cloud-->>Workflow: short-lived cloud credentials
    Workflow->>Cloud: deploy using temporary credentials
```

> [!IMPORTANT]
> **OIDC is the modern best practice for cloud deployments** — instead of storing long-lived cloud access keys as secrets (which can leak and don't expire), the workflow requests a short-lived, cryptographically-signed identity token that the cloud provider verifies against a trust policy. No static credentials to store, rotate, or leak. Requires `permissions: id-token: write`.

## 3.6 The `GITHUB_TOKEN` and Permissions

```yaml
permissions:                     # explicitly scope the auto-provided token
  contents: read
  pull-requests: write
  packages: write
  # Default can be read-all or restricted; explicit least-privilege is best practice

jobs:
  comment:
    runs-on: ubuntu-latest
    permissions:                 # can also be set per-job
      pull-requests: write
    steps:
      - run: gh pr comment ${{ github.event.number }} --body "Automated check passed"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> [!WARNING]
> Follow least-privilege: set `permissions:` explicitly (start with `contents: read`) and grant only what each job needs. An overly-permissive `GITHUB_TOKEN` combined with a compromised action or malicious PR is a supply-chain risk. Also, changes pushed using the default `GITHUB_TOKEN` do **not** trigger further workflows (preventing infinite loops).

## 3.7 Dynamic Matrices (`fromJSON`)

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set.outputs.matrix }}
    steps:
      - id: set
        run: |
          # Compute the matrix at runtime (e.g., detect changed packages)
          echo 'matrix={"package":["auth","billing","orders"]}' >> "$GITHUB_OUTPUT"

  test:
    needs: generate-matrix
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.generate-matrix.outputs.matrix) }}
    steps:
      - run: echo "Testing ${{ matrix.package }}"
```

Dynamic matrices are the key to **monorepo CI** — a first job detects which packages changed and emits a JSON matrix, so only affected packages get tested/built in parallel.

## 3.8 Service Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - run: npm test              # tests connect to localhost:5432 / localhost:6379
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
```

Service containers spin up dependencies (databases, caches, message queues) alongside your job — the CI equivalent of the Testcontainers pattern for integration testing referenced in [[02 Postgres]] and [[03 MongoDB]].

## 3.9 Running Jobs/Steps in Containers

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:                     # run the ENTIRE job inside this container
      image: node:20-alpine
      env:
        NODE_ENV: test
    steps:
      - uses: actions/checkout@v4
      - run: node --version        # runs inside node:20-alpine
```

## 3.10 Failure Scenarios

| Failure | Symptom | Common Cause | Fix |
|---|---|---|---|
| Job can't find files from a previous job | "No such file/directory" | Jobs don't share a filesystem; expecting build output to persist | Use artifacts (upload/download) to pass files between jobs |
| Secret is empty in a fork PR | Deploy step fails / auth error | Secrets aren't passed to `pull_request` from forks | Use `pull_request_target` carefully, or restructure so forks don't need secrets |
| Workflow doesn't trigger | Nothing runs on push | `paths:`/`branches:` filter excludes the change, or file not on default branch | Check filters; ensure the workflow exists on the relevant branch |
| Infinite workflow loop | Workflows keep re-triggering each other | A workflow pushes commits that re-trigger it | Default `GITHUB_TOKEN` pushes don't re-trigger; if using a PAT, guard with conditions |
| Cache never hits | Builds always slow | Cache key includes something that changes every run | Key on lockfile hash, not commit SHA or timestamp |
| Matrix job fails cancel all others | Lose results from passing combos | `fail-fast: true` (default) cancels the whole matrix on first failure | Set `fail-fast: false` |
| `pull_request_target` RCE risk | Security incident | Checking out + running untrusted fork code with secrets available | Never execute untrusted code in that context; use `pull_request` for untrusted PRs |
| Deployment race condition | Two deploys clobber each other | No concurrency control on the deploy job | Add a `concurrency` group with `cancel-in-progress: false` to serialize |

## 3.11 Security Best Practices

| Risk | Mitigation |
|---|---|
| Compromised third-party action | Pin actions to a full commit SHA (`uses: actions/checkout@<sha>`), not just a tag |
| Over-privileged token | Set `permissions:` explicitly to least-privilege |
| Long-lived cloud keys leaking | Use OIDC instead of stored cloud credentials |
| Secret exposure in logs | Don't echo secrets; rely on auto-masking but don't defeat it via transformations |
| `pull_request_target` privilege escalation | Never check out/run untrusted PR code with secrets present |
| Supply-chain injection via untrusted input | Never interpolate untrusted `github.event.*` data (like PR titles) directly into `run:` scripts — it enables script injection |

```yaml
# SCRIPT INJECTION RISK — PR title interpolated into shell
- run: echo "Title: ${{ github.event.pull_request.title }}"   # DANGEROUS if title is `"; rm -rf / #`
# SAFE — pass via environment variable, use the shell variable
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "Title: $TITLE"
```

---

# 4. Real-World System Design Usage

## 4.1 A Complete Production Pipeline

```mermaid
flowchart TB
    PR["Pull Request opened"] --> Lint["Lint + Format check"]
    PR --> UnitTest["Unit tests (matrix: node 18/20/22)"]
    PR --> IntTest["Integration tests (service containers)"]
    Lint --> Gate1{"All checks pass?"}
    UnitTest --> Gate1
    IntTest --> Gate1
    Gate1 -->|"yes"| Merge["Merge to main"]
    Merge --> Build["Build + push container image"]
    Build --> DeployStaging["Deploy to staging (auto)"]
    DeployStaging --> E2E["E2E tests against staging"]
    E2E --> DeployProd["Deploy to production (environment: manual approval)"]
    DeployProd --> Monitor["Post-deploy smoke tests + monitoring"]
```

## 4.2 Full Example: Build, Test, Deploy Pipeline

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}
          aws-region: us-east-1
      - run: ./deploy.sh
```

## 4.3 Big-Company Style Thinking

| Concern | GitHub Actions Design Response |
|---|---|
| Reliability | Required status checks block merges; matrix testing across versions/OSes |
| Speed | Caching, path-filtered/dynamic-matrix monorepo CI, concurrency cancellation |
| Security | OIDC (no static keys), SHA-pinned actions, least-privilege permissions, secret scanning |
| Consistency | Reusable workflows + composite actions enforce one standard pipeline org-wide |
| Deployment safety | Environments with required reviewers, serialized deploys via concurrency, staged rollout |
| Observability | Job summaries, status checks, deployment tracking, integration with monitoring/alerting |

## 4.4 Monorepo Strategy

```yaml
# Only run each package's CI when that package changes
on:
  push:
    paths:
      - 'packages/auth/**'
jobs:
  # ... or use dynamic matrix (§3.7) to compute changed packages at runtime
```

## 4.5 Integration with Other Systems

| System | Integration |
|---|---|
| Cloud (AWS/Azure/GCP) | OIDC + official credential-configuring actions |
| Container registries | GHCR, Docker Hub via `docker/build-push-action` |
| Kubernetes | `kubectl`/Helm steps, or GitOps (Actions updates a manifest repo) |
| Secrets managers | Vault/cloud secrets pulled at runtime via OIDC |
| Notifications | Slack/Teams/email on failure via Marketplace actions |
| Code quality | SonarQube, CodeQL (GitHub's own security scanning), coverage services |

---

# 5. Interview Preparation

## 5.1 What Interviewers Expect

For "GitHub Actions / CI/CD" topics, interviewers usually expect:

- You understand the workflow → jobs → steps hierarchy and what state lives where.
- You can write a basic build-test workflow with triggers, caching, and matrix.
- You understand secrets, artifacts vs. caching, and job dependencies (`needs`).

For senior/DevOps roles, they also expect:

- You know reusable workflows, composite actions, and dynamic matrices.
- You understand OIDC and why it's preferred over stored cloud keys.
- You can reason about security (SHA pinning, `pull_request_target`, script injection, least-privilege permissions).
- You can design a full CI/CD pipeline with environments, approvals, and concurrency control.

## 5.2 Most Important Questions and Answers

### Q1. What's the difference between a workflow, a job, and a step?

A workflow is the whole automated process (one YAML file) triggered by events. A job is a group of steps running on a single fresh runner; jobs run in parallel by default. A step is an individual task within a job — either a shell command (`run`) or a reusable action (`uses`). Steps share a filesystem/environment; jobs do not.

### Q2. Why don't jobs share files, and how do you pass data between them?

Each job runs on its own fresh, isolated runner VM, so there's no shared filesystem. To pass small string values, use job `outputs`; to pass files, use artifacts (`upload-artifact`/`download-artifact`). Caching is for optimization, not data transfer.

### Q3. What's the difference between caching and artifacts?

Caching is a performance optimization — it stores dependencies/build outputs to speed up future runs, and a cache miss just means a slower (not broken) run. Artifacts are explicit data transfer — files uploaded to be shared between jobs or downloaded, and a missing required artifact breaks a downstream job. Never rely on cache to pass required data.

### Q4. What is OIDC in GitHub Actions and why use it?

OIDC (OpenID Connect) lets a workflow authenticate to a cloud provider using a short-lived, cryptographically-signed identity token instead of stored long-lived credentials. The cloud verifies the token against a trust policy and issues temporary credentials. This eliminates static keys that can leak and don't expire — the modern best practice for cloud deployments. It requires `permissions: id-token: write`.

### Q5. What's the security risk with `pull_request_target`?

Unlike `pull_request` (which runs in the fork's context with no secrets), `pull_request_target` runs in the base repository's context with access to secrets, even for fork PRs. If such a workflow checks out and executes the untrusted PR's code, an attacker can steal secrets or gain code execution — a well-known privilege-escalation vector. Never run untrusted code in that context.

### Q6. How does the matrix strategy work, and what are `include`/`exclude`?

A matrix defines axes (e.g., `os`, `node-version`) and runs a job for every combination (Cartesian product) in parallel. `include` adds extra combinations or extra variables to existing combos; `exclude` removes specific combinations. `fail-fast: false` prevents one failing combo from cancelling the others.

### Q7. What's the difference between a reusable workflow and a composite action?

A composite action bundles multiple **steps** into one reusable step (for repeated step sequences like project setup). A reusable workflow bundles multiple **jobs** into a callable workflow via `on: workflow_call` (for repeated multi-job pipelines like a standard deploy). Composite = step-level reuse; reusable workflow = job-level reuse.

### Q8. How do you prevent multiple deployments from running simultaneously?

Use a `concurrency` group keyed to the deployment target with `cancel-in-progress: false`, which serializes runs in the same group (queuing rather than cancelling). For CI (not deploys), `cancel-in-progress: true` instead cancels superseded runs to save time.

### Q9. Why should you pin actions to a commit SHA instead of a tag?

Tags (like `@v4`) are mutable — a compromised or malicious maintainer could move a tag to point at malicious code, which your workflow would then execute (potentially with access to secrets). Pinning to a full commit SHA (`@<40-char-sha>`) guarantees you run exactly the reviewed code, mitigating this supply-chain risk.

### Q10. What is script injection in GitHub Actions and how do you prevent it?

If untrusted input (like a PR title or branch name from `github.event.*`) is interpolated directly into a `run:` script via `${{ }}`, an attacker can craft input containing shell commands that execute on the runner. Prevent it by passing such values through environment variables and referencing the shell variable (`$TITLE`) instead of interpolating the expression directly into the command.

## 5.3 Tricky Questions

### Why doesn't a commit pushed by a workflow using `GITHUB_TOKEN` trigger another workflow?

This is a deliberate safeguard against infinite loops — events created using the automatic `GITHUB_TOKEN` do not trigger further workflow runs. If you genuinely need a downstream trigger, you must use a Personal Access Token (PAT) or GitHub App token, and then carefully guard against loops with conditions.

### If all jobs run in parallel by default, how does a job "wait" for another?

Via the `needs:` keyword, which declares dependencies and turns the flat parallel set of jobs into a DAG. A job with `needs: [a, b]` only starts after both `a` and `b` complete successfully (by default). This is how you sequence build → test → deploy while still parallelizing independent jobs.

### Can a step's failure be ignored so the job continues?

Yes — `continue-on-error: true` on a step lets the job proceed even if that step fails (the step's `outcome` is still recorded as failure, but the job's `conclusion` isn't affected). Useful for non-critical steps like optional notifications or flaky auxiliary checks.

### What's the difference between `${{ }}` evaluated at different times?

Most expressions are evaluated when the workflow is compiled/step is reached, using contexts available at that point. Some contexts (like `steps.*` outputs or `env` set via `$GITHUB_ENV`) only become available after earlier steps run. A common bug is referencing a value in an `env:` block that isn't yet populated — understanding evaluation timing and which context is available where is key.

## 5.4 Common Candidate Mistakes

- Expecting files to persist between jobs (they don't — need artifacts).
- Using cache to pass required data between jobs (cache misses break it).
- Not setting `permissions:` explicitly (over-privileged `GITHUB_TOKEN`).
- Interpolating untrusted `github.event.*` data into `run:` (script injection).
- Using `pull_request_target` with untrusted checkout and secrets.
- Pinning actions to mutable tags instead of SHAs for security-sensitive workflows.
- Storing long-lived cloud keys instead of using OIDC.
- Forgetting `fail-fast: false` and losing results from passing matrix combos.

## 5.5 Interview Coding Checklist

- [ ] Use `needs:` to sequence jobs; understand they otherwise run in parallel.
- [ ] Pass files between jobs via artifacts, small values via outputs.
- [ ] Cache dependencies keyed on the lockfile hash.
- [ ] Set `permissions:` explicitly to least-privilege.
- [ ] Use OIDC for cloud auth, not stored keys.
- [ ] Pass untrusted input via env vars, never interpolate into `run:` directly.
- [ ] Add concurrency control for CI cancellation and deployment serialization.

---

# 6. Hands-On Thinking

## 6.1 Three Real-World Pipelines

### Project 1: Node.js Library CI

Concepts: matrix across Node versions/OSes, caching, lint + test + coverage, publish to npm on tagged release via OIDC/trusted publishing.

```yaml
on:
  push:
    tags: ['v*']
jobs:
  publish:
    permissions:
      id-token: write   # npm trusted publishing via OIDC
    # ... build, test, npm publish
```

### Project 2: Containerized Web App to Kubernetes

Concepts: build/push image to GHCR, OIDC to cloud, deploy to staging automatically, promote to production behind an environment approval, post-deploy smoke tests.

```mermaid
flowchart LR
    Build["Build image"] --> Push["Push to GHCR"]
    Push --> Staging["Deploy staging (auto)"]
    Staging --> Prod["Deploy prod (environment approval)"]
```

### Project 3: Monorepo with Selective CI

Concepts: dynamic matrix computing changed packages, reusable workflow per package type, composite action for shared setup, path-based triggering.

```yaml
jobs:
  changes:
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    # ... detect changed packages, emit JSON
  test:
    needs: changes
    strategy:
      matrix:
        package: ${{ fromJSON(needs.changes.outputs.packages) }}
```

## 6.2 Step-by-Step Design Approach

For any GitHub Actions pipeline:

1. Identify the triggering events and filters (which branches/paths/events should run what).
2. Decompose into jobs by parallelizable concern (lint, test, build, deploy) and sequence with `needs:`.
3. Decide data flow between jobs (artifacts for files, outputs for values).
4. Add caching keyed on lockfiles to speed up dependency install.
5. Set explicit least-privilege `permissions:` and use OIDC for cloud auth.
6. Gate deployments with environments (approvals) and serialize with concurrency.
7. Extract repeated logic into composite actions / reusable workflows.
8. Harden security: pin actions to SHAs, avoid script injection, guard `pull_request_target`.

## 6.3 Production Implementation Approach

```mermaid
flowchart TD
    A["Define triggers + filters"] --> B["Decompose into jobs + needs DAG"]
    B --> C["Data flow: artifacts + outputs"]
    C --> D["Caching for speed"]
    D --> E["Least-privilege permissions + OIDC"]
    E --> F["Environments + concurrency for deploy safety"]
    F --> G["Extract reusable workflows/composite actions"]
    G --> H["Security hardening (SHA pinning, injection guards)"]
    H --> I["Required status checks on the branch"]
```

## 6.4 Production Readiness Example

For a GitHub Actions pipeline, define:

- Required status checks configured on protected branches to block bad merges.
- OIDC-based cloud authentication (no long-lived stored keys).
- Explicit least-privilege `permissions:` on every workflow/job.
- Actions pinned to commit SHAs in security-sensitive workflows.
- Production deploys gated by an environment with required reviewers, serialized via concurrency.
- Caching configured and verified to actually hit.
- Reusable workflows/composite actions for org-wide consistency.
- Failure notifications wired to the team's alerting channel.

---

# 7. Deep Dive (Optional but Important)

## 7.1 Workflow Run Lifecycle Internals

```mermaid
flowchart TB
    Event["Event occurs (webhook)"] --> Match["GitHub matches event against workflow triggers"]
    Match --> Queue["Run queued"]
    Queue --> Assign["Runner assigned (GitHub-hosted spun up, or self-hosted picks up)"]
    Assign --> Provision["Runner provisions: fresh VM, tools preinstalled"]
    Provision --> Execute["Jobs execute (parallel/DAG per needs)"]
    Execute --> Report["Status reported back to commit/PR"]
    Report --> Teardown["GitHub-hosted runner destroyed (ephemeral)"]
```

GitHub-hosted runners are **ephemeral** — a brand-new VM per job, destroyed afterward. This is why nothing persists between runs (hence caching/artifacts) and why every run starts from a clean, reproducible state.

## 7.2 Expression Evaluation and Context Availability Timing

```text
Workflow parse time  -> `on:`, top-level `env:`, static structure evaluated
Job start            -> job-level contexts (needs.*, matrix.*) resolved
Step execution       -> steps.* (from prior steps), runtime env ($GITHUB_ENV) become available
```

Understanding *when* each context is populated explains a whole class of bugs — e.g., referencing `steps.foo.outputs.x` in a step's `env:` before step `foo` has run yields empty, because that output doesn't exist yet at evaluation time.

## 7.3 How Actions Are Resolved and Executed

```mermaid
flowchart TB
    Uses["uses: owner/repo@ref"] --> Fetch["GitHub fetches the action's code at that ref"]
    Fetch --> Type{"Action type?"}
    Type -->|"JavaScript"| JS["Runs via Node.js on the runner directly"]
    Type -->|"Docker"| Docker["Builds/pulls a container, runs the action inside it"]
    Type -->|"Composite"| Composite["Expands into its constituent steps inline"]
```

| Action Type | How It Runs |
|---|---|
| JavaScript | Executed by Node.js on the runner — fastest, cross-platform |
| Docker container | Runs inside a container — full environment control, Linux-only |
| Composite | Bundles other steps/actions, expanded inline |

## 7.4 Self-Hosted Runners

```mermaid
flowchart LR
    GitHub["GitHub"] -->|"job queued"| Runner["Self-hosted runner (your infra)"]
    Runner --> Execute["Executes job on your hardware/network"]
    Runner --> Access["Can access private networks, special hardware (GPU), custom tools"]
```

Self-hosted runners execute jobs on your own infrastructure — useful for GPU/special hardware, private network access, or cost control at scale. But they carry security responsibility: since they persist between jobs (unless ephemeral) and run on your network, running untrusted code (fork PRs) on them is dangerous.

> [!WARNING]
> **Never use self-hosted runners for public repository `pull_request` workflows** unless they're ephemeral and isolated. A malicious PR could run arbitrary code on your infrastructure, persist malware between jobs, or pivot into your private network.

## 7.5 Job Summaries and Outputs

```yaml
- name: Write a job summary
  run: |
    echo "## Test Results" >> "$GITHUB_STEP_SUMMARY"
    echo "- Passed: 42" >> "$GITHUB_STEP_SUMMARY"
    echo "- Failed: 0" >> "$GITHUB_STEP_SUMMARY"
```

`$GITHUB_STEP_SUMMARY` lets a job write rich Markdown displayed on the run's summary page — useful for surfacing test results, coverage, or deployment info without digging through logs.

## 7.6 Debugging Tools

| Tool/Technique | Purpose |
|---|---|
| Re-run with debug logging (`ACTIONS_RUNNER_DEBUG`, `ACTIONS_STEP_DEBUG` secrets) | Verbose internal logging |
| `act` (local runner) | Run workflows locally for faster iteration |
| Job summaries (`$GITHUB_STEP_SUMMARY`) | Surface structured output |
| Workflow visualization graph | See the job DAG and status in the UI |
| `tmate` action | SSH into a running runner for live debugging |

---

# Production Checklists

## Workflow Quality Checklist

- [ ] Triggers and path/branch filters correctly scope when workflows run.
- [ ] Jobs sequenced with `needs:` where order matters; parallel where independent.
- [ ] Files passed between jobs via artifacts, values via outputs (not cache).
- [ ] Dependency caching configured and verified to hit.
- [ ] Repeated logic extracted into composite actions / reusable workflows.
- [ ] `fail-fast: false` where full matrix results are wanted.

## Security Checklist

- [ ] `permissions:` set explicitly to least-privilege on every workflow/job.
- [ ] OIDC used for cloud authentication instead of stored long-lived keys.
- [ ] Third-party actions pinned to commit SHAs in sensitive workflows.
- [ ] No untrusted `github.event.*` input interpolated into `run:` scripts.
- [ ] `pull_request_target` not used to run untrusted code with secrets.
- [ ] Self-hosted runners not exposed to untrusted public PRs.

## Deployment Safety Checklist

- [ ] Production deploys gated by an environment with required reviewers.
- [ ] Concurrency control serializes deploys (`cancel-in-progress: false`).
- [ ] Required status checks block merges of failing code.
- [ ] Staged rollout (staging → E2E → production) where appropriate.
- [ ] Post-deploy smoke tests and rollback plan.

## Debugging Checklist

- [ ] Enable step debug logging when a run behaves unexpectedly.
- [ ] Check context-availability timing when an expression is empty.
- [ ] Check job isolation first when files "disappear" between jobs.
- [ ] Verify cache key stability when caches never hit.
- [ ] Test workflow changes on a branch before merging to the default branch.

---

# Learning Roadmap

## Phase 1: Beginner

Learn: workflow file structure, `on`/`jobs`/`steps`, `uses` vs `run`, runners, basic triggers.

Practice: a CI workflow that checks out, installs, and tests a project on push/PR.

## Phase 2: Intermediate

Learn: matrix builds, caching, artifacts, `needs`/job outputs, secrets/variables, conditional `if:`, expressions/contexts.

Practice: a multi-job build → test → deploy pipeline with a matrix and caching.

## Phase 3: Advanced

Learn: reusable workflows, composite actions, concurrency, environments/approvals, OIDC, service containers, dynamic matrices.

Practice: an OIDC-based cloud deployment with environment approval and serialized deploys.

## Phase 4: Production DevOps Engineer

Learn: security hardening (SHA pinning, injection, least-privilege), self-hosted runners, monorepo CI strategy, org-wide reusable workflow standards.

Practice: a full production pipeline with monorepo selective CI, reusable org workflows, hardened security, and staged deployments with rollback.

---

# Self-Review Completion Loop

The topic was reviewed against:

- Official GitHub Actions documentation (workflow syntax, events, contexts, security hardening, OIDC).
- Common production incident patterns (job isolation surprises, script injection, `pull_request_target` escalation, cache misuse).
- CI/CD best practices (least-privilege, keyless auth, deployment gating).
- Interview patterns for beginner through senior DevOps/backend roles.

## Gap Review Matrix

| Area | Covered? | Notes |
|---|---|---|
| CI/CD fundamentals | Yes | CI vs. Delivery vs. Deployment |
| Workflow structure | Yes | workflow → jobs → steps hierarchy |
| Every trigger (`on:`) | Yes | push/PR/schedule/dispatch/call/run/target and filters |
| `uses` vs `run` | Yes | Actions vs. shell commands |
| Job dependencies + parallelism | Yes | `needs:`, DAG, parallel default |
| Job outputs + artifacts | Yes | Data passing between isolated jobs |
| Matrix (all forms) | Yes | include/exclude/fail-fast/max-parallel/dynamic |
| Env vars + scopes | Yes | Workflow/job/step/`$GITHUB_ENV` |
| Expressions + contexts | Yes | All contexts, functions, status checks |
| Conditionals (`if:`) | Yes | success/failure/always/cancelled |
| Secrets + variables + `GITHUB_TOKEN` | Yes | With fork/masking caveats |
| Caching vs. artifacts | Yes | Distinct purposes emphasized |
| Reusable workflows | Yes | `workflow_call`, inputs/secrets/outputs |
| Composite actions | Yes | Step-level reuse |
| Concurrency | Yes | Cancellation + serialization patterns |
| Environments + approvals | Yes | Deployment protection rules |
| OIDC | Yes | Keyless cloud auth, sequence diagram |
| Permissions | Yes | Least-privilege token scoping |
| Dynamic matrices | Yes | `fromJSON`, monorepo CI |
| Service + job containers | Yes | Integration test dependencies |
| Security | Yes | SHA pinning, injection, `pull_request_target` |
| Self-hosted runners | Yes | Use cases + security warnings |
| Failure scenarios | Yes | Eight concrete patterns |
| Interview prep | Yes | Common + tricky questions |
| Hands-on projects | Yes | Three realistic pipelines |
| Internals | Yes | Run lifecycle, expression timing, action resolution |

No significant beginner-to-senior GitHub Actions gaps remain for the requested scope. Further specialization should split into separate deep dives: **Writing Custom JavaScript/Docker Actions**, **GitOps with GitHub Actions**, **Advanced Self-Hosted Runner Autoscaling**, **CodeQL/Security Scanning Pipelines**, and **Comparing GitHub Actions to Jenkins/GitLab CI**.

---

# Official References

- GitHub Actions Documentation: <https://docs.github.com/en/actions>
- Workflow Syntax Reference: <https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions>
- Contexts Reference: <https://docs.github.com/en/actions/learn-github-actions/contexts>
- Expressions Reference: <https://docs.github.com/en/actions/learn-github-actions/expressions>
- Security Hardening Guide: <https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions>
- OIDC Guide: <https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect>

---

## Final Summary

GitHub Actions structures all automation around a strict hierarchy — workflows react to events, jobs run in parallel on isolated fresh runners, and steps run sequentially sharing a filesystem — and mastering it means always knowing which level your state lives at. The YAML surface is vast (every trigger form, matrix variation, expression context, and reuse mechanism), but the senior-level differentiators are operational and security-focused: passing data correctly across isolated jobs (artifacts/outputs, never cache), authenticating to the cloud without static keys (OIDC), gating deployments with environments and concurrency, and hardening against the platform's real attack surface — script injection, `pull_request_target` escalation, mutable action tags, and over-privileged tokens.
