# AHR Consulting Agent Skills

Reusable, versioned guidance for working with AHR Consulting libraries.

Each skill follows the [Agent Skills specification](https://agentskills.io/specification).
The canonical skill content lives once under `plugins/<plugin>/skills/` and is consumed directly
by compatible agents. Claude Code metadata is intentionally limited to thin marketplace and plugin
manifests so the instructions cannot drift between tools.

## Available skills

| Skill | Purpose |
|---|---|
| `ahr-foundation` | Use `Ahr.Foundation` results, options, analyzers, and composition APIs correctly. |

## GitHub Copilot and compatible agents

Install the skill directory using the mechanism supported by your agent, or copy:

```text
plugins/ahr-foundation/skills/ahr-foundation
```

into the agent's skills directory. The folder contains the standard `SKILL.md` entry point.

## Claude Code

Add this repository as a marketplace, then install the plugin:

```text
/plugin marketplace add AHRConsulting/agent-skills
/plugin install ahr-foundation@ahr-consulting
```

For local validation before publication:

```bash
claude --plugin-dir ./plugins/ahr-foundation
```

The Claude Code invocation name is `/ahr-foundation:ahr-foundation`.

## Scope

This repository contains reusable instructions and supporting resources only. MCP servers, product
source code, and private project context belong in their own repositories.
