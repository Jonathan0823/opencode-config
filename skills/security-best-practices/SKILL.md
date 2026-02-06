---
name: security-best-practices
description: Security best practices, OWASP guidelines, secure coding patterns, and vulnerability prevention. Use when implementing authentication, handling user input, securing APIs, managing secrets, or reviewing code for security vulnerabilities.
license: MIT
compatibility: opencode
---

# Security Best Practices Skill

## Overview

This skill provides security guidelines following OWASP Top 10, secure coding patterns, authentication/authorization best practices, secrets management, and vulnerability prevention across multiple languages.

## OWASP Top 10 Summary

1. **Injection** - Use parameterized queries, never concatenate user input into SQL/commands
2. **Broken Authentication** - Implement strong passwords, secure sessions, rate limiting
3. **Sensitive Data Exposure** - Encrypt data at rest, use HTTPS, hash passwords
4. **XML External Entities (XXE)** - Disable external entities in XML parsers
5. **Broken Access Control** - Enforce authorization checks, implement resource-level controls
6. **Security Misconfiguration** - Secure defaults, minimal privileges, regular updates
7. **Cross-Site Scripting (XSS)** - Escape output, sanitize input, use CSP headers
8. **Insecure Deserialization** - Use JSON instead of pickle, validate data
9. **Known Vulnerabilities** - Regular dependency scanning, keep components updated
10. **Insufficient Logging** - Log security events, monitor for anomalies

## Quick Security Checklist

### Input Validation
- [ ] Validate all user input on server side
- [ ] Use allowlists, not denylists
- [ ] Sanitize data before display (prevent XSS)
- [ ] Validate file uploads (type, size, extension)

### Authentication
- [ ] Use strong password requirements (12+ chars, complexity)
- [ ] Hash passwords with bcrypt/Argon2 (not MD5/SHA1)
- [ ] Implement rate limiting on login endpoints
- [ ] Use secure session management (HttpOnly, Secure, SameSite)

### Authorization
- [ ] Check permissions on every request
- [ ] Implement principle of least privilege
- [ ] Use resource-level access controls
- [ ] Never rely on client-side checks

### Data Protection
- [ ] Encrypt sensitive data at rest
- [ ] Use TLS 1.2+ for all connections
- [ ] Set security headers (HSTS, CSP, X-Frame-Options)
- [ ] Never log sensitive data (passwords, tokens, PII)

### Secrets Management
- [ ] Use environment variables or secret managers
- [ ] Never commit secrets to version control
- [ ] Rotate secrets regularly
- [ ] Use different secrets per environment

## Common Vulnerabilities Prevention

### SQL Injection
```python
# ✅ SAFE: Parameterized query
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# ❌ UNSAFE: String concatenation
query = f"SELECT * FROM users WHERE id = '{user_id}'"
```

### XSS Prevention
```python
# ✅ SAFE: Template auto-escaping
return render_template('profile.html', username=username)

# ❌ UNSAFE: Raw HTML
return f"<div>{user_input}</div>"
```

### Command Injection
```python
# ✅ SAFE: Use list, not shell
subprocess.run(["ls", "-la", directory], shell=False)

# ❌ UNSAFE: Shell with user input
os.system(f"ls -la {directory}")
```

## Language-Specific Patterns

See detailed guides in references/:

- **[OWASP Top 10 Details](references/owasp-top10.md)** - Comprehensive prevention for all 10 categories
- **[Secure Coding - Python](references/secure-coding-python.md)** - Python-specific security patterns
- **[Secure Coding - JavaScript](references/secure-coding-javascript.md)** - Node.js/Frontend security
- **[Secure Coding - Go](references/secure-coding-go.md)** - Go security patterns
- **[Secrets Management](references/secrets-management.md)** - AWS, Vault, GCP secret management

## Security Headers

Always include these headers:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

## When to Use This Skill

Use this skill when:
- Implementing authentication and authorization systems
- Handling user input and data validation
- Setting up HTTPS and security headers
- Managing secrets and credentials
- Configuring CORS and CSP policies
- Reviewing code for security vulnerabilities
- Setting up logging and monitoring
- Configuring Docker and deployment security

## Related Skills

- `@docker-patterns` - Container security hardening
- `@ci-cd-pipelines` - Security scanning in CI/CD
- `@api-rest-design` - API security patterns
- `@postgresql-patterns` - Database security
- `@feature-development` - Secure development workflow
