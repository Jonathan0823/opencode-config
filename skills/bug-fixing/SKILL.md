---
name: bug-fixing
description: Systematic approach to debugging and fixing bugs
license: MIT
compatibility: opencode
---

# Bug Fixing Skill

## Overview

This skill provides a systematic approach to debugging and fixing bugs efficiently while preventing regressions.

## The Bug Fixing Process

### Phase 1: Reproduction (Critical!)

**Goal**: Create a reliable reproduction case.

#### 1.1 Gather Information
```
□ Bug report details: What happened vs expected
□ Environment: OS, browser, version, config
□ Steps to reproduce (user's description)
□ Error messages and stack traces
□ Logs (application, system, browser console)
□ Recent changes (git history, deployments)
□ Frequency: Always, sometimes, once?
```

#### 1.2 Reproduce Locally
```
□ Set up same environment
□ Follow reported steps exactly
□ Try variations of steps
□ Test on different browsers/devices (if applicable)
□ Test with different data/inputs
```

**If you can't reproduce:**
- Ask for more details
- Check production logs more carefully
- Add more logging
- Consider race conditions or timing issues
- Check for environment-specific issues

#### 1.3 Document Reproduction
```markdown
## Bug: [Brief description]

### Reproduction Steps
1. [Step 1]
2. [Step 2]
3. [Step 3]

### Expected
[What should happen]

### Actual
[What actually happens]

### Environment
- OS: [e.g., Ubuntu 22.04]
- Browser: [e.g., Chrome 120]
- Version: [commit hash or version]
- Config: [relevant settings]
```

### Phase 2: Investigation & Isolation

**Goal**: Find the root cause, not just the symptom.

#### 2.1 Binary Search Method
```
1. Identify entry point (user action/API call)
2. Add logging/checkpoints throughout code path
3. Run reproduction
4. Narrow down to specific function/line
5. Repeat until root cause found
```

#### 2.2 Debugging Techniques

**Logging Strategy**:
```go
// Add strategic logging
func processOrder(order *Order) error {
    log.Printf("[DEBUG] Processing order: %+v", order)
    
    if order.Total <= 0 {
        log.Printf("[DEBUG] Invalid order total: %f", order.Total)
        return fmt.Errorf("invalid order total: %f", order.Total)
    }
    
    user, err := userRepo.GetByID(order.UserID)
    if err != nil {
        log.Printf("[ERROR] Failed to get user %s: %v", order.UserID, err)
        return fmt.Errorf("get user: %w", err)
    }
    
    log.Printf("[DEBUG] Found user: %+v", user)
    // ...
}
```

**Debugger Usage**:
```
□ Set breakpoints at suspected locations
□ Inspect variable values
□ Step through execution
□ Watch expressions
□ Call stack analysis
```

#### 2.3 Root Cause Analysis

**Ask "Why" 5 Times**:
```
Problem: Application crashes on checkout

Why 1: Why does it crash?
→ Null pointer exception on user.email

Why 2: Why is user.email nil?
→ User was created without email validation

Why 3: Why was validation skipped?
→ OAuth signup doesn't require email

Why 4: Why doesn't OAuth require email?
→ We assumed all OAuth providers return email

Why 5: Why did we assume that?
→ We didn't test with GitHub accounts that hide email

Root Cause: Missing email handling for OAuth providers
```

#### 2.4 Common Bug Patterns

| Pattern | Cause | Solution |
|---------|-------|----------|
| Null/undefined | Missing checks | Add validation |
| Race condition | Concurrency | Synchronization, transactions |
| Off-by-one | Index errors | Boundary testing |
| State mismatch | Stale data | Cache invalidation, state sync |
| Type confusion | Dynamic typing | Type checking, validation |
| Resource leak | Missing cleanup | defer, finally, cleanup functions |

### Phase 3: Fix Implementation

**Goal**: Fix the root cause, not the symptom.

#### 3.1 Fix Guidelines
```
□ Fix the root cause, not the symptom
□ Minimal change principle
□ Don't break existing functionality
□ Add regression test first (TDD style)
□ Consider edge cases
□ Document why the fix works
```

#### 3.2 Fix Structure
```go
// 1. Regression test (write first)
func TestCheckout_WithGitHubOAuthUser(t *testing.T) {
    user := &User{
        ID:    "github-123",
        Name:  "Test User",
        Email: "", // GitHub OAuth with hidden email
    }
    
    err := checkoutService.ProcessOrder(user, order)
    assert.NoError(t, err) // Should not crash
}

// 2. The fix
func (s *CheckoutService) ProcessOrder(user *User, order *Order) error {
    // Handle missing email
    if user.Email == "" {
        // Option 1: Use default/fallback
        user.Email = s.generateEmailPlaceholder(user.ID)
        
        // Option 2: Prompt user (better UX)
        return ErrEmailRequired
        
        // Option 3: Use OAuth provider's API to fetch
        email, err := s.oauthService.FetchEmail(user.ID)
        if err != nil {
            return fmt.Errorf("fetch email: %w", err)
        }
        user.Email = email
    }
    
    // ... rest of checkout logic
}
```

#### 3.3 Validation Checklist
```
□ Does it fix the reported issue?
□ Does it fix the root cause?
□ Are there side effects?
□ Does it work for all cases?
□ Is the code clean and maintainable?
```

### Phase 4: Verification

**Goal**: Confirm the fix works and doesn't break anything.

#### 4.1 Testing
```
□ Reproduction test passes
□ New regression test passes
□ Existing tests still pass
□ Edge cases tested
□ Integration tests pass
□ Manual verification
```

#### 4.2 Test Commands
```bash
# Run specific test
go test -run TestCheckout_WithGitHubOAuthUser ./...

# Run all tests
go test ./...

# Run with race detection
go test -race ./...

# Run integration tests
go test -tags=integration ./...
```

### Phase 5: Deployment

**Goal**: Deploy safely with monitoring.

#### 5.1 Pre-deployment
```
□ Code reviewed
□ Tests passing
□ Documentation updated
□ Monitoring/alerting in place
```

#### 5.2 Deployment Strategy
```
□ Deploy to staging first
□ Smoke tests in staging
□ Deploy to production (gradual if possible)
□ Monitor error rates
□ Monitor performance
```

#### 5.3 Post-deployment
```
□ Watch for 24-48 hours
□ Check error logs
□ Verify metrics are healthy
□ Communicate fix to stakeholders
```

## Debugging Tools by Language

### Go
```bash
# Debug with delve
dlv debug main.go
dlv test
dlv attach <pid>

# Race detection
go run -race main.go

# Profile
import "runtime/pprof"
```

### Rust
```bash
# Debug with gdb/lldb
cargo build
gdb target/debug/myapp

# Backtrace on panic
RUST_BACKTRACE=1 cargo run

# Use log crate
log::debug!("Variable x = {}", x);
```

### TypeScript/JavaScript
```bash
# Node.js debugging
node --inspect-brk server.ts

# Chrome DevTools
# 1. Add 'debugger;' statements
# 2. Open DevTools
# 3. Sources tab → find file

# Console logging strategies
console.log("Debug:", { user, order, config });
console.table(data);
console.trace("Called from:");
```

## Best Practices

### 1. Add Logging Before Fixing
```go
// Temporary logging
log.Printf("[BUG] Variable state: x=%v, y=%v", x, y)
log.Printf("[BUG] Stack trace: %s", debug.Stack())
```

### 2. Create Minimal Reproduction
```go
// Create test case that reproduces the bug
func TestBug_MissingEmail(t *testing.T) {
    // Minimal setup to reproduce
    user := &User{ID: "1"}
    err := sendEmail(user, "Hello")
    
    // Before fix: should fail
    // assert.Error(t, err)
    
    // After fix: should handle gracefully
    assert.NoError(t, err)
}
```

### 3. Document the Fix
```go
// Fix for GitHub issue #123
// Users created via OAuth without email were causing nil pointer
// on checkout. Now we gracefully handle missing emails.
func (s *Service) ProcessOrder(user *User, order *Order) error {
    if user.Email == "" {
        return ErrEmailRequired
    }
    // ...
}
```

### 4. Prevent Future Bugs
```go
// Add validation at creation time
type User struct {
    Email string `validate:"required,email"`
}

// Or database constraint
// ALTER TABLE users ADD CONSTRAINT email_required CHECK (email IS NOT NULL AND email != '');
```

## When to Use

Use this skill when:
- Debugging a reported bug
- Investigating production issues
- Writing regression tests
- Reviewing bug fixes
- Preventing bugs through design

## Common Anti-patterns to Avoid

```go
// DON'T: Fix the symptom
func process(user *User) {
    if user == nil {
        return // Silent failure - bad!
    }
}

// DO: Fix the root cause
func process(user *User) error {
    if user == nil {
        return fmt.Errorf("user is nil")
    }
    // ...
}

// DON'T: Comment out code instead of fixing
// if (buggyCondition) { ... }

// DO: Fix properly or delete entirely
```
