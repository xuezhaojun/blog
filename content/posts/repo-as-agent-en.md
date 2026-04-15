---
title: "Repo-as-Agent: Building AI Agents with Git Repositories"
date: 2026-04-15
draft: false
tags: ["AI Agent", "Claude Code", "DevOps", "Software Engineering", "English"]
summary: "What if an AI Agent's identity, skills, and knowledge were all just files in a Git repository? This post introduces a pattern we've used in practice — no new frameworks, just directories, files, and Git to build production-grade AI Agents."
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## Where Does an Agent Actually Live?

When people say "build an AI Agent," what usually comes to mind is: a framework, a protocol, an orchestrator, a vector database, a plugin system. Before any real work begins, there's already a mountain of infrastructure.

But what if the answer could be far simpler?

**A Git repository + an AI coding assistant (Claude Code, Cursor, etc.) = a complete Agent.**

No new frameworks. No new protocols. Directories, files, Git — the most familiar tools every engineer already uses. The repository *is* the Agent. Its identity, skills, knowledge, and workflows all exist as files, version-controlled with Git, and reviewed through the same code review process as any other code.

## Repository Structure

Here's what an actual Agent repository looks like:

```
my-team-agent/
├── CLAUDE.md                  # Agent identity and behavioral rules
├── skills/                    # 20+ reusable task definitions
│   ├── bug-analyze/
│   ├── workspace-clone/
│   ├── jira-triage/
│   └── ...
├── workflows/                 # Scheduled or triggered multi-step processes
│   ├── daily-bug-triage.md
│   ├── daily-standup-prep.md
│   └── weekly-pr-report.md
├── solutions/                 # Lessons learned — known issues and solutions
├── repos/                     # 20+ related projects (read-only references)
├── team-members/              # Team roster and component ownership
├── docs/                      # Reference documents, loaded on demand
├── build/
│   └── Dockerfile             # Reproducible execution environment
└── deploy/                    # Kubernetes deployment manifests
```

The `CLAUDE.md` at the root is the entry point. When an AI assistant opens this repository, it reads this file first to understand who it is, what it can do, and where its knowledge lives. All subsequent interactions are built on top of this context.

The key point here: **everything is auditable**. Who added a skill? When was a workflow changed? Why was a solution deprecated? `git log` and `git blame` can answer all these questions. Team members review Agent changes exactly the same way they review code — through Pull Requests.

## Cross-Repository Visibility

In real enterprise environments, almost no task exists within a single repository. A bug might surface at the API gateway layer, but the root cause is in the authentication service, and the fix requires changes to a shared SDK. If the Agent can only see one repository at a time, it simply cannot perform end-to-end problem analysis.

The `repos/` directory solves this. It contains shallow clones of 20+ related projects that the Agent can read but won't modify:

```yaml
# repos/repos.yaml
categories:
  core:                    # Core projects the team directly owns
    - api-server
    - auth-service
    - sdk-go
    - cli-tool
  dependencies:            # Upstream dependencies with custom patches
    - network-proxy
    - grpc-fork
  build:                   # Build and CI configuration
    - ci-config
    - release-pipeline
  cross-team:              # Cross-team shared components
    - monitoring-stack
    - notification-service
```

With this global view, the Agent can trace call chains across repositories, understand dependency relationships, and analyze how changes in one project affect others. It sees the same code landscape that a senior engineer carries in their head — except it can `grep` through all of it in seconds.

## Git Worktree: Parallel Task Isolation

When tasks are executed end-to-end — from analysis to coding to testing to creating a PR — each task takes a long time. You can't execute them sequentially, sitting around waiting for one to finish before starting the next. But if two tasks modify the same repository simultaneously, they'll conflict with each other.

Git Worktree is the native solution. Starting from a bare clone, you can create multiple working directories, each on its own branch:

```bash
# Task 1: Fix a bug in the auth service
git worktree add ../workspace/auth-fix-1234 bugfix/auth-1234

# Task 2: Add a new feature to the same auth service
git worktree add ../workspace/auth-feature-5678 feature/oauth-5678
```

Each task operates in its own directory and branch, completely independent. After the task is done and the PR is merged, the worktree is automatically cleaned up.

This pattern is already built into the Agent's workspace management skill. When the Agent receives a task, it automatically creates an isolated worktree, works inside it, and cleans up when finished. Multiple tasks on the same repository can truly run in parallel.

## Team and Project Knowledge

Being able to read code isn't enough. To complete end-to-end tasks, the Agent also needs to understand the organization and processes behind the code.

Imagine this scenario: the Agent analyzes a bug, determines it belongs to the import controller component, writes a fix, creates a PR, and then... needs to notify the responsible person on Slack. Who owns the import controller? What's their Slack ID? Which channel should the message go to?

The `team-members/` directory answers these questions:

```markdown
## Core Team

| Name       | GitHub       | Email              | Components                        |
|------------|-------------|--------------------|-----------------------------------|
| Alice Chen | alice-c     | alice@example.com  | api-server, sdk-go                |
| Bob Park   | bob-park    | bob@example.com    | auth-service, cluster-proxy       |
| Carol Wu   | carol-wu    | carol@example.com  | import-controller, lifecycle-mgr  |
```

Beyond team structure, the Agent also knows:
- **Project management processes** — how to create Jira tickets, which fields are required, how ticket statuses flow
- **Release strategies** — which branch maps to which product version, what the backport rules are
- **Communication channels** — where to post bug analysis reports, which channel gets the weekly summary

In short, this is everything you'd tell a new hire during onboarding — not just "how to write code," but "how we collaborate."

## Progressive Knowledge Loading

AI models have a hard limit on their Context Window — the amount of text they can process at once. If you stuff all documentation, all skill definitions, and the entire team roster in at startup, the Agent drowns in irrelevant information and actually performs worse.

The solution is **Progressive Disclosure**: only load the knowledge the current task requires.

Each skill file explicitly declares its document dependencies:

```markdown
## Reference Loading

| Document              | When to Load                          |
|-----------------------|---------------------------------------|
| team-members.md       | Always (for assignment lookups)       |
| docs/jira.md          | When creating or updating Jira tickets|
| docs/build-release.md | When the bug involves release branches|
| repos/repos.yaml      | When identifying which repo to analyze|
```

When the user says "show me today's new bugs," the Agent loads the bug handling workflow, team member mappings, and repository list — but skips the release strategy and CI configuration docs. When the task is "backport this fix to release-2.8," the Agent loads the branching strategy and version mapping — but skips the Jira API documentation.

Different tasks, different knowledge sets. The Context Window stays focused on what truly matters.

## Dockerfile: Reproducible Execution Environment

We hit a classic problem early on: the same workflow runs perfectly on one engineer's machine but breaks on another's. The reason is always environment inconsistency — different operating systems, different tool versions, a missing CLI tool.

Agent execution depends on multiple layers:

1. **OS and Shell** — macOS and Linux differ in path handling, command arguments, and even `sed` syntax
2. **Tool versions** — `kubectl`, `helm`, `jq`, `yq`, language runtimes — any version mismatch can cause hard-to-debug errors
3. **Agent runtime** — different AI coding tools (Claude Code, Cursor, OpenCode) have different capabilities and tool-calling behaviors

The Dockerfile locks everything down:

```dockerfile
FROM debian:bookworm-slim

# Core tools
RUN apt-get update && apt-get install -y \
    git jq curl wget openssl unzip make gcc g++

# Go toolchain (pinned version)
ENV GO_VERSION=1.24.4
RUN curl -fsSL https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz \
    | tar -C /usr/local -xz

# CLI tools (pinned versions)
RUN curl -LO "https://dl.k8s.io/release/v1.31.4/bin/linux/amd64/kubectl" && ...
RUN curl -fsSL https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz \
    | tar -xz && ...
RUN curl -LO https://github.com/mikefarah/yq/releases/download/v4.45.1/yq_linux_amd64 \
    && ...
```

Once the Dockerfile is checked into the repository, any engineer can build the exact same environment. No more "works on my machine" problems. The Agent runs inside a container with a fully deterministic toolchain, independent of the host machine.

## From Local Tool to Always-On Agent

Everything above works great as a local solution — clone the Agent repository, start the AI assistant, and get to work. But the natural next step is: **deploy the Agent to the cloud so it runs 24/7**.

With the Dockerfile standardizing the environment, you can build a container image and deploy it to a Kubernetes cluster. We built [KubeOpenCode](https://github.com/kubeopencode/kubeopencode), an AI Agent management platform on Kubernetes, to do exactly this. Once deployed, the Agent becomes a persistent service:

- **Always online** — no need to keep a terminal open on your laptop
- **Auto-scaling** — scales down when idle to save resources, wakes up automatically when tasks arrive
- **Live knowledge updates** — the Agent repository syncs from Git every few minutes; when a team member merges a new skill, the running Agent gains the new capability without a restart
- **Scheduled workflows** — cron-triggered tasks that don't require manual initiation

```yaml
# Example: Automatically triage new bugs at 9 AM on weekdays
apiVersion: kubeopencode.io/v1alpha1
kind: CronTask
metadata:
  name: daily-bug-triage
spec:
  schedule: "0 9 * * 1-5"    # Monday-Friday 9:00
  taskTemplate:
    spec:
      description: |
        Run the daily bug triage workflow.
        Collect new bugs, analyze relevance, assign owners,
        post summary to Slack.
```

With scheduled tasks, the Agent shifts from passive response to proactive work. It triages new bugs and assigns owners before the team's standup. It auto-generates a PR activity report every Friday. It cleans up expired bot PRs every Monday. What the team sees in the morning is no longer a pile of unsorted backlog, but an already-categorized, pre-analyzed task list.

Credentials (API tokens, Git authentication) are centrally managed in the cluster's Kubernetes Secrets — no more passing around `.env` files or having everyone configure things individually.

## The Core Formula

The entire pattern boils down to a simple formula:

```
Code Repository + AI Coding Assistant = Agent
```

The repository provides identity, skills, knowledge, and workflows. The AI assistant provides reasoning, execution, and natural language interaction. Together, they form an Agent with these properties:

- **Version-controlled** — every change is recorded, reviewable, and reversible
- **Auditable** — `git log` shows the complete history of the Agent's evolution
- **Collaborative** — team members contribute skills and knowledge through PRs
- **Portable** — works with any AI coding assistant that can read files
- **Reproducible** — Dockerfile ensures consistent behavior across environments
- **Scalable** — deploy to Kubernetes for 24/7 operation and scheduled tasks

No vendor lock-in to a specific Agent framework. No proprietary plugin formats. No complex orchestration layer. Just files, Git, and the engineering practices you use every day.

## Getting Started

If you want to try this pattern:

1. **Create a repository** with a `CLAUDE.md` (or similar entry file) describing the Agent's role and rules
2. **Add a `skills/` directory** and write one or two task definitions — start with work the team does repeatedly
3. **Add a `docs/` directory** with reference materials the Agent needs — team contacts, project specs, API docs
4. **Point your AI assistant at this repository** and start assigning it tasks

No Kubernetes, no Docker, no infrastructure needed to get started. A local repository plus an AI coding tool is all it takes. The infrastructure layer (Dockerfile, cloud deployment, scheduled tasks) can be added incrementally as needs grow.

The best part? When a teammate asks "what can this Agent do?" — just send them the repo link. All the answers are in the files.
