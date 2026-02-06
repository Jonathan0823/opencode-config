---
name: git-team-workflow
description: Git workflows, branching strategies, and team collaboration patterns
license: MIT
compatibility: opencode
---

# Git Team Workflow Skill

## Overview

This skill provides guidelines for effective Git workflows, branching strategies, and team collaboration practices.

## Branching Strategies

### 1. GitFlow (Traditional)

```
main (production-ready)
  ↑
develop (integration)
  ↑
feature/login
feature/checkout
hotfix/security-fix
release/v1.2.0
```

**When to use:**
- Scheduled release cycles
- Multiple versions in production
- Large teams with formal processes

**Workflow:**
```bash
# Start feature

git checkout develop
git checkout -b feature/user-authentication

# Work on feature
git add .
git commit -m "feat(auth): implement JWT authentication"

# Push and create PR
git push origin feature/user-authentication
# Create PR to develop

# After PR merged
git checkout develop
git pull origin develop

# Start release
git checkout -b release/v1.2.0
# Bump version, update changelog
git commit -m "chore(release): prepare v1.2.0"

# Merge to main and develop
git checkout main
git merge release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags

git checkout develop
git merge release/v1.2.0
```

### 2. GitHub Flow (Simplified)

```
main (production-ready)
  ↑
feature/payment
feature/notifications
bugfix/cart-issue
```

**When to use:**
- Continuous deployment
- Small to medium teams
- Fast iteration cycles

**Workflow:**
```bash
# Start feature
git checkout main
git pull origin main
git checkout -b feature/payment-gateway

# Work and commit frequently
git add .
git commit -m "feat(payment): integrate Stripe"
git push origin feature/payment-gateway

# Create PR to main
# After review and CI pass, merge to main
# Deploy automatically
```

### 3. Trunk-Based Development

```
main (production-ready)
  ↑
short-lived branches (hours, not days)
```

**When to use:**
- Continuous deployment
- High-performing teams
- Strong CI/CD practices

**Rules:**
- Main branch always deployable
- Branches live < 1 day
- Feature flags for incomplete features
- Pair programming recommended

## Commit Guidelines

### 1. Conventional Commits

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting (no code change)
- `refactor`: Code refactoring
- `test`: Tests
- `chore`: Maintenance
- `perf`: Performance improvement
- `ci`: CI/CD changes

**Examples:**
```bash
feat(auth): add OAuth2 Google login

Implement Google OAuth2 authentication flow using
Passport.js. Includes token refresh and user profile
synchronization.

Closes #123

---

fix(api): resolve race condition in order processing

Add mutex lock to prevent concurrent order updates
that caused inventory inconsistencies.

Fixes #456

---

docs(readme): update installation instructions

Add Docker setup guide and environment variable
documentation.

---

refactor(database): optimize user queries

Replace N+1 queries with JOIN for user profile loading.
Improves response time by 40%.

---

test(auth): add integration tests for login

Cover happy path, invalid credentials, and locked
account scenarios.
```

### 2. Commit Best Practices

```bash
# DO: Commit early and often
git add src/auth/login.go
git commit -m "feat(auth): implement login handler"

git add src/auth/middleware.go
git commit -m "feat(auth): add JWT middleware"

# DO: Atomic commits (one logical change)
# DON'T: Mix unrelated changes

# DO: Meaningful commit messages
git commit -m "fix: resolve null pointer in user service"

# DON'T: Vague messages
git commit -m "fix stuff"  # Bad!

# DO: Reference issues
git commit -m "feat: add password reset

Implements #234"
```

## Pull Request Workflow

### 1. PR Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No console errors

## Related Issues
Fixes #123
Relates to #456
```

### 2. PR Best Practices

```bash
# DO: Keep PRs small (< 400 lines)
# DO: One PR per feature/fix
# DO: Add descriptive title
# DO: Request review from relevant team members

# DON'T: Include unrelated changes
# DON'T: Commit sensitive data
# DON'T: Leave debugging code
```

### 3. Code Review Process

**Reviewer Checklist:**
```
□ Code follows team standards
□ Tests are included and pass
□ No security vulnerabilities
□ Performance implications considered
□ Documentation updated
□ Commit messages are clear
```

**Review Comments:**
```
# Suggestion (non-blocking)
**suggestion**: Consider extracting this into a separate function

# Required change (blocking)
**issue**: This will fail with nil pointer. Please add nil check.

# Question
**question**: Why are we using recursion here instead of iteration?

# Praise
**praise**: Great use of the strategy pattern! 👍
```

## Git Commands Cheat Sheet

### Daily Workflow

```bash
# Start work
git pull origin main
git checkout -b feature/my-feature

# Regular commits
git status
git diff
git add -p  # Stage interactively
git commit -m "feat: add feature"

# Sync with main
git fetch origin
git rebase origin/main

# Or merge
git merge origin/main

# Push
git push origin feature/my-feature

# Create PR via GitHub CLI
gh pr create --title "feat: add my feature" --body "Description"
```

### Advanced Operations

```bash
# Interactive rebase (clean history)
git rebase -i HEAD~5
# Commands: pick, reword, squash, fixup, drop

# Stash work
git stash push -m "work in progress"
git stash list
git stash pop
git stash drop

# Cherry-pick
git cherry-pick abc123

# Bisect (find bad commit)
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh

# Reflog (recover lost commits)
git reflog
git checkout HEAD@{5}

# Clean untracked files
git clean -fd
```

## Team Collaboration

### 1. Branch Protection Rules

```yaml
# GitHub branch protection
main:
  protection:
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
    required_status_checks:
      strict: true
      contexts:
        - "ci/tests"
        - "ci/lint"
    required_linear_history: true
    allow_force_pushes: false
    allow_deletions: false
```

### 2. Release Process

```bash
# Version bumping
npm version patch  # 1.0.0 -> 1.0.1
npm version minor  # 1.0.0 -> 1.1.0
npm version major  # 1.0.0 -> 2.0.0

# Or manual
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin v1.2.0

# Generate changelog
npx conventional-changelog -p angular -i CHANGELOG.md -s
```

### 3. CI/CD Integration

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Check commit messages
        run: npx commitlint --from=HEAD~10
```

## Best Practices

### DO:
- ✅ Commit early and often
- ✅ Write clear commit messages
- ✅ Keep PRs small and focused
- ✅ Review code before merging
- ✅ Use feature branches
- ✅ Sync with main regularly
- ✅ Use meaningful branch names
- ✅ Delete merged branches

### DON'T:
- ❌ Commit to main directly
- ❌ Commit sensitive data
- ❌ Keep branches alive too long
- ❌ Force push to shared branches
- ❌ Ignore merge conflicts
- ❌ Commit debugging code
- ❌ Use vague commit messages

## Common Patterns

### Feature Development
```bash
# 1. Create feature branch
git checkout -b feature/user-profile

# 2. Make commits
git commit -m "feat(profile): add profile page"
git commit -m "feat(profile): add avatar upload"
git commit -m "test(profile): add unit tests"

# 3. Push and create PR
git push origin feature/user-profile
gh pr create

# 4. Address review comments
git commit -m "refactor(profile): address PR feedback"
git push

# 5. Squash and merge (via GitHub)
```

### Hotfix Process
```bash
# 1. Create hotfix from main
git checkout main
git checkout -b hotfix/security-patch

# 2. Fix and commit
git commit -m "fix(security): patch vulnerability"

# 3. Create PR to main
git push origin hotfix/security-patch

# 4. After merge to main, cherry-pick to develop
git checkout develop
git cherry-pick abc123
```

## When to Use

Use this skill when:
- Planning branching strategy
- Writing commit messages
- Creating pull requests
- Reviewing code
- Resolving merge conflicts
- Setting up CI/CD
- Onboarding new team members
