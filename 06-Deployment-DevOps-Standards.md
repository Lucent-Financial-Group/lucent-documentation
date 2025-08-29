# Deployment and DevOps Standards for Lucent Platform

## Overview

This document establishes deployment strategies, containerization standards, CI/CD pipeline configurations, and DevOps best practices for the Lucent DeFi platform. These standards ensure reliable, scalable, and secure deployment across all environments.

## Containerization Strategy

### Docker Standards

#### Base Image Selection

```dockerfile
# Backend services - Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS runtime
RUN apk add --no-cache dumb-init
ENV NODE_ENV production
USER node
WORKDIR /app
COPY --chown=node:node --from=builder /app/node_modules ./node_modules
COPY --chown=node:node . .
EXPOSE 3000
ENTRYPOINT ["dumb-init", "--"]
CMD ["npm", "run", "start:prod"]

# Frontend - Nginx serving static files
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Service-Specific Dockerfiles

```dockerfile
# user-service/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY tsconfig*.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build

FROM node:20-alpine AS runtime
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nestjs -u 1001
WORKDIR /app
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --chown=nestjs:nodejs package*.json ./

USER nestjs
EXPOSE 3001
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3001/health || exit 1

CMD ["node", "dist/main.js"]

# wallet-service/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY tsconfig*.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build

FROM node:20-alpine AS runtime
# Install security updates
RUN apk upgrade --no-cache
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nestjs -u 1001

# Install Azure CLI for Key Vault access
RUN apk add --no-cache azure-cli

WORKDIR /app
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --chown=nestjs:nodejs package*.json ./

USER nestjs
EXPOSE 3002
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3002/health || exit 1

CMD ["node", "dist/main.js"]
```

### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Databases
  mongodb:
    image: mongo:7
    container_name: lucent-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
      - ./docker/mongodb/init:/docker-entrypoint-initdb.d:ro
    networks:
      - lucent-network

  redis:
    image: redis:7-alpine
    container_name: lucent-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - lucent-network

  # API Gateway
  api-gateway:
    build:
      context: ./apps/api-gateway
      dockerfile: Dockerfile.dev
    container_name: lucent-api-gateway
    environment:
      - NODE_ENV=development
      - PORT=3000
      - USER_SERVICE_URL=http://user-service:3001
      - WALLET_SERVICE_URL=http://wallet-service:3002
      - PLAY_SERVICE_URL=http://play-service:3003
    ports:
      - "3000:3000"
    depends_on:
      - user-service
      - wallet-service
      - play-service
    volumes:
      - ./apps/api-gateway:/app
      - /app/node_modules
    networks:
      - lucent-network

  # Microservices
  user-service:
    build:
      context: ./apps/user-service
      dockerfile: Dockerfile.dev
    container_name: lucent-user-service
    environment:
      - NODE_ENV=development
      - PORT=3001
      - MONGODB_URL=mongodb://admin:password123@mongodb:27017/lucent-users?authSource=admin
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev-secret-key
    ports:
      - "3001:3001"
    depends_on:
      - mongodb
      - redis
    volumes:
      - ./apps/user-service:/app
      - /app/node_modules
    networks:
      - lucent-network

  wallet-service:
    build:
      context: ./apps/wallet-service
      dockerfile: Dockerfile.dev
    container_name: lucent-wallet-service
    environment:
      - NODE_ENV=development
      - PORT=3002
      - MONGODB_URL=mongodb://admin:password123@mongodb:27017/lucent-wallets?authSource=admin
      - REDIS_URL=redis://redis:6379
      - AZURE_KEY_VAULT_URL=${AZURE_KEY_VAULT_URL}
      - AZURE_CLIENT_ID=${AZURE_CLIENT_ID}
      - AZURE_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
      - AZURE_TENANT_ID=${AZURE_TENANT_ID}
    ports:
      - "3002:3002"
    depends_on:
      - mongodb
      - redis
    volumes:
      - ./apps/wallet-service:/app
      - /app/node_modules
    networks:
      - lucent-network

  play-service:
    build:
      context: ./apps/play-service
      dockerfile: Dockerfile.dev
    container_name: lucent-play-service
    environment:
      - NODE_ENV=development
      - PORT=3003
      - MONGODB_URL=mongodb://admin:password123@mongodb:27017/lucent-plays?authSource=admin
      - REDIS_URL=redis://redis:6379
    ports:
      - "3003:3003"
    depends_on:
      - mongodb
      - redis
    volumes:
      - ./apps/play-service:/app
      - /app/node_modules
    networks:
      - lucent-network

  # Frontend
  frontend:
    build:
      context: ./apps/frontend
      dockerfile: Dockerfile.dev
    container_name: lucent-frontend
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:3000/api/v1
      - NEXT_PUBLIC_WS_URL=ws://localhost:3000
    ports:
      - "3010:3000"
    volumes:
      - ./apps/frontend:/app
      - /app/node_modules
    networks:
      - lucent-network

volumes:
  mongodb_data:
  redis_data:

networks:
  lucent-network:
    driver: bridge
```

## Kubernetes Deployment

### Namespace and Configuration

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lucent-platform
  labels:
    name: lucent-platform

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lucent-config
  namespace: lucent-platform
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  API_VERSION: "v1"
  CORS_ORIGIN: "https://app.lucent.com"
  RATE_LIMIT_WINDOW: "15"
  RATE_LIMIT_MAX: "100"

---
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucent-secrets
  namespace: lucent-platform
type: Opaque
data:
  jwt-secret: <base64-encoded-secret>
  mongodb-password: <base64-encoded-password>
  redis-password: <base64-encoded-password>
  azure-client-secret: <base64-encoded-secret>
```

### Database Deployments

```yaml
# k8s/mongodb.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: lucent-platform
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:7
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: admin
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lucent-secrets
              key: mongodb-password
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: lucent-platform
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017

---
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: lucent-platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "256Mi"
            cpu: "125m"
          limits:
            memory: "512Mi"
            cpu: "250m"
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: lucent-platform
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

### Microservice Deployments

```yaml
# k8s/user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: lucent-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: lucent/user-service:latest
        ports:
        - containerPort: 3001
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: lucent-config
              key: NODE_ENV
        - name: PORT
          value: "3001"
        - name: MONGODB_URL
          value: "mongodb://admin:$(MONGODB_PASSWORD)@mongodb-service:27017/lucent-users?authSource=admin"
        - name: MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lucent-secrets
              key: mongodb-password
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: lucent-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "125m"
          limits:
            memory: "512Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: lucent-platform
spec:
  selector:
    app: user-service
  ports:
  - port: 3001
    targetPort: 3001
  type: ClusterIP

---
# k8s/wallet-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-service
  namespace: lucent-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wallet-service
  template:
    metadata:
      labels:
        app: wallet-service
    spec:
      serviceAccountName: azure-key-vault-sa
      containers:
      - name: wallet-service
        image: lucent/wallet-service:latest
        ports:
        - containerPort: 3002
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: lucent-config
              key: NODE_ENV
        - name: PORT
          value: "3002"
        - name: MONGODB_URL
          value: "mongodb://admin:$(MONGODB_PASSWORD)@mongodb-service:27017/lucent-wallets?authSource=admin"
        - name: MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lucent-secrets
              key: mongodb-password
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        - name: AZURE_KEY_VAULT_URL
          value: "https://lucent-keyvault.vault.azure.net/"
        - name: AZURE_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: azure-credentials
              key: client-id
        - name: AZURE_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: azure-credentials
              key: client-secret
        - name: AZURE_TENANT_ID
          valueFrom:
            secretKeyRef:
              name: azure-credentials
              key: tenant-id
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3002
          initialDelaySeconds: 45
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3002
          initialDelaySeconds: 10
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: wallet-service
  namespace: lucent-platform
spec:
  selector:
    app: wallet-service
  ports:
  - port: 3002
    targetPort: 3002
  type: ClusterIP
```

### Ingress Configuration

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lucent-ingress
  namespace: lucent-platform
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.lucent.com
    - app.lucent.com
    secretName: lucent-tls-secret
  rules:
  - host: api.lucent.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway-service
            port:
              number: 3000
  - host: app.lucent.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, wallet-service, play-service, frontend]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
        cache-dependency-path: apps/${{ matrix.service }}/package-lock.json

    - name: Install dependencies
      run: |
        cd apps/${{ matrix.service }}
        npm ci

    - name: Run linting
      run: |
        cd apps/${{ matrix.service }}
        npm run lint

    - name: Run unit tests
      run: |
        cd apps/${{ matrix.service }}
        npm run test:unit

    - name: Run integration tests
      if: matrix.service != 'frontend'
      run: |
        cd apps/${{ matrix.service }}
        npm run test:integration

    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: apps/${{ matrix.service }}/coverage/lcov.info
        flags: ${{ matrix.service }}

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    strategy:
      matrix:
        service: [user-service, wallet-service, play-service, task-service, portfolio-service, notification-service, frontend]

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: apps/${{ matrix.service }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig

    - name: Deploy to staging
      run: |
        # Update image tags in deployment files
        sed -i 's|image: lucent/.*:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/user-service:develop|g' k8s/staging/user-service.yaml
        sed -i 's|image: lucent/.*:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/wallet-service:develop|g' k8s/staging/wallet-service.yaml
        
        # Apply manifests
        kubectl apply -f k8s/staging/ --recursive
        
        # Wait for rollout to complete
        kubectl rollout status deployment/user-service -n lucent-staging
        kubectl rollout status deployment/wallet-service -n lucent-staging

    - name: Run smoke tests
      run: |
        # Wait for services to be ready
        sleep 30
        
        # Run health checks
        curl -f https://staging-api.lucent.com/health || exit 1
        
        # Run smoke tests
        npm run test:smoke -- --env staging

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig

    - name: Deploy to production
      run: |
        # Blue-green deployment strategy
        ./scripts/blue-green-deploy.sh
        
    - name: Run production tests
      run: |
        # Run comprehensive tests against production
        npm run test:e2e -- --env production
        
    - name: Notify Slack
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Deployment Scripts

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

NAMESPACE="lucent-platform"
NEW_VERSION="${GITHUB_SHA:0:7}"
SERVICES=("user-service" "wallet-service" "play-service" "task-service" "portfolio-service" "notification-service")

echo "Starting blue-green deployment for version: $NEW_VERSION"

# Function to deploy a service
deploy_service() {
    local service=$1
    echo "Deploying $service..."
    
    # Create new deployment with green suffix
    sed "s/name: $service$/name: $service-green/g" k8s/production/$service.yaml | \
    sed "s/image: .*$/image: $REGISTRY\/$IMAGE_NAME\/$service:$NEW_VERSION/g" | \
    kubectl apply -f -
    
    # Wait for deployment to be ready
    kubectl rollout status deployment/$service-green -n $NAMESPACE --timeout=300s
    
    # Run health checks
    if ! kubectl exec -n $NAMESPACE deployment/$service-green -- curl -f http://localhost:300${service#*-}/health; then
        echo "Health check failed for $service-green"
        kubectl delete deployment $service-green -n $NAMESPACE
        exit 1
    fi
    
    echo "$service-green is healthy"
}

# Deploy all services to green
for service in "${SERVICES[@]}"; do
    deploy_service $service
done

echo "All green deployments are ready. Starting traffic switch..."

# Switch traffic to green deployments
for service in "${SERVICES[@]}"; do
    kubectl patch service $service -n $NAMESPACE -p '{"spec":{"selector":{"app":"'$service'-green"}}}'
    echo "Traffic switched to $service-green"
done

echo "Waiting 60 seconds to monitor green deployment..."
sleep 60

# Run smoke tests against new deployment
if npm run test:smoke -- --env production; then
    echo "Smoke tests passed. Cleaning up blue deployments..."
    
    # Remove old blue deployments
    for service in "${SERVICES[@]}"; do
        kubectl delete deployment $service -n $NAMESPACE --ignore-not-found=true
        kubectl patch deployment $service-green -n $NAMESPACE -p '{"metadata":{"name":"'$service'"}}'
        echo "Cleaned up old $service deployment"
    done
    
    echo "Blue-green deployment completed successfully!"
else
    echo "Smoke tests failed. Rolling back to blue deployment..."
    
    # Rollback traffic to blue
    for service in "${SERVICES[@]}"; do
        kubectl patch service $service -n $NAMESPACE -p '{"spec":{"selector":{"app":"'$service'"}}}'
        kubectl delete deployment $service-green -n $NAMESPACE
    done
    
    echo "Rollback completed"
    exit 1
fi
```

## Monitoring and Observability

### Prometheus Configuration

```yaml
# monitoring/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    rule_files:
      - "/etc/prometheus/rules/*.yml"

    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)

      - job_name: 'lucent-services'
        static_configs:
          - targets:
              - 'user-service:3001'
              - 'wallet-service:3002'
              - 'play-service:3003'
              - 'task-service:3004'
              - 'portfolio-service:3005'
              - 'notification-service:3006'
        metrics_path: /metrics
        scrape_interval: 10s

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - 'alertmanager:9093'

---
# monitoring/alerting-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  lucent-alerts.yml: |
    groups:
    - name: lucent-platform
      rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Service {{ $labels.job }} has been down for more than 1 minute."

      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s for {{ $labels.job }}"

      - alert: MongoDBDown
        expr: up{job="mongodb"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB is down"
          description: "MongoDB instance has been down for more than 1 minute."

      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"
          description: "Redis instance has been down for more than 1 minute."
```

### Logging Configuration

```yaml
# monitoring/fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*lucent*.log
      pos_file /var/log/lucent.log.pos
      tag kubernetes.lucent.*
      read_from_head true
      format json
      time_format %Y-%m-%dT%H:%M:%S.%NZ
    </source>

    <filter kubernetes.lucent.**>
      @type kubernetes_metadata
      ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      verify_ssl true
      bearer_token_file /var/run/secrets/kubernetes.io/serviceaccount/token
    </filter>

    <filter kubernetes.lucent.**>
      @type parser
      key_name log
      reserve_data true
      <parse>
        @type json
      </parse>
    </filter>

    <match kubernetes.lucent.**>
      @type elasticsearch
      host elasticsearch-service.monitoring.svc.cluster.local
      port 9200
      logstash_format true
      logstash_prefix lucent
      type_name _doc
    </match>
```

## Security Configuration

### Network Policies

```yaml
# security/network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lucent-network-policy
  namespace: lucent-platform
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  
  # Allow inter-service communication
  - from:
    - namespaceSelector:
        matchLabels:
          name: lucent-platform
    ports:
    - protocol: TCP
      port: 3001
    - protocol: TCP
      port: 3002
    - protocol: TCP
      port: 3003
  
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
  
  # Allow database connections
  - to:
    - podSelector:
        matchLabels:
          app: mongodb
    ports:
    - protocol: TCP
      port: 27017
  
  # Allow external HTTPS for blockchain APIs
  - to: []
    ports:
    - protocol: TCP
      port: 443

---
# Security context for services
apiVersion: v1
kind: SecurityPolicy
metadata:
  name: lucent-security-policy
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  allowedCapabilities: []
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

This comprehensive deployment and DevOps standards document provides the foundation for reliable, scalable, and secure deployment of the Lucent platform across all environments, ensuring consistent practices and automated processes throughout the development lifecycle.