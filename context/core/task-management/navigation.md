<!-- Context: core/navigation | Priority: critical | Version: 1.0 | Updated: 2026-02-15 -->

# Task Management Navigation

**Purpose**: JSON-driven task breakdown and tracking system

**Last Updated**: 2026-02-14

---

## Structure

```
core/task-management/
├── navigation.md
├── standards/
│   ├── task-schema.md           # Base JSON schema (v1.0)
│   └── enhanced-task-schema.md  # Extended schema (v2.0) - line precision, domain modeling, contracts
├── guides/
│   ├── splitting-tasks.md       # Task decomposition
│   └── managing-tasks.md        # Workflow guide
└── lookup/
    └── task-commands.md         # CLI script reference
```

---

## Quick Routes

| Task | Path | Priority |
|------|------|----------|
| **Generate spec from a plan** | `guides/managing-tasks.md` | ⭐⭐⭐⭐⭐ |
| **Understand base schema** | `standards/task-schema.md` | ⭐⭐⭐⭐⭐ |
| **Use enhanced features** | `standards/enhanced-task-schema.md` | ⭐⭐⭐⭐ |
| **Split a feature** | `guides/splitting-tasks.md` | ⭐⭐⭐⭐⭐ |
| **Manage task lifecycle** | `guides/managing-tasks.md` | ⭐⭐⭐⭐ |
| **Use CLI commands** | `lookup/task-commands.md` | ⭐⭐⭐⭐ |

---

## Loading Strategy

### For Creating Basic Tasks:
1. Load `standards/task-schema.md` (understand base structure)
2. Load `guides/splitting-tasks.md` (decomposition approach)
3. Reference `lookup/task-commands.md` (validate after creation)

### For Generating Specs from Plans:
1. Load `guides/managing-tasks.md` (spec + lifecycle flow)
2. Load `guides/splitting-tasks.md` (how specs become tasks)
3. Reference `standards/task-schema.md` (expected JSON output)

### For Multi-Stage Orchestration:
1. Load `standards/enhanced-task-schema.md` (advanced features)
2. Load `standards/task-schema.md` (base structure reference)
3. Load `guides/splitting-tasks.md` (decomposition approach)
4. Reference planning agents: ArchitectureAnalyzer, StoryMapper, PrioritizationEngine, ContractManager, ADRManager

### For Managing Tasks:
1. Load `guides/managing-tasks.md` (workflow)
2. Reference `lookup/task-commands.md` (CLI usage)

---

## Related

- **Active tasks** → `.tmp/tasks/{feature}/` (at project root)
- **Spec artifacts** → `.tmp/tasks/{feature}/spec.md`
- **Completed tasks** → `.tmp/tasks/completed/{feature}/`
- **TaskManager agent** → `.opencode/agent/subagents/core/task-manager.md`
- **Planning agents** → `.opencode/agent/subagents/planning/` (ArchitectureAnalyzer, StoryMapper, PrioritizationEngine, ContractManager, ADRManager)
- **Multi-stage workflow** → `../workflows/multi-stage-orchestration.md`
- **Core navigation** → `../navigation.md`
