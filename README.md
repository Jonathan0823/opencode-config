# OpenCode AI Configuration Repository

Portable configuration pack for OpenCode: agent prompts, slash commands, context docs, reusable skills, runtime config, and metadata.

> Inspired by [OpenAgents Control (OAC)](https://github.com/darrenhinde/OpenAgentsControl) by Darren Hinde.

## What’s Included

- `agent/` — core agents and subagents
- `command/` — slash commands
- `context/` — standards, workflows, and project intelligence
- `skills/` — reusable skill modules
- `config/agent-metadata.json` — registry/installer metadata
- `opencode.json` — runtime permissions and MCP config
- `install.sh` — installer/bootstrap script

## Quick Start

### Global config

```bash
mkdir -p ~/.config/opencode
cp -r agent command context skills config opencode.json ~/.config/opencode/
```

### Project-local config

```bash
mkdir -p .opencode
cp -r agent command context skills config opencode.json .opencode/
```

## Using It

- Use agents with `@code-reviewer`, `@researcher`, `@refactorer`, `@explainer`, `@implementer`
- Use skills with `@skill-name`
- OpenCode reads config from the local project first, then global config

## Repository Structure

```
.
├── agent/
├── command/
├── config/
├── context/
├── install.sh
├── opencode.json
├── package.json
└── README.md
```

## Configuration Notes

- `opencode.json` controls permissions and the Context7 MCP connection
- `config/agent-metadata.json` stores metadata used by the registry/install flow
- `context/navigation.md` is the entry point for context discovery

## Attribution

OpenAgents Control (OAC) and its installer/workflow are created by Darren Hinde. This repository adapts that foundation into a portable OpenCode configuration source.

## Related Resources

- [OpenCode AI Documentation](https://opencode.ai/docs)
- [Context7 MCP](https://mcp.context7.com/)
