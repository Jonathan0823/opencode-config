# OpenCode AI Configuration Pack

Control your AI patterns. Get repeatable results.

This repo is a ready-to-use OpenCode config built around the OpenAgents Control workflow: agents that learn your patterns, load only the context they need, and pause for approval before execution.

## Why it helps

- **Pattern control** — teach agents your standards once, reuse them everywhere
- **Approval gates** — review plans before anything changes
- **MVI context loading** — load only what matters, when it matters
- **Editable agents** — prompts are markdown, not baked-in black boxes
- **Team-ready** — share the same config and context across your team
- **Live docs** — use Context7 for up-to-date library and framework docs

## What’s included

- **Agents** — OpenAgent, OpenCoder, and specialist subagents
- **Commands** — context, validation, commit, cleanup, and workflow helpers
- **Context** — standards, workflows, and project intelligence
- **Skills** — reusable workflows for frontend, backend, security, DevOps, docs, and more
- **Runtime config** — OpenCode permissions and MCP settings

## Example use cases

- Generate code that matches your API and component conventions
- Review a change for security, maintainability, and consistency
- Pull current docs for an external library before implementing
- Turn project patterns into shared context your team can reuse

## How to use it

- **Project-local**: place the config in `.opencode/`
- **Global**: place the config in `~/.config/opencode/`
- OpenCode will load the nearest config automatically

## Repository layout

```text
agent/
command/
config/
context/
skills/
opencode.json
README.md
```

## Quick examples

```text
@code-reviewer review this change for security issues
@researcher find the current docs for this library
@implementer build this feature using our patterns
```

## Attribution

Powered by [OpenAgents Control (OAC)](https://github.com/darrenhinde/OpenAgentsControl) by Darren Hinde.
