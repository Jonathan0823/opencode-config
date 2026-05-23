---
description: Analyze the codebase for recurring patterns and duplication
agent: build
subtask: true
---

Analyze the current codebase for recurring implementation patterns.

## Scope

$ARGUMENTS

## Workflow

1. Identify the language or feature area in scope.
2. Search for similar implementations and repeated structures.
3. Group matches by pattern type.
4. Summarize the findings with file and line references.
5. Call out refactoring opportunities only when they are concrete.

## Output

- Pattern name
- Matching files
- Why the pattern matters
- Suggested consolidation or cleanup
