---
name: pr-writing
description: Write concise, review-ready pull requests with clear scope, testing, and reviewer guidance
license: MIT
compatibility: opencode
metadata:
  audience: contributors
  workflow: github
---

# PR Writing Skill

## What I do

- Produce clean, review-ready pull request summaries.
- Keep PRs small, focused, and easy to merge.
- Ensure communication includes context, testing, and reviewer focus areas.
- Provide reusable templates for consistent PR quality.

## Workflow

### 1) Communicate early
- Link the related issue (or state why none exists).
- Announce ownership when starting work.
- Clarify scope before implementation.

### 2) Keep scope small
- Fix one problem per PR.
- Avoid bundling unrelated changes.
- Split larger work into logical commits.

### 3) Keep changes clean
- Follow project lint/style rules.
- Preserve existing architecture and patterns.
- Remove dead code and noisy changes.

### 4) Test before opening
- State what you tested (manual + automated).
- Mention environments used.
- Call out known gaps or deferred tests.

### 5) Write the summary for reviewers
Include:
- Related issue/context
- What changed and why
- How it was tested
- What reviewers should focus on
- Notes/risks/migrations (if any)

## PR Body Template

```markdown
## Related issue
- Closes #<issue-number>

## Summary
- <change 1>
- <change 2>
- <change 3>

## Scope
- In scope: <items>
- Out of scope: <items>

## Testing
- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual verification

### Test details
- Commands run:
  - `<command>`
- Results:
  - `<result>`

## Reviewer focus
- Please review: <critical files/logic>
- Risk areas: <edge cases>

## Notes
- Migrations/config changes: <none | details>
- Follow-ups: <optional>
```

## Quick Quality Checklist

- [ ] Single clear purpose
- [ ] Minimal diff / no unrelated edits
- [ ] Clear commit history
- [ ] Tests described and reproducible
- [ ] Reviewer guidance included
- [ ] Backward compatibility considered

## When to use me

Use this skill when opening a pull request or when improving an existing PR description for faster, higher-quality review.
