# fn-notifications Deployment Guide

**Complete deployment guide for the Notifications domain service with enterprise resilience patterns and multi-channel delivery orchestration.**

## Prerequisites

### Cloud Infrastructure
- **Confluent Cloud** Kafka cluster for event streaming
- **PostgreSQL** instance (Supabase or cloud provider)
- **Twilio** account for SMS and WhatsApp delivery
- **Email service** (SendGrid, Mailgun, or AWS SES)
- **Kubernetes cluster** for container orchestration
- **Redis** cluster for caching (optional but recommended)

### Required Tools
- **Docker** 20.10+ for containerization
- **kubectl** for Kubernetes management
- **Helm** 3.0+ for Kubernetes deployments
- **Elixir** 1.15+ and **OTP** 26+ for local development
- **Mix** build tool for Elixir applications

### Platform Compatibility
- **Apple Silicon Support**: Local development uses `platform: linux/amd64` in docker-compose files for PostgreSQL compatibility
- **Multi-platform Builds**: Production supports both AMD64 and ARM64 architectures
- **See**: [Platform Compatibility Guide](../CLOUD-SETUP.md#platform-compatibility)

### Access & Credentials
- Confluent Cloud API keys and cluster endpoints
- Database connection string with proper permissions
- Twilio Account SID, Auth Token, and phone numbers
- Email service API keys and configuration
- Container registry access for deployment

## Environment Configuration

### Environment Variables

**Core Service Configuration**:
```bash
# Service Configuration
PORT=4000
MIX_ENV=prod                        # dev | test | prod
PHX_HOST=notifications.findlynow.com
PHX_SERVER=true
SECRET_KEY_BASE=your-64-char-secret-key-base
LIVE_VIEW_SIGNING_SALT=your-32-char-salt

# Database (PostgreSQL)
DATABASE_URL=postgresql://username:password@host:5432/database?sslmode=require
POOL_SIZE=10
QUEUE_TARGET=50
QUEUE_INTERVAL=5000

# Confluent Cloud Kafka Configuration
KAFKA_HOSTS=pkc-xyz.region.provider.confluent.cloud:9092
KAFKA_USERNAME=your-confluent-api-key
KAFKA_PASSWORD=your-confluent-api-secret
KAFKA_SSL=true
KAFKA_SASL_MECHANISM=PLAIN
KAFKA_GROUP_ID=fn-notifications-production
KAFKA_TOPICS=fn-posts.post.created,fn-posts.post.matched,fn-posts.post.claimed,fn-posts.post.resolved,fn-users.user.registered

# Email Delivery (Swoosh + SendGrid)
EMAIL_ADAPTER=Swoosh.Adapters.Sendgrid
SENDGRID_API_KEY=your-sendgrid-api-key
FROM_EMAIL=notifications@findlynow.com
FROM_NAME=Findly Now

# SMS/WhatsApp Delivery (Twilio)
TWILIO_ACCOUNT_SID=your-twilio-account-sid
TWILIO_AUTH_TOKEN=your-twilio-auth-token
TWILIO_PHONE_NUMBER=+1234567890
TWILIO_WHATSAPP_NUMBER=+14155238886

# Caching (Cachex with optional Redis backend)
CACHE_ADAPTER=cachex                # cachex | redis
REDIS_URL=redis://username:password@host:6379/0
USER_PREFERENCES_CACHE_TTL=300000   # 5 minutes in milliseconds

# Circuit Breaker Configuration
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_TIMEOUT_MS=60000
CIRCUIT_BREAKER_RESET_TIMEOUT_MS=300000

# Bulkhead Resource Limits
EMAIL_POOL_MAX_CONCURRENCY=10
EMAIL_POOL_TIMEOUT_MS=30000
SMS_POOL_MAX_CONCURRENCY=5
SMS_POOL_TIMEOUT_MS=15000
WHATSAPP_POOL_MAX_CONCURRENCY=5
WHATSAPP_POOL_TIMEOUT_MS=15000

# Oban Background Jobs
OBAN_QUEUES=default:10,notifications:20,retries:5
OBAN_PRUNING_MAX_AGE=3600          # 1 hour
OBAN_RESCUE_ATTEMPTS=3

# Observability
TELEMETRY_ENABLED=true
PROMETHEUS_METRICS_ENABLED=true
LOG_LEVEL=info                      # debug | info | warn | error
LOGGER_JSON=true

# Security
CORS_ORIGINS=https://app.findlynow.com,https://admin.findlynow.com
CSRF_PROTECTION=true
SECURE_COOKIES=true
```

**Development Environment (.env.dev)**:
```bash
PORT=4000
MIX_ENV=dev
PHX_HOST=localhost
SECRET_KEY_BASE=dev-secret-key-base-minimum-64-characters-required-for-security
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fn_notifications_dev
KAFKA_HOSTS=localhost:9092
KAFKA_SSL=false
EMAIL_ADAPTER=Swoosh.Adapters.Local
CACHE_ADAPTER=cachex
LOG_LEVEL=debug
LOGGER_JSON=false
CORS_ORIGINS=http://localhost:3000
```

**Production Environment (.env.prod)**:
```bash
PORT=4000
MIX_ENV=prod
PHX_HOST=notifications.findlynow.com
PHX_SERVER=true
SECRET_KEY_BASE=${SECRET_KEY_BASE}
DATABASE_URL=${DATABASE_URL}
KAFKA_HOSTS=${KAFKA_HOSTS}
KAFKA_USERNAME=${KAFKA_USERNAME}
KAFKA_PASSWORD=${KAFKA_PASSWORD}
SENDGRID_API_KEY=${SENDGRID_API_KEY}
TWILIO_ACCOUNT_SID=${TWILIO_ACCOUNT_SID}
TWILIO_AUTH_TOKEN=${TWILIO_AUTH_TOKEN}
LOG_LEVEL=info
LOGGER_JSON=true
```

## Database Setup

### PostgreSQL Configuration

**1. Database Creation and Setup**:
```sql
-- Create database and user
CREATE DATABASE fn_notifications_production;
CREATE USER fn_notifications WITH PASSWORD 'secure-password';
GRANT ALL PRIVILEGES ON DATABASE fn_notifications_production TO fn_notifications;

-- Connect to the database
\c fn_notifications_production

-- Grant schema permissions
GRANT ALL ON SCHEMA public TO fn_notifications;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO fn_notifications;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO fn_notifications;
```

**2. Apply Database Schema**:
```bash
# Using psql with schema file
psql "$DATABASE_URL" -f schema.sql

# Or using Mix alias
export DATABASE_URL="your-database-connection-string"
mix schema.deploy

# Verify schema application
psql "$DATABASE_URL" -c "
  SELECT table_name, column_count
  FROM (
    SELECT table_name, COUNT(*) as column_count
    FROM information_schema.columns
    WHERE table_schema = 'public'
      AND table_name IN ('notifications', 'user_preferences', 'oban_jobs')
    GROUP BY table_name
  ) t;
"
```

**3. Database Performance Optimization**:
```sql
-- Optimize for notification workload
ALTER DATABASE fn_notifications_production SET work_mem = '256MB';
ALTER DATABASE fn_notifications_production SET shared_preload_libraries = 'pg_stat_statements';
ALTER DATABASE fn_notifications_production SET max_connections = 100;

-- Create additional performance indexes if needed
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_notifications_metadata_gin
ON notifications USING gin (metadata);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_notifications_retry_eligible
ON notifications (failed_at, retry_count)
WHERE status = 'failed' AND retry_count < max_retries;
```

### Schema Validation

```bash
# Verify all required tables exist
psql "$DATABASE_URL" -c "
  SELECT
    t.table_name,
    t.table_type,
    c.column_count
  FROM information_schema.tables t
  LEFT JOIN (
    SELECT table_name, COUNT(*) as column_count
    FROM information_schema.columns
    WHERE table_schema = 'public'
    GROUP BY table_name
  ) c ON t.table_name = c.table_name
  WHERE t.table_schema = 'public'
    AND t.table_type = 'BASE TABLE'
  ORDER BY t.table_name;
"

# Verify Oban job processing tables
psql "$DATABASE_URL" -c "
  SELECT
    schemaname,
    tablename,
    attname,
    typname,
    atttypmod
  FROM pg_attribute
  JOIN pg_class ON attrelid = pg_class.oid
  JOIN pg_type ON atttypid = pg_type.oid
  JOIN pg_namespace ON pg_class.relnamespace = pg_namespace.oid
  WHERE schemaname = 'public'
    AND tablename = 'oban_jobs'
    AND attnum > 0;
"
```

## Kafka Configuration

### Confluent Cloud Setup

**1. Create Confluent Cloud Cluster**:
```bash
# Using Confluent CLI
confluent kafka cluster create fn-notifications-production \
  --cloud gcp \
  --region us-central1 \
  --type basic

# Note cluster ID and bootstrap servers
export CLUSTER_ID="lkc-xyz123"
export BOOTSTRAP_SERVERS="pkc-xyz.us-central1.gcp.confluent.cloud:9092"
```

**2. Create Required Topics**:
```bash
# Posts domain events (consumed by notifications)
confluent kafka topic create fn-posts.post.created \
  --cluster $CLUSTER_ID \
  --partitions 6 \
  --config retention.ms=2592000000  # 30 days

confluent kafka topic create fn-posts.post.matched \
  --cluster $CLUSTER_ID \
  --partitions 6 \
  --config retention.ms=2592000000

confluent kafka topic create fn-posts.post.claimed \
  --cluster $CLUSTER_ID \
  --partitions 6 \
  --config retention.ms=604800000   # 7 days (urgent)

confluent kafka topic create fn-posts.post.resolved \
  --cluster $CLUSTER_ID \
  --partitions 3 \
  --config retention.ms=2592000000

# User domain events
confluent kafka topic create fn-users.user.registered \
  --cluster $CLUSTER_ID \
  --partitions 3 \
  --config retention.ms=2592000000

# Internal notification events (published by notifications)
confluent kafka topic create fn-notifications.notification.sent \
  --cluster $CLUSTER_ID \
  --partitions 3 \
  --config retention.ms=604800000

confluent kafka topic create fn-notifications.delivery.failed \
  --cluster $CLUSTER_ID \
  --partitions 3 \
  --config retention.ms=604800000
```

**3. Create API Keys and ACLs**:
```bash
# Create API key for notifications service
confluent api-key create --resource $CLUSTER_ID \
  --description "FN Notifications Service Production"

# Create service account for fine-grained access control
confluent iam service-account create fn-notifications-service \
  --description "Service account for FN Notifications"

# Grant necessary permissions
confluent kafka acl create \
  --allow \
  --service-account sa-xyz123 \
  --operation read \
  --topic "fn-posts.*"

confluent kafka acl create \
  --allow \
  --service-account sa-xyz123 \
  --operation write \
  --topic "fn-notifications.*"

confluent kafka acl create \
  --allow \
  --service-account sa-xyz123 \
  --operation read \
  --consumer-group "fn-notifications-*"
```

**4. Test Kafka Connectivity**:
```bash
# Test consumer connectivity
confluent kafka topic consume fn-posts.post.created \
  --cluster $CLUSTER_ID \
  --api-key $API_KEY \
  --api-secret $API_SECRET \
  --from-beginning

# Test producer connectivity
echo '{"test": "message"}' | \
confluent kafka topic produce fn-notifications.notification.sent \
  --cluster $CLUSTER_ID \
  --api-key $API_KEY \
  --api-secret $API_SECRET
```

## External Service Configuration

### Twilio Setup

**1. Account Configuration**:
```bash
# Verify Twilio credentials and services
curl -X GET "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID.json" \
  -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"

# List available phone numbers
curl -X GET "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/IncomingPhoneNumbers.json" \
  -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"

# Check WhatsApp Business profile
curl -X GET "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/Messages.json?From=whatsapp:$TWILIO_WHATSAPP_NUMBER" \
  -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"
```

**2. WhatsApp Business Setup**:
```bash
# Submit WhatsApp Business profile for approval
curl -X POST "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/Messages.json" \
  -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN" \
  -d "From=whatsapp:$TWILIO_WHATSAPP_NUMBER" \
  -d "To=whatsapp:+1234567890" \
  -d "Body=Test WhatsApp Business message from Findly Now"
```

### Email Service Setup

**1. SendGrid Configuration**:
```bash
# Verify SendGrid API key
curl -X GET "https://api.sendgrid.com/v3/user/account" \
  -H "Authorization: Bearer $SENDGRID_API_KEY" \
  -H "Content-Type: application/json"

# Create sender identity
curl -X POST "https://api.sendgrid.com/v3/verified_senders" \
  -H "Authorization: Bearer $SENDGRID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "nickname": "Findly Now Notifications",
    "from_email": "notifications@findlynow.com",
    "from_name": "Findly Now",
    "reply_to": "support@findlynow.com",
    "address": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94105",
    "country": "US"
  }'

# Set up webhook for delivery events
curl -X POST "https://api.sendgrid.com/v3/user/webhooks/event/settings" \
  -H "Authorization: Bearer $SENDGRID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "url": "https://notifications.findlynow.com/api/webhooks/sendgrid",
    "group_resubscribe": true,
    "delivered": true,
    "group_unsubscribe": true,
    "spam_report": true,
    "bounce": true,
    "deferred": true,
    "unsubscribe": true,
    "processed": true,
    "open": true,
    "click": true,
    "dropped": true
  }'
```

**2. Alternative Email Providers**:

**Mailgun Configuration**:
```bash
# Verify Mailgun domain
curl -X GET "https://api.mailgun.net/v3/domains/findlynow.com" \
  -u "api:$MAILGUN_API_KEY"

# Send test email
curl -X POST "https://api.mailgun.net/v3/findlynow.com/messages" \
  -u "api:$MAILGUN_API_KEY" \
  -F from='Findly Now <notifications@findlynow.com>' \
  -F to='test@example.com' \
  -F subject='Test Email from Findly Now' \
  -F text='This is a test email from the notifications service.'
```

**AWS SES Configuration**:
```bash
# Verify domain identity
aws ses verify-domain-identity --domain findlynow.com

# Get verification status
aws ses get-identity-verification-attributes --identities findlynow.com

# Set up SNS topic for bounce/complaint handling
aws sns create-topic --name findly-now-email-events
```

## Docker Containerization

### Multi-stage Dockerfile

```dockerfile
# Multi-stage build for Elixir/Phoenix application
ARG ELIXIR_VERSION=1.15.7
ARG OTP_VERSION=26.1.2
ARG DEBIAN_VERSION=bullseye-20231009-slim

# Build stage
FROM hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION} AS builder

# Install build dependencies
RUN apt-get update -y && apt-get install -y \
  build-essential \
  git \
  nodejs \
  npm \
  && apt-get clean && rm -f /var/lib/apt/lists/*_*

# Set build environment
ENV MIX_ENV=prod

# Install hex and rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# Create build directory
WORKDIR /app

# Copy mix files
COPY mix.exs mix.lock ./

# Install mix dependencies
RUN mix deps.get --only=prod && \
    mix deps.compile

# Copy application source
COPY config config
COPY lib lib
COPY priv priv

# Copy assets and compile them
COPY assets assets
RUN cd assets && npm install && npm run deploy
RUN mix phx.digest

# Compile the application
RUN mix compile

# Create release
RUN mix release

# Runtime stage
FROM debian:${DEBIAN_VERSION} AS runner

# Install runtime dependencies
RUN apt-get update -y && apt-get install -y \
  libstdc++6 \
  openssl \
  libncurses5 \
  locales \
  ca-certificates \
  && apt-get clean && rm -f /var/lib/apt/lists/*_*

# Set locale
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# Create app user
RUN groupadd -r app && useradd -r -g app app

# Create app directory
WORKDIR /app

# Copy release from builder
COPY --from=builder --chown=app:app /app/_build/prod/rel/fn_notifications ./

# Switch to app user
USER app

# Expose port
EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/app/bin/fn_notifications", "rpc", "FnNotifications.Health.check()"]

# Set runtime environment
ENV MIX_ENV=prod
ENV PORT=4000
ENV PHX_SERVER=true

# Start the application
CMD ["/app/bin/fn_notifications", "start"]
```

### Build and Push Images

```bash
# Build production image
docker build -t fn-notifications:latest .
docker build -t fn-notifications:v1.0.0 .

# Tag for registry
docker tag fn-notifications:latest gcr.io/your-project/fn-notifications:latest
docker tag fn-notifications:v1.0.0 gcr.io/your-project/fn-notifications:v1.0.0

# Push to Google Container Registry
docker push gcr.io/your-project/fn-notifications:latest
docker push gcr.io/your-project/fn-notifications:v1.0.0

# Or push to Docker Hub
docker tag fn-notifications:latest your-dockerhub/fn-notifications:latest
docker push your-dockerhub/fn-notifications:latest
```

## Kubernetes Deployment

### Namespace and Configuration

**1. Create Namespace**:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fn-notifications
  labels:
    app: fn-notifications
    environment: production
```

**2. ConfigMap Configuration**:
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fn-notifications-config
  namespace: fn-notifications
data:
  PORT: "4000"
  MIX_ENV: "prod"
  PHX_SERVER: "true"
  PHX_HOST: "notifications.findlynow.com"

  # Database configuration
  POOL_SIZE: "10"
  QUEUE_TARGET: "50"
  QUEUE_INTERVAL: "5000"

  # Kafka configuration
  KAFKA_SSL: "true"
  KAFKA_SASL_MECHANISM: "PLAIN"
  KAFKA_GROUP_ID: "fn-notifications-production"

  # Email configuration
  EMAIL_ADAPTER: "Swoosh.Adapters.Sendgrid"
  FROM_EMAIL: "notifications@findlynow.com"
  FROM_NAME: "Findly Now"

  # Circuit breaker configuration
  CIRCUIT_BREAKER_FAILURE_THRESHOLD: "5"
  CIRCUIT_BREAKER_TIMEOUT_MS: "60000"
  CIRCUIT_BREAKER_RESET_TIMEOUT_MS: "300000"

  # Resource pool configuration
  EMAIL_POOL_MAX_CONCURRENCY: "10"
  EMAIL_POOL_TIMEOUT_MS: "30000"
  SMS_POOL_MAX_CONCURRENCY: "5"
  SMS_POOL_TIMEOUT_MS: "15000"
  WHATSAPP_POOL_MAX_CONCURRENCY: "5"
  WHATSAPP_POOL_TIMEOUT_MS: "15000"

  # Oban configuration
  OBAN_QUEUES: "default:10,notifications:20,retries:5"
  OBAN_PRUNING_MAX_AGE: "3600"
  OBAN_RESCUE_ATTEMPTS: "3"

  # Observability
  TELEMETRY_ENABLED: "true"
  PROMETHEUS_METRICS_ENABLED: "true"
  LOG_LEVEL: "info"
  LOGGER_JSON: "true"

  # Security
  CORS_ORIGINS: "https://app.findlynow.com,https://admin.findlynow.com"
  CSRF_PROTECTION: "true"
  SECURE_COOKIES: "true"
```

**3. Secrets Configuration**:
```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: fn-notifications-secrets
  namespace: fn-notifications
type: Opaque
data:
  SECRET_KEY_BASE: <base64-encoded-secret-key-base>
  LIVE_VIEW_SIGNING_SALT: <base64-encoded-signing-salt>
  DATABASE_URL: <base64-encoded-database-url>

  # Kafka credentials
  KAFKA_HOSTS: <base64-encoded-kafka-hosts>
  KAFKA_USERNAME: <base64-encoded-kafka-username>
  KAFKA_PASSWORD: <base64-encoded-kafka-password>

  # Email service credentials
  SENDGRID_API_KEY: <base64-encoded-sendgrid-key>

  # Twilio credentials
  TWILIO_ACCOUNT_SID: <base64-encoded-twilio-sid>
  TWILIO_AUTH_TOKEN: <base64-encoded-twilio-token>
  TWILIO_PHONE_NUMBER: <base64-encoded-phone-number>
  TWILIO_WHATSAPP_NUMBER: <base64-encoded-whatsapp-number>

  # Optional Redis credentials
  REDIS_URL: <base64-encoded-redis-url>
```

### Deployment Configuration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fn-notifications
  namespace: fn-notifications
  labels:
    app: fn-notifications
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fn-notifications
  template:
    metadata:
      labels:
        app: fn-notifications
        version: v1
    spec:
      containers:
      - name: fn-notifications
        image: gcr.io/your-project/fn-notifications:v1.0.0
        ports:
        - containerPort: 4000
          name: http
        envFrom:
        - configMapRef:
            name: fn-notifications-config
        - secretRef:
            name: fn-notifications-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 4000
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /api/health
            port: 4000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}
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
  name: fn-notifications-service
  namespace: fn-notifications
  labels:
    app: fn-notifications
spec:
  selector:
    app: fn-notifications
  ports:
  - name: http
    port: 80
    targetPort: 4000
  type: ClusterIP
```

**2. Ingress Configuration**:
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fn-notifications-ingress
  namespace: fn-notifications
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "200"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.findlynow.com,https://admin.findlynow.com"
    nginx.ingress.kubernetes.io/websocket-services: "fn-notifications-service"
spec:
  tls:
  - hosts:
    - notifications.findlynow.com
    secretName: notifications-findlynow-com-tls
  rules:
  - host: notifications.findlynow.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fn-notifications-service
            port:
              number: 80
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fn-notifications-hpa
  namespace: fn-notifications
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fn-notifications
  minReplicas: 3
  maxReplicas: 15
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
        value: 20
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

## Deployment Process

### CI/CD Pipeline

**1. GitHub Actions Workflow**:
```yaml
# .github/workflows/deploy.yml
name: Deploy Notifications Service

on:
  push:
    branches: [main]
    paths: ['fn-notifications/**']

env:
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GKE_CLUSTER: fn-production
  GKE_ZONE: us-central1-a
  IMAGE_NAME: fn-notifications

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: fn_notifications_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3

    - name: Setup Elixir
      uses: erlef/setup-beam@v1
      with:
        elixir-version: '1.15.7'
        otp-version: '26.1.2'

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          deps
          _build
        key: ${{ runner.os }}-mix-${{ hashFiles('**/mix.lock') }}
        restore-keys: ${{ runner.os }}-mix-

    - name: Install dependencies
      run: |
        cd fn-notifications
        mix deps.get

    - name: Run tests
      run: |
        cd fn-notifications
        mix test
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fn_notifications_test

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

    - name: Get GKE credentials
      run: gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE

    - name: Build Docker image
      run: |
        cd fn-notifications
        docker build -t gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:$GITHUB_SHA .
        docker tag gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:$GITHUB_SHA gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:latest

    - name: Push Docker image
      run: |
        docker push gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:$GITHUB_SHA
        docker push gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:latest

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/fn-notifications fn-notifications=gcr.io/$GCP_PROJECT_ID/$IMAGE_NAME:$GITHUB_SHA -n fn-notifications
        kubectl rollout status deployment/fn-notifications -n fn-notifications --timeout=300s

    - name: Verify deployment
      run: |
        kubectl get pods -n fn-notifications
        kubectl get services -n fn-notifications

        # Wait for rollout and test health endpoint
        sleep 30
        kubectl port-forward svc/fn-notifications-service 4000:80 -n fn-notifications &
        sleep 10
        curl -f http://localhost:4000/api/health || exit 1
```

**2. Manual Deployment Commands**:
```bash
# Apply all Kubernetes manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# Verify deployment
kubectl get all -n fn-notifications

# Check logs
kubectl logs -f deployment/fn-notifications -n fn-notifications

# Port forward for local testing
kubectl port-forward svc/fn-notifications-service 4000:80 -n fn-notifications
```

### Rolling Updates and Rollbacks

```bash
# Update to new version
kubectl set image deployment/fn-notifications fn-notifications=gcr.io/your-project/fn-notifications:v1.1.0 -n fn-notifications

# Monitor rollout
kubectl rollout status deployment/fn-notifications -n fn-notifications

# Rollback if needed
kubectl rollout undo deployment/fn-notifications -n fn-notifications

# Check rollout history
kubectl rollout history deployment/fn-notifications -n fn-notifications

# Check specific revision
kubectl rollout history deployment/fn-notifications --revision=2 -n fn-notifications
```

## Monitoring and Observability

### Health Checks and Metrics

**Application Health Implementation**:
```elixir
# lib/fn_notifications_web/controllers/health_controller.ex
defmodule FnNotificationsWeb.HealthController do
  use FnNotificationsWeb, :controller

  def health(conn, _params) do
    checks = %{
      database: check_database(),
      kafka: check_kafka(),
      email_service: check_email_service(),
      sms_service: check_sms_service(),
      cache: check_cache()
    }

    overall_status = if Enum.all?(checks, fn {_, status} -> status.healthy end) do
      "healthy"
    else
      "unhealthy"
    end

    status_code = if overall_status == "healthy", do: 200, else: 503

    conn
    |> put_status(status_code)
    |> json(%{
      status: overall_status,
      timestamp: DateTime.utc_now(),
      version: Application.spec(:fn_notifications, :vsn),
      checks: checks,
      circuit_breakers: circuit_breaker_status()
    })
  end

  defp check_database do
    case Ecto.Adapters.SQL.query(FnNotifications.Repo, "SELECT 1", []) do
      {:ok, _} -> %{healthy: true, response_time_ms: 0}
      {:error, reason} -> %{healthy: false, error: inspect(reason)}
    end
  end

  defp check_kafka do
    # Implementation depends on your Kafka client
    %{healthy: true, response_time_ms: 0}
  end

  defp circuit_breaker_status do
    %{
      email_circuit: get_circuit_state(:email),
      sms_circuit: get_circuit_state(:sms),
      whatsapp_circuit: get_circuit_state(:whatsapp)
    }
  end
end
```

### Prometheus Metrics

**ServiceMonitor Configuration**:
```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fn-notifications-metrics
  namespace: fn-notifications
  labels:
    app: fn-notifications
spec:
  selector:
    matchLabels:
      app: fn-notifications
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
    honorLabels: true
```

**Custom Prometheus Metrics**:
```elixir
# lib/fn_notifications/telemetry.ex
defmodule FnNotifications.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {:telemetry_poller, measurements: periodic_measurements(), period: 10_000},
      {TelemetryMetricsPrometheus, metrics: metrics()}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end

  def metrics do
    [
      # Notification metrics
      counter("fn_notifications.notification.sent.total",
        tags: [:channel, :notification_type]
      ),
      counter("fn_notifications.notification.delivered.total",
        tags: [:channel, :notification_type]
      ),
      counter("fn_notifications.notification.failed.total",
        tags: [:channel, :notification_type, :failure_reason]
      ),
      summary("fn_notifications.notification.delivery_time",
        unit: {:native, :millisecond},
        tags: [:channel]
      ),

      # Circuit breaker metrics
      last_value("fn_notifications.circuit_breaker.state",
        tags: [:service],
        description: "Circuit breaker state (0=closed, 1=open, 2=half_open)"
      ),
      counter("fn_notifications.circuit_breaker.failures.total",
        tags: [:service]
      ),

      # Resource pool metrics
      last_value("fn_notifications.pool.active_connections",
        tags: [:pool]
      ),
      last_value("fn_notifications.pool.utilization",
        tags: [:pool]
      ),

      # Phoenix metrics
      summary("phoenix.endpoint.stop.duration",
        unit: {:native, :millisecond},
        tags: [:route]
      ),
      counter("phoenix.endpoint.stop.total",
        tags: [:method, :route, :status]
      ),

      # Broadway metrics
      counter("broadway.processor.message.stop.total",
        tags: [:name, :status]
      ),
      summary("broadway.processor.message.stop.duration",
        unit: {:native, :millisecond},
        tags: [:name]
      )
    ]
  end

  defp periodic_measurements do
    [
      # VM metrics
      {FnNotifications.Telemetry, :vm_measurements, []},
      # Custom application metrics
      {FnNotifications.Telemetry, :notification_measurements, []},
      {FnNotifications.Telemetry, :circuit_breaker_measurements, []}
    ]
  end
end
```

### Alert Rules

**Prometheus Alert Rules**:
```yaml
# alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: fn-notifications-alerts
  namespace: fn-notifications
spec:
  groups:
  - name: fn-notifications.rules
    rules:
    - alert: NotificationsServiceDown
      expr: up{job="fn-notifications"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Notifications service is down"
        description: "Notifications service has been down for more than 1 minute"

    - alert: HighNotificationFailureRate
      expr: |
        (
          rate(fn_notifications_notification_failed_total[5m]) /
          rate(fn_notifications_notification_sent_total[5m])
        ) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High notification failure rate"
        description: "Notification failure rate is {{ $value | humanizePercentage }} over 5 minutes"

    - alert: CircuitBreakerOpen
      expr: fn_notifications_circuit_breaker_state{state="open"} == 1
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Circuit breaker is open for {{ $labels.service }}"
        description: "Circuit breaker for {{ $labels.service }} has been open for more than 1 minute"

    - alert: HighDeliveryLatency
      expr: |
        histogram_quantile(0.95,
          rate(fn_notifications_notification_delivery_time_bucket[5m])
        ) > 10000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High notification delivery latency"
        description: "95th percentile delivery time is {{ $value }}ms"

    - alert: BroadwayConsumerLag
      expr: |
        rate(broadway_processor_message_stop_total{status="error"}[5m]) > 10
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High Broadway consumer error rate"
        description: "Broadway is experiencing {{ $value }} errors per second"

    - alert: DatabaseConnectionPoolExhaustion
      expr: |
        fn_notifications_pool_utilization{pool="database"} > 0.9
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Database connection pool nearly exhausted"
        description: "Database pool utilization is {{ $value | humanizePercentage }}"
```

## Security and Compliance

### Network Security

**Network Policy**:
```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fn-notifications-network-policy
  namespace: fn-notifications
spec:
  podSelector:
    matchLabels:
      app: fn-notifications
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 4000
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
  # Allow HTTPS outbound for external services
  - to: []
    ports:
    - protocol: TCP
      port: 443
  # Allow database connections
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow Kafka connections
  - to: []
    ports:
    - protocol: TCP
      port: 9092
```

### Pod Security Standards

**Pod Security Policy**:
```yaml
# pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: fn-notifications-psp
  namespace: fn-notifications
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'configMap'
  - 'persistentVolumeClaim'
```

## Backup and Disaster Recovery

### Database Backup Strategy

```bash
# Automated daily backups
#!/bin/bash
# backup-notifications-db.sh

DB_NAME="fn_notifications_production"
BACKUP_DIR="/backups/notifications"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/notifications_backup_${DATE}.sql"

# Create backup directory
mkdir -p $BACKUP_DIR

# Create compressed backup
pg_dump "$DATABASE_URL" | gzip > "${BACKUP_FILE}.gz"

# Verify backup
if [ $? -eq 0 ]; then
    echo "Backup successful: ${BACKUP_FILE}.gz"

    # Upload to cloud storage
    gsutil cp "${BACKUP_FILE}.gz" gs://fn-backups/notifications/

    # Clean up local backups older than 7 days
    find $BACKUP_DIR -name "*.gz" -mtime +7 -delete
else
    echo "Backup failed!"
    exit 1
fi
```

### Disaster Recovery Plan

**Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)**:
- **RTO**: 30 minutes for service restoration
- **RPO**: 1 hour maximum data loss
- **Multi-region deployment** for high availability
- **Database standby** in different region/zone
- **Kafka topic replication** across availability zones

**Recovery Procedures**:

1. **Database Recovery**:
```bash
# Restore from backup
gunzip -c notifications_backup_20240115_090000.sql.gz | psql "$DATABASE_URL"

# Verify data integrity
psql "$DATABASE_URL" -c "SELECT COUNT(*) FROM notifications;"
psql "$DATABASE_URL" -c "SELECT COUNT(*) FROM user_preferences;"
```

2. **Service Recovery**:
```bash
# Scale up in disaster recovery region
kubectl scale deployment fn-notifications --replicas=5 -n fn-notifications-dr

# Switch traffic to DR region
kubectl patch ingress fn-notifications-ingress -n fn-notifications-dr \
  --type='json' -p='[{"op": "replace", "path": "/spec/rules/0/host", "value": "notifications.findlynow.com"}]'

# Update DNS to point to DR region
# (This would be done through your DNS provider)
```

3. **Data Validation**:
```bash
# Verify service functionality
curl -f https://notifications.findlynow.com/api/health

# Test notification sending
curl -X POST https://notifications.findlynow.com/api/v1/test-delivery \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test-user", "channels": ["email"]}'
```

---

*For domain architecture and business rules, see [domain-architecture.md](domain-architecture.md). For API usage, see [api-documentation.md](api-documentation.md). For system-wide deployment patterns, see [../fn-infra/](../fn-infra/).*