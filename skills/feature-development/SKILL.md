---
name: feature-development
description: End-to-end feature development workflow from research to deployment
license: MIT
compatibility: opencode
---

# Feature Development Skill

## Overview

This skill provides a structured workflow for developing features from concept to deployment, ensuring consistent quality and maintainability.

## The Feature Development Process

### Phase 1: Research & Discovery (0-2 hours)

**Goal**: Understand the problem deeply before writing code.

#### 1.1 Requirements Analysis
```
□ Identify stakeholder needs
□ Clarify acceptance criteria
□ Define scope boundaries (what's in/out)
□ Identify dependencies and blockers
□ Estimate complexity
```

**Questions to ask**:
- Who are the users?
- What problem does this solve?
- What does "done" look like?
- Are there existing patterns/solutions to leverage?
- What are the edge cases?

#### 1.2 Technical Research
```
□ Review existing codebase for similar features
□ Check for reusable components/utilities
□ Identify required libraries/frameworks
□ Research best practices for this type of feature
□ Consider performance implications
□ Review security requirements
```

#### 1.3 Create Research Summary
Document findings in comments or a separate file:
```markdown
## Feature: [Name]

### Requirements
- [List key requirements]

### Technical Approach
- [Architecture decision]

### Dependencies
- [List dependencies]

### Risks
- [Potential blockers]

### References
- [Links to docs, similar code]
```

### Phase 2: Planning & Design (1-3 hours)

**Goal**: Create a clear roadmap before coding.

#### 2.1 Architecture Design
- **Choose patterns**: MVC, Repository, CQRS, etc.
- **Define data models**: Entities, DTOs, API contracts
- **Design APIs**: Endpoints, request/response schemas
- **Plan UI components**: Component hierarchy, state management

#### 2.2 Task Breakdown
Break feature into small, testable chunks:
```markdown
## Implementation Plan

### Task 1: [Name]
- **Scope**: [What this task accomplishes]
- **Files to touch**: [List]
- **Estimated time**: [Hours]
- **Dependencies**: [None/Task X]

### Task 2: [Name]
...
```

**Guideline**: Each task should be completable in 2-4 hours.

#### 2.3 Define Interfaces First
Create contracts/interfaces before implementations:

```go
// repository.go - Define interface first
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// Implementation comes later
type postgresUserRepo struct {
    db *sql.DB
}
```

### Phase 3: Implementation (50-70% of time)

**Goal**: Build the feature incrementally with tests.

#### 3.1 Development Order
1. **Data layer first**: Models, repositories, database schema
2. **Business logic**: Services, domain logic
3. **API layer**: Controllers, handlers, routes
4. **UI layer**: Components, pages (if applicable)
5. **Integration**: Wire everything together

#### 3.2 Code Quality Gates
Before committing:
```
□ Code follows language conventions
□ No obvious security issues (input validation, auth)
□ Error handling is comprehensive
□ Logging is appropriate
□ No hardcoded values (use config)
□ No commented-out code
□ Functions are focused and small
```

#### 3.3 Commit Strategy
```
Commit 1: feat(data): add user model and migrations
Commit 2: feat(repo): implement user repository
Commit 3: feat(service): add user service layer
Commit 4: feat(api): create user endpoints
Commit 5: feat(ui): add user list component
Commit 6: test: add unit tests for user service
Commit 7: test: add integration tests
```

### Phase 4: Testing (20-30% of time)

**Goal**: Ensure feature works correctly and reliably.

#### 4.1 Test Pyramid
```
    /\
   /  \  E2E Tests (few)
  /____\
 /      \ Integration Tests (some)
/________\
Unit Tests (many)
```

#### 4.2 Unit Testing
- **Coverage target**: 80%+ for business logic
- **Mock external dependencies**: DB, API, filesystem
- **Test edge cases**: Null, empty, max values
- **Arrange-Act-Assert pattern**:

```typescript
// Good test structure
it('should create user with valid data', async () => {
  // Arrange
  const userData = { name: 'John', email: 'john@example.com' };
  const mockRepo = { create: jest.fn().mockResolvedValue({ id: '1', ...userData }) };
  const service = new UserService(mockRepo);
  
  // Act
  const result = await service.create(userData);
  
  // Assert
  expect(result).toHaveProperty('id');
  expect(mockRepo.create).toHaveBeenCalledWith(userData);
});
```

#### 4.3 Integration Testing
- Test component interactions
- Use test database or in-memory DB
- Test API endpoints with real HTTP calls

```typescript
// Example: API integration test
describe('POST /api/users', () => {
  it('should create a new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'John', email: 'john@example.com' });
    
    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty('id');
  });
  
  it('should return 400 for invalid email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'John', email: 'invalid' });
    
    expect(response.status).toBe(400);
  });
});
```

#### 4.4 E2E Testing (Critical Paths Only)
- Test complete user journeys
- Focus on happy paths and critical error paths
- Use tools like Playwright, Cypress, or Selenium

### Phase 5: Review & Refinement (10-20% of time)

**Goal**: Polish and prepare for deployment.

#### 5.1 Code Review Checklist
```
□ Does it meet requirements?
□ Is code readable and maintainable?
□ Are edge cases handled?
□ Is error handling comprehensive?
□ Are there any security issues?
□ Is performance acceptable?
□ Is it tested?
□ Is documentation updated?
```

#### 5.2 Performance Review
```
□ Database queries are optimized (N+1 avoided)
□ No unnecessary re-renders (frontend)
□ Lazy loading implemented where needed
□ Caching strategy appropriate
□ Bundle size acceptable (frontend)
```

#### 5.3 Documentation
Update:
- README.md (setup instructions)
- API documentation (OpenAPI/Swagger)
- Code comments (complex logic)
- Changelog

### Phase 6: Deployment (varies)

**Goal**: Deploy safely to production.

#### 6.1 Pre-deployment Checklist
```
□ All tests passing
□ Migrations tested
□ Feature flags configured (if using)
□ Rollback plan ready
□ Monitoring alerts configured
```

#### 6.2 Deployment Strategy
- **Blue-green**: Zero downtime
- **Canary**: Gradual rollout
- **Feature flags**: Enable/disable without deploy

#### 6.3 Post-deployment
```
□ Monitor error rates
□ Check performance metrics
□ Verify feature works in production
□ Communicate to stakeholders
```

## Best Practices

### 1. Progressive Enhancement
Build MVP first, then enhance:
```
MVP → Add validation → Add caching → Add analytics → Optimize
```

### 2. Feature Flags
Use feature flags for risky changes:
```go
if featureflags.IsEnabled("new-checkout") {
    return newCheckoutProcess()
}
return oldCheckoutProcess()
```

### 3. Database Migrations
Always have rollback plan:
```sql
-- Migration: Add users table
CREATE TABLE users (...);

-- Rollback: Drop users table
-- DROP TABLE users;
```

### 4. Monitoring
Add metrics from day one:
```go
// Track performance
start := time.Now()
result := processOrder(order)
duration := time.Since(start)
metrics.RecordHistogram("order_processing_duration", duration)

// Track errors
if err != nil {
    metrics.IncrementCounter("order_processing_errors")
    logger.Error("failed to process order", zap.Error(err))
}
```

## When to Use

Use this skill when:
- Starting a new feature
- Planning a complex implementation
- Breaking down large requirements
- Reviewing feature development process
- Onboarding new team members
