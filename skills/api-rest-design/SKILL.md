---
name: api-rest-design
description: RESTful API design patterns, versioning, and best practices
license: MIT
compatibility: opencode
---

# API REST Design Skill

## Overview

This skill provides guidelines for designing RESTful APIs that are intuitive, maintainable, and scalable.

## Core Principles

### 1. Resource-Based Design

```
✅ Good: /users, /orders, /products
❌ Bad: /getUsers, /createOrder, /deleteProduct

Resources should be nouns, not verbs.
HTTP methods define the action.
```

### 2. HTTP Methods

| Method | Action | Idempotent | Safe |
|--------|--------|------------|------|
| GET | Read resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Full update/replace | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

```http
GET    /users          # List all users
GET    /users/123      # Get specific user
POST   /users          # Create new user
PUT    /users/123      # Full update user 123
PATCH  /users/123      # Partial update user 123
DELETE /users/123      # Delete user 123
```

## URL Design

### 1. Resource Hierarchy

```http
✅ Good:
/users/123/orders           # Orders for user 123
/orders/456/items           # Items in order 456
/products/789/reviews       # Reviews for product 789

✅ Good (Alternative - flatter):
/orders?userId=123          # Filter orders by user
/order-items?orderId=456    # Filter items by order
```

### 2. Query Parameters

```http
✅ Filtering:
/users?status=active&role=admin
/orders?status=pending&from=2024-01-01&to=2024-01-31

✅ Sorting:
/users?sort=-createdAt,name      # Sort by createdAt DESC, name ASC
/products?sort=price,-popularity  # Sort by price ASC, popularity DESC

✅ Pagination:
/users?page=2&limit=20           # Offset-based
/users?cursor=eyJpZCI6MTIzfQ==&limit=20  # Cursor-based (recommended)

✅ Field Selection:
/users?fields=id,email,name      # Sparse fieldsets
```

### 3. URL Naming

```
✅ Good:
- /users (plural)
- /user-profiles (kebab-case)
- /order-items (compound words)

❌ Bad:
- /user (singular - inconsistent)
- /userProfiles (camelCase)
- /user_profiles (snake_case)
- /UserProfiles (PascalCase)
```

## Request/Response Design

### 1. Request Body (POST/PUT/PATCH)

```json
{
  "email": "user@example.com",
  "username": "johndoe",
  "profile": {
    "firstName": "John",
    "lastName": "Doe"
  },
  "preferences": {
    "newsletter": true,
    "notifications": {
      "email": true,
      "sms": false
    }
  }
}
```

### 2. Response Structure

```json
{
  "data": {
    "id": "usr_123",
    "email": "user@example.com",
    "username": "johndoe",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 3. Collection Response

```json
{
  "data": [
    { "id": "usr_1", "email": "user1@example.com" },
    { "id": "usr_2", "email": "user2@example.com" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "hasMore": true
  },
  "links": {
    "self": "/users?page=1&limit=20",
    "next": "/users?page=2&limit=20",
    "prev": null
  }
}
```

## HTTP Status Codes

### Success Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 OK | Success | GET, PUT, PATCH, DELETE |
| 201 Created | Resource created | POST |
| 202 Accepted | Async processing started | Async operations |
| 204 No Content | Success, no body | DELETE, empty response |
| 206 Partial Content | Partial success | Range requests |

### Client Error Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 400 Bad Request | Invalid input | Validation errors |
| 401 Unauthorized | Authentication required | Missing/invalid auth |
| 403 Forbidden | Insufficient permissions | Valid auth, no access |
| 404 Not Found | Resource doesn't exist | Invalid ID |
| 409 Conflict | Resource conflict | Duplicate entry |
| 422 Unprocessable | Semantic errors | Business logic errors |
| 429 Too Many Requests | Rate limit exceeded | Throttling |

### Server Error Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 500 Internal Server Error | Unexpected error | Unhandled exceptions |
| 502 Bad Gateway | Upstream error | Proxy/gateway issues |
| 503 Service Unavailable | Temporarily down | Maintenance/overload |

## Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Must be between 0 and 150"
      }
    ],
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

## Authentication & Security

### 1. Authentication Methods

```http
✅ API Keys:
GET /api/users
Authorization: ApiKey abc123xyz

✅ Bearer Tokens (JWT):
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

✅ OAuth 2.0:
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=CLIENT_ID&
client_secret=CLIENT_SECRET
```

### 2. Security Headers

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
```

### 3. Input Validation

```go
// Request DTO with validation
type CreateUserRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Username string `json:"username" validate:"required,min=3,max=50,alphanum"`
    Age      int    `json:"age" validate:"gte=0,lte=150"`
    Password string `json:"password" validate:"required,min=8"`
}

// Validation
if err := validate.Struct(req); err != nil {
    return c.Status(400).JSON(ErrorResponse{
        Code:    "VALIDATION_ERROR",
        Message: "Invalid request data",
        Details: formatValidationErrors(err),
    })
}
```

## API Versioning

### 1. URL Path Versioning (Recommended)

```http
/v1/users
/v2/users

✅ Pros:
- Clear and explicit
- Easy to route
- Cache-friendly

❌ Cons:
- URLs change
```

### 2. Header Versioning

```http
Accept: application/vnd.api+json;version=2
Api-Version: 2

✅ Pros:
- Clean URLs
- Flexible

❌ Cons:
- Harder to discover
- Cache issues
```

### 3. Deprecation Strategy

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jun 2024 00:00:00 GMT
Link: </v2/users>; rel="successor-version"

{
  "warning": "This API version is deprecated and will be removed on 2024-06-01"
}
```

## OpenAPI/Swagger Documentation

```yaml
openapi: 3.0.0
info:
  title: Users API
  version: 1.0.0
  description: API for user management

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
    
    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Validation error

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        email:
          type: string
          format: email
        username:
          type: string
        createdAt:
          type: string
          format: date-time
```

## Rate Limiting

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200

429 Too Many Requests
Retry-After: 60

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 60 seconds.",
    "retryAfter": 60
  }
}
```

## Best Practices

### DO:
- ✅ Use HTTPS in production
- ✅ Validate all inputs
- ✅ Use consistent naming conventions
- ✅ Version your API from day one
- ✅ Implement rate limiting
- ✅ Return appropriate status codes
- ✅ Support filtering, sorting, pagination
- ✅ Document with OpenAPI/Swagger
- ✅ Use caching headers
- ✅ Implement idempotency keys for POST

### DON'T:
- ❌ Expose internal errors to clients
- ❌ Use GET for operations with side effects
- ❌ Store sensitive data in URLs
- ❌ Ignore HTTP caching
- ❌ Break backward compatibility without versioning
- ❌ Return plain text instead of JSON
- ❌ Use overly nested resources (>3 levels)

## When to Use

Use this skill when:
- Designing new APIs
- Reviewing API designs
- Implementing API endpoints
- Writing API documentation
- Planning API versioning strategy
