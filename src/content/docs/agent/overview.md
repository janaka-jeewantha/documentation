---
title: Overview
description: What the Xianix Agent is and how it works end-to-end.
---

The Xianix Agent is a long-running .NET console application that connects your code platform (GitHub, Azure DevOps) to AI-powered automation. It listens for webhook events, evaluates rules, and orchestrates isolated Docker containers to run Claude Code plugins against your repositories.

## System Architecture

```
GitHub / Azure DevOps
        │ webhook
        ▼
  Xians Webhooks
  (app.xians.ai)
        │ dispatches task
        ▼
    TheAgent
  (.NET Console App)
        │ spawns & mounts socket
        ▼
     Executor
  (Docker container)
        │ runs
        ▼
 ClaudeCode Plugins
 (review, triage…)
```

| Layer | What it does |
|---|---|
| **GitHub / Azure DevOps** | Source of events — PR opened, comment posted, etc. |
| **Xians Webhooks** | Receives platform events and routes them to the registered agent. |
| **TheAgent** | Long-running .NET process that polls Xians, interprets tasks, and orchestrates execution. |
| **Executor** | Isolated Docker container spawned per task; has no persistent state. |
| **ClaudeCode Plugins** | Skills (e.g. PR review, requirement analysis) that run inside the Executor and call the LLM. |

## How an Event Flows

1. A webhook arrives at Xians from GitHub or Azure DevOps (e.g. a PR is opened).
2. The agent's `WebhookWorkflow` receives the event and passes it to `EventOrchestrator`.
3. `RulesEvaluator` matches the event against `rules.json`, extracts inputs, and identifies the plugin to run.
4. A `ProcessingWorkflow` is signalled — it creates an isolated workspace, clones the repository, installs the plugin, and invokes it.
5. The plugin runs inside an ephemeral Docker container and posts results back to the platform.

## Repositories

| Repo | Description |
|---|---|
| `the-agent` | .NET agent to deploy on Xians, workflows, orchestrator, rules engine |
| `plugins-official` | Official Claude Code plugins maintained by the 99x / Xianix team |
