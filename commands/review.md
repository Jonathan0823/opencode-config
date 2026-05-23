---
description: Run a strict code review on changes or files
agent: CodeReviewer
subtask: true
---

Review code for quality, security, performance, and maintainability.

## Scope

$ARGUMENTS

If no scope is provided, review the current diff (`git diff`).

If file paths are provided, review those specific files.

## Review Format

Output findings in this order:

### Critical Issues (must fix)
- Security vulnerabilities, data loss risks, logic bugs

### Architecture & Design
- Coupling, cohesion, pattern violations, abstraction leaks

### Maintainability
- Readability, naming, complexity, duplication

### Nitpicks (optional)
- Style inconsistencies, minor refactors

For each finding, include:
- File:line reference
- Why it's a problem
- Concrete suggestion

## Rules

- Do NOT make edits unless explicitly asked
- Findings-first output (not conversational preamble)
- Be specific: reference exact lines, not vague areas
- If no significant issues found, say so concisely
