---
name: ci-cd-pipelines
description: GitHub Actions workflows for continuous integration, automated testing, security scanning, and deployment automation. Use when setting up CI/CD pipelines, creating GitHub Actions workflows, automating tests, or deploying applications.
license: MIT
compatibility: opencode
---

# CI/CD Pipelines Skill

## Overview

This skill provides GitHub Actions workflows for building, testing, securing, and deploying applications. Covers CI/CD best practices, environment promotion strategies, and integration with Docker and cloud platforms.

## Quick Start

### Basic CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
```

### Basic CD Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to server
        run: |
          echo "${{ secrets.SSH_KEY }}" > key.pem
          chmod 600 key.pem
          ssh -i key.pem user@server "cd /app && git pull && docker-compose up -d"
```

## Core Concepts

### Workflow Structure

```yaml
name: Workflow Name

on:
  # Event triggers
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # Weekly

env:
  # Global environment variables
  NODE_VERSION: '20'

jobs:
  job-name:
    runs-on: ubuntu-latest
    steps:
      - name: Step name
        run: echo "Hello World"
```

### Environment Promotion

```
develop → staging → production
   ↑          ↑           ↑
  PRs      QA Test    Live Deploy
```

## Common Patterns

### Matrix Testing

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x, 21.x]
    os: [ubuntu-latest, windows-latest]

steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

### Caching Dependencies

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      ~/.cache/pip
      ~/.cargo/registry
    key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json') }}
```

### Security Scanning

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload scan results
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'
```

## Docker Integration

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:latest
      ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## Deployment Strategies

### Blue-Green Deployment

See `references/deployment-strategies.md` for detailed implementation.

### Rolling Updates

```yaml
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy with zero downtime
      run: |
        # Update one instance at a time
        for server in server1 server2 server3; do
          ssh $server "docker-compose pull && docker-compose up -d"
          sleep 10
          curl -f http://$server/health || exit 1
        done
```

## When to Use This Skill

Use this skill when:
- Setting up GitHub Actions workflows
- Creating CI/CD pipelines for automated testing
- Implementing security scanning in CI
- Configuring deployment automation
- Managing environment-specific deployments
- Integrating Docker with CI/CD
- Setting up branch protection with CI checks

## Related Skills

- `@docker-patterns` - Container builds and deployment
- `@security-best-practices` - Security scanning and hardening
- `@git-team-workflow` - Branch protection and PR workflows
- `@testing-strategies` - Test configuration and coverage
