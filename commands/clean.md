---
description: Clean and normalize code in the current scope
agent: build
subtask: true
---

Clean the codebase or selected files.

## Scope

$ARGUMENTS

## Workflow

1. Remove obvious debug code and dead comments.
2. Format code with the project-standard formatter when available.
3. Remove unused imports and obvious lint issues.
4. Run the repo's required validation for the affected language stack.
5. Report what was changed and what still needs manual work.
