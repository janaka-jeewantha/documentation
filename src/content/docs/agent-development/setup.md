---
title: Setup
description: How to configure and run the Xianix Agent locally or in Docker.
---

## Prerequisites

- .NET SDK (see `global.json` for the pinned version)
- Docker (for running executor containers)
- A Xians account and API key ([app.xians.ai](https://app.xians.ai))
- An LLM API key (Anthropic)
- A GitHub or Azure DevOps token

## Configuration

Copy `.env.example` to `.env` and fill in all required values:

```bash
cp .env.example .env
```

Key variables:

| Variable | Description |
|---|---|
| `XIANS_SERVER_URL` | Xians platform URL, e.g. `https://app.xians.ai` |
| `XIANS_API_KEY` | Your Xians API key |
| `LLM_API_KEY` | Anthropic API key for Claude |
| `GITHUB_TOKEN` | GitHub PAT (required if using GitHub webhooks) |
| `AZURE_DEVOPS_TOKEN` | Azure DevOps PAT (required if using Azure DevOps webhooks) |
| `EXECUTOR_IMAGE` | Docker image for the executor, default `99xio/xianix-executor:latest` |

See `.env.example` in the repo for the full list.

## Running Locally

```bash
dotnet run --project TheAgent/TheAgent.csproj
```

A healthy agent prints 4 workflow queues and starts listening:

```
│ REGISTERED WORKFLOWS (4)                                      │
  Agent:    Xianix AI-DLC Agent
  Workflow: Xianix AI-DLC Agent:Activation Workflow
  ...
✓ Worker listening on queue 'xianix:...:Processing Workflow'
✓ Worker listening on queue 'xianix:...:Activation Workflow'
✓ Worker listening on queue 'xianix:...:Supervisor Workflow'
✓ Worker listening on queue 'xianix:...:Integrator Workflow'
```

## Running in Docker

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --env-file .env \
  99xio/xianix-agent:latest
```

The Docker socket mount is required — the agent spawns executor containers via the Docker API.

## Running Tests

```bash
dotnet test TheAgent.Tests/TheAgent.Tests.csproj
```

## Simulating Webhooks

With the agent running, fire simulated GitHub webhook events using the scripts in `Scripts/`:

```bash
export WEBHOOK_URL=https://app.xians.ai/webhooks/<your-agent-id>
./Scripts/simulate-pr-opened.sh    # should respond { "status": "success" }
./Scripts/simulate-pr-closed.sh    # should respond { "status": "ignored" }
```

## Building Docker Images

### Executor

```bash
cd Executor/
docker build -t 99xio/xianix-executor:latest .
```

### TheAgent

```bash
cd TheAgent/
docker build -t 99xio/xianix-agent:latest .
```

### Publishing

Both images are published automatically when you push a version tag:

```bash
VERSION=v1.0.0
git tag $VERSION
git push origin $VERSION
```

Tags are derived from the version (e.g. `v1.2.3` produces `1.2.3`, `1.2`, `1`, and `latest`). Pre-release tags (e.g. `v1.0.0-rc1`) skip the `latest` tag.
