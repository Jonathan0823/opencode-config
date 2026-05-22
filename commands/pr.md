---
description: Create a pull request draft for review and approval
agent: plan
subtask: true
---

Create a pull request draft. This command plans first — it never creates a PR without explicit approval.

## Phase 1 — Plan (run in plan mode)

1. **Check branch and status**:
   ```bash
   git branch --show-current
   git status --porcelain
   git log --oneline origin/main..HEAD
   git diff origin/main...HEAD --stat
   ```
   Fall back to `main` or `master` if `origin/main` doesn't exist.

2. **Determine base branch**: Check `git remote show origin` for HEAD branch.

3. **Build the PR draft**:
   - Title: conventional commit format, under 72 chars
   - Body: use the `pr-writing` skill template:
     ```
     ## Related issue
     - Closes #<issue-number>

     ## Summary
     - <change 1>
     - <change 2>

     ## Scope
     - In scope: <items>
     - Out of scope: <items>

     ## Testing
     - [ ] Unit tests
     - [ ] Integration tests
     - [ ] Manual verification

     ## Reviewer focus
     - Please review: <critical files/logic>
     - Risk areas: <edge cases>

     ## Notes
     - Migrations/config changes: <none | details>
     - Follow-ups: <optional>
     ```
   - Do NOT include verbose line-by-line file listings

4. **Show the PR draft** to the user:
   ```
   Branch: <branch> → <base-branch>

   Title:
   type(scope): description

   Body:
   <full body>
   ```

5. **Ask for approval**:
   - "Create this PR?" → yes / no / edit title / edit body / change scope

## Phase 2 — Execute (only after user approval)

6. **Create PR**:
   ```bash
   gh pr create --title "type(scope): description" --body "<body>" --base <base-branch>
   ```

7. **Output**: Show the PR URL when complete.

## Hard Rules

- One PR = one logical change.
- Never auto-create a PR.
- Never create a PR on protected branches without explicit override.
- Never use a title that combines multiple outcomes.
