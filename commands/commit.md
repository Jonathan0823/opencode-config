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
   - If yes, the user runs the commit manually outside this command.

## Hard Rules

- One commit = one responsibility.
- If the diff reads like two changes, split it.
- Never auto-push.
- Never commit on protected branches unless explicitly overridden.
- Never use a subject that combines multiple outcomes.
- This command is plan-only. Execution happens outside the command after user approval.

---

**STOP HERE.** Do not stage, commit, or push. Wait for the user to explicitly approve and run the commands manually.
