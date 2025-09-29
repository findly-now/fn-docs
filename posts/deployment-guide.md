# fn-posts Deployment Guide

**Complete deployment guide for the Posts domain service across development, staging, and production environments.**

## Prerequisites

### Cloud Infrastructure
- **Google Cloud Platform** account with billing enabled
- **Google Cloud Storage** bucket for photo storage
- **Supabase** project for managed PostgreSQL with PostGIS
- **Confluent Cloud** Kafka cluster for event publishing
- **Kubernetes cluster** (GKE, EKS, or AKS) for container orchestration

### Required Tools
- **Docker** 20.10+ for containerization
- **kubectl** for Kubernetes management
- **Helm** 3.0+ for Kubernetes deployments
- **Go** 1.25+ for local development
- **Make** for build automation

### Platform Compatibility
- **Apple Silicon Support**: Local development uses `platform: linux/amd64` in docker-compose files for PostgreSQL compatibility
- **Multi-platform Builds**: Production supports both AMD64 and ARM64 architectures
- **See**: [Platform Compatibility Guide](../CLOUD-SETUP.md#platform-compatibility)

### Access & Credentials
- GCP service account with Storage Admin permissions
- Supabase database connection string with PostGIS enabled
- Confluent Cloud Kafka bootstrap servers and API keys
- Container registry access (Google Container Registry or Docker Hub)

## Environment Configuration

### Environment Variables

**Core Service Configuration**:
```bash
# Service Configuration
PORT=8080
GIN_MODE=release                    # development | test | release
LOG_LEVEL=info                      # debug | info | warn | error
ENVIRONMENT=production              # development | staging | production

# Database (Supabase PostgreSQL with PostGIS)
DATABASE_URL=postgresql://username:password@host:5432/database?sslmode=require
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=10
DB_CONN_MAX_LIFETIME=5m

# Google Cloud Storage
GOOGLE_CLOUD_PROJECT=your-gcp-project-id
GCS_BUCKET_NAME=fn-photos-production
GCS_CREDENTIALS_JSON=base64-encoded-service-account-key
STORAGE_PUBLIC_URL=https://storage.googleapis.com/fn-photos-production

# Kafka Event Publishing (Confluent Cloud)
KAFKA_BOOTSTRAP_SERVERS=pkc-xyz.region.provider.confluent.cloud:9092
KAFKA_USERNAME=your-confluent-api-key
KAFKA_PASSWORD=your-confluent-api-secret
KAFKA_TOPIC_PREFIX=production-fn-posts
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN

# Authentication & Security
JWT_SECRET=your-256-bit-secret-key
JWT_EXPIRATION=24h
ALLOWED_ORIGINS=https://app.findlynow.com,https://admin.findlynow.com
CORS_ENABLED=true

# Performance & Limits
MAX_PHOTO_SIZE_MB=10
MAX_PHOTOS_PER_POST=10
MAX_REQUEST_SIZE_MB=50
REQUEST_TIMEOUT=30s
GEOSPATIAL_QUERY_TIMEOUT=5s

# Health Checks
HEALTH_CHECK_INTERVAL=30s
READINESS_TIMEOUT=10s
LIVENESS_TIMEOUT=5s
```

**Development Environment (.env.dev)**:
```bash
PORT=8080
GIN_MODE=debug
LOG_LEVEL=debug
ENVIRONMENT=development
DATABASE_URL=postgresql://postgres:password@localhost:5432/fn_posts_dev?sslmode=disable
GCS_BUCKET_NAME=fn-photos-dev
KAFKA_TOPIC_PREFIX=dev-fn-posts
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001
```

**Production Environment (.env.prod)**:
```bash
PORT=8080
GIN_MODE=release
LOG_LEVEL=info
ENVIRONMENT=production
DATABASE_URL=${SUPABASE_DATABASE_URL}
GCS_BUCKET_NAME=fn-photos-production
KAFKA_TOPIC_PREFIX=production-fn-posts
ALLOWED_ORIGINS=https://app.findlynow.com
```

## Database Setup

### Supabase Configuration

**1. Create Supabase Project**:
```bash
# Create new Supabase project via web interface
# Note the connection string and API keys
```

**2. Enable PostGIS Extension**:
```sql
-- Execute in Supabase SQL Editor
CREATE EXTENSION IF NOT EXISTS postgis;

-- Verify PostGIS installation
SELECT PostGIS_Version();
```

**3. Apply Database Schema**:
```bash
# Using psql with connection string
psql "$DATABASE_URL" -f script.sql

# Or using Makefile
export DATABASE_URL="your-supabase-connection-string"
make migrate-up
```

**4. Configure Database Performance**:
```sql
-- Optimize for geospatial queries (execute as superuser if possible)
ALTER DATABASE your_database SET shared_preload_libraries = 'postgis';
ALTER DATABASE your_database SET work_mem = '256MB';
ALTER DATABASE your_database SET max_connections = 100;
```

### Schema Validation

```bash
# Verify schema is correctly applied
psql "$DATABASE_URL" -c "
  SELECT table_name, column_name, data_type
  FROM information_schema.columns
  WHERE table_schema = 'public'
    AND table_name IN ('posts', 'post_photos')
  ORDER BY table_name, ordinal_position;
"

# Verify PostGIS indexes
psql "$DATABASE_URL" -c "
  SELECT indexname, tablename, indexdef
  FROM pg_indexes
  WHERE tablename IN ('posts', 'post_photos')
    AND indexdef LIKE '%GIST%';
"
```

## Google Cloud Storage Setup

### Bucket Configuration

**1. Create Storage Bucket**:
```bash
# Create bucket with appropriate location and storage class
gsutil mb -p your-gcp-project-id -c STANDARD -l us-central1 gs://fn-photos-production

# Set bucket lifecycle for automatic cleanup of orphaned files
gsutil lifecycle set bucket-lifecycle.json gs://fn-photos-production
```

**2. Bucket Lifecycle Configuration (bucket-lifecycle.json)**:
```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "Delete"},
        "condition": {
          "age": 365,
          "matchesPrefix": ["tmp/"]
        }
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {
          "age": 90,
          "matchesStorageClass": ["STANDARD"]
        }
      }
    ]
  }
}
```

**3. Set Bucket Permissions**:
```bash
# Make bucket publicly readable for photos
gsutil iam ch allUsers:objectViewer gs://fn-photos-production

# Create service account for application access
gcloud iam service-accounts create fn-posts-storage \
  --display-name="Posts Service Storage Account"

# Grant service account access to bucket
gsutil iam ch serviceAccount:fn-posts-storage@your-project.iam.gserviceaccount.com:objectAdmin gs://fn-photos-production

# Generate and download service account key
gcloud iam service-accounts keys create fn-posts-storage-key.json \
  --iam-account=fn-posts-storage@your-project.iam.gserviceaccount.com
```

**4. Configure CDN (Optional)**:
```bash
# Create load balancer and CDN for global photo distribution
gcloud compute backend-buckets create fn-photos-backend \
  --gcs-bucket-name=fn-photos-production

gcloud compute url-maps create fn-photos-lb \
  --default-backend-bucket=fn-photos-backend

gcloud compute target-https-proxies create fn-photos-proxy \
  --url-map=fn-photos-lb \
  --ssl-certificates=your-ssl-cert

gcloud compute forwarding-rules create fn-photos-forwarding-rule \
  --global \
  --target-https-proxy=fn-photos-proxy \
  --ports=443
```

## Kafka Setup (Confluent Cloud)

### Cluster Configuration

**1. Create Confluent Cloud Cluster**:
```bash
# Using Confluent CLI
confluent kafka cluster create fn-posts-production \
  --cloud gcp \
  --region us-central1 \
  --type basic

# Note the cluster ID and bootstrap servers
```

**2. Create Topics**:
```bash
# Posts domain events
confluent kafka topic create production-fn-posts.post.created \
  --partitions 6 \
  --config retention.ms=2592000000  # 30 days

confluent kafka topic create production-fn-posts.post.status_updated \
  --partitions 6 \
  --config retention.ms=2592000000

confluent kafka topic create production-fn-posts.post.resolved \
  --partitions 6 \
  --config retention.ms=2592000000

confluent kafka topic create production-fn-posts.photo.added \
  --partitions 3 \
  --config retention.ms=604800000   # 7 days
```

**3. Create API Keys**:
```bash
# Create API key for posts service
confluent api-key create --resource your-cluster-id \
  --description "Posts service production API key"

# Note the API key and secret for configuration
```

**4. Schema Registry (Optional)**:
```bash
# Enable Schema Registry for Avro schemas
confluent schema-registry cluster enable --cloud gcp --geo us

# Register event schemas
confluent schema-registry schema create \
  --subject production-fn-posts.post.created-value \
  --schema-file post-created-schema.avro
```

## Docker Containerization

### Dockerfile Optimization

```dockerfile
# Multi-stage build for production
FROM golang:1.25-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git ca-certificates tzdata

# Set working directory
WORKDIR /app

# Copy go mod and sum files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a -installsuffix cgo \
    -o main cmd/main.go

# Production stage
FROM scratch

# Copy timezone data and certificates
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the binary
COPY --from=builder /app/main /main

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/main", "-health-check"]

# Run the application
ENTRYPOINT ["/main"]
```

### Build and Push Images

```bash
# Build image for production
docker build -t fn-posts:latest .
docker tag fn-posts:latest gcr.io/your-project/fn-posts:latest
docker tag fn-posts:latest gcr.io/your-project/fn-posts:v1.0.0

# Push to Google Container Registry
docker push gcr.io/your-project/fn-posts:latest
docker push gcr.io/your-project/fn-posts:v1.0.0

# Or push to Docker Hub
docker tag fn-posts:latest your-dockerhub/fn-posts:latest
docker push your-dockerhub/fn-posts:latest
```

## Kubernetes Deployment

### Namespace and Resources

**1. Create Namespace**:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fn-posts
  labels:
    app: fn-posts
    environment: production
```

**2. Create ConfigMap**:
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fn-posts-config
  namespace: fn-posts
data:
  PORT: "8080"
  GIN_MODE: "release"
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
  GCS_BUCKET_NAME: "fn-photos-production"
  KAFKA_TOPIC_PREFIX: "production-fn-posts"
  MAX_PHOTO_SIZE_MB: "10"
  MAX_PHOTOS_PER_POST: "10"
  HEALTH_CHECK_INTERVAL: "30s"
```

**3. Create Secrets**:
```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: fn-posts-secrets
  namespace: fn-posts
type: Opaque
data:
  DATABASE_URL: <base64-encoded-database-url>
  GCS_CREDENTIALS_JSON: <base64-encoded-service-account-key>
  KAFKA_USERNAME: <base64-encoded-kafka-username>
  KAFKA_PASSWORD: <base64-encoded-kafka-password>
  JWT_SECRET: <base64-encoded-jwt-secret>
```

### Deployment Configuration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fn-posts
  namespace: fn-posts
  labels:
    app: fn-posts
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fn-posts
  template:
    metadata:
      labels:
        app: fn-posts
        version: v1
    spec:
      containers:
      - name: fn-posts
        image: gcr.io/your-project/fn-posts:v1.0.0
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: fn-posts-config
        - secretRef:
            name: fn-posts-secrets
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
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
      imagePullSecrets:
      - name: gcr-json-key
```

### Service and Ingress

**1. Service Configuration**:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fn-posts-service
  namespace: fn-posts
  labels:
    app: fn-posts
spec:
  selector:
    app: fn-posts
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
```

**2. Ingress Configuration**:
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fn-posts-ingress
  namespace: fn-posts
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.findlynow.com"
spec:
  tls:
  - hosts:
    - api.findlynow.com
    secretName: api-findlynow-com-tls
  rules:
  - host: api.findlynow.com
    http:
      paths:
      - path: /posts
        pathType: Prefix
        backend:
          service:
            name: fn-posts-service
            port:
              number: 80
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fn-posts-hpa
  namespace: fn-posts
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fn-posts
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

## Deployment Process

### CI/CD Pipeline

**1. GitHub Actions Workflow**:
```yaml
# .github/workflows/deploy.yml
name: Deploy Posts Service

on:
  push:
    branches: [main]
    paths: ['fn-posts/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-go@v4
      with:
        go-version: '1.25'
    - name: Run tests
      run: |
        cd fn-posts
        make e2e-test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Configure GCP credentials
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}

    - name: Configure Docker
      run: gcloud auth configure-docker

    - name: Build and push Docker image
      run: |
        cd fn-posts
        docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-posts:${{ github.sha }} .
        docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-posts:${{ github.sha }}

    - name: Deploy to Kubernetes
      run: |
        gcloud container clusters get-credentials fn-production --zone us-central1-a
        kubectl set image deployment/fn-posts fn-posts=gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-posts:${{ github.sha }} -n fn-posts
        kubectl rollout status deployment/fn-posts -n fn-posts
```

**2. Manual Deployment**:
```bash
# Apply Kubernetes manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# Verify deployment
kubectl get pods -n fn-posts
kubectl logs -f deployment/fn-posts -n fn-posts

# Check service health
kubectl port-forward svc/fn-posts-service 8080:80 -n fn-posts
curl http://localhost:8080/health
```

### Rolling Updates

```bash
# Update to new version
kubectl set image deployment/fn-posts fn-posts=gcr.io/your-project/fn-posts:v1.1.0 -n fn-posts

# Monitor rollout
kubectl rollout status deployment/fn-posts -n fn-posts

# Rollback if needed
kubectl rollout undo deployment/fn-posts -n fn-posts

# Check rollout history
kubectl rollout history deployment/fn-posts -n fn-posts
```

## Monitoring and Observability

### Health Checks

**Application Health Endpoints**:
```go
// Health check implementation
func (h *HealthHandler) Health(c *gin.Context) {
    status := "healthy"
    checks := map[string]string{}

    // Database connectivity
    if err := h.db.Ping(); err != nil {
        checks["database"] = "unhealthy: " + err.Error()
        status = "unhealthy"
    } else {
        checks["database"] = "healthy"
    }

    // GCS connectivity
    if err := h.gcsClient.Ping(); err != nil {
        checks["storage"] = "unhealthy: " + err.Error()
        status = "unhealthy"
    } else {
        checks["storage"] = "healthy"
    }

    // Kafka connectivity
    if err := h.kafkaClient.Ping(); err != nil {
        checks["kafka"] = "unhealthy: " + err.Error()
        status = "unhealthy"
    } else {
        checks["kafka"] = "healthy"
    }

    httpStatus := 200
    if status == "unhealthy" {
        httpStatus = 503
    }

    c.JSON(httpStatus, gin.H{
        "status": status,
        "checks": checks,
        "timestamp": time.Now().UTC(),
        "version": version.BuildVersion,
    })
}
```

### Logging Configuration

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: fn-posts
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/fn-posts*.log
      pos_file /var/log/fluentd-fn-posts.log.pos
      tag kubernetes.fn-posts
      format json
    </source>

    <filter kubernetes.fn-posts>
      @type kubernetes_metadata
      @log_level warn
    </filter>

    <match kubernetes.fn-posts>
      @type google_cloud
      project_id your-gcp-project-id
      zone us-central1-a
      vm_id vm-instance
      use_metadata_service true
    </match>
```

### Metrics and Alerting

**Prometheus Metrics**:
```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fn-posts-metrics
  namespace: fn-posts
spec:
  selector:
    matchLabels:
      app: fn-posts
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
```

**Alert Rules**:
```yaml
# alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: fn-posts-alerts
  namespace: fn-posts
spec:
  groups:
  - name: fn-posts.rules
    rules:
    - alert: PostsServiceDown
      expr: up{job="fn-posts"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Posts service is down"
        description: "Posts service has been down for more than 1 minute"

    - alert: HighErrorRate
      expr: rate(http_requests_total{job="fn-posts",status=~"5.."}[5m]) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate in Posts service"
        description: "Error rate is {{ $value }} per second"

    - alert: HighResponseTime
      expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="fn-posts"}[5m])) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High response time in Posts service"
        description: "95th percentile response time is {{ $value }} seconds"
```

## Security Considerations

### Network Security

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fn-posts-network-policy
  namespace: fn-posts
spec:
  podSelector:
    matchLabels:
      app: fn-posts
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
      port: 8080
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443  # HTTPS
    - protocol: TCP
      port: 5432 # PostgreSQL
    - protocol: TCP
      port: 9092 # Kafka
```

### Security Scanning

```bash
# Scan Docker image for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd):/src -w /src \
  aquasec/trivy image gcr.io/your-project/fn-posts:latest

# Scan Kubernetes manifests
kubectl apply --dry-run=server -f k8s/ | kubectl diff -f -
```

## Backup and Disaster Recovery

### Database Backups

```bash
# Automated backup via Supabase (configured in Supabase dashboard)
# Manual backup
pg_dump "$DATABASE_URL" > fn-posts-backup-$(date +%Y%m%d).sql

# Restore from backup
psql "$DATABASE_URL" < fn-posts-backup-20240115.sql
```

### Photo Storage Backup

```bash
# Cross-region replication (configure in GCS)
gsutil rsync -r -d gs://fn-photos-production gs://fn-photos-backup-us-east1
```

### Disaster Recovery Plan

1. **RTO (Recovery Time Objective)**: 15 minutes
2. **RPO (Recovery Point Objective)**: 1 hour
3. **Multi-region deployment** for high availability
4. **Database standby** in different region
5. **Photo storage replication** across regions
6. **Kafka topic replication** across availability zones

---

*For architecture decisions, see [domain-architecture.md](domain-architecture.md). For API usage, see [api-documentation.md](api-documentation.md). For system-wide deployment patterns, see [../fn-infra/](../fn-infra/).*