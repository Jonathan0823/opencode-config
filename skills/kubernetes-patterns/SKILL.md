---
name: kubernetes-patterns
description: Kubernetes deployment patterns, resource management, networking, security, and production best practices. Use when deploying containerized applications to Kubernetes, managing clusters, configuring ingress, or implementing scaling strategies.
license: MIT
compatibility: opencode
---

# Kubernetes Patterns Skill

## Overview

This skill provides comprehensive Kubernetes deployment patterns for production workloads including resource management, networking configuration, security hardening, auto-scaling, and multi-environment setups.

## Quick Start

### Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
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
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### Service Exposure

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

## Core Concepts

### 1. Resource Management
- **Requests**: Guaranteed resources for scheduling
- **Limits**: Maximum resources allowed
- **Quality of Service**: Guaranteed, Burstable, BestEffort

### 2. Pod Lifecycle
- **Pending**: Scheduling/initialization
- **Running**: All containers running
- **Succeeded**: Completed successfully
- **Failed**: Terminated with error
- **Unknown**: State cannot be determined

### 3. Service Types
- **ClusterIP**: Internal cluster access only
- **NodePort**: Exposes port on each node
- **LoadBalancer**: Cloud provider load balancer
- **ExternalName**: Maps to external DNS

## Production Patterns

### Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Graceful Shutdown

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]

t terminationGracePeriodSeconds: 60
```

### Security Context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

## Scaling Strategies

### Horizontal Pod Autoscaler

```yaml
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
  maxReplicas: 20
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### Vertical Pod Autoscaler

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        minAllowed:
          cpu: 50m
          memory: 100Mi
        maxAllowed:
          cpu: 1000m
          memory: 1Gi
        controlledResources: ["cpu", "memory"]
```

## Detailed References

See comprehensive guides in references/:

- **[Deployment Patterns](references/deployment-patterns.md)** - Deployments, StatefulSets, DaemonSets, Jobs
- **[Networking](references/networking.md)** - Services, Ingress, Network Policies, Service Mesh
- **[Security](references/security.md)** - RBAC, Pod Security, Network Policies, Secrets
- **[Storage](references/storage.md)** - Volumes, Persistent Volumes, ConfigMaps, StatefulSets

## When to Use This Skill

Use this skill when:
- Deploying containerized applications to Kubernetes
- Configuring auto-scaling and resource management
- Setting up ingress and load balancing
- Implementing security policies and RBAC
- Managing secrets and configurations
- Troubleshooting pod issues
- Planning cluster capacity
- Migrating from Docker Compose to Kubernetes

## Related Skills

- `@docker-patterns` - Containerization basics
- `@ci-cd-pipelines` - CI/CD with Kubernetes
- `@security-best-practices` - Container and cluster security
- `@observability-monitoring` - Monitoring Kubernetes workloads
- `@microservices-patterns` - Service mesh and distributed systems
