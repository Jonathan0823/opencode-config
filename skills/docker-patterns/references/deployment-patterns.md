# Deployment Patterns

## Blue-Green Deployment

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

## Rolling Updates

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

## Docker Swarm Deployment

```yaml
# docker-stack.yml
version: '3.8'

services:
  app:
    image: myapp:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
        preferences:
          - spread: node.availability.zone
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    networks:
      - frontend
      - backend
    configs:
      - source: app_config
        target: /app/config.yml
    secrets:
      - api_key
      - db_password

  db:
    image: postgres:15-alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  db_data:
    driver: local

configs:
  app_config:
    external: true

secrets:
  api_key:
    external: true
  db_password:
    external: true
```

### Deploy Stack

```bash
# Initialize swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-stack.yml myapp

# Update service
docker service update --image myapp:v2.0 myapp_app

# Rolling update with health check
docker service update \
  --image myapp:v2.0 \
  --update-delay 10s \
  --update-parallelism 1 \
  --update-failure-action rollback \
  myapp_app

# Rollback
docker service update --rollback myapp_app

# View logs
docker service logs myapp_app

# Scale service
docker service scale myapp_app=5
```

## Kubernetes Deployment

### Basic Deployment

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:latest
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
            - name: NODE_ENV
              value: "production"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      securityContext:
        fsGroup: 1000
```

### Service

```yaml
# k8s-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

### Horizontal Pod Autoscaler

```yaml
# k8s-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  # ... rest of spec
```

### Apply and Manage

```bash
# Apply manifests
kubectl apply -f k8s-deployment.yaml
kubectl apply -f k8s-service.yaml
kubectl apply -f k8s-hpa.yaml

# Rolling update
kubectl set image deployment/myapp myapp=myregistry/myapp:v2.0

# Check rollout status
kubectl rollout status deployment/myapp

# View rollout history
kubectl rollout history deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Scale deployment
kubectl scale deployment myapp --replicas=5

# View logs
kubectl logs -f deployment/myapp
```

## Environment Promotion

### GitHub Actions Workflow

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker push myapp:${{ github.sha }}
      
      - name: Deploy to staging
        run: |
          kubectl config use-context staging
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
      
      - name: Smoke tests on staging
        run: |
          sleep 30
          curl -f https://staging.myapp.com/health
          npm run test:smoke -- --env=staging
      
      - name: Deploy to production
        if: success()
        run: |
          kubectl config use-context production
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
      
      - name: Production smoke tests
        run: |
          sleep 30
          curl -f https://myapp.com/health
          npm run test:smoke -- --env=production
```

## Database Migrations

### Init Container Pattern

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
        - name: db-migrate
          image: myregistry/myapp-migrate:latest
          command: ["npm", "run", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
      containers:
        - name: myapp
          image: myregistry/myapp:latest
          # ...
```

## Rollback Strategies

### Automatic Rollback on Failure

```yaml
# docker-compose.prod.yml
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

### Manual Rollback

```bash
# Docker Swarm
docker service update --rollback myapp_app

# Kubernetes
kubectl rollout undo deployment/myapp

# Docker Compose
docker-compose -f docker-compose.prod.yml pull myapp:previous
docker-compose -f docker-compose.prod.yml up -d
```

## Monitoring Deployments

### Health Checks

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1
```

```yaml
# docker-compose.yml
services:
  app:
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Deployment Notifications

```yaml
# .github/workflows/deploy.yml
name: Deploy

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
          docker-compose -f docker-compose.prod.yml up -d
      
      - name: Notify Slack - Start
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "🚀 Deployment started: ${{ github.sha }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Health check
        run: |
          sleep 30
          curl -f http://localhost/health
      
      - name: Notify Slack - Success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "✅ Deployment successful: ${{ github.sha }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify Slack - Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "❌ Deployment failed: ${{ github.sha }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```
