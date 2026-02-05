---
name: security-best-practices
description: Security best practices, OWASP guidelines, secure coding patterns, and vulnerability prevention
---

# Security Best Practices Skill

## Overview

This skill provides comprehensive security guidelines following OWASP Top 10, secure coding patterns, authentication/authorization best practices, secrets management, and vulnerability scanning across multiple languages and frameworks.

## OWASP Top 10 Prevention

### 1. Injection Attacks (SQL, NoSQL, Command, LDAP)

```python
# ❌ DON'T: String concatenation for SQL queries
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)  # Vulnerable to SQL injection

# ✅ DO: Use parameterized queries
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))

# ✅ DO: Use ORM with proper escaping
user = User.objects.filter(username=username).first()

# JavaScript/Node.js
// ❌ DON'T: Template strings for queries
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ✅ DO: Parameterized queries with prepared statements
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId]);

// Go
// ❌ DON'T: String formatting
query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", userID)

// ✅ DO: Use database driver parameters
query := "SELECT * FROM users WHERE id = ?"
rows, err := db.Query(query, userID)
```

### 2. Broken Authentication

```python
# ✅ DO: Implement strong password requirements
import re
from typing import Optional

def validate_password(password: str) -> Optional[str]:
    """Validate password strength. Returns error message if invalid."""
    if len(password) < 12:
        return "Password must be at least 12 characters"
    if not re.search(r'[A-Z]', password):
        return "Password must contain at least one uppercase letter"
    if not re.search(r'[a-z]', password):
        return "Password must contain at least one lowercase letter"
    if not re.search(r'\d', password):
        return "Password must contain at least one digit"
    if not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
        return "Password must contain at least one special character"
    return None

# ✅ DO: Implement proper session management
import secrets
import hashlib
from datetime import datetime, timedelta

class SessionManager:
    def __init__(self):
        self.sessions = {}
    
    def create_session(self, user_id: str) -> str:
        """Create a secure session token."""
        # Generate cryptographically secure token
        token = secrets.token_urlsafe(32)
        session_hash = hashlib.sha256(token.encode()).hexdigest()
        
        self.sessions[session_hash] = {
            'user_id': user_id,
            'created_at': datetime.utcnow(),
            'expires_at': datetime.utcnow() + timedelta(hours=24)
        }
        
        return token
    
    def validate_session(self, token: str) -> Optional[dict]:
        """Validate session token."""
        session_hash = hashlib.sha256(token.encode()).hexdigest()
        session = self.sessions.get(session_hash)
        
        if not session:
            return None
        
        if datetime.utcnow() > session['expires_at']:
            del self.sessions[session_hash]
            return None
        
        return session

# ✅ DO: Implement rate limiting
from functools import wraps
import time
from typing import Dict, List

class RateLimiter:
    def __init__(self, max_requests: int = 5, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests: Dict[str, List[float]] = {}
    
    def is_allowed(self, key: str) -> bool:
        now = time.time()
        window_start = now - self.window_seconds
        
        # Clean old requests
        if key in self.requests:
            self.requests[key] = [t for t in self.requests[key] if t > window_start]
        else:
            self.requests[key] = []
        
        # Check if under limit
        if len(self.requests[key]) >= self.max_requests:
            return False
        
        self.requests[key].append(now)
        return True

def rate_limit(max_requests: int = 5, window_seconds: int = 60):
    limiter = RateLimiter(max_requests, window_seconds)
    
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            # Use IP or user ID as key
            key = kwargs.get('user_id', 'anonymous')
            if not limiter.is_allowed(key):
                raise Exception("Rate limit exceeded")
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

### 3. Sensitive Data Exposure

```python
# ✅ DO: Encrypt sensitive data at rest
from cryptography.fernet import Fernet
import os

class EncryptionService:
    def __init__(self):
        # Load key from secure environment
        key = os.environ.get('ENCRYPTION_KEY')
        if not key:
            raise ValueError("ENCRYPTION_KEY not set")
        self.cipher = Fernet(key)
    
    def encrypt(self, data: str) -> bytes:
        """Encrypt sensitive data."""
        return self.cipher.encrypt(data.encode())
    
    def decrypt(self, encrypted_data: bytes) -> str:
        """Decrypt sensitive data."""
        return self.cipher.decrypt(encrypted_data).decode()

# ✅ DO: Hash passwords with salt
import bcrypt

def hash_password(password: str) -> str:
    """Hash password using bcrypt."""
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(password.encode(), salt).decode()

def verify_password(password: str, hashed: str) -> bool:
    """Verify password against hash."""
    return bcrypt.checkpw(password.encode(), hashed.encode())

# ✅ DO: Use HTTPS and secure headers
# FastAPI example
from fastapi import FastAPI
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# Force HTTPS
app.add_middleware(HTTPSRedirectMiddleware)

# Restrict hosts
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com", "*.example.com"])

@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    return response
```

### 4. XML External Entities (XXE)

```python
# ❌ DON'T: Parse XML with external entities enabled
import xml.etree.ElementTree as ET

# Vulnerable to XXE
tree = ET.parse(xml_file)

# ✅ DO: Disable external entities
from defusedxml import ElementTree as ET

tree = ET.parse(xml_file)  # Safe from XXE

# Java
// ❌ DON'T: Enable external entities
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", false);

// ✅ DO: Disable external entities
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

### 5. Broken Access Control

```python
# ✅ DO: Implement proper authorization checks
from functools import wraps
from typing import Callable

class AuthorizationError(Exception):
    pass

def require_permission(permission: str):
    """Decorator to require specific permission."""
    def decorator(f: Callable):
        @wraps(f)
        def wrapper(user, *args, **kwargs):
            if permission not in user.permissions:
                raise AuthorizationError(f"Missing permission: {permission}")
            return f(user, *args, **kwargs)
        return wrapper
    return decorator

# Usage
@require_permission("users:read")
def get_user(user, user_id: int):
    return User.objects.get(id=user_id)

@require_permission("users:delete")
def delete_user(user, user_id: int):
    User.objects.filter(id=user_id).delete()

# ✅ DO: Implement resource-level access control
class ResourceAccessControl:
    def can_access(self, user, resource, action: str) -> bool:
        """Check if user can perform action on resource."""
        # Check ownership
        if resource.owner_id == user.id:
            return True
        
        # Check role-based permissions
        if user.role == "admin":
            return True
        
        # Check specific permissions
        return self._check_permission(user, resource, action)
    
    def _check_permission(self, user, resource, action: str) -> bool:
        # Implementation specific to your authorization model
        pass
```

### 6. Security Misconfiguration

```yaml
# ✅ DO: Secure configuration files
# docker-compose.yml example
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - DEBUG=false
      - SECRET_KEY=${SECRET_KEY}  # From environment
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
```

### 7. Cross-Site Scripting (XSS)

```python
# ✅ DO: Escape output in templates
# Jinja2 example
from markupsafe import Markup, escape

@app.route('/profile/<username>')
def profile(username):
    # Automatically escaped by Jinja2
    return render_template('profile.html', username=username)

# ✅ DO: Use Content Security Policy
# In HTML response headers
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}';

# ✅ DO: Sanitize user input
import bleach

def sanitize_html(user_input: str) -> str:
    """Sanitize HTML to prevent XSS."""
    allowed_tags = ['p', 'br', 'strong', 'em', 'u']
    allowed_attributes = {}
    return bleach.clean(user_input, tags=allowed_tags, attributes=allowed_attributes)
```

### 8. Insecure Deserialization

```python
# ❌ DON'T: Use pickle for untrusted data
import pickle

data = pickle.loads(untrusted_data)  # Dangerous!

# ✅ DO: Use safe serialization formats
import json

# Safe alternative
data = json.loads(untrusted_data)

# ✅ DO: If you must use pickle, implement signing
import hmac
import hashlib
import pickle

def secure_pickle_loads(data: bytes, secret_key: bytes) -> any:
    """Load pickled data with HMAC verification."""
    if len(data) < 32:
        raise ValueError("Invalid data")
    
    signature = data[:32]
    payload = data[32:]
    
    expected = hmac.new(secret_key, payload, hashlib.sha256).digest()
    if not hmac.compare_digest(signature, expected):
        raise ValueError("Invalid signature")
    
    return pickle.loads(payload)
```

### 9. Using Components with Known Vulnerabilities

```bash
# ✅ DO: Regular dependency scanning
# Python
pip install safety
safety check

# Node.js
npm audit
npm audit fix

# Java
./mvnw dependency:check

# ✅ DO: Use dependency scanning in CI/CD
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          format: 'sarif'
          output: 'trivy-results.sarif'
```

### 10. Insufficient Logging and Monitoring

```python
# ✅ DO: Implement comprehensive logging
import logging
import structlog
from datetime import datetime

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

def log_security_event(event_type: str, user_id: str, details: dict):
    """Log security-related events."""
    logger.info(
        "security_event",
        event_type=event_type,
        user_id=user_id,
        timestamp=datetime.utcnow().isoformat(),
        ip_address=details.get('ip'),
        user_agent=details.get('user_agent'),
        success=details.get('success', True)
    )

# Usage
log_security_event(
    "login_attempt",
    user_id="user123",
    details={
        "ip": request.client.host,
        "user_agent": request.headers.get("user-agent"),
        "success": False
    }
)
```

## Secure Coding Patterns by Language

### Python

```python
# ✅ DO: Use secrets module for cryptographic operations
import secrets
import string

def generate_secure_token(length: int = 32) -> str:
    """Generate cryptographically secure random token."""
    alphabet = string.ascii_letters + string.digits
    return ''.join(secrets.choice(alphabet) for _ in range(length))

def generate_password_reset_token() -> str:
    """Generate secure password reset token."""
    return secrets.token_urlsafe(32)

# ✅ DO: Validate file uploads
import magic
from pathlib import Path

def validate_upload(file_path: str, allowed_extensions: set) -> bool:
    """Validate uploaded file."""
    path = Path(file_path)
    
    # Check extension
    if path.suffix.lower() not in allowed_extensions:
        return False
    
    # Check MIME type
    mime = magic.from_file(file_path, mime=True)
    allowed_mimes = {'image/jpeg', 'image/png', 'application/pdf'}
    if mime not in allowed_mimes:
        return False
    
    return True

# ✅ DO: Use pathlib for safe path handling
from pathlib import Path

def safe_file_access(base_dir: str, filename: str) -> Path:
    """Safely access files preventing path traversal."""
    base = Path(base_dir).resolve()
    file_path = (base / filename).resolve()
    
    # Ensure file is within base directory
    if not str(file_path).startswith(str(base)):
        raise ValueError("Path traversal attempt detected")
    
    return file_path
```

### JavaScript/TypeScript

```javascript
// ✅ DO: Use helmet for Express security
const helmet = require('helmet');
const express = require('express');
const app = express();

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));

// ✅ DO: Use csurf for CSRF protection
const csurf = require('csurf');
const csrfProtection = csurf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.render('send', { csrfToken: req.csrfToken() });
});

// ✅ DO: Validate and sanitize input
const { body, validationResult } = require('express-validator');

app.post('/user',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  body('username').trim().escape(),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Process valid input
  }
);
```

### Go

```go
// ✅ DO: Use bcrypt for password hashing
import "golang.org/x/crypto/bcrypt"

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func CheckPasswordHash(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// ✅ DO: Use constant-time comparison for secrets
import "crypto/subtle"

func VerifySecret(provided, expected string) bool {
    return subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) == 1
}

// ✅ DO: Set secure cookie attributes
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    sessionToken,
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteStrictMode,
    MaxAge:   86400,
    Path:     "/",
})
```

## Secrets Management

### Environment Variables

```bash
# ✅ DO: Use .env files for local development (never commit)
# .env.example (committed)
DATABASE_URL=postgresql://user:password@localhost/dbname
SECRET_KEY=your-secret-key-here
API_KEY=your-api-key-here

# ✅ DO: Use different values for different environments
# .env.production (not committed, set in deployment platform)
DATABASE_URL=${DATABASE_URL}  # From secrets manager
SECRET_KEY=${SECRET_KEY}      # From secrets manager
DEBUG=false
```

### Secrets Managers

```python
# ✅ DO: Use AWS Secrets Manager
import boto3
from botocore.exceptions import ClientError

def get_secret(secret_name: str) -> dict:
    """Get secret from AWS Secrets Manager."""
    client = boto3.client('secretsmanager')
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except ClientError as e:
        logger.error(f"Failed to retrieve secret: {e}")
        raise

# ✅ DO: Use HashiCorp Vault
import hvac

client = hvac.Client(url='https://vault.example.com')
client.auth.kubernetes.login(role='my-app', jwt='/var/run/secrets/kubernetes.io/serviceaccount/token')

secret = client.secrets.kv.v2.read_secret_version(path='myapp/database')
db_password = secret['data']['data']['password']

# ✅ DO: Rotate secrets regularly
# Implement automatic rotation in CI/CD
# Use short-lived credentials where possible
```

## When to Use This Skill

Use this skill when:
- Implementing authentication and authorization
- Handling user input and data validation
- Setting up HTTPS and security headers
- Managing secrets and credentials
- Configuring CORS and CSP policies
- Reviewing code for security vulnerabilities
- Setting up logging and monitoring
- Configuring Docker and deployment security

## Related Skills

- `@code-reviewer` - For security-focused code reviews
- `@docker-patterns` - Container security
- `@api-rest-design` - API security patterns
- `@postgresql-patterns` - Database security
- `@feature-development` - Secure development workflow
