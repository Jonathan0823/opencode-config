# Container Security

## Security Options in Docker Compose

```yaml
# docker-compose.yml with security options
version: '3.8'

services:
  app:
    build: .
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
      - /var/tmp:noexec,nosuid,size=50m
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Dockerfile Security Hardening

```dockerfile
# Use minimal base image
FROM alpine:3.18

# Create non-root user
RUN addgroup -g 1000 -S appgroup && \
    adduser -u 1000 -S appuser -G appgroup

# Install only necessary packages
RUN apk add --no-cache ca-certificates

# Set working directory
WORKDIR /app

# Copy application with proper ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

# Use read-only filesystem
# docker run --read-only --tmpfs /tmp myimage

# Drop all capabilities
# docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage
```

## Network Security

### Internal Networks

```yaml
version: '3.8'

services:
  app:
    build: .
    networks:
      - frontend
      - backend

  db:
    image: postgres:15-alpine
    networks:
      - backend
    # Database not accessible from external network

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external connectivity
```

### Disable Inter-Container Communication

```yaml
version: '3.8'

services:
  app:
    build: .
    networks:
      - isolated

networks:
  isolated:
    driver: bridge
    internal: true
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"
```

## Secrets Management

### Docker Secrets

```yaml
version: '3.8'

services:
  app:
    build: .
    secrets:
      - api_key
      - db_password
    environment:
      - API_KEY_FILE=/run/secrets/api_key
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  api_key:
    file: ./secrets/api_key.txt
  db_password:
    file: ./secrets/db_password.txt
```

Reading secrets in application:

```python
import os

def get_secret(secret_name):
    """Read secret from Docker secrets or environment."""
    secret_path = f"/run/secrets/{secret_name}"
    
    if os.path.exists(secret_path):
        with open(secret_path, 'r') as f:
            return f.read().strip()
    
    # Fallback to environment variable
    return os.environ.get(secret_name.upper())

# Usage
api_key = get_secret('api_key')
```

## Resource Limits

```yaml
version: '3.8'

services:
  app:
    build: .
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
          pids: 100
        reservations:
          cpus: '0.25'
          memory: 256M
    
    # Runtime limits
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
```

## Image Scanning

### Using Trivy

```bash
# Scan image for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image myapp:latest

# Scan filesystem
docker run --rm -v $(pwd):/app aquasec/trivy:latest fs /app

# Scan and output to file
docker run --rm -v $(pwd):/app -v $(pwd)/reports:/reports \
  aquasec/trivy:latest fs --format sarif -o /reports/trivy.sarif /app
```

### Using Docker Scout

```bash
# Enable Docker Scout
docker scout enroll

# Scan image
docker scout cves myapp:latest

# Compare two images
docker scout compare --to myapp:old myapp:new
```

### Using Snyk

```bash
# Scan image
snyk container test myapp:latest

# Monitor for vulnerabilities
snyk container monitor myapp:latest
```

## Security Scanning in CI/CD

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Fail on high/critical vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          exit-code: '1'
          severity: 'HIGH,CRITICAL'
```

## Content Trust (Docker Content Trust)

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Sign and push image
docker trust sign myregistry/myapp:latest

# Inspect signature
docker trust inspect --pretty myregistry/myapp:latest
```

## Best Practices Checklist

### Image Build
- [ ] Use specific image tags (not `latest`)
- [ ] Use minimal base images (alpine, scratch, distroless)
- [ ] Run as non-root user
- [ ] Copy files with specific ownership
- [ ] Remove unnecessary packages and files
- [ ] Scan images before deployment
- [ ] Sign images with Docker Content Trust

### Runtime Security
- [ ] Enable user namespaces
- [ ] Drop unnecessary capabilities
- [ ] Use read-only root filesystem
- [ ] Set resource limits (CPU, memory, PIDs)
- [ ] Use tmpfs for writable directories
- [ ] Enable Docker Content Trust
- [ ] Use secrets management (not env vars for sensitive data)

### Network Security
- [ ] Use internal networks for backend services
- [ ] Expose only necessary ports
- [ ] Disable inter-container communication if not needed
- [ ] Use TLS for external communication

### Monitoring
- [ ] Enable audit logging
- [ ] Monitor container resource usage
- [ ] Set up alerts for security events
- [ ] Regular vulnerability scanning
