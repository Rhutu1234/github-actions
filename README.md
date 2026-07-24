# GitHub Actions: CI/CD Automation Built into GitHub

*A practical guide to GitHub Actions — the automation platform built directly into GitHub for CI/CD pipelines, covering workflow syntax, triggers, jobs and runners, secrets management, reusable workflows, matrix builds, and deployment patterns for .NET applications.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Concepts](#1-core-concepts)
3. [Workflow Syntax](#2-workflow-syntax)
4. [Triggers](#3-triggers)
5. [Jobs, Dependencies, and Runners](#4-jobs-dependencies-and-runners)
6. [A Complete .NET CI Pipeline](#5-a-complete-net-ci-pipeline)
7. [Secrets and Variables](#6-secrets-and-variables)
8. [Matrix Builds](#7-matrix-builds)
9. [Caching Dependencies](#8-caching-dependencies)
10. [Reusable Workflows and Composite Actions](#9-reusable-workflows-and-composite-actions)
11. [Deployment Patterns](#10-deployment-patterns)
12. [Environments and Approval Gates](#11-environments-and-approval-gates)
13. [Security Considerations](#12-security-considerations)
14. [Quick Reference Table](#quick-reference-table)
15. [Conclusion](#conclusion)

---

## Introduction

GitHub Actions is GitHub's built-in automation platform — workflows defined as YAML files living in your repository (`.github/workflows/`) that run in response to events: a push, a pull request, a schedule, or a manual trigger. It covers everything from a simple "run the tests on every PR" pipeline to a full multi-stage build, test, and deployment pipeline spanning multiple cloud providers.

```yaml
name: CI
on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore
      - run: dotnet test --no-build
```

That's a complete, working CI pipeline — checkout the code, install .NET, restore, build, and test — triggered automatically on every push and pull request, with zero infrastructure to provision or maintain.

---

## 1. Core Concepts

### Workflows, jobs, and steps

- A **workflow** is a YAML file defining one or more automated processes, triggered by events.
- A **job** is a set of steps that run together on the same runner (virtual machine or container).
- A **step** is an individual task — running a shell command, or invoking a reusable **action**.
- An **action** is a reusable, packaged unit of automation (like `actions/checkout` or `actions/setup-dotnet`), either from GitHub itself, the community (via the GitHub Marketplace), or your own repository.

```
.github/workflows/ci.yml   ← one workflow file
  └── job: build-and-test
        ├── step: checkout code
        ├── step: setup .NET
        ├── step: restore
        ├── step: build
        └── step: test
```

### Runners

A **runner** is the machine (virtual or physical) that actually executes a job's steps. GitHub provides **GitHub-hosted runners** (`ubuntu-latest`, `windows-latest`, `macos-latest`) — fresh VMs provisioned per job, pre-loaded with common tooling and torn down afterward — or you can register **self-hosted runners** on your own infrastructure for specialized hardware, network access requirements, or cost control at high volume.

---

## 2. Workflow Syntax

### Anatomy of a workflow file

```yaml
name: CI Pipeline               # display name shown in the GitHub UI

on:                              # what triggers this workflow
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:                              # environment variables available to every job
  DOTNET_VERSION: '9.0.x'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --logger trx
```

### Expressions and contexts

```yaml
- name: Print branch name
  run: echo "Building branch ${{ github.ref_name }}"

- name: Conditional step
  if: github.event_name == 'pull_request'
  run: echo "This only runs for pull requests"
```

GitHub Actions exposes several built-in **contexts** (`github`, `env`, `secrets`, `matrix`, `steps`, `job`, `needs`) accessible via `${{ }}` expression syntax — `github.ref_name` (the branch/tag name), `github.event_name` (what triggered the run), `github.sha` (the commit SHA), and many more, letting workflow behavior adapt dynamically to the specific event that triggered it.

---

## 3. Triggers

### Common event triggers

```yaml
on:
  push:
    branches: [main, develop]
    paths: ['src/**', '!src/**/*.md']   # only trigger if these paths changed

  pull_request:
    branches: [main]

  schedule:
    - cron: '0 2 * * *'   # daily at 2:00 AM UTC

  workflow_dispatch:        # manual trigger, with optional inputs
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]

  release:
    types: [published]
```

- **`push`/`pull_request`** — the most common triggers for CI, optionally scoped to specific branches or file paths (`paths` filtering avoids re-running a full pipeline for a documentation-only change, for instance).
- **`schedule`** — cron-based triggers for periodic tasks (nightly builds, scheduled cleanup jobs, dependency update checks).
- **`workflow_dispatch`** — enables a manual "Run workflow" button in the GitHub UI, optionally with typed inputs, useful for on-demand deployments or maintenance tasks.
- **`release`** — triggers when a GitHub Release is published, a natural hook for a production deployment pipeline.

### Reacting to other workflows

```yaml
on:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
    branches: [main]
```

`workflow_run` lets one workflow trigger based on another workflow's completion — a common pattern for separating "build and test" from "deploy," so deployment only proceeds after CI has genuinely succeeded on the exact commit being deployed.

---

## 4. Jobs, Dependencies, and Runners

### Running jobs in parallel (the default)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [ /* ... */ ]

  test:
    runs-on: ubuntu-latest
    steps: [ /* ... */ ]
```

Jobs within a workflow run in parallel by default, each on its own fresh runner — `lint` and `test` above would execute simultaneously, not sequentially, unless you explicitly declare a dependency between them.

### Sequencing jobs with `needs`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [ /* ... */ ]

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps: [ /* ... */ ]
```

`needs` establishes an explicit dependency — `deploy` waits for `build` to complete successfully before starting, and is skipped entirely if `build` fails, which is exactly the behavior you want for a pipeline where deployment should never proceed against a build that didn't pass.

### Passing data between jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get_version.outputs.version }}
    steps:
      - id: get_version
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

Since each job runs on its own isolated runner (no shared filesystem or memory between them), passing data between jobs requires explicit `outputs` — a common pattern for computing a version number, build artifact path, or other value in one job and consuming it in a dependent job.

### Choosing a runner

```yaml
jobs:
  build-windows:
    runs-on: windows-latest   # for .NET Framework or Windows-specific builds
  build-linux:
    runs-on: ubuntu-latest    # cheaper, faster, and the default for most .NET Core/.NET 5+ workloads
  build-mac:
    runs-on: macos-latest     # for iOS/macOS-targeted builds (e.g., MAUI)
```

For most modern .NET applications (.NET 5 and later, cross-platform by default), `ubuntu-latest` is both the cheapest and fastest option among GitHub-hosted runners — reserve `windows-latest` for workloads with a genuine Windows-specific dependency, and `macos-latest` specifically for builds that need to produce iOS/macOS outputs.

---

## 5. A Complete .NET CI Pipeline

```yaml
name: .NET CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run unit tests
        run: dotnet test --no-build --configuration Release --logger trx --results-directory TestResults

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: 'TestResults/*.trx'
          reporter: dotnet-trx

      - name: Publish
        if: github.ref == 'refs/heads/main'
        run: dotnet publish -c Release -o ./publish

      - name: Upload build artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: app-package
          path: ./publish
```

This pipeline restores, builds, and tests on every push and pull request, publishes test results in a readable format even if tests fail (`if: always()` ensures this step runs regardless of prior step outcomes), and — only on the `main` branch — publishes the application and uploads it as a build artifact ready for a subsequent deployment job to pick up.

---

## 6. Secrets and Variables

### Repository and organization secrets

```yaml
steps:
  - name: Deploy to Azure
    run: az webapp deploy --name my-api --resource-group my-rg
    env:
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
```

Secrets (configured under repository or organization **Settings → Secrets and variables → Actions**) are encrypted at rest, never displayed in logs (GitHub automatically redacts any secret value that appears in output), and accessed via the `secrets` context — never hardcode credentials directly in a workflow file, since that file is plain text, version-controlled, and often visible to anyone with read access to the repository.

### Variables (non-secret configuration)

```yaml
env:
  ASPNETCORE_ENVIRONMENT: ${{ vars.ENVIRONMENT_NAME }}
```

**Variables** (as opposed to secrets) are for non-sensitive configuration values that still benefit from being centrally managed rather than hardcoded in every workflow file — an environment name, a region, a feature flag default.

### OIDC: avoiding long-lived cloud credentials entirely

```yaml
permissions:
  id-token: write   # required for OIDC
  contents: read

steps:
  - name: Azure Login via OIDC
    uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Rather than storing a long-lived cloud service principal secret, GitHub Actions supports **OpenID Connect (OIDC)** federation with Azure, AWS, and GCP — the workflow requests a short-lived token that the cloud provider trusts based on a pre-configured federated identity relationship (tied to the specific repository/branch), eliminating the need to store, rotate, or risk leaking a long-lived cloud credential at all. This is the currently recommended approach for cloud deployments from GitHub Actions, a meaningful security improvement over storing a static secret.

---

## 7. Matrix Builds

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet-version: ['8.0.x', '9.0.x']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet-version }}
      - run: dotnet test
```

A **matrix** runs the same job across every combination of the specified dimensions — the example above runs tests across 4 combinations (2 operating systems × 2 .NET versions) automatically, without duplicating the job definition four times. This is the standard pattern for confirming a library or application behaves correctly across multiple supported OS/runtime version combinations.

### Excluding specific combinations

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    dotnet-version: ['8.0.x', '9.0.x']
    exclude:
      - os: macos-latest
        dotnet-version: '8.0.x'
```

`exclude` (and its counterpart `include`, for adding specific extra combinations beyond the full cross-product) lets you skip combinations that don't need testing, keeping matrix size and CI cost reasonable.

### Fail-fast behavior

```yaml
strategy:
  fail-fast: false   # let all matrix jobs run to completion, even if one fails
  matrix:
    os: [ubuntu-latest, windows-latest]
```

By default, a matrix cancels all remaining in-progress jobs the moment any one combination fails — `fail-fast: false` disables this, useful when you want a complete picture of exactly which combinations pass and fail, rather than stopping at the first failure.

---

## 8. Caching Dependencies

```yaml
- name: Cache NuGet packages
  uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
    restore-keys: |
      ${{ runner.os }}-nuget-
```

Since GitHub-hosted runners are fresh, ephemeral VMs for every job, dependencies (NuGet packages, npm modules, Docker layers) get re-downloaded from scratch on every run unless explicitly cached. `actions/cache` stores a directory keyed by a hash of relevant lock/project files — if the key matches an existing cache entry (meaning dependencies haven't changed), the cache is restored instead of re-downloading everything, often producing a substantial speedup for dependency-heavy builds.

### Built-in caching via `setup-dotnet`

```yaml
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '9.0.x'
    cache: true
    cache-dependency-path: '**/packages.lock.json'
```

Recent versions of `setup-dotnet` support built-in dependency caching without needing a separate, manually configured `actions/cache` step — simpler to set up for the common case, though it requires a `packages.lock.json` file (generated via `dotnet restore --use-lock-file` or enabled via a project setting) to compute a reliable cache key.

---

## 9. Reusable Workflows and Composite Actions

### Reusable workflows: share an entire pipeline

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build
on:
  workflow_call:
    inputs:
      dotnet-version:
        required: true
        type: string
    secrets:
      nuget-api-key:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ inputs.dotnet-version }}
      - run: dotnet build
```

```yaml
# .github/workflows/ci.yml — calling the reusable workflow
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      dotnet-version: '9.0.x'
    secrets:
      nuget-api-key: ${{ secrets.NUGET_API_KEY }}
```

`workflow_call` lets one workflow file be invoked by others, with explicit typed inputs and secrets — the standard mechanism for sharing an entire multi-job pipeline definition across many repositories in an organization, avoiding copy-pasted, slowly-diverging pipeline YAML in every repo.

### Composite actions: share a sequence of steps

```yaml
# .github/actions/setup-and-restore/action.yml
name: 'Setup .NET and Restore'
runs:
  using: composite
  steps:
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '9.0.x'
    - run: dotnet restore
      shell: bash
```

```yaml
# using it in a workflow
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-and-restore
```

Composite actions bundle a reusable *sequence of steps* (rather than an entire job/workflow) into a single referenceable action — a good fit for a common setup pattern (like "checkout, setup .NET, restore") repeated across many jobs or workflows within the same repository.

---

## 10. Deployment Patterns

### Deploying to Azure App Service

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-package
          path: ./publish

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: my-api
          package: ./publish
```

### Deploying to AWS ECS

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
    aws-region: us-east-1

- name: Build and push image
  run: |
    docker build -t product-api .
    aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
    docker tag product-api:latest ${{ secrets.ECR_REGISTRY }}/product-api:${{ github.sha }}
    docker push ${{ secrets.ECR_REGISTRY }}/product-api:${{ github.sha }}

- name: Update ECS service
  run: aws ecs update-service --cluster my-cluster --service product-api-service --force-new-deployment
```

### Deploying to a Kubernetes cluster (AKS)

```yaml
- uses: azure/aks-set-context@v4
  with:
    resource-group: my-rg
    cluster-name: my-cluster

- run: kubectl set image deployment/product-api product-api=myregistry.azurecr.io/product-api:${{ github.sha }}
```

These follow directly from the deployment mechanics covered in this series' Azure Compute and AWS Compute guides — GitHub Actions is the automation layer triggering the same `az webapp deploy`, ECS service update, or `kubectl` commands you'd otherwise run by hand, but consistently, on every merge, without manual intervention.

---

## 11. Environments and Approval Gates

```yaml
jobs:
  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: echo "Deploying to production"
```

A GitHub **Environment** (configured under repository Settings → Environments) can require manual approval from designated reviewers before a job targeting it proceeds, restrict which branches can deploy to it, and hold environment-specific secrets scoped only to that environment — a production environment might require a lead engineer's explicit approval, while a staging environment deploys automatically with no gate at all.

```yaml
environment: production
```

This single line is what triggers GitHub to check the environment's configured protection rules — if approval is required, the job pauses at that point and waits for a reviewer to approve it directly in the GitHub UI before continuing, giving a genuine human gate on production deployments without needing a separate external tool.

---

## 12. Security Considerations

### Pin actions to a specific version, ideally a commit SHA

```yaml
# ✅ Safer: pinned to a specific commit SHA
- uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3

# Less safe: a mutable tag that could be repointed to different code later
- uses: actions/checkout@v4
```

Using a floating tag (`@v4`) trusts that the action's maintainer never republishes malicious code under that same tag — pinning to an exact commit SHA is the most secure option for security-sensitive pipelines, since a SHA reference can't be silently changed after the fact, though it does mean manually updating the SHA to pick up legitimate updates.

### Least-privilege `GITHUB_TOKEN` permissions

```yaml
permissions:
  contents: read
  pull-requests: write
```

The automatically-provided `GITHUB_TOKEN` defaults to fairly broad permissions in many repository configurations — explicitly declaring only the specific permissions a workflow actually needs (read-only repo content, write access to pull request comments, but nothing else) limits the blast radius if a workflow or one of its dependencies is ever compromised.

### Be cautious with `pull_request_target` and untrusted forks

```yaml
# pull_request_target runs with the base repo's permissions and secrets,
# even for PRs from forks — dangerous if it also checks out and runs the fork's code
on: pull_request_target
```

`pull_request_target` (unlike plain `pull_request`) runs with access to repository secrets even for pull requests originating from external forks — this is sometimes necessary (e.g., to comment on a PR from a fork), but combining it with checking out and executing the fork's own code is a well-known way to accidentally hand secrets to an untrusted contributor's code; if this combination is genuinely needed, the code that runs with secret access should be reviewed and controlled by the base repository, not blindly pulled from the fork.

### Avoid injecting untrusted input directly into `run` commands

```yaml
# ❌ Vulnerable to script injection via a maliciously crafted PR title
- run: echo "Building PR: ${{ github.event.pull_request.title }}"

# ✅ Safer: pass untrusted values as an environment variable instead
- run: echo "Building PR: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

Directly interpolating untrusted, user-controllable values (a PR title, an issue body, a branch name) into a `run` step's shell command is a genuine script-injection vector — a maliciously crafted PR title containing shell metacharacters could execute arbitrary commands in the runner. Passing the same value through an environment variable instead avoids this, since the shell no longer directly interpolates the untrusted string into the command itself.

---

## Quick Reference Table

| Concept | Purpose |
|---|---|
| Workflow / job / step / action | The nested structure of a GitHub Actions pipeline |
| `on:` triggers | What starts a workflow run (push, PR, schedule, manual, etc.) |
| `needs:` | Sequences jobs, waiting on a dependency's success |
| `runs-on:` | Selects the runner OS/environment for a job |
| `secrets` context | Encrypted, redacted-in-logs credential access |
| OIDC federation | Short-lived cloud credentials without storing long-lived secrets |
| `strategy: matrix` | Runs the same job across multiple parameter combinations |
| `actions/cache` | Persists dependencies between runs to speed up builds |
| `workflow_call` | Reusable, shareable entire workflow definitions |
| Composite action | Reusable, shareable sequence of steps |
| Environments | Approval gates, branch restrictions, and scoped secrets per deployment target |
| Pinning to a commit SHA | Protects against a compromised or repointed action tag |

---

## Conclusion

GitHub Actions' core value is proximity: your CI/CD pipeline lives in the same repository as your code, versioned and reviewed the same way, with no separate system to provision or maintain, and a vast marketplace of ready-made actions covering nearly every common automation need. From a five-line "build and test on every push" workflow to a full multi-environment deployment pipeline with approval gates and OIDC-federated cloud credentials, the underlying model stays the same: events trigger workflows, workflows run jobs on runners, and jobs execute steps — simple building blocks that compose into pipelines as sophisticated as a project actually needs.

The security practices matter as much as the pipeline mechanics — pin actions to trusted versions, scope `GITHUB_TOKEN` permissions tightly, prefer OIDC over long-lived cloud secrets, and be deliberate about how untrusted input (PR titles, fork contents) flows through a workflow — since a CI/CD pipeline with broad permissions and access to production secrets is exactly the kind of high-value target worth defending carefully, not an afterthought bolted onto the "just get the build green" goal.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the caching trick that took your build from ten minutes down to two.*
