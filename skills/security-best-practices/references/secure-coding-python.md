# Secure Coding Patterns - Python

## Cryptographic Operations

```python
import secrets
import string

def generate_secure_token(length: int = 32) -> str:
    """Generate cryptographically secure random token."""
    alphabet = string.ascii_letters + string.digits
    return ''.join(secrets.choice(alphabet) for _ in range(length))

def generate_password_reset_token() -> str:
    """Generate secure password reset token."""
    return secrets.token_urlsafe(32)
```

## File Upload Validation

```python
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
```

## Path Traversal Prevention

```python
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

## Input Validation

```python
from typing import Optional
import re

def validate_email(email: str) -> bool:
    """Validate email format."""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

def sanitize_filename(filename: str) -> str:
    """Sanitize filename to prevent path traversal."""
    # Remove any path components
    filename = Path(filename).name
    
    # Remove null bytes
    filename = filename.replace('\x00', '')
    
    # Remove dangerous characters
    filename = re.sub(r'[<>:"|?*]', '', filename)
    
    return filename
```

## Secure HTTP Headers

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Configure CORS securely
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Specific origins only
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=3600,
)

@app.middleware("http")
async def security_headers(request, call_next):
    response = await call_next(request)
    
    # Security headers
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"
    
    return response
```

## SQL Injection Prevention

```python
# Using SQLAlchemy (ORM)
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

engine = create_engine('postgresql://user:pass@localhost/db')
Session = sessionmaker(bind=engine)

# ✅ SAFE: Using ORM
session = Session()
user = session.query(User).filter(User.username == username).first()

# ✅ SAFE: Using parameterized queries
with engine.connect() as conn:
    result = conn.execute(
        text("SELECT * FROM users WHERE username = :username"),
        {"username": username}
    )

# ❌ UNSAFE: String formatting
query = f"SELECT * FROM users WHERE username = '{username}'"
```

## Command Injection Prevention

```python
import subprocess
from typing import List

def safe_command(command: List[str]) -> str:
    """Execute command safely without shell injection."""
    try:
        result = subprocess.run(
            command,
            capture_output=True,
            text=True,
            check=True,
            shell=False  # Never use shell=True with user input
        )
        return result.stdout
    except subprocess.CalledProcessError as e:
        raise RuntimeError(f"Command failed: {e}")

# ✅ SAFE: Using list of arguments
safe_command(["ls", "-la", directory])

# ❌ UNSAFE: Using shell=True with user input
subprocess.run(f"ls -la {directory}", shell=True)  # Dangerous!
```

## Secrets Management

```python
import os
from functools import lru_cache

@lru_cache()
def get_secret(key: str) -> str:
    """Get secret from environment with validation."""
    value = os.environ.get(key)
    if not value:
        raise ValueError(f"Secret {key} not set in environment")
    return value

# Usage
DATABASE_URL = get_secret("DATABASE_URL")
SECRET_KEY = get_secret("SECRET_KEY")
```

## Secure Cookie Configuration

```python
from fastapi import Response

response = Response()
response.set_cookie(
    key="session_id",
    value=session_token,
    httponly=True,      # Prevent JavaScript access
    secure=True,        # HTTPS only
    samesite="Strict",  # CSRF protection
    max_age=3600,       # 1 hour
    path="/",
)
```
