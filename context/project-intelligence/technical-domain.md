<!-- Context: project-intelligence/technical | Priority: critical | Version: 1.0 | Updated: 2026-02-25 -->

# Technical Domain

**Purpose**: Tech stack, architecture, development patterns for this project.
**Last Updated**: 2026-02-25

## Quick Reference
**Update Triggers**: Tech stack changes | New patterns | Architecture decisions
**Audience**: Developers, AI agents

## Primary Stack
| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Framework | Next.js | Latest | Frontend framework for React applications |
| Language | TypeScript | Latest | Type-safe JavaScript for robust applications |
| Backend | Go | Latest | High-performance backend services |
| Backend | Rust | Latest | Systems programming for performance and safety |
| Database | PostgreSQL | Latest | Robust relational database |
| Database | SQL Server | Latest | Microsoft's relational database management system |

## Code Patterns
### API Endpoint
*(Skipped by user - no specific API pattern provided)*

### Component
*(Skipped by user - no specific component pattern provided)*

## Naming Conventions
| Type | Convention | Example |
|------|-----------|---------|
| Files | kebab-case | user-profile.tsx |
| Components | PascalCase | UserProfile |
| Functions | camelCase | getUserProfile |
| Database | snake_case | user_profiles |

## Code Standards
- TypeScript strict mode
- Prefer server components for SEO websites
- Go clean architecture
- Domain driven structuring
- Sometime use taskfile for running scripts

## Security Requirements
- Validate all user input
- Use parameterized queries
- Sanitize before rendering

## 📂 Codebase References
**Implementation**: `src/app/`, `src/components/`, `src/lib/`, `src/db/`, `internal/`, `pkg/` - Common directories for Next.js, Go, and Rust projects.
**Config**: `package.json`, `tsconfig.json`, `go.mod`, `Cargo.toml`

## Related Files
- Business Domain (example: business-domain.md)
- Decisions Log (example: decisions-log.md)
