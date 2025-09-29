# fn-matcher Deployment Guide

**Complete deployment guide for the Matcher domain service across development, staging, and production environments.**

## Prerequisites

### Cloud Infrastructure
- **PostgreSQL + PostGIS** database for spatial matching operations
- **Redis** cluster for caching and rate limiting
- **Confluent Cloud** Kafka cluster for event processing
- **Kubernetes cluster** (GKE, EKS, or AKS) for container orchestration

### Required Tools
- **Docker** 20.10+ for containerization
- **kubectl** for Kubernetes management
- **Helm** 3.0+ for Kubernetes deployments
- **Rust** 1.80+ for local development
- **SQLx CLI** for database migrations
- **Make** for build automation

### Access & Credentials
- PostgreSQL connection with CREATE, INSERT, UPDATE, DELETE permissions
- Redis connection string with write permissions
- Confluent Cloud Kafka bootstrap servers and API keys
- Container registry access (Google Container Registry or Docker Hub)

## Environment Configuration

### Environment Variables

**Core Service Configuration**:
```bash
# Service Configuration
SERVER_PORT=8003
RUST_LOG=fn_matcher=info,tower_http=info,sqlx=warn
ENVIRONMENT=production              # development | staging | production
SERVICE_NAME=fn-matcher

# Database (PostgreSQL with PostGIS)
DATABASE_URL=postgresql://username:password@host:5432/fn_matcher?sslmode=require
DATABASE_MAX_CONNECTIONS=20
DATABASE_MIN_CONNECTIONS=5
DATABASE_MAX_LIFETIME=1800s
DATABASE_CONNECT_TIMEOUT=30s
DATABASE_IDLE_TIMEOUT=600s

# Redis (Caching and Rate Limiting)
REDIS_URL=redis://username:password@host:6379/0
REDIS_MAX_CONNECTIONS=10
REDIS_CONNECTION_TIMEOUT=5s
REDIS_POOL_TIMEOUT=10s

# Kafka Event Processing (Standardized Configuration)
KAFKA_BOOTSTRAP_SERVERS=your-cluster.confluent.cloud:9092
KAFKA_SASL_USERNAME=your-api-key
KAFKA_SASL_PASSWORD=your-api-secret
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN
KAFKA_CONSUMER_GROUP_ID=fn-matcher-production

# Kafka Topics (Standardized Names)
KAFKA_TOPICS_POSTS=posts.events                    # Input: Posts events
KAFKA_TOPICS_ENRICHMENT=media-ai.enrichment        # Input: Media AI enriched posts
KAFKA_TOPICS_MATCHING=posts.matching               # Output: Matching events

# Configurable Matching Algorithm Weights
MATCHING_LOCATION_WEIGHT=0.3        # Location proximity weight (30%)
MATCHING_VISUAL_WEIGHT=0.4          # Visual similarity weight (40%)
MATCHING_TEXT_WEIGHT=0.2            # Text similarity weight (20%)
MATCHING_TEMPORAL_WEIGHT=0.1        # Temporal proximity weight (10%)

# Confidence Thresholds
CONFIDENCE_AUTO_ALERT=0.90          # Auto-alert both parties (90%+)
CONFIDENCE_SUGGEST_MATCH=0.70       # Suggest match to users (70%+)
CONFIDENCE_STORE_POTENTIAL=0.50     # Store for manual review (50%+)
CONFIDENCE_DISCARD_BELOW=0.50       # Discard matches below this

# Performance Tuning
MAX_CONCURRENT_MATCHES=100          # Maximum concurrent matching operations
MATCH_SEARCH_RADIUS_KM=50           # Maximum search radius in kilometers
MATCH_TEMPORAL_WINDOW_HOURS=168     # Match against items reported within 7 days
BATCH_SIZE_MATCH_PROCESSING=20      # Event batch size for processing

# Rate Limiting
RATE_LIMIT_MATCHING_PER_MINUTE=1000  # Max matching operations per minute
RATE_LIMIT_API_PER_MINUTE=100        # Max API calls per minute per client
```

**Development Environment**:
```bash
# Development overrides
SERVER_PORT=8003
RUST_LOG=fn_matcher=debug,tower_http=debug,sqlx=debug
DATABASE_URL=postgresql://postgres:postgres@localhost:5434/fn_matcher
REDIS_URL=redis://localhost:6380

# Local Kafka (no authentication)
KAFKA_BOOTSTRAP_SERVERS=localhost:9094
KAFKA_SECURITY_PROTOCOL=PLAINTEXT
KAFKA_CONSUMER_GROUP_ID=fn-matcher-dev

# Local topic names (same as production)
KAFKA_TOPICS_POSTS=posts.events
KAFKA_TOPICS_ENRICHMENT=media-ai.enrichment
KAFKA_TOPICS_MATCHING=posts.matching
```

**Staging Environment**:
```bash
# Staging configuration
ENVIRONMENT=staging
RUST_LOG=fn_matcher=info,tower_http=info
# Use staging Kafka cluster and database
KAFKA_GROUP_ID=fn-matcher-staging
DATABASE_URL=postgresql://staging-user:password@staging-db:5432/fn_matcher_staging
```

## Database Setup

### PostgreSQL with PostGIS Installation

**Create Database and Extensions**:
```sql
-- Create database
CREATE DATABASE fn_matcher;

-- Connect to database
\c fn_matcher;

-- Enable PostGIS extension for spatial operations
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- Verify PostGIS installation
SELECT PostGIS_Full_Version();
```

**Connection Pooling Configuration**:
```bash
# For production workloads
DATABASE_MAX_CONNECTIONS=50
DATABASE_MIN_CONNECTIONS=10

# Connection health checks
DATABASE_MAX_LIFETIME=3600s    # 1 hour max connection lifetime
DATABASE_IDLE_TIMEOUT=600s     # 10 minute idle timeout
```

### SQLx Migrations

**Run Database Migrations**:
```bash
# Install SQLx CLI
cargo install sqlx-cli --no-default-features --features postgres

# Set database URL
export DATABASE_URL=postgresql://username:password@host:5432/fn_matcher

# Run migrations
sqlx migrate run

# Verify migration status
sqlx migrate info
```

**Production Migration Process**:
```bash
# 1. Backup database before migration
pg_dump -h production-host -U username fn_matcher > backup_before_migration.sql

# 2. Test migration on staging first
sqlx migrate run --database-url $STAGING_DATABASE_URL

# 3. Run migration on production during maintenance window
sqlx migrate run --database-url $PRODUCTION_DATABASE_URL

# 4. Verify application functionality
curl http://production-host:8003/health
```

## Docker Configuration

### Dockerfile Optimization

The service uses a multi-stage Dockerfile for optimized production builds:

```dockerfile
# Build stage
FROM rust:1.80-slim as builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN cargo fetch
COPY src ./src
RUN cargo build --release

# Runtime stage
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/fn-matcher /usr/local/bin/
EXPOSE 8003
CMD ["fn-matcher"]
```

### Docker Compose Development

**Local Development Stack**:
```yaml
version: '3.8'
services:
  fn-matcher:
    build: .
    ports:
      - "8003:8003"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/fn_matcher
      - REDIS_URL=redis://redis:6379
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - postgres
      - redis
      - kafka

  postgres:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: fn_matcher
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Container Build Commands

```bash
# Build development image
docker build -t fn-matcher:dev .

# Build production image with optimization
docker build -t fn-matcher:latest --target production .

# Build with specific Rust version
docker build -t fn-matcher:rust-1.80 --build-arg RUST_VERSION=1.80 .

# Multi-platform build for deployment
docker buildx build --platform linux/amd64,linux/arm64 -t fn-matcher:latest .
```

## Kubernetes Deployment

### Kubernetes Manifests

**Deployment Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fn-matcher
  namespace: findly-now
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fn-matcher
  template:
    metadata:
      labels:
        app: fn-matcher
    spec:
      containers:
      - name: fn-matcher
        image: gcr.io/findly-now/fn-matcher:latest
        ports:
        - containerPort: 8003
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: fn-matcher-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: fn-matcher-secrets
              key: redis-url
        - name: KAFKA_BROKERS
          value: "kafka-cluster.confluent.cloud:9092"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8003
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8003
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Service Configuration**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fn-matcher-service
  namespace: findly-now
spec:
  selector:
    app: fn-matcher
  ports:
  - port: 8003
    targetPort: 8003
  type: ClusterIP
```

**HorizontalPodAutoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fn-matcher-hpa
  namespace: findly-now
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fn-matcher
  minReplicas: 2
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

### Helm Chart Deployment

**Install via Helm**:
```bash
# Add Findly Now Helm repository
helm repo add findly-now https://charts.findlynow.com
helm repo update

# Install fn-matcher
helm install fn-matcher findly-now/fn-matcher \
  --namespace findly-now \
  --create-namespace \
  --set image.tag=latest \
  --set database.url=$DATABASE_URL \
  --set redis.url=$REDIS_URL \
  --set kafka.brokers=$KAFKA_BROKERS

# Upgrade deployment
helm upgrade fn-matcher findly-now/fn-matcher \
  --namespace findly-now \
  --set image.tag=v1.2.0

# Check deployment status
helm status fn-matcher -n findly-now
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy fn-matcher

on:
  push:
    branches: [main]
    paths: ['fn-matcher/**']

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: fn_matcher_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v4

    - name: Setup Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: 1.80.0
        components: rustfmt, clippy

    - name: Run tests
      working-directory: fn-matcher
      run: |
        cargo fmt --check
        cargo clippy -- -D warnings
        cargo test

    - name: Run E2E tests
      working-directory: fn-matcher
      run: make e2e-test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Setup Google Cloud
      uses: google-github-actions/setup-gcloud@v1
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ secrets.GCP_PROJECT_ID }}

    - name: Configure Docker for GCR
      run: gcloud auth configure-docker

    - name: Build and push Docker image
      working-directory: fn-matcher
      run: |
        docker build -t gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-matcher:${{ github.sha }} .
        docker push gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-matcher:${{ github.sha }}

    - name: Deploy to Kubernetes
      run: |
        gcloud container clusters get-credentials production-cluster --region us-central1
        kubectl set image deployment/fn-matcher fn-matcher=gcr.io/${{ secrets.GCP_PROJECT_ID }}/fn-matcher:${{ github.sha }} -n findly-now
        kubectl rollout status deployment/fn-matcher -n findly-now
```

## Monitoring and Observability

### Health Checks

**Health Check Endpoints**:
```bash
# Basic health check
curl http://localhost:8003/health

# Detailed health with dependencies
curl http://localhost:8003/health/detailed

# Readiness check for Kubernetes
curl http://localhost:8003/health/ready

# Liveness check for Kubernetes
curl http://localhost:8003/health/live
```

**Health Check Implementation**:
```rust
#[derive(Serialize)]
struct HealthStatus {
    service: String,
    version: String,
    status: String,
    database: ComponentHealth,
    redis: ComponentHealth,
    kafka: ComponentHealth,
    matching_algorithm: ComponentHealth,
}

pub async fn health_check_detailed(State(app_state): State<AppState>) -> impl IntoResponse {
    let database_health = check_database_connection(&app_state.db_pool).await;
    let redis_health = check_redis_connection(&app_state.redis_client).await;
    let kafka_health = check_kafka_connection(&app_state.kafka_client).await;
    let algorithm_health = check_matching_algorithm_health().await;

    let overall_status = if [&database_health, &redis_health, &kafka_health, &algorithm_health]
        .iter()
        .all(|health| health.status == "healthy")
    {
        "healthy"
    } else {
        "unhealthy"
    };

    let health_status = HealthStatus {
        service: "fn-matcher".to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        status: overall_status.to_string(),
        database: database_health,
        redis: redis_health,
        kafka: kafka_health,
        matching_algorithm: algorithm_health,
    };

    let status_code = if overall_status == "healthy" {
        StatusCode::OK
    } else {
        StatusCode::SERVICE_UNAVAILABLE
    };

    (status_code, Json(health_status))
}
```

### Metrics and Logging

**Structured Logging Configuration**:
```rust
use tracing::{info, warn, error, instrument};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "fn_matcher=info".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();
}

#[instrument(skip(self), fields(lost_post_id = %lost_post.id, found_post_id = %found_post.id))]
pub async fn calculate_match_confidence(
    &self,
    lost_post: &PostData,
    found_post: &PostData,
) -> Result<ConfidenceScore, MatchingError> {
    info!("Starting match confidence calculation");

    let confidence = self.matching_engine.calculate_score(lost_post, found_post).await?;

    info!(
        confidence_score = %confidence.value(),
        "Match confidence calculated successfully"
    );

    Ok(confidence)
}
```

**Prometheus Metrics**:
```rust
use prometheus::{Counter, Histogram, Registry};

lazy_static! {
    static ref MATCHES_PROCESSED: Counter = Counter::new(
        "matches_processed_total",
        "Total number of matches processed"
    ).unwrap();

    static ref MATCH_CONFIDENCE_HISTOGRAM: Histogram = Histogram::with_opts(
        prometheus::HistogramOpts::new(
            "match_confidence_scores",
            "Distribution of match confidence scores"
        )
    ).unwrap();

    static ref MATCHING_DURATION: Histogram = Histogram::with_opts(
        prometheus::HistogramOpts::new(
            "matching_duration_seconds",
            "Time spent calculating matches"
        )
    ).unwrap();
}

// Usage in matching service
impl MatchingService {
    pub async fn process_match(&self, post_data: PostData) -> Result<(), MatchingError> {
        let timer = MATCHING_DURATION.start_timer();

        let matches = self.find_matches(post_data).await?;

        for match_result in matches {
            MATCHES_PROCESSED.inc();
            MATCH_CONFIDENCE_HISTOGRAM.observe(match_result.confidence.value());
        }

        timer.observe_duration();
        Ok(())
    }
}
```

## Performance Optimization

### Database Performance

**Index Optimization**:
```sql
-- Spatial index for location-based matching (most critical)
CREATE INDEX CONCURRENTLY idx_posts_location_gist
ON posts USING GIST (location);

-- Compound index for item type and status
CREATE INDEX CONCURRENTLY idx_posts_type_status_created
ON posts (item_type, status, created_at DESC);

-- Temporal index for recent posts
CREATE INDEX CONCURRENTLY idx_posts_created_at_btree
ON posts (created_at DESC)
WHERE status = 'active';

-- Analyze for query planning
ANALYZE posts;
```

**Connection Pool Tuning**:
```bash
# Production database pool configuration
DATABASE_MAX_CONNECTIONS=50        # Adjust based on CPU cores and workload
DATABASE_MIN_CONNECTIONS=10        # Keep minimum connections warm
DATABASE_MAX_LIFETIME=3600s        # Rotate connections every hour
DATABASE_IDLE_TIMEOUT=600s         # Close idle connections after 10 minutes
DATABASE_CONNECT_TIMEOUT=30s       # Connection establishment timeout
```

### Redis Caching Strategy

**Cache Configuration**:
```rust
#[derive(Clone)]
pub struct CacheConfig {
    pub match_candidates_ttl: Duration,
    pub user_preferences_ttl: Duration,
    pub location_clusters_ttl: Duration,
}

impl Default for CacheConfig {
    fn default() -> Self {
        Self {
            match_candidates_ttl: Duration::from_secs(300),    // 5 minutes
            user_preferences_ttl: Duration::from_secs(1800),   // 30 minutes
            location_clusters_ttl: Duration::from_secs(3600),  // 1 hour
        }
    }
}
```

### Kafka Consumer Optimization

**Consumer Configuration**:
```bash
# Kafka consumer performance tuning
KAFKA_CONSUMER_GROUP_ID=fn-matcher-production
KAFKA_CONSUMER_SESSION_TIMEOUT=30000         # 30 seconds
KAFKA_CONSUMER_HEARTBEAT_INTERVAL=10000      # 10 seconds
KAFKA_CONSUMER_MAX_POLL_RECORDS=100          # Batch size
KAFKA_CONSUMER_FETCH_MIN_BYTES=1024          # Minimum fetch size
KAFKA_CONSUMER_FETCH_MAX_WAIT=500            # Maximum wait time
KAFKA_CONSUMER_AUTO_OFFSET_RESET=earliest
```

## Security Configuration

### Environment Security

**Secret Management**:
```bash
# Use Kubernetes secrets for sensitive data
kubectl create secret generic fn-matcher-secrets \
  --from-literal=database-url="postgresql://user:pass@host:5432/db" \
  --from-literal=redis-url="redis://user:pass@host:6379" \
  --from-literal=kafka-username="api-key" \
  --from-literal=kafka-password="api-secret" \
  --namespace findly-now
```

**Network Security**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fn-matcher-network-policy
  namespace: findly-now
spec:
  podSelector:
    matchLabels:
      app: fn-matcher
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8003
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

## Troubleshooting

### Common Issues

**Database Connection Issues**:
```bash
# Check database connectivity
psql -h host -U username -d fn_matcher -c "SELECT 1;"

# Verify PostGIS extension
psql -h host -U username -d fn_matcher -c "SELECT PostGIS_Full_Version();"

# Check connection pool status
curl http://localhost:8003/health/detailed | jq '.database'
```

**Kafka Consumer Lag**:
```bash
# Check consumer lag
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-matcher-production \
  --describe

# Reset consumer group if needed (posts.events topic)
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-matcher-production \
  --reset-offsets --to-latest \
  --topic posts.events \
  --execute

# Check topic configuration
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --describe --topic posts.events
```

**Performance Debugging**:
```bash
# Check matching performance metrics
curl http://localhost:8003/metrics | grep matching_duration

# View detailed logs
kubectl logs -f deployment/fn-matcher -n findly-now

# Database query performance
psql -h host -U username -d fn_matcher -c "
  SELECT query, calls, total_time, mean_time
  FROM pg_stat_statements
  WHERE query LIKE '%ST_DWithin%'
  ORDER BY total_time DESC;"
```

### Emergency Procedures

**Service Recovery**:
```bash
# Restart deployment
kubectl rollout restart deployment/fn-matcher -n findly-now

# Scale down temporarily
kubectl scale deployment fn-matcher --replicas=0 -n findly-now

# Scale back up
kubectl scale deployment fn-matcher --replicas=3 -n findly-now

# Rollback to previous version
kubectl rollout undo deployment/fn-matcher -n findly-now
```

**Database Emergency**:
```bash
# Create emergency database backup
pg_dump -h host -U username fn_matcher > emergency_backup_$(date +%Y%m%d_%H%M%S).sql

# Restart database connections
kubectl delete pods -l app=postgres -n findly-now

# Check database locks
psql -h host -U username -d fn_matcher -c "
  SELECT pid, usename, query, state
  FROM pg_stat_activity
  WHERE state != 'idle';"
```

## Performance Benchmarks

### Expected Performance Metrics

- **Match Calculation**: <100ms for single post matching
- **Batch Processing**: 1000+ posts per minute
- **Database Queries**: <50ms for spatial queries within 10km radius
- **Memory Usage**: <512MB per replica under normal load
- **CPU Usage**: <50% per replica under normal load

### Load Testing

```bash
# Load test with wrk
wrk -t12 -c400 -d30s --script=load_test.lua http://localhost:8003/api/matches

# Kafka load testing
kafka-producer-perf-test --topic posts.lifecycle \
  --num-records 10000 \
  --record-size 1024 \
  --throughput 1000 \
  --producer-props bootstrap.servers=$KAFKA_BROKERS
```

---

*This deployment guide provides comprehensive instructions for deploying the fn-matcher service across all environments. For architectural details, see [domain-architecture.md](domain-architecture.md). For API specifications, see [api-documentation.md](api-documentation.md).*