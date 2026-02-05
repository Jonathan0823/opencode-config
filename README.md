# OpenCode AI Configuration Repository

This repository defines AI agents and skills for enhancing code review workflows using OpenCode AI.

## Overview

OpenCode-config provides:
- A **code reviewer AI agent** focused on security, performance, and maintainability
- **Git workflow skills** (commit message standards and release management)
- **Interaction standards** for AI agents

## Project Structure

```
opencode-config/
├── .git/                 # Git repository
├── .gitignore           # Git ignore rules
├── AGENTS.md            # AI agent interaction standards
├── bun.lock             # Bun lock file
├── opencode.json        # Main configuration file
├── package.json         # Node.js dependencies
└── skills/              # AI skills directory
    ├── git-commit/      # Commit message skill
    │   └── SKILL.md    # Skill documentation
    └── git-release/     # Release management skill
        └── SKILL.md    # Skill documentation
```

## Features

### AI Agents

#### Code Reviewer Agent
- Mode: Subagent with read-only permissions
- Focus areas: Security, performance, and maintainability
- Uses OpenCode AI SDK for integration

### Skills System

#### Git Commit Skill
Enforces conventional commit message standards:
- Format: `<type>(<scope>): <subject>`
- Types: chore, feat, fix, docs, style, refactor, test, perf, ci
- Promotes single-responsibility commits
- Includes linting and testing guidelines

#### Git Release Skill
- Drafts release notes from merged PRs
- Proposes version bumps
- Provides copy-pasteable release creation commands

## Technology Stack

- **Runtime**: Node.js/TypeScript
- **Package Manager**: Bun
- **Dependencies**:
  - `@opencode-ai/plugin: 1.1.51` (core integration)
  - `zod` (validation library)
- **Configuration Format**: JSON for agent definitions, YAML for skill metadata

## Installation

```bash
# Clone the repository
git clone https://github.com/Jonathan0823/opencode-config.git
cd opencode-config

# Install dependencies
bun install
```

## Usage

This is a **configuration repository** - it doesn't have a traditional "run" command. The configurations are used by:

1. **OpenCode AI Platform**: Load the `opencode.json` to activate defined agents
2. **Skills Integration**: Use the skills directory for Git workflow automation
3. **Agent Configuration**: Apply AGENTS.md standards to AI interactions

## Agent Standards

This project follows established interaction standards:

- **Clarity and Transparency**: Clear communication of capabilities
- **User-Centric Design**: Prioritizes user needs with customization options
- **Reviewing & Debugging**: Detailed error explanations and troubleshooting

## License

Skills are MIT licensed. See individual SKILL.md files for details.

## Repository

- **URL**: https://github.com/Jonathan0823/opencode-config.git
