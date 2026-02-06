# Dockerfile Patterns

## Multi-Stage Builds

### Node.js Application

```dockerfile
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

### Python (FastAPI/Django)

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

### Go Application

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

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/main", "-health-check"] || exit 1

ENTRYPOINT ["/main"]
```

### Rust Application

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

## Security Best Practices

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

## Layer Caching Optimization

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

## BuildKit Features

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

## Health Checks

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
