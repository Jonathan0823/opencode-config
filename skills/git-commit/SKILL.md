---
name: git-commit
description: Create consistent commit messages
license: MIT
compatibility: opencode
metadata:
  audience: maintainers
  workflow: github
---

## What I do

- Create commit messages based on conventional commits
- Commit message with "type(scope): description" format
- Suggests additional details for the commit body if needed
- Encourages single responsibility in commits
- Split to multiple commits if necessary following best practices and Single Responsibility Principle

1. **Commit Message Format**:
   - Use clear and concise commit messages.
   - Follow the format: `<type>(<scope>): <subject>`
     - `<type>`: chore, feat, fix, docs, style, refactor, test, perf, ci
     - `<scope>`: optional, specify the area of the codebase affected
     - `<subject>`: brief description of the change
   - Example: `feat(auth): add OAuth2 support`
2. **Descriptive Messages**:
   - Provide detailed descriptions in the commit body if necessary.
3. Single Responsibility:
   - Each commit only focuses on a single change or feature.
   - Avoid bundling unrelated changes in one commit.
   - Example: Instead of `fix and update UI`, use two separate commits: `fix(button): correct button alignment` and `feat(ui): update color scheme`.
4. Testing:
   - Ensure all tests pass before committing changes if applicable.
5. Linting:
   - Run linting tools to maintain code quality.

## When to use me

Use this when you are about to make a commit and want to ensure it follows conventional commit standards, and a brief description of the change.
