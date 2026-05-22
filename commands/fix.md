---
description: Diagnose, reproduce, and fix a bug
agent: build
subtask: true
---

Debug and fix the reported issue.

## Bug Report

$ARGUMENTS

## Process

1. **Understand**: If the issue is vague, ask for:
   - Expected vs actual behavior
   - Steps to reproduce
   - Environment (OS, versions)

2. **Reproduce**: Write or run a minimal reproduction.

3. **Diagnose**: Identify root cause. Search for related code paths.

4. **Propose**: Show the root cause and proposed fix. Ask for approval.

5. **Fix**: Apply the minimal change needed.

6. **Verify**: Run tests to confirm the fix works.

7. **Report**: Summarize what was wrong, what was changed, and how it was verified.
