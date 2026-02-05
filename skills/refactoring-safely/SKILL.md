---
name: refactoring-safely
description: Safe refactoring practices with tests and incremental changes
---

# Safe Refactoring Skill

## Overview

This skill provides guidelines for refactoring code safely while maintaining functionality and preventing regressions.

## Golden Rules of Refactoring

```
1. NEVER refactor without tests
2. Make small, incremental changes
3. Run tests after EVERY change
4. Use version control (commit frequently)
5. Have a rollback plan
```

## The Safe Refactoring Process

### Phase 1: Preparation

#### 1.1 Understand the Code
```
□ Read and understand the current implementation
□ Identify dependencies
□ Document current behavior
□ Note edge cases
□ Check for side effects
```

#### 1.2 Ensure Test Coverage
```
□ Write tests BEFORE refactoring if none exist
□ Aim for >80% coverage on code to refactor
□ Include edge cases in tests
□ Add characterization tests (document current behavior)

// Characterization test - captures current behavior
func TestUserService_GetUser_Characterization(t *testing.T) {
    service := NewUserService(mockRepo)
    
    // Test current behavior before refactoring
    user, err := service.GetUser("123")
    
    // Even if behavior seems wrong, document it
    assert.Equal(t, "John", user.Name)
    assert.Nil(t, err)
}
```

#### 1.3 Create Safety Net
```bash
# Ensure you can revert easily
git checkout -b refactor/user-service

# Commit current state
git add .
git commit -m "chore: pre-refactor checkpoint"
```

### Phase 2: Incremental Refactoring

#### 2.1 Start with Low-Risk Changes

**Order of Safety (safest first):**
1. Rename variables/functions (mechanical)
2. Extract methods/functions
3. Extract classes/modules
4. Change data structures
5. Modify algorithms
6. Change interfaces/contracts

#### 2.2 Rename Safely
```go
// Before
func calc(a, b int) int {
    return a + b
}

// After (with IDE rename)
func calculateSum(firstOperand, secondOperand int) int {
    return firstOperand + secondOperand
}

// Test should pass immediately after rename
```

#### 2.3 Extract Method
```go
// Before
func processOrder(order *Order) error {
    // Validate order
    if order.Total <= 0 {
        return fmt.Errorf("invalid total")
    }
    if len(order.Items) == 0 {
        return fmt.Errorf("no items")
    }
    
    // Calculate taxes
    tax := order.Total * 0.08
    order.Tax = tax
    order.Total += tax
    
    // Save to DB
    return db.Save(order)
}

// After - extract methods
func processOrder(order *Order) error {
    if err := validateOrder(order); err != nil {
        return err
    }
    
    calculateTaxes(order)
    
    return db.Save(order)
}

func validateOrder(order *Order) error {
    if order.Total <= 0 {
        return fmt.Errorf("invalid total")
    }
    if len(order.Items) == 0 {
        return fmt.Errorf("no items")
    }
    return nil
}

func calculateTaxes(order *Order) {
    tax := order.Total * 0.08
    order.Tax = tax
    order.Total += tax
}
```

#### 2.4 Extract Class/Module
```go
// Before: God class with many responsibilities
type OrderService struct {
    db *sql.DB
}

func (s *OrderService) CreateOrder(data OrderData) error  { ... }
func (s *OrderService) ValidateOrder(order Order) error   { ... }
func (s *OrderService) CalculateTaxes(order Order) error  { ... }
func (s *OrderService) SendNotification(order Order) error{ ... }

// After: Separated responsibilities
type OrderService struct {
    db         *sql.DB
    validator  *OrderValidator
    calculator *TaxCalculator
    notifier   *NotificationService
}

func NewOrderService(db *sql.DB, notifier NotificationService) *OrderService {
    return &OrderService{
        db:         db,
        validator:  NewOrderValidator(),
        calculator: NewTaxCalculator(),
        notifier:   notifier,
    }
}

func (s *OrderService) CreateOrder(data OrderData) error {
    order := NewOrder(data)
    
    if err := s.validator.Validate(order); err != nil {
        return err
    }
    
    s.calculator.ApplyTaxes(order)
    
    if err := s.db.Save(order); err != nil {
        return err
    }
    
    return s.notifier.SendOrderConfirmation(order)
}
```

### Phase 3: Common Refactoring Patterns

#### 3.1 Replace Conditional with Polymorphism
```go
// Before
func calculateShipping(order Order) float64 {
    switch order.Type {
    case "standard":
        return 5.99
    case "express":
        return 15.99
    case "free":
        return 0
    default:
        return 5.99
    }
}

// After
type ShippingCalculator interface {
    Calculate(order Order) float64
}

type StandardShipping struct{}
func (s StandardShipping) Calculate(order Order) float64 { return 5.99 }

type ExpressShipping struct{}
func (e ExpressShipping) Calculate(order Order) float64 { return 15.99 }

type FreeShipping struct{}
func (f FreeShipping) Calculate(order Order) float64 { return 0 }

func NewShippingCalculator(orderType string) ShippingCalculator {
    switch orderType {
    case "express":
        return ExpressShipping{}
    case "free":
        return FreeShipping{}
    default:
        return StandardShipping{}
    }
}
```

#### 3.2 Introduce Parameter Object
```go
// Before
func createUser(
    firstName, lastName, email string,
    age int,
    street, city, zip string,
) (*User, error) {
    // ...
}

// After
type CreateUserRequest struct {
    FirstName string
    LastName  string
    Email     string
    Age       int
    Address   Address
}

type Address struct {
    Street string
    City   string
    Zip    string
}

func createUser(req CreateUserRequest) (*User, error) {
    // Validation logic here
    // ...
}
```

#### 3.3 Replace Magic Numbers with Constants
```go
// Before
if timeout > 30000 { ... }

// After
const (
    DefaultTimeoutMs = 30000
    MaxRetries       = 3
    DefaultPageSize  = 20
)

if timeout > DefaultTimeoutMs { ... }
```

#### 3.4 Remove Dead Code
```go
// Before
func process(data Data) Result {
    // Old implementation commented out
    // for i, item := range data.Items { ... }
    
    // New implementation
    return newProcess(data)
}

// After
func process(data Data) Result {
    return newProcess(data)
}
```

### Phase 4: Testing During Refactoring

#### 4.1 Commit After Each Change
```bash
# After each safe change
git add .
git commit -m "refactor: extract validateOrder method"

# Run tests
go test ./...

# If tests pass, continue
# If tests fail, debug or revert
git stash  # or git reset --hard HEAD~1
```

#### 4.2 Test Coverage Checklist
```
□ Unit tests for extracted methods
□ Integration tests still pass
□ Edge cases handled
□ Error paths tested
□ Performance acceptable
```

### Phase 5: Database Refactoring

#### 5.1 Expand-Contract Pattern
```sql
-- Phase 1: Expand (add new structure)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Update code to write to both columns
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name);

-- Phase 2: Migrate (update all data)
-- Backfill data
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name) 
WHERE full_name IS NULL;

-- Phase 3: Contract (remove old structure)
-- Update code to read from new column only
-- Then:
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

#### 5.2 Zero-Downtime Migrations
```sql
-- 1. Add new column (nullable)
ALTER TABLE orders ADD COLUMN total_cents BIGINT;

-- 2. Update code to write to both columns
-- 3. Backfill data in batches
UPDATE orders 
SET total_cents = total * 100 
WHERE total_cents IS NULL 
LIMIT 1000;

-- 4. Make new column NOT NULL after backfill
ALTER TABLE orders ALTER COLUMN total_cents SET NOT NULL;

-- 5. Update code to read from new column only
-- 6. Drop old column
ALTER TABLE orders DROP COLUMN total;
```

## Language-Specific Techniques

### Go
```go
// Use interfaces for testability
// Before: Hard to test
type Service struct {
    db *sql.DB
}

// After: Easy to mock
type Database interface {
    Query(query string, args ...interface{}) (*sql.Rows, error)
}

type Service struct {
    db Database
}

// Embed interfaces for composition
type ReadWriter interface {
    io.Reader
    io.Writer
}
```

### TypeScript
```typescript
// Use type guards
function isAdmin(user: User): user is Admin {
    return user.role === 'admin';
}

if (isAdmin(user)) {
    user.adminOnlyMethod(); // Type-safe
}

// Discriminated unions
type Response = 
    | { type: 'success'; data: User }
    | { type: 'error'; error: Error }
    | { type: 'loading' };

function handleResponse(response: Response) {
    switch (response.type) {
        case 'success':
            return response.data;
        case 'error':
            return response.error;
        case 'loading':
            return null;
    }
}
```

### Rust
```rust
// Use newtype pattern for type safety
struct UserId(String);
struct OrderId(String);

fn get_user(id: UserId) -> User { ... }
fn get_order(id: OrderId) -> Order { ... }

// Can't mix up IDs:
let user_id = UserId("123".to_string());
get_order(user_id); // Compile error!

// Use Result for error handling
fn parse_config(path: &str) -> Result<Config, ConfigError> {
    let content = fs::read_to_string(path)?;
    let config: Config = toml::from_str(&content)?;
    Ok(config)
}
```

## When to Refactor

### Good Times to Refactor
```
□ Before adding new features (clean up first)
□ After fixing bugs (prevent future bugs)
□ During code review (incremental improvement)
□ When code is hard to test (improve testability)
□ When performance is poor (optimize structure)
□ When onboarding new developers (improve clarity)
```

### Bad Times to Refactor
```
✗ Right before a deadline
✗ Without tests
✗ While debugging unrelated issues
✗ When you're tired or rushed
✗ Without version control
✗ When you don't understand the code
```

## Common Refactoring Mistakes

```go
// DON'T: Change behavior while refactoring
// Refactoring should preserve existing behavior

// DON'T: Refactor and add features simultaneously
// Do one thing at a time

// DON'T: Skip tests
// Tests are your safety net

// DON'T: Make large changes at once
// Small, incremental changes are safer

// DON'T: Refactor code you're not touching
// Leave working code alone
```

## When to Use

Use this skill when:
- Improving code structure without changing behavior
- Paying down technical debt
- Making code more maintainable
- Preparing for new features
- Improving testability
- Code reviews
