---
description: Create consistent commit messages
agent: build
---

Create a commit message following conventional commits standard.

## Commit Message Format

- Use clear and concise commit messages
- Follow the format: `<type>(<scope>): <subject>`
  - `<type>`: chore, feat, fix, docs, style, refactor, test, perf, ci
  - `<scope>`: optional, specify the area of the codebase affected
  - `<subject>`: brief description of the change
- Example: `feat(auth): add OAuth2 support`

## Best Practices

1. **Descriptive Messages**: Provide detailed descriptions in the commit body if necessary
2. **Single Responsibility**: Each commit only focuses on a single change or feature
   - Avoid bundling unrelated changes in one commit
   - Example: Instead of `fix and update UI`, use two separate commits:
     - `fix(button): correct button alignment`
     - `feat(ui): update color scheme`
3. **Testing**: Ensure all tests pass before committing changes if applicable
4. **Linting**: Run linting tools to maintain code quality

## Instructions

Before creating the commit:

1. Run `git status` to see all untracked files
2. Run `git diff` to see both staged and unstaged changes that will be committed
3. Run `git log` to see recent commit messages to follow this repository's commit message style
4. Analyze all staged changes and draft a commit message that:
   - Summarizes the nature of the changes (new feature, enhancement, bug fix, refactor, etc.)
   - Ensures the message accurately reflects the changes and their purpose
   - Follows conventional commit format: `<type>(<scope>): <subject>`
5. Add relevant untracked files to the staging area
6. Create the commit with the message
7. Run `git status` after the commit completes to verify success

If multiple unrelated changes are staged, suggest splitting them into separate commits following Single Responsibility Principle.
