---
description: Create a pull request draft for review and approval
agent: plan
subtask: false
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

3. **Build the PR draft** using the `pr-writing` skill template:
   - Title: conventional commit format, under 72 chars
   - Body sections (fill every section, do not skip):
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

     ### Test details
     - Commands run:
       - <command>
     - Results:
       - <result>

     ## Reviewer focus
     - Please review: <critical files/logic>
     - Risk areas: <edge cases>

     ## Notes
     - Migrations/config changes: <none | details>
     - Follow-ups: <optional>
     ```
   - Do NOT include verbose line-by-line file listings

4. **Show the full PR draft** to the user exactly as it will be submitted, including every section from the template.

5. **Ask for approval**:
   - "Create this PR?" → yes / no / edit title / edit body / change scope
   - If yes, the user creates the PR manually outside this command.

6. **Execution guidance** (shown after approval, not executed by this command):
   - Preferred: use GitHub MCP `github_create_pull_request` with the drafted title and body
   - Fallback: `gh pr create --title "..." --body "..." --base <base-branch>`

## Hard Rules

- One PR = one logical change.
- Never auto-create a PR.
- Never create a PR on protected branches without explicit override.
- Never use a title that combines multiple outcomes.
- This command is plan-only. Execution happens outside the command after user approval.

---

**STOP HERE.** Do not create the PR. Wait for the user to explicitly approve. After approval, the user creates the PR using GitHub MCP (preferred) or `gh pr create` (fallback).
