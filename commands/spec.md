---
description: Generate a spec and task bundle from a feature plan
agent: TaskManager
subtask: false
---

Generate a specification and task bundle from the user's plan.

## User Input

$ARGUMENTS

## Process

1. **Interpret plan**: Treat the user's plan as the source of truth.

2. **Load methodology**: Load the `task-management` skill and TaskManager guidance.

3. **Ask clarifying questions**: If ambiguous, ask up to 3 targeted questions:
   - What problem is being solved?
   - Who are the users or actors?
   - What defines success?

4. **Generate spec**: Create `spec.md` and the task bundle under `.tmp/tasks/{feature-slug}/`.

5. **Present**: Show the spec summary and task bundle summary.

6. **Save**: Write the spec and task bundle after approval.

7. **Next steps**: Suggest implementation tasks or `/plan` follow-up.
