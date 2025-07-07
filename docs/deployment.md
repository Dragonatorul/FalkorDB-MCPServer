# Production Deployment Guide

Complete guide for deploying FalkorDB MCP Server in production environments with Docker, orchestration platforms, and best practices.

## Quick Deployment Options

### Option 1: Docker Run (Simple)

```bash
# Start FalkorDB
docker run -d \
  --name falkordb \
  -p 6379:6379 \
  -v falkordb_data:/data \
  falkordb/falkordb:latest

# Start MCP Server
docker run -d \
  --name falkordb-mcp \
  -p 3000:3000 \
  -e FALKORDB_HOST=falkordb \
  -e MCP_API_KEY=your-secure-api-key \
  --link falkordb \
  ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
```

### Option 2: Docker Compose (Recommended)

```yaml
# docker-compose.yml
version: '3.8'

services:
  falkordb:
    image: falkordb/falkordb:latest
    container_name: falkordb
    ports:
      - "6379:6379"
    volumes:
      - falkordb_data:/data
    environment:
      - FALKORDB_SAVE_INTERVAL=3600
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  mcp-server:
    image: ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
    container_name: falkordb-mcp
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - FALKORDB_HOST=falkordb
      - FALKORDB_PORT=6379
      - MCP_API_KEY=${MCP_API_KEY}
      - ENABLE_MULTI_TENANCY=${ENABLE_MULTI_TENANCY:-false}
      - MULTI_TENANT_AUTH_MODE=${MULTI_TENANT_AUTH_MODE:-api-key}
      - TENANT_GRAPH_PREFIX=${TENANT_GRAPH_PREFIX:-false}
      - BEARER_JWKS_URI=${BEARER_JWKS_URI}
      - BEARER_ISSUER=${BEARER_ISSUER}
      - BEARER_ALGORITHM=${BEARER_ALGORITHM:-RS256}
      - BEARER_AUDIENCE=${BEARER_AUDIENCE}
    depends_on:
      falkordb:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  falkordb_data:
    driver: local

networks:
  default:
    driver: bridge
```

**Environment file (.env):**
```env
# Required
MCP_API_KEY=your-very-secure-api-key-here

# Multi-tenancy (optional)
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# JWT Configuration (required if multi-tenancy enabled)
BEARER_JWKS_URI=https://your-auth-provider.com/.well-known/jwks.json
BEARER_ISSUER=https://your-auth-provider.com
BEARER_AUDIENCE=falkordb-mcp-production
```

**Deploy:**
```bash
# Start the stack
docker-compose up -d

# View logs
docker-compose logs -f

# Stop the stack
docker-compose down
```

## Kubernetes Deployment

### Basic Kubernetes Manifests

**Namespace:**
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: falkordb-mcp
  labels:
    app: falkordb-mcp
```

**ConfigMap:**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falkordb-mcp-config
  namespace: falkordb-mcp
data:
  NODE_ENV: "production"
  FALKORDB_HOST: "falkordb-service"
  FALKORDB_PORT: "6379"
  ENABLE_MULTI_TENANCY: "true"
  MULTI_TENANT_AUTH_MODE: "bearer"
  TENANT_GRAPH_PREFIX: "true"
  BEARER_ALGORITHM: "RS256"
```

**Secret:**
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: falkordb-mcp-secret
  namespace: falkordb-mcp
type: Opaque
stringData:
  MCP_API_KEY: "your-secure-api-key"
  BEARER_JWKS_URI: "https://auth.company.com/.well-known/jwks.json"
  BEARER_ISSUER: "https://auth.company.com"
  BEARER_AUDIENCE: "falkordb-mcp-production"
```

**FalkorDB Deployment:**
```yaml
# falkordb-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falkordb
  namespace: falkordb-mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: falkordb
  template:
    metadata:
      labels:
        app: falkordb
    spec:
      containers:
      - name: falkordb
        image: falkordb/falkordb:latest
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: falkordb-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: falkordb-service
  namespace: falkordb-mcp
spec:
  selector:
    app: falkordb
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: falkordb-pvc
  namespace: falkordb-mcp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**MCP Server Deployment:**
```yaml
# mcp-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falkordb-mcp-server
  namespace: falkordb-mcp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: falkordb-mcp-server
  template:
    metadata:
      labels:
        app: falkordb-mcp-server
    spec:
      containers:
      - name: mcp-server
        image: ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: falkordb-mcp-config
        - secretRef:
            name: falkordb-mcp-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
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
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: falkordb-mcp-service
  namespace: falkordb-mcp
spec:
  selector:
    app: falkordb-mcp-server
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

**Ingress:**
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: falkordb-mcp-ingress
  namespace: falkordb-mcp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - mcp-api.yourdomain.com
    secretName: falkordb-mcp-tls
  rules:
  - host: mcp-api.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: falkordb-mcp-service
            port:
              number: 80
```

**Deploy to Kubernetes:**
```bash
# Apply all manifests
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f falkordb-deployment.yaml
kubectl apply -f mcp-server-deployment.yaml
kubectl apply -f ingress.yaml

# Check deployment status
kubectl get pods -n falkordb-mcp
kubectl get services -n falkordb-mcp
kubectl logs -f deployment/falkordb-mcp-server -n falkordb-mcp
```

## Helm Chart Deployment

### Helm Chart Structure

```
falkordb-mcp-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml
│   └── pvc.yaml
└── charts/
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: falkordb-mcp
description: A Helm chart for FalkorDB MCP Server
type: application
version: 1.1.0
appVersion: "1.1.0"
dependencies:
- name: falkordb
  version: "1.0.0"
  repository: "https://charts.falkordb.com"
  condition: falkordb.enabled
```

**values.yaml:**
```yaml
# Default values for falkordb-mcp
replicaCount: 3

image:
  repository: ghcr.io/dragonatorul/falkordb-mcpserver
  tag: "1.1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: mcp-api.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: falkordb-mcp-tls
      hosts:
        - mcp-api.yourdomain.com

config:
  nodeEnv: production
  enableMultiTenancy: true
  multiTenantAuthMode: bearer
  tenantGraphPrefix: true
  bearer:
    algorithm: RS256

secrets:
  mcpApiKey: "your-secure-api-key"
  bearerJwksUri: "https://auth.company.com/.well-known/jwks.json"
  bearerIssuer: "https://auth.company.com"
  bearerAudience: "falkordb-mcp-production"

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

falkordb:
  enabled: true
  persistence:
    enabled: true
    size: 10Gi
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
    requests:
      cpu: 250m
      memory: 512Mi
```

**Install with Helm:**
```bash
# Add the repository (if available)
helm repo add falkordb-mcp https://charts.yourdomain.com

# Install from local chart
helm install falkordb-mcp ./falkordb-mcp-chart \
  --namespace falkordb-mcp \
  --create-namespace \
  --values production-values.yaml

# Upgrade
helm upgrade falkordb-mcp ./falkordb-mcp-chart \
  --namespace falkordb-mcp

# Uninstall
helm uninstall falkordb-mcp --namespace falkordb-mcp
```

## Cloud-Specific Deployments

### AWS ECS with Fargate

**Task Definition:**
```json
{
  "family": "falkordb-mcp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "falkordb",
      "image": "falkordb/falkordb:latest",
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 6379,
          "protocol": "tcp"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "falkordb-data",
          "containerPath": "/data"
        }
      ]
    },
    {
      "name": "mcp-server",
      "image": "ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0",
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        },
        {
          "name": "FALKORDB_HOST",
          "value": "localhost"
        },
        {
          "name": "FALKORDB_PORT",
          "value": "6379"
        }
      ],
      "secrets": [
        {
          "name": "MCP_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:falkordb-mcp-secrets:MCP_API_KEY::"
        }
      ],
      "dependsOn": [
        {
          "containerName": "falkordb",
          "condition": "HEALTHY"
        }
      ]
    }
  ],
  "volumes": [
    {
      "name": "falkordb-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-12345678",
        "transitEncryption": "ENABLED"
      }
    }
  ]
}
```

### Google Cloud Run

**cloudbuild.yaml:**
```yaml
steps:
  # Build the container image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/falkordb-mcp:$COMMIT_SHA', '.']
  
  # Push the container image to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/falkordb-mcp:$COMMIT_SHA']
  
  # Deploy container image to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
    - 'run'
    - 'deploy'
    - 'falkordb-mcp'
    - '--image'
    - 'gcr.io/$PROJECT_ID/falkordb-mcp:$COMMIT_SHA'
    - '--region'
    - 'us-central1'
    - '--platform'
    - 'managed'
    - '--allow-unauthenticated'
    - '--set-env-vars'
    - 'NODE_ENV=production,FALKORDB_HOST=your-falkordb-host'
    - '--set-secrets'
    - 'MCP_API_KEY=projects/$PROJECT_ID/secrets/mcp-api-key:latest'
```

### Azure Container Instances

**ARM Template:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "containerGroupName": {
      "type": "string",
      "defaultValue": "falkordb-mcp"
    }
  },
  "resources": [
    {
      "type": "Microsoft.ContainerInstance/containerGroups",
      "apiVersion": "2021-03-01",
      "name": "[parameters('containerGroupName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "containers": [
          {
            "name": "falkordb",
            "properties": {
              "image": "falkordb/falkordb:latest",
              "ports": [
                {
                  "port": 6379,
                  "protocol": "TCP"
                }
              ],
              "resources": {
                "requests": {
                  "cpu": 0.5,
                  "memoryInGB": 1
                }
              }
            }
          },
          {
            "name": "mcp-server",
            "properties": {
              "image": "ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0",
              "ports": [
                {
                  "port": 3000,
                  "protocol": "TCP"
                }
              ],
              "environmentVariables": [
                {
                  "name": "NODE_ENV",
                  "value": "production"
                },
                {
                  "name": "FALKORDB_HOST",
                  "value": "localhost"
                }
              ],
              "resources": {
                "requests": {
                  "cpu": 0.5,
                  "memoryInGB": 1
                }
              }
            }
          }
        ],
        "osType": "Linux",
        "ipAddress": {
          "type": "Public",
          "ports": [
            {
              "port": 3000,
              "protocol": "TCP"
            }
          ]
        },
        "restartPolicy": "Always"
      }
    }
  ]
}
```

## Load Balancing and High Availability

### NGINX Load Balancer

**nginx.conf:**
```nginx
upstream falkordb_mcp_backend {
    least_conn;
    server mcp-server-1:3000 max_fails=3 fail_timeout=30s;
    server mcp-server-2:3000 max_fails=3 fail_timeout=30s;
    server mcp-server-3:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name mcp-api.yourdomain.com;
    
    location / {
        proxy_pass http://falkordb_mcp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Health check
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    location /health {
        access_log off;
        proxy_pass http://falkordb_mcp_backend;
    }
}
```

### HAProxy Configuration

**haproxy.cfg:**
```
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend falkordb_mcp_frontend
    bind *:80
    default_backend falkordb_mcp_backend

backend falkordb_mcp_backend
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    
    server mcp1 mcp-server-1:3000 check
    server mcp2 mcp-server-2:3000 check
    server mcp3 mcp-server-3:3000 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
```

## Monitoring and Observability

### Prometheus Monitoring

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'falkordb-mcp'
    static_configs:
      - targets: ['mcp-server-1:3000', 'mcp-server-2:3000', 'mcp-server-3:3000']
    scrape_interval: 30s
    metrics_path: /metrics
    
  - job_name: 'falkordb'
    static_configs:
      - targets: ['falkordb:6379']
    scrape_interval: 30s
```

### Grafana Dashboard

**Dashboard Configuration:**
```json
{
  "dashboard": {
    "title": "FalkorDB MCP Server",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Database Connections",
        "type": "singlestat",
        "targets": [
          {
            "expr": "falkordb_connected_clients",
            "legendFormat": "Active Connections"
          }
        ]
      }
    ]
  }
}
```

### Log Aggregation

**Fluentd Configuration:**
```yaml
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter docker.falkordb-mcp.**>
  @type parser
  key_name log
  <parse>
    @type json
    time_key timestamp
    time_type string
    time_format %Y-%m-%dT%H:%M:%S.%LZ
  </parse>
</filter>

<match docker.falkordb-mcp.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name falkordb-mcp-logs
  type_name _doc
</match>
```

## Security Considerations

### SSL/TLS Termination

**Let's Encrypt with Certbot:**
```bash
# Install certbot
sudo apt-get install certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d mcp-api.yourdomain.com

# Auto-renewal
sudo crontab -e
# Add: 0 12 * * * /usr/bin/certbot renew --quiet
```

### Firewall Configuration

**UFW Rules:**
```bash
# Allow SSH
sudo ufw allow 22

# Allow HTTP/HTTPS
sudo ufw allow 80
sudo ufw allow 443

# Allow specific application ports
sudo ufw allow from trusted-ip to any port 3000

# Enable firewall
sudo ufw enable
```

### Network Security

**Docker Network Isolation:**
```yaml
# docker-compose.yml
networks:
  internal:
    driver: bridge
    internal: true
  external:
    driver: bridge

services:
  falkordb:
    networks:
      - internal
  
  mcp-server:
    networks:
      - internal
      - external
    ports:
      - "3000:3000"
```

## Backup and Recovery

### Database Backup

**Automated Backup Script:**
```bash
#!/bin/bash
# backup-falkordb.sh

BACKUP_DIR="/backups/falkordb"
DATE=$(date +%Y%m%d_%H%M%S)
CONTAINER_NAME="falkordb"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup FalkorDB data
docker exec $CONTAINER_NAME redis-cli BGSAVE
docker cp $CONTAINER_NAME:/data/dump.rdb $BACKUP_DIR/falkordb_$DATE.rdb

# Compress backup
gzip $BACKUP_DIR/falkordb_$DATE.rdb

# Keep only last 7 days of backups
find $BACKUP_DIR -name "*.rdb.gz" -mtime +7 -delete

echo "Backup completed: falkordb_$DATE.rdb.gz"
```

**Cron Job:**
```bash
# Backup every 6 hours
0 */6 * * * /scripts/backup-falkordb.sh >> /var/log/falkordb-backup.log 2>&1
```

### Disaster Recovery

**Recovery Procedure:**
```bash
# Stop containers
docker-compose down

# Restore data
gunzip -c /backups/falkordb/falkordb_20250107_120000.rdb.gz > /var/lib/docker/volumes/falkordb_data/_data/dump.rdb

# Restart containers
docker-compose up -d

# Verify recovery
curl http://localhost:3000/health
```

## Performance Tuning

### Container Resource Limits

**Optimized Docker Compose:**
```yaml
services:
  falkordb:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
    
  mcp-server:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### FalkorDB Optimization

**FalkorDB Configuration:**
```conf
# falkordb.conf
save 900 1
save 300 10
save 60 10000

maxmemory 2gb
maxmemory-policy allkeys-lru

tcp-keepalive 300
timeout 300
```

### Application Tuning

**Node.js Performance:**
```dockerfile
# Dockerfile optimizations
FROM node:18-alpine

# Enable production optimizations
ENV NODE_ENV=production
ENV NODE_OPTIONS="--max-old-space-size=512"

# Use non-root user
USER node

# Optimize for single-threaded workload
CMD ["node", "--max-old-space-size=512", "dist/index.js"]
```

---

*This documentation was generated by Claude Code, an AI assistant, to provide comprehensive deployment guidance for the FalkorDB MCP Server in production environments.*