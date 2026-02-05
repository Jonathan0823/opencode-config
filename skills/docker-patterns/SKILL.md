---
name: docker-patterns
description: Docker best practices, multi-stage builds, container security, and Docker Compose patterns
---

# Docker Patterns Skill

## Overview

This skill provides comprehensive guidelines for building production-ready Docker containers, multi-stage builds, security hardening, Docker Compose orchestration, and deployment best practices.

## Dockerfile Best Practices

### 1. Multi-Stage Builds

```dockerfile
# ✅ DO: Use multi-stage builds to minimize image size
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy source and build
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "dist/main.js"]
```

### 2. Language-Specific Examples

#### Python (FastAPI/Django)

```dockerfile
# Build stage
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim AS production

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Go

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Install git and ca-certificates
RUN apk add --no-cache git ca-certificates

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Production stage
FROM scratch

# Copy CA certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /app/main /main

# Use non-root user (scratch doesn't have adduser, use numeric UID)
USER 65534:65534

# Expose port
EXPOSE 8080

# Health check (using wget from busybox)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/main", "-health-check"] || exit 1

ENTRYPOINT ["/main"]
```

#### Rust

```dockerfile
# Build stage
FROM rust:1.75-slim AS builder

WORKDIR /app

# Install dependencies
RUN apt-get update && apt-get install -y pkg-config libssl-dev

# Copy manifests
COPY Cargo.toml Cargo.lock ./

# Build dependencies (caching layer)
RUN mkdir src && \
    echo 'fn main() {}' > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Copy source and build
COPY src ./src
RUN touch src/main.rs && \
    cargo build --release

# Production stage
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy binary
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp

# Switch to non-root user
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD myapp --health-check || exit 1

CMD ["myapp"]
```

### 3. Security Best Practices

```dockerfile
# ✅ DO: Use specific image tags, not 'latest'
FROM node:18.19.0-alpine3.18

# ✅ DO: Run as non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# ✅ DO: Copy with specific ownership
COPY --chown=appuser:appgroup . /app

USER appuser

# ✅ DO: Use read-only root filesystem
# docker run --read-only ...

# ✅ DO: Drop unnecessary capabilities
# docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ...

# ✅ DO: Set resource limits
# docker run --memory=512m --cpus=1.0 ...
```

### 4. Layer Caching Optimization

```dockerfile
# ✅ DO: Order instructions by change frequency (least to most)
FROM node:18-alpine

WORKDIR /app

# 1. Copy package files (rarely change)
COPY package*.json ./

# 2. Install dependencies (change when packages update)
RUN npm ci --only=production

# 3. Copy application code (changes frequently)
COPY . .

# 4. Build (changes frequently)
RUN npm run build

CMD ["node", "dist/main.js"]

# ✅ DO: Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# ✅ DO: Use .dockerignore to exclude unnecessary files
# .dockerignore content:
/*
!package*.json
!src/
!public/
!tsconfig.json
node_modules/
.env
.env.local
.env.*.local
.DS_Store
*.log
coverage/
.vscode/
.idea/
.git/
.gitignore
README.md
Dockerfile*
docker-compose*
```

## Docker Compose Patterns

### 1. Development Environment

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    volumes:
      - .:/app
      - /app/node_modules  # Anonymous volume to prevent overwriting
    depends_on:
      - db
      - redis
    command: npm run dev

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Optional: Admin tools
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  postgres_data:
  redis_data:
```

### 2. Production Environment

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - SECRET_KEY=${SECRET_KEY}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static_files:/usr/share/nginx/html/static:ro
    depends_on:
      - app

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  static_files:
    driver: local
```

### 3. Multi-Environment Setup

```yaml
# docker-compose.base.yml
version: '3.8'

services:
  app:
    build:
      context: .
    environment:
      - APP_NAME=myapp
    volumes:
      - app_logs:/app/logs

  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  app_logs:
  postgres_data:

---
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DEBUG=true
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp_dev

---
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      dockerfile: Dockerfile
      target: production
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M

  db:
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    deploy:
      resources:
        limits:
          memory: 1G

# Usage:
# docker-compose -f docker-compose.base.yml -f docker-compose.dev.yml up
# docker-compose -f docker-compose.base.yml -f docker-compose.prod.yml up
```

## Container Security

### 1. Security Options

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

### 2. Network Security

```yaml
version: '3.8'

services:
  app:
    build: .
    networks:
      - frontend
      - backend
    expose:
      - "3000"  # Not published to host, only accessible within network

  db:
    image: postgres:15-alpine
    networks:
      - backend
    # No ports exposed - only accessible from app service

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
```

## Health Checks and Monitoring

### 1. Dockerfile Health Checks

```dockerfile
# Node.js application
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})" || exit 1

# Python application
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# Go application
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/health || exit 1
```

### 2. Docker Compose Health Checks

```yaml
version: '3.8'

services:
  app:
    build: .
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## Image Scanning and Optimization

### 1. Scan for Vulnerabilities

```bash
# Using Docker Scout
docker scout cves myapp:latest

# Using Trivy
trivy image myapp:latest

# Using Snyk
snyk container test myapp:latest
```

### 2. Image Size Optimization

```bash
# Check image size
docker images myapp

# Analyze image layers
docker history myapp:latest

# Multi-stage build size comparison
# Without multi-stage: 1.2GB
# With multi-stage: 150MB

# Use dive tool to analyze
 dive myapp:latest
```

### 3. BuildKit Features

```dockerfile
# syntax=docker/dockerfile:1

# Use BuildKit mount cache for faster builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Use BuildKit secrets for sensitive data
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

# Use BuildKit SSH for private repos
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git
```

## Deployment Patterns

### 1. Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

# Build new version (green)
docker build -t myapp:green .

# Start green environment
docker-compose -f docker-compose.green.yml up -d

# Health check
sleep 10
if curl -f http://localhost:3001/health; then
    # Switch traffic to green
    docker-compose -f docker-compose.nginx.yml up -d
    
    # Stop blue environment
    docker-compose -f docker-compose.blue.yml down
    
    # Tag green as new blue
    docker tag myapp:green myapp:blue
else
    echo "Health check failed, rolling back"
    docker-compose -f docker-compose.green.yml down
    exit 1
fi
```

### 2. Rolling Updates

```yaml
# docker-compose.yml with rolling updates
version: '3.8'

services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        monitor: 60s
```

## When to Use This Skill

Use this skill when:
- Creating Dockerfiles for production applications
- Setting up development environments with Docker Compose
- Optimizing image sizes with multi-stage builds
- Hardening containers for security
- Setting up health checks and monitoring
- Managing multiple environments (dev/staging/prod)
- Implementing CI/CD pipelines with Docker
- Troubleshooting container issues

## Related Skills

- `@security-best-practices` - Container security
- `@ci-cd` - Continuous integration and deployment
- `@feature-development` - Development workflow
- `@python-patterns` - Python-specific patterns
- `@go-conventions` - Go-specific patterns
- `@ts-react-nextjs` - Node.js patterns
