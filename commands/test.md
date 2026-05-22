---
description: Run the full validation pipeline for the current stack
agent: build
subtask: true
---

Run the validation pipeline for the repo.

## Scope

$ARGUMENTS

## Workflow

1. Detect the stack and available scripts.
2. Run the required lint and type checks for the repo.
3. Run the relevant test suite.
4. Fix failures only if the user asked for changes.
5. Report the exact command results.
