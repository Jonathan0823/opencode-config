### Interaction Standards for AI Agents

This document outlines the standards and best practices for designing and implementing AI agents to ensure effective, ethical, and user-friendly interactions.

#### 1. Clarity and Transparency

- AI agents should clearly communicate their capabilities and limitations to users.
- Responses should be concise, relevant, and free from ambiguity.
- When delegating to subagents, explain which subagent is being used and why.

#### 2. User-Centric Design

- Design interactions that prioritize user needs and preferences.
- Provide options for users to customize their experience with the AI agent.
- Always ask for questions when user intent is unclear or when multiple interpretations are possible.
- Offer to use specific subagents when appropriate: "Should I use @researcher to look into this?"

### 3. Reviewing & Debugging

- Provide clear explanations for any errors or unexpected behaviors.
- Offer suggestions for troubleshooting and resolving issues.
- Maintain logs of interactions to help identify patterns and improve performance.

---

## Subagent Usage Guide

### Available Subagents

| Subagent | Purpose | When to Use |
|----------|---------|-------------|
| `@code-reviewer` | Code review | Reviewing code for quality, security, performance |
| `@researcher` | Research | Investigating libraries, patterns, solutions |
| `@refactorer` | Refactoring | Improving code structure safely |
| `@explainer` | Explanation | Understanding code, generating documentation |
| `@implementer` | Implementation | Building features end-to-end |

### Usage Patterns

**For Research:**
```
User: "What's the best way to implement authentication in Go?"
→ Use @researcher
```

**For Code Review:**
```
User: "Please review this code"
→ Use @code-reviewer
```

**For Refactoring:**
```
User: "Refactor this to be more maintainable"
→ Use @refactorer
```

**For Documentation:**
```
User: "Explain how this works and add docs"
→ Use @explainer
```

**For Feature Implementation:**
```
User: "Build a user authentication system"
→ Use @implementer
```

---

## Code Quality Standards

### Language-Specific Guidelines

**Go:**
- Follow `go-conventions` skill
- Use interfaces for testability
- Explicit error handling
- Small, focused functions

**Rust:**
- Follow `rust-patterns` skill
- Ownership and borrowing rules
- Result/Option types for error handling
- Type safety over convenience

**TypeScript:**
- Follow `ts-react-nextjs` or `ts-vue-svelte` skill
- Strict TypeScript configuration
- Explicit types for public APIs
- Avoid `any` type

**Python:**
- Type hints for function signatures
- Docstrings for public APIs
- Explicit error handling
- PEP 8 style guide

### General Standards

- **Function Length**: Max 50 lines (aim for 20-30)
- **Cyclomatic Complexity**: Max 10
- **Test Coverage**: 80%+ for business logic
- **Documentation**: Public APIs must be documented
- **Error Handling**: Every error case must be handled
- **Security**: Input validation, auth checks, no secrets in code

---

## Development Workflow

### Feature Development Process

1. **Research** → Use `@researcher` for investigation
2. **Plan** → Define interfaces, break into tasks
3. **Implement** → Use `@implementer` or code directly
4. **Test** → Unit, integration, manual tests
5. **Review** → Use `@code-reviewer`
6. **Document** → Use `@explainer`

### Bug Fixing Process

1. **Reproduce** → Create reliable test case
2. **Investigate** → Use `@researcher` if needed
3. **Fix** → Minimal change, root cause
4. **Verify** → Tests pass
5. **Deploy** → Monitor for regression

### Refactoring Process

1. **Ensure Tests** → Write tests if none exist
2. **Small Changes** → One change at a time
3. **Run Tests** → After every change
4. **Commit** → Frequent commits
5. **Review** → Use `@refactorer`

---

## Git Workflow

### Branch Naming

```
feature/user-authentication
bugfix/login-redirect
hotfix/security-patch
refactor/order-service
```

### Commit Messages

Follow conventional commits:
```
feat(auth): add JWT middleware
fix(api): resolve race condition
docs(readme): update installation
refactor(db): optimize queries
test(auth): add integration tests
```

### Pull Request Process

1. Create feature branch
2. Make atomic commits
3. Push branch
4. Create PR with description
5. Request review
6. Address feedback
7. Merge (squash or rebase)
8. Delete branch

---

## Communication Standards

### When Starting Work

```
"I'll help you with [task]. Let me start by [approach]."
"This involves [complexity]. I'll use [@subagent] for [specific part]."
```

### When Asking Questions

```
"To proceed, I need to clarify: [specific question]"
"There are a few approaches: [option A] or [option B]. Which do you prefer?"
```

### When Explaining Code

```
"This code [does X]. It works by [mechanism]."
"The issue is [problem], caused by [root cause]."
"The fix [solution], which [explanation]."
```

### When Completing Tasks

```
"Done! I've [what was done]."
"Next steps could be: [suggestions]."
"This change [impact]."
```

---

## Team Environment Compatibility

### For Solo Development

- Commit frequently (every 2-4 hours of work)
- Write descriptive commit messages
- Document architecture decisions
- Keep README updated

### For Team Collaboration

- Use feature branches
- Require code review before merge
- Document breaking changes
- Update CHANGELOG
- Communicate in PRs clearly

### CI/CD Integration

```
□ Tests must pass
□ Linting must pass
□ Commit messages validated
□ No secrets in code
□ Documentation updated (if needed)
```

---

## Error Handling

### When Errors Occur

1. **Explain** what went wrong
2. **Why** it happened (if known)
3. **How** to fix it (suggestions)
4. **Prevent** it from happening again

### Example Error Responses

```
❌ Bad: "Error occurred"

✅ Good: "Failed to connect to database. This is likely because the 
PostgreSQL service isn't running. Try: docker-compose up postgres"

✅ Good: "Build failed due to TypeScript error in src/auth/service.ts:45.
The function signature doesn't match the interface. Should I fix this?"
```

---

## Performance Guidelines

### Code Performance

- Profile before optimizing
- Optimize for readability first
- Measure impact of changes
- Document performance-critical code

### Response Performance

- Respond within 5 seconds
- For long operations, show progress
- Stream responses when possible
- Cache repeated operations

---

## Security Guidelines

### Never

- Expose secrets or API keys
- Log sensitive data (passwords, tokens)
- Trust user input (always validate)
- Skip authentication/authorization checks
- Use eval() or equivalent

### Always

- Validate input at API boundaries
- Use parameterized queries
- Implement proper error handling
- Apply principle of least privilege
- Keep dependencies updated

---

## Skill Usage

### When to Use Skills

Use skills when:
- Working with specific technologies (Go, Rust, React, etc.)
- Following specific workflows (feature dev, bug fix, refactoring)
- Needing best practices for specific domains (API design, testing)
- Writing documentation or setting up Git workflows

### Skill Priority

1. **Language skills** → When writing/reviewing code
2. **Workflow skills** → When following processes
3. **Database skills** → When working with data
4. **Quality skills** → For architecture and design

---

## Continuous Improvement

### Regular Reviews

- Weekly: Review AGENTS.md for updates
- Monthly: Add new skills as needed
- Quarterly: Evaluate subagent effectiveness

### Feedback Loop

- Track which subagents are used most
- Note common patterns that need skills
- Update documentation based on usage
- Refine prompts based on results
