---
title: Prerequisites
description: Everything you need before installing the Xianix Agent.
---

Before installing the Xianix Agent, make sure you have the following accounts and tools in place.

## Accounts

| Account | Required for |
|---|---|
| [Xians](https://xians.ai) | Agent control plane self-hosted or 99x's [Agentri](https://agentri.ai/) account |
| [Anthropic](https://console.anthropic.com) | API key for Claude (LLM) |
| GitHub and/or Azure DevOps | Repo admin access to configure access and wekhooks |

## Tokens & Keys

Collect these before running setup:

| Credential | Where to get it |
|---|---|
| **Xians API key** | Agent Studio → Settings in [app.xians.ai](https://app.xians.ai) |
| **Anthropic API key** | [console.anthropic.com/keys](https://console.anthropic.com/keys) |
| **GitHub PAT** | [github.com/settings/tokens](https://github.com/settings/tokens) — scopes: `repo`, `read:org` |
| **Azure DevOps PAT** | `https://dev.azure.com/<org>/_usersSettings/tokens` — scopes: `Code (Read & Write)`, `Pull Request Threads (Read & Write)` |

You only need a GitHub token **or** an Azure DevOps token depending on which platform you're connecting to. You can add both if you need to support both platforms.

## Docker Access

The agent needs access to the Docker socket to spawn executor containers. On Linux/macOS, ensure the user running the agent is in the `docker` group, or use `sudo`:

```bash
sudo usermod -aG docker $USER
```

On Docker Desktop (macOS/Windows), Docker socket access is available by default.
