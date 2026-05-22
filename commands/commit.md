---
description: Create a commit draft for review and approval
agent: plan
subtask: true
---

Create a commit draft. This command plans first — it never stages or commits without explicit approval.

## Phase 1 — Plan (run in plan mode)

1. **Check branch and status**:
   ```bash
   git branch --show-current
   git status --porcelain
   git diff --stat
   ```

2. **Build the commit draft**:
   - current branch
   - changed files
   - single responsibility summary
   - risk level
   - proposed commit message in conventional format: `type(scope): description`

3. **Enforce branch safety**:
   - refuse direct commits on protected branches like `main` or `master`
   - recommend creating a feature branch first

4. **Check SRP**:
   - only one logical change per commit
   - if the diff contains multiple responsibilities, split it before continuing
   - reject commit subjects that imply mixed scope, especially subjects containing ` and ` or ` & `

5. **Show the commit draft** to the user:
   ```
   Branch: <branch>
   Files: <file list>
   Risk: <low | medium | high>

   Proposed commit:
   type(scope): description

   <one-line summary of what this commit does and why>
   ```

6. **Ask for approval**:
   - "Commit this?" → yes / no / edit message / change file selection

## Phase 2 — Execute (only after user approval)

7. **Stage files** only after user confirms:
   ```bash
   git add <file1> <file2>
   ```

8. **Re-check staged diff**: confirm it still has one responsibility.

9. **Commit**:
   ```bash
   git commit -m "type(scope): description"
   ```

10. **Show result**: commit hash and summary. Ask if user wants to push.

## Hard Rules

- One commit = one responsibility.
- If the diff reads like two changes, split it.
- Never auto-push.
- Never commit on protected branches unless explicitly overridden.
- Never use a subject that combines multiple outcomes.
- This command never stages or commits without explicit user approval.
