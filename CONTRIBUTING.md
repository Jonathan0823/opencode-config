# Contributing to OpenCode Config

Thank you for your interest in contributing to the OpenCode Config repository! This document provides guidelines for contributing new skills, improving existing ones, and maintaining code quality.

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/YOUR_USERNAME/opencode-config.git`
3. Create a new branch: `git checkout -b feature/your-feature-name`
4. Make your changes
5. Test your changes locally
6. Commit with descriptive messages following [conventional commits](https://www.conventionalcommits.org/)
7. Push to your fork: `git push origin feature/your-feature-name`
8. Create a Pull Request

## Skill Development Guidelines

### Skill Structure

Each skill should follow this structure:

```
skills/<skill-name>/
└── SKILL.md
```

### Skill Template

```markdown
---
name: skill-name
description: Brief description of what this skill covers
---

# Skill Name

## Overview

Brief explanation of the skill's purpose and scope.

## Core Principles

### 1. Principle Name

Explanation with code examples:

```language
// Good example
function goodExample() {
  // Implementation
}

// Bad example
function badExample() {
  // Implementation
}
```

## Framework-Specific Patterns

### Framework Name

```
## When to Use This Skill

List specific scenarios when this skill should be applied.

## Related Skills

- `@other-skill` - Description
```

### Writing Guidelines

1. **Use clear, concise language** - Avoid unnecessary jargon
2. **Include practical examples** - Every concept should have code examples
3. **Show good vs bad practices** - Use ✅ DO and ❌ DON'T format
4. **Be comprehensive but focused** - Cover important patterns without being overwhelming
5. **Include error handling** - Show how to handle common error scenarios
6. **Add cross-references** - Link to related skills
7. **Use consistent formatting** - Follow the existing skill format

### Code Example Standards

- Use syntax highlighting for all code blocks
- Include comments explaining non-obvious code
- Show realistic, production-ready examples
- Include imports and necessary context

### Frontmatter Requirements

All skills must include YAML frontmatter:

```yaml
---
name: skill-name
description: Brief, clear description (max 100 characters)
---
```

## Submitting Changes

### Commit Message Format

Follow conventional commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

Types:
- `feat`: New skill or feature
- `fix`: Bug fix in existing skill
- `docs`: Documentation improvements
- `refactor`: Code refactoring without functional changes
- `chore`: Maintenance tasks

Examples:
- `feat(skills): add python-patterns skill`
- `fix(go-conventions): correct error handling example`
- `docs(readme): update installation instructions`

### Pull Request Process

1. **Title**: Use clear, descriptive title
2. **Description**: Explain what changed and why
3. **Checklist**:
   - [ ] Skill follows the template structure
   - [ ] All code examples are tested/valid
   - [ ] Documentation is clear and complete
   - [ ] No breaking changes (or clearly documented)
   - [ ] Related skills are cross-referenced

### Review Process

- All PRs require at least one review
- Address review comments promptly
- Maintain respectful, constructive communication

## Code of Conduct

### Our Standards

- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on constructive feedback
- Accept responsibility for mistakes

### Unacceptable Behavior

- Harassment or discrimination
- Trolling or insulting comments
- Personal or political attacks
- Publishing others' private information

## Questions?

- Open an issue for questions or suggestions
- Join discussions in existing issues
- Check existing skills for examples

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

Thank you for contributing to OpenCode Config!
