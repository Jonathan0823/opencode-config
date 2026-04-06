# OpenCode AI Configuration Repository

This repository is a portable configuration source for OpenCode, providing specialized subagents for code review, research, refactoring, documentation, and feature implementation, plus **28 specialized skills** covering development workflows, security, DevOps, databases, and best practices.

> Inspired by [OpenAgents Control (OAC)](https://github.com/darrenhinde/OpenAgentsControl) by Darren Hinde.

## Features

### 🤖 AI Subagents

- **Code Reviewer** (`@code-reviewer`) - Reviews code for security, performance, and maintainability issues
- **Researcher** (`@researcher`) - Investigates libraries, frameworks, patterns, and technical solutions  
- **Refactorer** (`@refactorer`) - Safely refactors code following best practices and language conventions
- **Explainer** (`@explainer`) - Explains code concepts and generates comprehensive documentation
- **Implementer** (`@implementer`) - Implements features end-to-end following development workflows

### 🛠️ Development Skills (28 Total)

#### 🔧 Development Workflow (6 skills)
- **Git Workflow** - Conventional commits, automated releases, branching strategies
- **Feature Development** - End-to-end workflow from research to deployment
- **Bug Fixing** - Systematic debugging and fixing methodologies
- **Refactoring** - Safe refactoring practices with tests
- **Code Documentation** - Documentation standards and patterns
- **Testing** - Testing strategies for production code

#### 🎨 Frontend (3 skills)
- **Frontend Design** - Production-grade UI components and interfaces
- **TypeScript/React/Next.js** - Modern React 19 and Next.js 15 patterns
- **TypeScript/Vue/Svelte** - Vue 3 and Svelte 5 patterns

#### 💻 Backend Languages (3 skills)
- **Go Conventions** - Idiomatic Go patterns and best practices
- **Python Patterns** - Python with type hints and async/await
- **Rust Patterns** - Ownership, lifetimes, and safe Rust code

#### 🗄️ Database (3 skills)
- **PostgreSQL** - Patterns, optimization, and best practices
- **MongoDB** - Schema design and query optimization
- **SQL Server** - T-SQL patterns and best practices

#### 🔒 Security & Best Practices (4 skills)
- **Security Best Practices** - OWASP guidelines, secure coding, vulnerability prevention
- **API REST Design** - RESTful API design patterns and versioning
- **Docker Patterns** - Container best practices and multi-stage builds
- **Code Documentation** - Documentation standards and patterns

#### 🚀 DevOps & Infrastructure (5 skills)
- **CI/CD Pipelines** - GitHub Actions workflows and deployment strategies
- **Kubernetes Patterns** - K8s deployment, networking, and security
- **Performance Optimization** - Caching, profiling, database optimization
- **Observability & Monitoring** - Logging, metrics, distributed tracing
- **Microservices Patterns** - Service communication, resilience, event-driven

## 🚀 Quick Start

### Prerequisites

- **OpenCode AI CLI** must be installed separately. See [OpenCode AI installation guide](https://opencode.ai/docs/installation)
- *(Optional)* **Context7 API key** if you want to use the MCP documentation lookup integration

### Step 0: Clone This Repo

Start by cloning this repository. It already contains the agent, command, context, and skill structure.

```bash
git clone <this-repo-url>
cd <repo-name>
```

### Step 1: Use This Repo as the Config Source

After cloning, this repository becomes the source of truth for agents, commands, context, skills, and runtime config.

- `agent/`
- `command/`
- `context/`
- `skills/`
- `opencode.json`

### Setup

#### Option 1: Project-Specific Configuration

```bash
cp opencode.json /path/to/your/project/
cp -r skills/ /path/to/your/project/
```

#### Option 2: Global Configuration

```bash
mkdir -p ~/.config/opencode
cp opencode.json ~/.config/opencode/
cp -r skills/ ~/.config/opencode/
```

#### Option 3: Selective Copy

```bash
# Example: Copy only specific skills
cp -r skills/git-commit/ ~/.config/opencode/skills/
cp -r skills/docker-patterns/ ~/.config/opencode/skills/
cp -r skills/kubernetes-patterns/ ~/.config/opencode/skills/
```

### Configuration Discovery

OpenCode AI discovers configurations in this order:
1. **Current directory** (project-specific)
2. **Parent directories** (walking up)
3. **`~/.config/opencode/`** (global)

If you install OAC globally, this repo's files should live under that config root. If you use a project-local setup, keep the same folder structure inside your project.

```bash
# Run OpenCode AI - config loads automatically
opencode
```

## 📋 Complete Skills Catalog

### Using Skills

Invoke skills using the `@` syntax:

```
# Git workflow
@git-commit Help me create a commit message

# Security review
@security-best-practices Review this authentication code

# Containerization
@docker-patterns Create a Dockerfile for this Node.js app

# CI/CD
@ci-cd-pipelines Set up GitHub Actions for testing and deployment

# Kubernetes
@kubernetes-patterns Deploy this to K8s with proper resource limits

# Multiple skills
@docker-patterns @ci-cd-pipelines Containerize and set up CI/CD
```

### 🔧 Development Workflow Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [git-commit](skills/git-commit/SKILL.md) | Conventional commit format: `<type>(<scope>): <subject>` | Making commits |
| [git-release](skills/git-release/SKILL.md) | Automated release notes and version bumping | Preparing releases |
| [git-team-workflow](skills/git-team-workflow/SKILL.md) | Branching strategies and collaboration patterns | Git workflow setup |
| [feature-development](skills/feature-development/SKILL.md) | End-to-end feature development workflow | Building new features |
| [bug-fixing](skills/bug-fixing/SKILL.md) | Systematic debugging and bug fixing | Fixing bugs |
| [refactoring-safely](skills/refactoring-safely/SKILL.md) | Safe refactoring with tests | Refactoring code |

### 🎨 Frontend Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [frontend-design](skills/frontend-design/SKILL.md) | Production-grade UI components and interfaces | Building UIs |
| [ts-react-nextjs](skills/ts-react-nextjs/SKILL.md) | TypeScript, React 19, Next.js 15 patterns | React development |
| [ts-vue-svelte](skills/ts-vue-svelte/SKILL.md) | TypeScript, Vue 3, Svelte 5 patterns | Vue/Svelte development |

### 💻 Backend Language Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [go-conventions](skills/go-conventions/SKILL.md) | Go idioms, interfaces, error handling | Writing Go code |
| [python-patterns](skills/python-patterns/SKILL.md) | Python type hints, async/await, FastAPI/Django | Python development |
| [rust-patterns](skills/rust-patterns/SKILL.md) | Ownership, borrowing, Result/Option types | Writing Rust code |

### 🗄️ Database Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [postgresql-patterns](skills/postgresql-patterns/SKILL.md) | PostgreSQL optimization and best practices | PostgreSQL development |
| [sqlserver-patterns](skills/sqlserver-patterns/SKILL.md) | SQL Server T-SQL patterns | SQL Server development |
| [mongodb-patterns](skills/mongodb-patterns/SKILL.md) | MongoDB schema design and queries | MongoDB development |

### 🔒 Security & API Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [security-best-practices](skills/security-best-practices/SKILL.md) | OWASP guidelines, secure coding patterns | Implementing security |
| [api-rest-design](skills/api-rest-design/SKILL.md) | RESTful API patterns and versioning | Designing APIs |
| [docker-patterns](skills/docker-patterns/SKILL.md) | Container best practices, multi-stage builds | Containerizing apps |
| [testing-strategies](skills/testing-strategies/SKILL.md) | Testing patterns for production code | Writing tests |
| [code-documentation](skills/code-documentation/SKILL.md) | Documentation standards and patterns | Writing docs |

### 🚀 DevOps & Infrastructure Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [ci-cd-pipelines](skills/ci-cd-pipelines/SKILL.md) | GitHub Actions workflows and deployment strategies | Setting up CI/CD |
| [kubernetes-patterns](skills/kubernetes-patterns/SKILL.md) | K8s deployment, networking, security | Deploying to Kubernetes |
| [performance-optimization](skills/performance-optimization/SKILL.md) | Caching, profiling, database optimization | Optimizing performance |
| [observability-monitoring](skills/observability-monitoring/SKILL.md) | Logging, metrics, distributed tracing | Setting up monitoring |
| [microservices-patterns](skills/microservices-patterns/SKILL.md) | Service communication, resilience, event-driven | Building microservices |

### 🛠️ Meta Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| [skill-creator](skills/skill-creator/SKILL.md) | Guide for creating effective skills | Creating new skills |
| [context7](skills/context7/SKILL.md) | Context7 documentation lookup integration | Looking up library docs |
| [task-management](skills/task-management/SKILL.md) | Task CLI for tracking subtasks | Managing complex workflows |

## 📊 Stats

- **Total Skills:** 28
- **With Progressive Disclosure:** 6 (security, docker, ci-cd, kubernetes, performance, microservices, observability)
- **Average SKILL.md Size:** ~140 lines (optimized for context window)
- **Subagent Categories:** 3 (core, code, development)
- **Command Definitions:** 10+
- **License:** MIT (all skills)

## 🏗️ Architecture

### Repository Structure

```
.
├── agent/          # Core agent prompts and specialized subagents
├── command/        # Slash command definitions
├── commands/       # Additional command packs
├── context/        # Context system, standards, workflows, project intelligence
├── skills/         # Reusable skill modules
├── tool/           # Tooling utilities
├── opencode.json   # Main runtime configuration
├── AGENTS.md       # Agent behavior and interaction standards
└── README.md
```

### Skill Structure

```
skills/
├── git-commit/        # Git commit conventions
├── git-release/       # Release management
├── git-team-workflow/ # Git workflow patterns
├── go-conventions/    # Go language patterns
├── rust-patterns/     # Rust language patterns
├── ts-react-nextjs/   # React/Next.js patterns
├── ts-vue-svelte/     # Vue/Svelte patterns
├── frontend-design/   # UI/UX design
├── api-rest-design/   # API design patterns
├── security-best-practices/  # Security + references/
├── docker-patterns/   # Docker + references/
├── ci-cd-pipelines/   # CI/CD + references/
├── kubernetes-patterns/      # K8s + references/
├── performance-optimization/ # Performance + references/
├── microservices-patterns/   # Microservices + references/
├── observability-monitoring/ # Observability + references/
├── postgresql-patterns/      # PostgreSQL patterns
├── sqlserver-patterns/       # SQL Server patterns
├── mongodb-patterns/         # MongoDB patterns
├── testing-strategies/       # Testing patterns
├── feature-development/      # Feature development
├── bug-fixing/               # Bug fixing
├── refactoring-safely/       # Safe refactoring
├── code-documentation/       # Documentation
├── skill-creator/            # Skill creation guide
├── context7/                 # Context7 documentation lookup
└── task-management/          # Task management CLI
```

### Subagent Architecture

Subagent definitions are organized under `agent/subagents/` by purpose:

- `core/` - Context discovery (ContextScout), external docs (ExternalScout), task planning (TaskManager)
- `code/` - Coding (CoderAgent), review (CodeReviewer), testing (TestEngineer), build validation (BuildAgent)
- `development/` - Frontend and DevOps specialists

This structure enables targeted delegation for implementation, review, testing, and documentation tasks.

### Command System

Built-in slash commands under `command/` and `commands/`:

- **Context management**: `/context`, `/add-context`
- **Validation**: `/validate-repo`, `/test`
- **Workflows**: `/optimize`, `/clean`, `/commit`

### Context System

The `context/` directory contains reusable standards and project guidance:

- `core/standards/` - Code quality, documentation, testing rules
- `core/workflows/` - Delegation and execution workflows
- `project-intelligence/` - Project-specific conventions
- `openagents-repo/` - Repository-specific operational guidance

### Progressive Disclosure

Skills with `references/` folder use progressive disclosure:
1. **SKILL.md** (~140 lines) loads first - overview and quick reference
2. **references/** loaded on-demand - detailed documentation

Example:
```
security-best-practices/
├── SKILL.md                    # Overview (130 lines)
└── references/
    ├── owasp-top10.md          # Detailed OWASP guide
    ├── secure-coding-python.md # Python security
    ├── secure-coding-javascript.md # JS security
    ├── secure-coding-go.md     # Go security
    └── secrets-management.md   # Secrets management
```

### Configuration

See [AGENTS.md](./AGENTS.md) for detailed information on:
- Interaction standards for AI agents
- Subagent usage guidelines
- Code quality standards
- Development workflows

### Agent Capabilities

| Agent | Mode | Permissions | Best For |
|-------|------|-------------|----------|
| `@code-reviewer` | Subagent | Read-only | Security, performance reviews |
| `@researcher` | Subagent | Read-only | Library research, patterns |
| `@refactorer` | Subagent | Read-write | Safe code refactoring |
| `@explainer` | Subagent | Read-write | Documentation generation |
| `@implementer` | Subagent | Read-write | Feature implementation |

## 🔧 Configuration

Your `opencode.json` should include:

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "deny",
      "experimental-*": "ask"
    }
  }
}
```

This enables all skills by default, denies internal skills, and asks for experimental ones.

## 🔐 Security Notes

- Secrets are loaded from environment variables (see `.env.example`)
- Never commit `.env` files — they are excluded via `.gitignore`
- MCP/API keys should use the `{env:VAR_NAME}` syntax in configuration files
- Run `cp .env.example .env` and add your keys before using MCP integrations

## 🤝 Contributing

Contributions are welcome! Please follow these guidelines:

1. Follow the existing skill structure and documentation format
2. Include practical examples in all skills
3. Add `license: MIT` and `compatibility: opencode` to YAML frontmatter
4. Keep SKILL.md under 200 lines, move details to `references/`
5. Update this README when adding new skills
6. Test configurations before submitting

For detailed contribution guidelines, see [skill-creator](skills/skill-creator/SKILL.md).

## 📄 License

All skills in this collection are licensed under the MIT License. See individual `SKILL.md` files for specific licensing details.

## Attribution

OpenAgents Control (OAC) and its installer/workflow are created by Darren Hinde. This repository adapts that foundation into a portable configuration source for OpenCode users.

## 🔗 Related Resources

- [OpenCode AI Documentation](https://opencode.ai/docs)
- [AGENTS.md](./AGENTS.md) - Complete agent interaction standards
- [Context7 MCP](https://mcp.context7.com/) - Documentation lookup service
- [Skill Creator Guide](skills/skill-creator/SKILL.md) - Creating custom skills
- [ENV_SETUP.md](./ENV_SETUP.md) - Security setup for API keys

---

**Repository**: https://github.com/Jonathan0823/opencode-config.git

**Maintained with ❤️ for the OpenCode community**
