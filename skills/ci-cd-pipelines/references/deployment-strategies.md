# Deployment Strategies Reference

## Overview

Different deployment strategies offer varying trade-offs between downtime, risk, and complexity.

## Deployment Strategy Comparison

| Strategy | Downtime | Risk | Complexity | Rollback Speed |
|----------|----------|------|------------|----------------|
| **Recreate** | High | Low | Low | Slow |
| **Rolling** | Zero | Medium | Low | Medium |
| **Blue-Green** | Zero | Low | Medium | Fast |
| **Canary** | Zero | Low | High | Fast |
| **A/B Testing** | Zero | Medium | High | Fast |

## Blue-Green Deployment

### Concept

Two identical production environments:
- **Blue**: Currently serving production traffic
- **Green**: Idle, ready for new version

After testing Green, switch traffic instantly. Blue remains for instant rollback.

### GitHub Actions Implementation

```yaml
name: Blue-Green Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine inactive environment
        id: env
        run: |
          # Check which environment is currently active
          ACTIVE=$(curl -s https://api.loadbalancer.com/active)
          if [ "$ACTIVE" = "blue" ]; then
            echo "target=green" >> $GITHUB_OUTPUT
            echo "current=blue" >> $GITHUB_OUTPUT
          else
            echo "target=blue" >> $GITHUB_OUTPUT
            echo "current=green" >> $GITHUB_OUTPUT
          fi
      
      - name: Deploy to inactive environment
        run: |
          TARGET=${{ steps.env.outputs.target }}
          echo "Deploying to $TARGET environment"
          
          # Deploy to target environment
          ssh deploy@$TARGET-server "\
            cd /var/www/$TARGET && \
            git pull origin main && \
            docker-compose -f docker-compose.$TARGET.yml up -d --build
          "
      
      - name: Health check
        run: |
          TARGET=${{ steps.env.outputs.target }}
          for i in {1..10}; do
            if curl -f https://$TARGET.yourapp.com/health; then
              echo "Health check passed"
              exit 0
            fi
            sleep 5
          done
          echo "Health check failed"
          exit 1
      
      - name: Switch traffic
        run: |
          TARGET=${{ steps.env.outputs.target }}
          CURRENT=${{ steps.env.outputs.current }}
          
          # Update load balancer to point to new environment
          curl -X POST https://api.loadbalancer.com/switch \
            -d "target=$TARGET"
          
          echo "Traffic switched from $CURRENT to $TARGET"
      
      - name: Verify production
        run: |
          # Verify production is serving new version
          VERSION=$(curl -s https://yourapp.com/version)
          echo "Production version: $VERSION"
      
      - name: Notify team
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}: Version ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Docker Compose Setup

```yaml
# docker-compose.blue.yml
version: '3.8'
services:
  app:
    image: myapp:${VERSION}
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
      - ENVIRONMENT=blue

# docker-compose.green.yml
version: '3.8'
services:
  app:
    image: myapp:${VERSION}
    ports:
      - "3002:3000"
    environment:
      - NODE_ENV=production
      - ENVIRONMENT=green
```

## Rolling Deployment

### Concept

Gradually replace old instances with new ones:
1. Take one old instance offline
2. Deploy new version
3. Health check
4. Bring online
5. Repeat until all instances updated

### GitHub Actions Implementation

```yaml
name: Rolling Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    strategy:
      matrix:
        server: [server1, server2, server3]
      max-parallel: 1  # Deploy to one server at a time
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to ${{ matrix.server }}
        run: |
          SERVER=${{ matrix.server }}
          echo "Deploying to $SERVER"
          
          # Remove from load balancer
          curl -X POST https://api.loadbalancer.com/drain \
            -d "server=$SERVER"
          
          # Wait for connections to drain
          sleep 10
          
          # Deploy
          ssh deploy@$SERVER "\
            cd /var/www/app && \
            git pull origin main && \
            docker-compose up -d --build
          "
          
          # Health check
          for i in {1..5}; do
            if ssh deploy@$SERVER "curl -f http://localhost:3000/health"; then
              break
            fi
            sleep 5
          done
          
          # Add back to load balancer
          curl -X POST https://api.loadbalancer.com/enable \
            -d "server=$SERVER"
          
          echo "Deployment to $SERVER complete"
```

## Canary Deployment

### Concept

Roll out to small subset of users first:
1. Deploy new version to small percentage (e.g., 5%)
2. Monitor metrics and errors
3. Gradually increase percentage
4. Full rollout if metrics look good
5. Automatic rollback if errors detected

### GitHub Actions Implementation

```yaml
name: Canary Deployment

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker push myapp:${{ github.sha }}
      
      - name: Deploy canary (5%)
        run: |
          # Deploy canary version
          kubectl set image deployment/app-canary \
            app=myapp:${{ github.sha }}
          
          # Wait for rollout
          kubectl rollout status deployment/app-canary
      
      - name: Monitor canary (5 minutes)
        run: |
          sleep 300
          
          # Check error rate
          ERROR_RATE=$(curl -s "https://api.monitoring.com/metrics?\
            metric=error_rate&\
            deployment=canary&\
            duration=5m" | jq -r '.value')
          
          echo "Canary error rate: $ERROR_RATE"
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high, rolling back"
            kubectl rollout undo deployment/app-canary
            exit 1
          fi
      
      - name: Scale canary to 25%
        run: |
          kubectl scale deployment app-canary --replicas=5
          kubectl scale deployment app --replicas=15
      
      - name: Monitor canary (10 minutes)
        run: |
          sleep 600
          
          ERROR_RATE=$(curl -s "https://api.monitoring.com/metrics?\
            metric=error_rate&\
            deployment=canary&\
            duration=10m" | jq -r '.value')
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high, rolling back"
            kubectl rollout undo deployment/app-canary
            exit 1
          fi
      
      - name: Full rollout
        run: |
          # Update main deployment
          kubectl set image deployment/app \
            app=myapp:${{ github.sha }}
          
          # Scale down canary
          kubectl scale deployment app-canary --replicas=0
          
          # Wait for main rollout
          kubectl rollout status deployment/app
      
      - name: Notify
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Canary deployment ${{ job.status }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## Feature Flags

### Concept

Deploy code to production but hide behind flags:
- Enable features for specific users
- Gradual rollout
- Instant rollback (disable flag)
- A/B testing support

### GitHub Actions with Feature Flags

```yaml
name: Deploy with Feature Flags

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        run: |
          # Deploy code with new features disabled
          ssh deploy@production "cd /var/www/app && git pull && docker-compose up -d"
      
      - name: Enable feature for beta users
        run: |
          # Enable new feature for beta group
          curl -X POST https://api.featureflags.com/enable \
            -H "Authorization: Bearer ${{ secrets.FF_TOKEN }}" \
            -d '{
              "flag": "new-checkout-flow",
              "target": "beta-users"
            }'
      
      - name: Monitor beta rollout
        run: |
          sleep 3600  # Monitor for 1 hour
          
          # Check metrics
          CONVERSION=$(curl -s "https://api.analytics.com/metrics?\
            feature=new-checkout-flow&\
            segment=beta-users" | jq -r '.conversion_rate')
          
          echo "Beta conversion rate: $CONVERSION"
      
      - name: Gradual rollout
        run: |
          # 25% of users
          curl -X POST https://api.featureflags.com/percentage \
            -d '{"flag": "new-checkout-flow", "percentage": 25}'
          
          sleep 7200  # Wait 2 hours
          
          # 50% of users
          curl -X POST https://api.featureflags.com/percentage \
            -d '{"flag": "new-checkout-flow", "percentage": 50}'
          
          sleep 7200
          
          # 100% of users
          curl -X POST https://api.featureflags.com/percentage \
            -d '{"flag": "new-checkout-flow", "percentage": 100}'
```

## Database Migrations

### Zero-Downtime Migrations

```yaml
name: Deploy with Migration

on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run backward-compatible migrations
        run: |
          # Step 1: Add new column (nullable)
          psql $DATABASE_URL -c "ALTER TABLE users ADD COLUMN email_verified boolean;"
      
      - name: Deploy new code
        run: |
          # Deploy code that uses new column
          ssh deploy@production "cd /var/www/app && git pull && docker-compose up -d"
      
      - name: Backfill data
        run: |
          # Step 3: Backfill existing data
          psql $DATABASE_URL -c "
            UPDATE users 
            SET email_verified = true 
            WHERE email IS NOT NULL;
          "
      
      - name: Make column required
        run: |
          # Step 4: Add NOT NULL constraint
          psql $DATABASE_URL -c "
            ALTER TABLE users 
            ALTER COLUMN email_verified SET NOT NULL;
          "
```

## Rollback Strategies

### Automatic Rollback on Failure

```yaml
name: Deploy with Rollback

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Store previous version
        id: prev
        run: |
          PREV_VERSION=$(curl -s https://api.production.com/version)
          echo "version=$PREV_VERSION" >> $GITHUB_OUTPUT
      
      - name: Deploy
        run: |
          ssh deploy@production "cd /var/www/app && git pull && docker-compose up -d"
      
      - name: Health check
        run: |
          for i in {1..12}; do
            if curl -f https://api.production.com/health; then
              exit 0
            fi
            sleep 5
          done
          exit 1
        continue-on-error: false
      
      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, rolling back to ${{ steps.prev.outputs.version }}"
          ssh deploy@production "cd /var/www/app && git checkout ${{ steps.prev.outputs.version }} && docker-compose up -d"
```

### Manual Rollback

```yaml
name: Manual Rollback

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Rollback
        run: |
          ssh deploy@production "cd /var/www/app && git checkout ${{ github.event.inputs.version }} && docker-compose up -d"
      
      - name: Verify rollback
        run: |
          sleep 30
          curl -f https://api.production.com/health
```

## Environment Promotion

### Staging → Production

```yaml
name: Promote to Production

on:
  push:
    branches: [main]

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Get staging image
        id: image
        run: |
          # Get the image currently running in staging
          STAGING_IMAGE=$(kubectl get deployment app -n staging -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "image=$STAGING_IMAGE" >> $GITHUB_OUTPUT
      
      - name: Run smoke tests on staging
        run: |
          # Verify staging is healthy
          curl -f https://staging.yourapp.com/health
          
          # Run smoke tests
          npm run test:smoke -- --env=staging
      
      - name: Deploy to production
        run: |
          kubectl set image deployment/app -n production \
            app=${{ steps.image.outputs.image }}
      
      - name: Production smoke tests
        run: |
          sleep 30
          npm run test:smoke -- --env=production
```

## Monitoring Deployments

### Integration with Monitoring Tools

```yaml
name: Monitored Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Start deployment in monitoring
        run: |
          DEPLOY_ID=$(curl -X POST https://api.datadog.com/deployments \
            -d "service=myapp&version=${{ github.sha }}" | jq -r '.id')
          echo "deploy_id=$DEPLOY_ID" >> $GITHUB_ENV
      
      - name: Deploy
        run: |
          ssh deploy@production "cd /var/www/app && git pull && docker-compose up -d"
      
      - name: Monitor deployment
        run: |
          for i in {1..12}; do
            STATUS=$(curl -s "https://api.datadog.com/deployments/$DEPLOY_ID/status" | jq -r '.status')
            
            if [ "$STATUS" = "healthy" ]; then
              echo "Deployment healthy"
              exit 0
            elif [ "$STATUS" = "failed" ]; then
              echo "Deployment failed"
              exit 1
            fi
            
            sleep 10
          done
      
      - name: Mark deployment complete
        if: success()
        run: |
          curl -X POST "https://api.datadog.com/deployments/$DEPLOY_ID/complete"
      
      - name: Mark deployment failed
        if: failure()
        run: |
          curl -X POST "https://api.datadog.com/deployments/$DEPLOY_ID/fail"
```

## Best Practices

1. **Always have a rollback plan** - Know how to revert quickly
2. **Health checks** - Verify application health after deployment
3. **Monitoring** - Watch metrics during and after deployment
4. **Gradual rollout** - Don't deploy to 100% immediately
5. **Feature flags** - Deploy code disabled, enable gradually
6. **Database compatibility** - Keep migrations backward compatible
7. **Test in staging** - Verify in staging before production
8. **Automated smoke tests** - Run quick tests post-deployment
9. **Notifications** - Alert team of deployment status
10. **Documentation** - Document your deployment process
