# Secure Coding Patterns - Go

## Password Hashing with bcrypt

```go
package main

import (
    "golang.org/x/crypto/bcrypt"
    "fmt"
)

const bcryptCost = 12

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)
    if err != nil {
        return "", fmt.Errorf("failed to hash password: %w", err)
    }
    return string(bytes), nil
}

func CheckPasswordHash(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Usage
func main() {
    password := "securePassword123!"
    
    hash, err := HashPassword(password)
    if err != nil {
        panic(err)
    }
    
    match := CheckPasswordHash(password, hash)
    fmt.Printf("Password match: %v\n", match)
}
```

## Constant-Time Comparison

```go
package main

import (
    "crypto/subtle"
    "fmt"
)

// VerifySecret compares secrets in constant time to prevent timing attacks
func VerifySecret(provided, expected string) bool {
    return subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) == 1
}

// Usage - API key validation
func validateAPIKey(providedKey string) bool {
    expectedKey := getAPIKeyFromSecureStorage()
    return VerifySecret(providedKey, expectedKey)
}
```

## Secure Cookie Configuration

```go
package main

import (
    "net/http"
    "time"
)

func setSecureCookie(w http.ResponseWriter, name, value string) {
    cookie := &http.Cookie{
        Name:     name,
        Value:    value,
        HttpOnly: true,                      // Prevent JavaScript access
        Secure:   true,                      // HTTPS only
        SameSite: http.SameSiteStrictMode,   // CSRF protection
        MaxAge:   86400,                     // 24 hours
        Path:     "/",
        Expires:  time.Now().Add(24 * time.Hour),
    }
    http.SetCookie(w, cookie)
}
```

## SQL Injection Prevention

```go
package main

import (
    "database/sql"
    _ "github.com/lib/pq"
)

// ❌ DON'T: String formatting
func getUserUnsafe(db *sql.DB, username string) (*User, error) {
    query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
    row := db.QueryRow(query)
    // ...
}

// ✅ DO: Parameterized queries
func getUserSafe(db *sql.DB, username string) (*User, error) {
    query := "SELECT id, username, email FROM users WHERE username = $1"
    row := db.QueryRow(query, username)
    
    var user User
    err := row.Scan(&user.ID, &user.Username, &user.Email)
    if err != nil {
        return nil, err
    }
    return &user, nil
}

// ✅ DO: Using prepared statements
func getUserPrepared(db *sql.DB, username string) (*User, error) {
    stmt, err := db.Prepare("SELECT id, username, email FROM users WHERE username = $1")
    if err != nil {
        return nil, err
    }
    defer stmt.Close()
    
    var user User
    err = stmt.QueryRow(username).Scan(&user.ID, &user.Username, &user.Email)
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

## Input Validation

```go
package main

import (
    "fmt"
    "regexp"
    "strings"
    "unicode"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func ValidateEmail(email string) bool {
    return emailRegex.MatchString(email)
}

func ValidatePassword(password string) error {
    if len(password) < 12 {
        return fmt.Errorf("password must be at least 12 characters")
    }
    
    var hasUpper, hasLower, hasNumber, hasSpecial bool
    for _, char := range password {
        switch {
        case unicode.IsUpper(char):
            hasUpper = true
        case unicode.IsLower(char):
            hasLower = true
        case unicode.IsNumber(char):
            hasNumber = true
        case unicode.IsPunct(char) || unicode.IsSymbol(char):
            hasSpecial = true
        }
    }
    
    if !hasUpper || !hasLower || !hasNumber || !hasSpecial {
        return fmt.Errorf("password must contain uppercase, lowercase, number, and special character")
    }
    
    return nil
}

func SanitizeFilename(filename string) string {
    // Remove path components
    filename = filepath.Base(filename)
    
    // Remove null bytes
    filename = strings.ReplaceAll(filename, "\x00", "")
    
    // Remove dangerous characters
    filename = regexp.MustCompile(`[<>:"|?*]`).ReplaceAllString(filename, "")
    
    return filename
}
```

## JWT Implementation

```go
package main

import (
    "fmt"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

func GenerateJWT(userID, email, secret string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(1 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "your-app",
            Subject:   userID,
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}

func ValidateJWT(tokenString, secret string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return []byte(secret), nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, fmt.Errorf("invalid token")
}
```

## Secure HTTP Headers Middleware

```go
package main

import (
    "net/http"
)

func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Prevent MIME type sniffing
        w.Header().Set("X-Content-Type-Options", "nosniff")
        
        // Prevent clickjacking
        w.Header().Set("X-Frame-Options", "DENY")
        
        // XSS protection
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        
        // Enforce HTTPS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        
        // Control referrer information
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // Content Security Policy
        w.Header().Set("Content-Security-Policy", 
            "default-src 'self'; "+
            "script-src 'self'; "+
            "style-src 'self' 'unsafe-inline'; "+
            "img-src 'self' data: https:; "+
            "font-src 'self'; "+
            "connect-src 'self'; "+
            "media-src 'self'; "+
            "object-src 'none'; "+
            "frame-src 'none'; "+
            "base-uri 'self';")
        
        // Permissions Policy
        w.Header().Set("Permissions-Policy", 
            "geolocation=(), "+
            "microphone=(), "+
            "camera=(), "+
            "payment=()")
        
        next.ServeHTTP(w, r)
    })
}

// Usage
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)
    
    // Apply security middleware
    secureMux := SecurityHeadersMiddleware(mux)
    
    http.ListenAndServe(":8080", secureMux)
}
```

## Rate Limiting

```go
package main

import (
    "net/http"
    "sync"
    "time"
)

type RateLimiter struct {
    requests map[string][]time.Time
    mu       sync.RWMutex
    maxReq   int
    window   time.Duration
}

func NewRateLimiter(maxReq int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        requests: make(map[string][]time.Time),
        maxReq:   maxReq,
        window:   window,
    }
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-rl.window)
    
    // Clean old requests
    var validRequests []time.Time
    for _, req := range rl.requests[key] {
        if req.After(windowStart) {
            validRequests = append(validRequests, req)
        }
    }
    
    // Check limit
    if len(validRequests) >= rl.maxReq {
        rl.requests[key] = validRequests
        return false
    }
    
    // Add new request
    validRequests = append(validRequests, now)
    rl.requests[key] = validRequests
    
    return true
}

func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Use IP address as key
            key := r.RemoteAddr
            
            if !limiter.Allow(key) {
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
func main() {
    limiter := NewRateLimiter(100, 1*time.Minute)
    
    mux := http.NewServeMux()
    mux.HandleFunc("/api/", apiHandler)
    
    // Apply rate limiting
    rateLimitedMux := RateLimitMiddleware(limiter)(mux)
    
    http.ListenAndServe(":8080", rateLimitedMux)
}
```

## File Upload Validation

```go
package main

import (
    "fmt"
    "io"
    "mime/multipart"
    "net/http"
    "path/filepath"
    "strings"
)

var allowedMimeTypes = map[string]bool{
    "image/jpeg": true,
    "image/png":  true,
    "image/gif":  true,
    "application/pdf": true,
}

var allowedExtensions = map[string]bool{
    ".jpg":  true,
    ".jpeg": true,
    ".png":  true,
    ".gif":  true,
    ".pdf":  true,
}

func validateFileUpload(file multipart.File, header *multipart.FileHeader) error {
    // Check file size (e.g., 5MB limit)
    const maxSize = 5 * 1024 * 1024
    if header.Size > maxSize {
        return fmt.Errorf("file too large: max size is 5MB")
    }
    
    // Check extension
    ext := strings.ToLower(filepath.Ext(header.Filename))
    if !allowedExtensions[ext] {
        return fmt.Errorf("invalid file extension: %s", ext)
    }
    
    // Check MIME type (read first 512 bytes)
    buffer := make([]byte, 512)
    _, err := file.Read(buffer)
    if err != nil && err != io.EOF {
        return fmt.Errorf("failed to read file: %w", err)
    }
    
    // Reset file pointer
    file.Seek(0, 0)
    
    mimeType := http.DetectContentType(buffer)
    if !allowedMimeTypes[mimeType] {
        return fmt.Errorf("invalid file type: %s", mimeType)
    }
    
    return nil
}

func uploadHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // Parse multipart form (32MB max memory)
    err := r.ParseMultipartForm(32 << 20)
    if err != nil {
        http.Error(w, "Failed to parse form", http.StatusBadRequest)
        return
    }
    
    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "Failed to get file", http.StatusBadRequest)
        return
    }
    defer file.Close()
    
    // Validate file
    if err := validateFileUpload(file, header); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Save file...
    fmt.Fprintf(w, "File %s uploaded successfully", header.Filename)
}
```

## Path Traversal Prevention

```go
package main

import (
    "fmt"
    "path/filepath"
    "strings"
)

func safeFilePath(basePath, userInput string) (string, error) {
    // Clean and get absolute path
    base := filepath.Clean(basePath)
    target := filepath.Clean(filepath.Join(base, userInput))
    
    // Ensure target is within base directory
    if !strings.HasPrefix(target, base) {
        return "", fmt.Errorf("path traversal attempt detected")
    }
    
    return target, nil
}

// Usage
func serveFile(basePath, filename string) ([]byte, error) {
    safePath, err := safeFilePath(basePath, filename)
    if err != nil {
        return nil, err
    }
    
    return os.ReadFile(safePath)
}
```
