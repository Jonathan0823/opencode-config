# OpenCode AI Configuration Repository

AI agent configurations and skills for enhancing software development workflows using OpenCode AI. This repository provides specialized subagents for code review, research, refactoring, documentation, and feature implementation.

## Features

### 🤖 AI Subagents

- **Code Reviewer** (`@code-reviewer`) - Reviews code for security, performance, and maintainability issues
- **Researcher** (`@researcher`) - Investigates libraries, frameworks, patterns, and technical solutions  
- **Refactorer** (`@refactorer`) - Safely refactors code following best practices and language conventions
- **Explainer** (`@explainer`) - Explains code concepts and generates comprehensive documentation
- **Implementer** (`@implementer`) - Implements features end-to-end following development workflows

### 🛠️ Development Skills

- **Git Workflow Skills**
  - Conventional commit message standards
  - Automated release management and changelog generation
  - Git workflow and branching strategies

- **Language-Specific Skills**
  - Go conventions and idiomatic patterns
  - Rust patterns and ownership best practices
  - TypeScript/React/Next.js and Vue/Svelte patterns

- **Domain Skills**
  - API REST design patterns and best practices
  - SQL Server, PostgreSQL, and MongoDB patterns
  - Testing strategies and patterns
  - Feature development workflows
  - Bug fixing methodologies
  - Safe refactoring practices
  - Code documentation standards

## Installation

```bash
# Clone the repository
git clone https://github.com/Jonathan0823/opencode-config.git
cd opencode-config

# Install dependencies
bun install
```

### Prerequisites

- [Node.js](https://nodejs.org/) 18+ or [Bun](https://bun.sh/) runtime
- OpenCode AI platform access
- Context7 API key (for documentation lookup)

## Usage

This is a **configuration repository** used by the OpenCode AI platform. The configurations are loaded automatically when using the OpenCode AI CLI or IDE extensions.

### Loading Configuration

```bash
# The opencode.json configuration is automatically loaded
# when running OpenCode AI in this directory
opencode
```

### Using Subagents

Once loaded, invoke subagents using the `@` syntax:

```
# Code review
@code-reviewer Please review src/auth/service.ts

# Research
@researcher What's the best way to implement authentication in Go?

# Refactoring
@refactorer Refactor this to be more maintainable

# Documentation
@explainer Explain how this authentication system works

# Feature implementation
@implementer Build a user authentication system
```

### Using Skills

Skills are automatically available when working with specific technologies:

```
# When working with Go files, Go conventions skill is loaded
# When working with Git, Git commit/release skills are loaded
# When working with databases, appropriate patterns skills are loaded
```

## Architecture

### Project Structure

```
opencode-config/
├── opencode.json          # Main agent configuration
├── AGENTS.md              # AI agent interaction standards
├── package.json           # Node.js dependencies
├── bun.lock               # Bun lock file
└── skills/                # AI skills directory
    ├── git-commit/        # Conventional commits skill
    ├── git-release/       # Release management skill
    ├── git-team-workflow/ # Git workflow patterns
    ├── go-conventions/    # Go language patterns
    ├── rust-patterns/     # Rust language patterns
    ├── ts-react-nextjs/   # TypeScript/React patterns
    ├── ts-vue-svelte/     # TypeScript/Vue/Svelte patterns
    ├── api-rest-design/   # API design patterns
    ├── postgresql-patterns/ # PostgreSQL best practices
    ├── sqlserver-patterns/  # SQL Server patterns
    ├── mongodb-patterns/    # MongoDB patterns
    ├── testing-strategies/  # Testing patterns
    ├── feature-development/ # Feature development workflow
    ├── bug-fixing/          # Bug fixing methodologies
    ├── refactoring-safely/  # Safe refactoring practices
    └── code-documentation/  # Documentation standards
```

### Configuration Overview

See [AGENTS.md](./AGENTS.md) for detailed information on:
- Interaction standards for AI agents
- Subagent usage guidelines
- Code quality standards
- Development workflows
- Git workflows and branching strategies

The main configuration file `opencode.json` defines:
- **5 specialized subagents** with specific roles and permissions
- **MCP integration** with Context7 for documentation lookup
- **Permission models** for each subagent (read-only, read-write, etc.)

### Agent Capabilities

| Agent | Mode | Permissions | Best For |
|-------|------|-------------|----------|
| `@code-reviewer` | Subagent | Read-only | Security, performance, maintainability reviews |
| `@researcher` | Subagent | Read-only | Library research, pattern investigation |
| `@refactorer` | Subagent | Read-write | Safe code refactoring |
| `@explainer` | Subagent | Read-write | Documentation generation |
| `@implementer` | Subagent | Read-write | Feature implementation |

## Skills Reference

### Git Workflow Skills

- **[git-commit](./skills/git-commit/SKILL.md)** - Conventional commit format: `<type>(<scope>): <subject>`
- **[git-release](./skills/git-release/SKILL.md)** - Automated release notes and version bumping
- **[git-team-workflow](./skills/git-team-workflow/SKILL.md)** - Branching strategies and collaboration patterns

### Language Skills

- **[go-conventions](./skills/go-conventions/SKILL.md)** - Go idioms, interfaces, error handling
- **[rust-patterns](./skills/rust-patterns/SKILL.md)** - Ownership, borrowing, Result/Option types
- **[ts-react-nextjs](./skills/ts-react-nextjs/SKILL.md)** - TypeScript, React 19, Next.js 15 patterns
- **[ts-vue-svelte](./skills/ts-vue-svelte/SKILL.md)** - TypeScript, Vue 3, Svelte 5 patterns

### Domain Skills

- **[api-rest-design](./skills/api-rest-design/SKILL.md)** - RESTful API patterns and versioning
- **[postgresql-patterns](./skills/postgresql-patterns/SKILL.md)** - PostgreSQL optimization and best practices
- **[sqlserver-patterns](./skills/sqlserver-patterns/SKILL.md)** - SQL Server T-SQL patterns
- **[mongodb-patterns](./skills/mongodb-patterns/SKILL.md)** - MongoDB schema design and queries
- **[testing-strategies](./skills/testing-strategies/SKILL.md)** - Testing patterns for production code
- **[feature-development](./skills/feature-development/SKILL.md)** - End-to-end feature development workflow
- **[bug-fixing](./skills/bug-fixing/SKILL.md)** - Systematic debugging and bug fixing
- **[refactoring-safely](./skills/refactoring-safely/SKILL.md)** - Safe refactoring with tests
- **[code-documentation](./skills/code-documentation/SKILL.md)** - Documentation standards and best practices

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Follow the existing skill structure and documentation format
2. Ensure all skills include practical examples
3. Update relevant documentation when adding new features
4. Test configurations before submitting

For detailed contribution guidelines, please open an issue to discuss proposed changes.

## License

Individual skills are MIT licensed. See individual `SKILL.md` files for specific licensing details.

## Related Resources

- [OpenCode AI Documentation](https://opencode.ai/docs)
- [AGENTS.md](./AGENTS.md) - Complete agent interaction standards
- [Context7 MCP](https://mcp.context7.com/) - Documentation lookup service

---

**Repository**: https://github.com/Jonathan0823/opencode-config.git
