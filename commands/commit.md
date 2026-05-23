---
description: Create commit draft(s) for review and approval — SRP-first, may produce multiple drafts
agent: plan
subtask: false
---

Create one or more commit drafts. This command plans only — it never stages or commits without explicit approval.

## Phase 1 — Inspect

1. **Check branch and status**:
   ```bash
   git branch --show-current
   git status --porcelain
   git diff --stat
   ```

2. **Enforce branch safety**:
   - refuse direct commits on protected branches like `main` or `master`
   - recommend creating a feature branch first

## Phase 2 — Group by Responsibility

3. **Cluster changed files into responsibility groups**:
   - each group must represent exactly one logical change
   - group by concern, not by file path alone (e.g. "config/docs" together, "command workflow" together, "cleanup/removals" together)
   - if a file touches multiple concerns, assign it to the dominant concern and note the overlap
   - if the diff is too broad to cluster cleanly, stop and ask the user to narrow the scope before drafting

## Phase 3 — Draft

4. **Produce commit drafts**:
   - **one group only** → produce one commit draft
   - **multiple groups** → produce multiple numbered commit drafts in dependency order (cleanup/removal first, then config, then docs, then code)

   Each draft must show:
   ```
   ### Draft N — <responsibility label>
   Branch: <branch>
   Files: <file list>
   Risk: <low | medium | high>

   Proposed commit:
   type(scope): description

   <one-line summary of what this commit does and why it is a single responsibility>
   ```

## Phase 4 — Approve

5. **Ask for approval per draft**:
   - "Approve Draft N?" → yes / no / edit message / change file selection
   - If yes, the user runs the matching `git add` / `git commit` manually outside this command.
   - Move to the next draft only after the current one is resolved.

## Hard Rules

- One draft = one responsibility.
- No `and` / `&` in commit subjects.
- No mixed-scope commits.
- Never auto-push.
- Never commit on protected branches unless explicitly overridden.
- This command is plan-only. Execution happens outside the command after user approval.

---

**STOP HERE.** Do not stage, commit, or push. Wait for the user to explicitly approve and run the commands manually.
