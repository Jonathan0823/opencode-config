---
name: issue-writing
description: Write clear, actionable GitHub issues with reproducible details, triage metadata, and review-ready context
license: MIT
compatibility: opencode
metadata:
  audience: contributors
  workflow: github
---

# Issue Writing Skill

## What I do

- Create concise, actionable GitHub issues that are easy to triage.
- Standardize issue structure for bugs, enhancements, and questions.
- Improve collaboration with clear reproduction steps, expected vs actual behavior, and impact.
- Add prioritization guidance (labels, milestones, assignees).

## Core principles

1. **Clarity first**: The title and first paragraph should explain the problem immediately.
2. **Reproducibility**: Anyone should be able to reproduce the issue from the steps.
3. **Context over noise**: Include relevant logs/screenshots/env details; avoid vague text.
4. **Actionability**: State desired outcome and acceptance criteria.
5. **Triage-friendly**: Use labels, priority, milestone, and ownership.

## What to include in every issue

- **Descriptive title**
  - Example: `Bug: App crashes when clicking Login without credentials`
- **Problem statement**
  - What happened, where, and why it matters.
- **Expected vs actual behavior**
- **Steps to reproduce** (numbered)
- **Environment**
  - OS, browser/runtime, app version/commit, relevant config
- **Evidence**
  - screenshots, stack traces, logs, request IDs, recordings
- **Impact**
  - who is affected, severity, workaround if any
- **Triage metadata**
  - labels, priority, milestone, assignee

## Templates

### 1) Bug report template

```markdown
## Description
<Clear and concise description of the problem>

## Steps to Reproduce
1. <Step 1>
2. <Step 2>
3. <Step 3>

## Expected Behavior
<What should happen>

## Actual Behavior
<What actually happens>

## Environment
- OS: <e.g., Ubuntu 24.04>
- Browser/Runtime: <e.g., Chrome 124, Node 20>
- App Version/Commit: <sha/version>
- Config/Flags: <if relevant>

## Evidence
- Logs/Stack trace:
  ```
  <paste logs>
  ```
- Screenshot/Video: <link or attachment>

## Impact
- Severity: <high/medium/low>
- Affected users/scope: <who/how many>
- Workaround: <if any>

## Suggested Fix (optional)
<If you have a likely direction>
```

### 2) Enhancement template

```markdown
## Problem
<What user/team problem exists today>

## Proposed Change
<What should be added/changed>

## Alternatives Considered
- <Alternative A>
- <Alternative B>

## Acceptance Criteria
- [ ] <criteria 1>
- [ ] <criteria 2>

## Impact
- User impact: <positive/negative>
- Technical impact: <complexity, risks>

## Dependencies
<Related issues, APIs, migrations>
```

### 3) Question / clarification template

```markdown
## Question
<What exactly do you need clarified?>

## Context
<What you were trying to do and where>

## What you tried
- <attempt 1>
- <attempt 2>

## Desired outcome
<What answer or decision is needed>
```

## Prioritization & triage guidance

- Add labels such as: `bug`, `enhancement`, `question`, `needs-triage`, `priority:high|medium|low`.
- Assign milestones for release planning.
- Assign an owner when someone starts work.
- Keep one issue = one problem.

## Good vs weak issue examples

### Weak
- Title: `Login broken`
- Body: `Doesn't work on my machine.`

### Good
- Title: `Bug: Login button triggers 500 when username is empty`
- Body includes: exact reproduction, expected/actual behavior, logs, environment, and impact.

## Quick checklist before submitting

- [ ] Title is specific and searchable
- [ ] Repro steps are complete and deterministic
- [ ] Expected vs actual behavior is explicit
- [ ] Environment details included
- [ ] Evidence attached (logs/screenshots)
- [ ] Labels/priority/milestone set
- [ ] Issue is scoped to one problem

## When to use me

Use this skill when creating or refining GitHub issues so maintainers can triage quickly and contributors can implement with minimal back-and-forth.
