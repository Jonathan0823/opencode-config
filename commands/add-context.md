---
description: Capture project patterns and create project intelligence
agent: build
subtask: true
---

Create or update project intelligence for the current repository.

## User Input

$ARGUMENTS

## Workflow

1. Detect the stack, coding patterns, and naming conventions used in the repo.
2. Ask a short set of targeted questions if anything is unclear.
3. Summarize the patterns you found before writing anything.
4. Create or update the relevant context files with MVI-style content.
5. Update navigation entries if new context files were added.
6. Show the result and ask for confirmation before writing broader changes.

## Rules

- Keep context concise and focused.
- Prefer concrete examples over long explanations.
- Link the context back to actual code paths when possible.
