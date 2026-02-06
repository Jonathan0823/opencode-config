---
name: go-conventions
description: Go conventions, best practices, and idiomatic patterns for production code
license: MIT
compatibility: opencode
---

# Go Conventions Skill

## Overview

This skill provides guidelines for writing idiomatic, production-ready Go code following Go conventions, best practices, and patterns optimized for backend development with Gin and standard library.

## Core Principles

### 1. Idiomatic Go

- Follow Go's philosophy: "Simple is better"
- Prefer composition over inheritance
- Use small interfaces
- Write self-documenting code with clear variable names
- Keep functions short and focused

### 2. Error Handling

```go
// DO: Explicit error handling with context
result, err := processData(input)
if err != nil {
    return fmt.Errorf("failed to process data: %w", err)
}

// DON'T: Ignore errors
result, _ := processData(input) // Never do this

// DO: Wrap errors for context
return fmt.Errorf("database operation failed: %w", err)

// DO: Use custom error types for domain-specific errors
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s: %s", e.Field, e.Message)
}
```

### 3. Concurrency Patterns

```go
// DO: Use context for cancellation
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    // ...
}

// DO: Use channels with select for timeouts
select {
case result := <-ch:
    return result, nil
case <-time.After(timeout):
    return nil, errors.New("timeout")
case <-ctx.Done():
    return nil, ctx.Err()
}

// DO: Use sync.WaitGroup for goroutine coordination
var wg sync.WaitGroup
for _, task := range tasks {
    wg.Add(1)
    go func(t Task) {
        defer wg.Done()
        process(t)
    }(task)
}
wg.Wait()
```

### 4. Struct and Interface Design

```go
// DO: Small interfaces (Interface Segregation)
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// DO: Constructor patterns for complex structs
func NewService(db *sql.DB, cache *redis.Client, logger *zap.Logger) *Service {
    return &Service{
        db:     db,
        cache:  cache,
        logger: logger,
    }
}

// DO: Method receivers - pointer vs value
func (s *Service) Update(data Data) error  // Pointer for mutation
func (c Config) Validate() error            // Value for read-only
```

### 5. Package Organization

```
myproject/
├── cmd/
│   └── api/              # Main application entry point
│       └── main.go
├── internal/
│   ├── domain/           # Business logic (entities, use cases)
│   ├── repository/       # Data access layer
│   ├── service/          # Business services
│   └── handler/          # HTTP handlers
├── pkg/
│   ├── middleware/       # Shared middleware
│   └── utils/           # Shared utilities
└── go.mod
```

## Gin-Specific Patterns

### 1. Handler Structure

```go
// DO: Use dependency injection
func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    user, err := h.service.CreateUser(c.Request.Context(), req)
    if err != nil {
        handleError(c, err)
        return
    }
    
    c.JSON(http.StatusCreated, user)
}

// DO: Centralized error handling
func handleError(c *gin.Context, err error) {
    switch e := err.(type) {
    case *ValidationError:
        c.JSON(http.StatusBadRequest, gin.H{"error": e.Error()})
    case *NotFoundError:
        c.JSON(http.StatusNotFound, gin.H{"error": e.Error()})
    default:
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    }
}
```

### 2. Middleware Patterns

```go
// DO: Authentication middleware
func AuthMiddleware(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }
        
        claims, err := validateToken(token, secret)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
            return
        }
        
        c.Set("user", claims)
        c.Next()
    }
}

// DO: Logging middleware
func LoggerMiddleware(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        
        c.Next()
        
        logger.Info("request",
            zap.String("method", c.Request.Method),
            zap.String("path", path),
            zap.Int("status", c.Writer.Status()),
            zap.Duration("duration", time.Since(start)),
        )
    }
}
```

## Database Patterns (PostgreSQL)

```go
// DO: Repository pattern
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id int64) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, opts ListOptions) ([]User, error)
}

// DO: Transaction handling
func (r *userRepo) CreateWithProfile(ctx context.Context, user *User, profile *Profile) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    if err := r.createUserTx(ctx, tx, user); err != nil {
        return err
    }
    
    if err := r.createProfileTx(ctx, tx, profile); err != nil {
        return err
    }
    
    return tx.Commit()
}
```

## Testing Patterns

```go
// DO: Table-driven tests
func TestCalculateTotal(t *testing.T) {
    tests := []struct {
        name     string
        items    []Item
        expected float64
    }{
        {
            name:     "empty cart",
            items:    []Item{},
            expected: 0,
        },
        {
            name: "single item",
            items: []Item{{Price: 10, Quantity: 2}},
            expected: 20,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := calculateTotal(tt.items)
            assert.Equal(t, tt.expected, result)
        })
    }
}

// DO: Mock interfaces for testing
type MockUserService struct {
    mock.Mock
}

func (m *MockUserService) GetUser(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*User), args.Error(1)
}
```

## When to Use

Use this skill when:
- Writing or reviewing Go code
- Designing Go architecture
- Refactoring Go projects
- Implementing Go backend services with Gin
- Writing Go tests
- Setting up Go project structure
