# OpenCode AI Configuration Pack

Control your AI patterns. Get repeatable results.

This repository is a production-oriented OpenCode configuration that applies the OpenAgents Control workflow: context-aware planning, approval gates, modular delegation, and consistent implementation standards.

If you’ve ever had AI generate code that _works_ but doesn’t match your architecture, naming, or quality bar, this config is built to solve exactly that.

---

## Why use this config

### 1) Your patterns become default behavior

Agents are designed to load project context before execution, so generated output follows your conventions instead of generic defaults.

### 2) Approval-first workflow

Plans are proposed before execution. You stay in control of edits, commands, and implementation scope.

### 3) MVI-style context loading

Minimal Viable Information (MVI) keeps context focused and efficient: load what is needed, avoid unnecessary prompt bloat.

### 4) Team consistency by design

The context and command system makes it easier for teams to produce consistent code and documentation across contributors.

### 5) Live external docs when needed

External library lookups are supported via Context7 workflows, helping avoid outdated patterns.

---

## What you get

- **Core agents** for general execution and coding workflows
- **Specialized subagents** for planning, implementation, testing, review, and documentation
- **Command system** for structured, repeatable workflows
- **Context system** for standards, workflows, and project intelligence
- **Skill library** across frontend, backend, infra, security, testing, and documentation
- **Runtime config** through `opencode.json`
- **MCP servers** for live docs (Context7) and GitHub integration

---

## How this helps in practice

With a typical generic setup:

- AI generates code fast
- You spend time reshaping it to fit project standards

With this config:

- Agents discover relevant context first
- They propose an approach with explicit checkpoints
- They execute incrementally with validation and approval

Result: less rework, cleaner diffs, more predictable output.

---

## Installation (Linux)

Copy this repository into your global OpenCode config path (excluding git internals):

```bash
mkdir -p ~/.config/opencode
rsync -av --exclude '.git' ./ ~/.config/opencode/
```

That’s it. OpenCode will load this config from `~/.config/opencode`.

### Environment setup

Copy `.env.example` to `.env` and fill in your API keys:

```bash
cp .env.example .env
# Edit .env with your keys
```

Available variables:

- `CONTEXT7_API_KEY` — for Context7 live docs (get from https://context7.com)
- GitHub token — stored in `~/.config/opencode/.secrets/github-key`

The `.env` file is gitignored. Never commit secrets.

### Platform support

- ✅ Tested on **Linux**
- ✅ Tested on **Windows via WSL**
- ⚠️ Other environments may work, but are not officially tested yet

---

## Configuration resolution (important)

OpenCode configuration is loaded by proximity:

1. Project-local (`.opencode/`)
2. Parent directories
3. Global (`~/.config/opencode/`)

**Practical recommendation**:

- Use **project-local** for shared team behavior
- Use **global** for your personal defaults

---

## MCP Servers

This config includes MCP (Model Context Protocol) connections for enhanced tooling and workflows.

### Context7

Context7 provides live documentation lookups so agents can fetch up-to-date library docs without relying on training data.

- Used by the `Context7` skill and `@researcher` agent
- Enables real-time library docs, code examples, and API references
- Configured in `opencode.json` under the `context7` key
- API key set via `CONTEXT7_API_KEY` in `.env`

### GitHub MCP

GitHub MCP enables AI agents to interact with GitHub directly—creating issues, PRs, managing repos, and more without leaving the workflow.

- Used for GitHub-aware workflows (issue creation, PR management, repo operations)
- Connects to GitHub Copilot MCP endpoint
- Requires a GitHub personal access token

#### Secret handling

- Store the GitHub token in `~/.config/opencode/.secrets/github-key`
- The `.secrets/` directory is ignored by git
- Do not commit credentials into the repository
- Token is referenced in `opencode.json` via `{file:~/.config/opencode/.secrets/github-key}`

#### Notes

- This setup uses a locally stored token for auth
- Upstream GitHub MCP also supports OAuth-based setups if you want to switch later

---

## Example prompts

```text
@implementer build this endpoint using our API conventions
@code-reviewer review this diff for security and maintainability risks
@researcher compare library options for this feature
@explainer summarize this module and generate docs
```

---

## Core workflow model

This config emphasizes a structured lifecycle:

1. **Discover** context and constraints
2. **Propose** a plan
3. **Approve** before execution
4. **Implement** incrementally
5. **Validate** and hand off

This is intentionally optimized for reliability and maintainability over "fire-and-forget" automation.

---

## Repo structure

```text
agent/          # Core agents and specialized subagents
command/        # Slash commands and workflow entrypoints
config/         # Metadata and config support files
context/        # Standards, workflows, project intelligence
skills/         # Reusable capability modules
opencode.json   # Runtime permissions/MCP configuration
README.md
```

---

## Customization guide

You can safely adapt this config to your team:

- Update context files in `context/project-intelligence/`
- Tune agent behavior in `agent/`
- Add workflow commands under `command/`
- Extend capabilities through `skills/`

Tip: keep edits incremental and tested. If behavior changes significantly, document it in README or context notes.

---

## FAQ

### Will this replace my existing config?

If you copy these files into an existing config location, overlapping files will be replaced.

### Should I use local or global config?

Use local for team/project consistency. Use global for personal defaults across repositories.

### Do I need every skill enabled?

No. You can keep only the skills and commands your team actively uses.

### Can I still use upstream OAC resources?

Yes. This repo is compatible with OAC workflows and patterns and can be evolved alongside upstream guidance.

---

## Attribution

Powered by [OpenAgents Control (OAC)](https://github.com/darrenhinde/OpenAgentsControl) by Darren Hinde.

---

## Related resources

- [OpenAgents Control Repository](https://github.com/darrenhinde/OpenAgentsControl)
- [OpenCode Documentation](https://opencode.ai/docs)
- [AGENTS.md](./AGENTS.md)
