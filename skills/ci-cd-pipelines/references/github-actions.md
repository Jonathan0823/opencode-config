# GitHub Actions Reference

## Workflow Syntax

### Event Triggers

```yaml
on:
  # Push events
  push:
    branches: [main, develop]
    tags: ['v*']
    paths:
      - 'src/**'
      - '!src/**/*.md'
  
  # Pull request events
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  
  # Scheduled events
  schedule:
    - cron: '0 0 * * 0'  # Every Sunday at midnight
  
  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
  
  # External events
  repository_dispatch:
    types: [deploy]
```

### Job Configuration

```yaml
jobs:
  job-name:
    # Runner selection
    runs-on: ubuntu-latest
    
    # Or self-hosted
    runs-on: self-hosted
    
    # Or matrix
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 21]
        include:
          - os: ubuntu-latest
            node: 21
            experimental: true
        exclude:
          - os: windows-latest
            node: 18
    
    # Dependencies
    needs: [build, test]
    
    # Environment
    environment: production
    
    # Concurrency
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    
    # Timeout
    timeout-minutes: 30
    
    # Outputs
    outputs:
      version: ${{ steps.version.outputs.value }}
```

### Step Types

```yaml
steps:
  # Checkout code
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0
      token: ${{ secrets.PAT }}
  
  # Setup language
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'
  
  # Setup multiple languages
  - uses: actions/setup-python@v5
    with:
      python-version: '3.11'
  
  # Run command
  - name: Run tests
    run: npm test
    working-directory: ./app
    shell: bash
    env:
      API_KEY: ${{ secrets.API_KEY }}
  
  # Multi-line commands
  - name: Build application
    run: |
      echo "Building..."
      npm run build
      echo "Build complete"
  
  # Conditional steps
  - name: Deploy production
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
  
  # Continue on error
  - name: Optional step
    continue-on-error: true
    run: ./optional.sh
```

## Complete Examples

### Node.js Application

```yaml
name: Node.js CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
      
      - name: Build
        run: npm run build
```

### Python Application

```yaml
name: Python CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov flake8
      
      - name: Lint with flake8
        run: flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
      
      - name: Test with pytest
        run: pytest --cov=./ --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

### Go Application

```yaml
name: Go CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
      
      - name: Download dependencies
        run: go mod download
      
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...
      
      - name: Build
        run: go build -v ./...
      
      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest
```

### Rust Application

```yaml
name: Rust CI

on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Cache cargo registry
        uses: actions/cache@v4
        with:
          path: ~/.cargo/registry
          key: ${{ runner.os }}-cargo-registry-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Cache cargo index
        uses: actions/cache@v4
        with:
          path: ~/.cargo/git
          key: ${{ runner.os }}-cargo-index-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Cache cargo build
        uses: actions/cache@v4
        with:
          path: target
          key: ${{ runner.os }}-cargo-build-target-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Build
        run: cargo build --verbose
      
      - name: Run tests
        run: cargo test --verbose
      
      - name: Check formatting
        run: cargo fmt -- --check
      
      - name: Run clippy
        run: cargo clippy -- -D warnings
```

## Secrets and Variables

### Repository Secrets

```yaml
steps:
  - name: Use secret
    run: |
      echo "${{ secrets.SSH_KEY }}" > key.pem
      chmod 600 key.pem
    
  - name: Use environment variable
    run: echo "API URL: ${{ vars.API_URL }}"
```

### Environment Secrets

```yaml
deploy:
  runs-on: ubuntu-latest
  environment: production
  steps:
    - uses: actions/checkout@v4
    
    - name: Deploy
      env:
        API_KEY: ${{ secrets.API_KEY }}
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      run: ./deploy.sh
```

### OIDC Token Authentication

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/my-role
          aws-region: us-east-1
```

## Reusable Workflows

### Calling a Reusable Workflow

```yaml
jobs:
  call-workflow:
    uses: owner/repo/.github/workflows/reusable.yml@main
    with:
      node-version: '20'
      environment: staging
    secrets:
      API_KEY: ${{ secrets.API_KEY }}
```

### Creating a Reusable Workflow

```yaml
# .github/workflows/reusable.yml
name: Reusable Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: 'development'
    secrets:
      API_KEY:
        required: true
    outputs:
      version:
        description: "Build version"
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      
      - name: Get version
        id: version
        run: echo "value=$(cat package.json | jq -r .version)" >> $GITHUB_OUTPUT
      
      - name: Build
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: npm run build:${{ inputs.environment }}
```

## Advanced Features

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-files
          path: dist/
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-files
      - run: npm test
  
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Conditional Jobs

```yaml
jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to staging"
  
  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploying to production"
```

### Services (Database, Cache, etc.)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests with services
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        run: npm test
```

### Composite Actions

```yaml
# .github/actions/setup/action.yml
name: 'Setup Project'
description: 'Setup Node.js and install dependencies'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - run: npm ci
      shell: bash
    
    - run: npm run build
      shell: bash
```

Usage:
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup
    with:
      node-version: '18'
```

## Best Practices

1. **Pin action versions** - Use specific versions, not `@main`
2. **Use caching** - Cache dependencies for faster builds
3. **Limit permissions** - Use minimal required permissions
4. **Secure secrets** - Never log secrets or commit them
5. **Fail fast** - Use `fail-fast: false` for matrix builds
6. **Timeout jobs** - Set appropriate timeouts
7. **Use environments** - For deployment jobs with protection rules
8. **Matrix testing** - Test on multiple versions and OS
