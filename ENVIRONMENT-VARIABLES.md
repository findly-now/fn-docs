# Findly Now - Environment Variables Reference

**Document Ownership**: This document OWNS all environment variable configuration across the entire Findly Now ecosystem.

## Overview

This comprehensive guide documents all environment variables required for proper configuration of the Findly Now microservices architecture. All services use environment variables for configuration management, ensuring secure and flexible deployment across different environments.

## Common Patterns

### Variable Naming Conventions
- **Service Prefixes**: Each service uses consistent prefixes for related variables
- **Uppercase**: All environment variables use UPPERCASE_WITH_UNDERSCORES
- **Hierarchical Structure**: Related variables grouped with common prefixes
- **Boolean Values**: Use "true"/"false" strings for boolean configurations

### Environment Validation
- All services validate required environment variables on startup
- Missing required variables cause immediate startup failure with clear error messages
- Optional variables have documented default values

## Kafka Configuration

### Standard Kafka Variables (All Services)
All services that interact with Kafka require these base variables:

```bash
# Kafka Connection
KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-confluent-api-key"
KAFKA_SASL_PASSWORD="your-confluent-api-secret"

# Security Configuration
KAFKA_SECURITY_PROTOCOL="SASL_SSL"
KAFKA_SASL_MECHANISM="PLAIN"

# Consumer Group Configuration (when applicable)
KAFKA_CONSUMER_GROUP_ID="service-specific-group-id"
```

### Topic Configuration Variables
Services use these variables to configure topic names:

```bash
# Core Event Topics (Standardized)
KAFKA_POSTS_TOPIC="posts.events"              # All posts domain events
KAFKA_ENRICHMENT_TOPIC="posts.enhancements"   # AI enhanced posts
KAFKA_MATCHING_TOPIC="posts.matching"         # Matching results
KAFKA_NOTIFICATION_TOPIC="notifications.events" # Notification events
```

## Service-Specific Configuration

### fn-posts (Go Service)

#### Required Variables
```bash
# Server Configuration
PORT="8080"
GIN_MODE="release"  # Options: debug, release, test

# Database (Supabase)
DATABASE_URL="postgresql://user:pass@host:port/db?sslmode=require"
DB_MAX_OPEN_CONNS="10"
DB_MAX_IDLE_CONNS="5"
DB_CONN_MAX_LIFETIME="1h"

# Google Cloud Storage
GCS_BUCKET_NAME="your-gcs-bucket"
GCS_CREDENTIALS_JSON="base64-encoded-service-account-json"

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-confluent-api-key"
KAFKA_SASL_PASSWORD="your-confluent-api-secret"
KAFKA_TOPIC="posts.events"  # Output topic for posts events

# Authentication
JWT_SECRET="your-jwt-secret-key"
GOOGLE_CLIENT_ID="your-google-oauth-client-id"

# Geospatial Configuration
DEFAULT_SEARCH_RADIUS_KM="5.0"
MAX_SEARCH_RADIUS_KM="50.0"
```

#### Optional Variables
```bash
# Performance Tuning
REQUEST_TIMEOUT="30s"
UPLOAD_MAX_SIZE="10MB"
MAX_PHOTOS_PER_POST="10"

# Logging
LOG_LEVEL="info"  # Options: debug, info, warn, error
LOG_FORMAT="json"  # Options: json, text
```

### fn-notifications (Elixir Service)

#### Required Variables
```bash
# Phoenix Configuration
PORT="4000"
SECRET_KEY_BASE="your-phoenix-secret-key-base"
PHX_SERVER="true"

# Database
DATABASE_URL="postgresql://user:pass@host:port/db?sslmode=require"
POOL_SIZE="10"

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-confluent-api-key"
KAFKA_SASL_PASSWORD="your-confluent-api-secret"

# Topic Subscriptions
KAFKA_POSTS_TOPIC="posts.events"
KAFKA_MATCHER_TOPIC="posts.matching"
KAFKA_ENRICHMENT_TOPIC="posts.enhancements"

# Notification Providers
SENDGRID_API_KEY="your-sendgrid-api-key"
TWILIO_ACCOUNT_SID="your-twilio-account-sid"
TWILIO_AUTH_TOKEN="your-twilio-auth-token"
WHATSAPP_PHONE_NUMBER="your-whatsapp-business-number"

# Email Configuration
FROM_EMAIL="noreply@findlynow.com"
REPLY_TO_EMAIL="support@findlynow.com"
```

#### Optional Variables
```bash
# Broadway Configuration
BROADWAY_CONCURRENCY="10"
BROADWAY_BATCH_SIZE="100"
BROADWAY_BATCH_TIMEOUT="5000"

# Rate Limiting
EMAIL_RATE_LIMIT="100"  # emails per minute
SMS_RATE_LIMIT="50"     # SMS per minute

# Notification Preferences
DEFAULT_EMAIL_ENABLED="true"
DEFAULT_SMS_ENABLED="false"
DEFAULT_PUSH_ENABLED="true"

# Logging
LOG_LEVEL="info"
```

### fn-media-ai (Python Service)

#### Required Variables
```bash
# FastAPI Configuration
PORT="8000"
HOST="0.0.0.0"
WORKERS="4"

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-confluent-api-key"
KAFKA_SASL_PASSWORD="your-confluent-api-secret"

# Topic Configuration
KAFKA_CONSUMER_TOPICS='["posts.events"]'      # Input: Posts events (JSON array)
KAFKA_POST_ENHANCED_TOPIC="posts.enhancements"  # Output: Enhanced posts

# Google Cloud Storage
GCS_BUCKET_NAME="your-gcs-bucket"
GCS_CREDENTIALS_JSON="base64-encoded-service-account-json"

# AI Model Configuration
VISION_MODEL_PATH="/app/models/vision_model"
OCR_MODEL_PATH="/app/models/ocr_model"
CLASSIFICATION_MODEL_PATH="/app/models/classification_model"

# Processing Configuration
MAX_IMAGE_SIZE="10MB"
SUPPORTED_FORMATS="jpg,jpeg,png,webp"
PROCESSING_TIMEOUT="30"  # seconds
```

#### Optional Variables
```bash
# AI Processing Configuration
VISION_CONFIDENCE_THRESHOLD="0.7"
OCR_CONFIDENCE_THRESHOLD="0.8"
BATCH_SIZE="5"
GPU_ENABLED="false"

# Performance Tuning
MAX_CONCURRENT_REQUESTS="10"
MODEL_CACHE_SIZE="3"  # number of models to keep in memory

# Logging
LOG_LEVEL="INFO"
UVICORN_LOG_LEVEL="info"
```

### fn-matcher (Rust Service)

#### Required Variables
```bash
# Server Configuration
HOST="0.0.0.0"
PORT="3000"

# Database
DATABASE_URL="postgresql://user:pass@host:port/db?sslmode=require"
DATABASE_MAX_CONNECTIONS="10"

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-confluent-api-key"
KAFKA_SASL_PASSWORD="your-confluent-api-secret"

# Topic Configuration
KAFKA_CONSUMER_TOPICS="posts.events,posts.enhancements" # Input topics
KAFKA_PRODUCER_TOPIC_MATCHING="posts.matching"          # Output: Matching events
KAFKA_PRODUCER_TOPIC_NOTIFICATIONS="notifications.events" # Output: Notifications

# Algorithm Configuration (Configurable Weights)
MATCHING_LOCATION_WEIGHT="0.3"   # Weight for location similarity (0.0-1.0)
MATCHING_VISUAL_WEIGHT="0.4"     # Weight for visual similarity (0.0-1.0)
MATCHING_TEXT_WEIGHT="0.2"       # Weight for text similarity (0.0-1.0)
MATCHING_TEMPORAL_WEIGHT="0.1"   # Weight for temporal proximity (0.0-1.0)
```

#### Optional Variables
```bash
# Matching Thresholds
MATCHING_MIN_CONFIDENCE="0.6"    # Minimum confidence for a match
MATCHING_MAX_DISTANCE_KM="10.0"  # Maximum distance for location matching
MATCHING_MAX_TIME_DAYS="30"      # Maximum time difference for temporal matching

# Performance Configuration
MATCHING_BATCH_SIZE="50"
MATCHING_WORKER_THREADS="4"
CACHE_TTL_SECONDS="3600"         # Cache time-to-live

# Logging
RUST_LOG="info"                  # Options: trace, debug, info, warn, error
LOG_FORMAT="json"                # Options: json, pretty
```

## Infrastructure Configuration

### fn-infra (Kubernetes Deployment)

#### Environment-Specific Variables
```bash
# Environment Configuration
ENVIRONMENT="dev"        # Options: dev, staging, prod
CLUSTER_NAME="findly-cluster"
NAMESPACE="findly-now"

# Image Configuration
IMAGE_REGISTRY="gcr.io/your-project"
IMAGE_TAG="latest"

# Resource Limits
POSTS_CPU_LIMIT="500m"
POSTS_MEMORY_LIMIT="512Mi"
NOTIFICATIONS_CPU_LIMIT="300m"
NOTIFICATIONS_MEMORY_LIMIT="256Mi"
MEDIA_AI_CPU_LIMIT="1000m"
MEDIA_AI_MEMORY_LIMIT="1Gi"
MATCHER_CPU_LIMIT="500m"
MATCHER_MEMORY_LIMIT="512Mi"

# Autoscaling Configuration
POSTS_MIN_REPLICAS="2"
POSTS_MAX_REPLICAS="10"
NOTIFICATIONS_MIN_REPLICAS="1"
NOTIFICATIONS_MAX_REPLICAS="5"
```

## Security and Credentials

### Secret Management
All sensitive variables should be managed through secure secret management systems:

- **Kubernetes**: Use Kubernetes Secrets
- **Local Development**: Use `.env` files (never committed to git)
- **CI/CD**: Use encrypted environment variables in GitHub Actions

### Required Secrets
```bash
# Never expose these in logs or configuration files
KAFKA_SASL_PASSWORD="***"
JWT_SECRET="***"
SECRET_KEY_BASE="***"
GCS_CREDENTIALS_JSON="***"
SENDGRID_API_KEY="***"
TWILIO_AUTH_TOKEN="***"
DATABASE_URL="***"
```

## Environment-Specific Configurations

### Development Environment (Local Docker)
```bash
# Development-specific overrides
GIN_MODE="debug"
LOG_LEVEL="debug"
PHX_SERVER="true"
RUST_LOG="debug"
GPU_ENABLED="false"

# Local Kafka configuration (no authentication)
KAFKA_BOOTSTRAP_SERVERS="kafka:29092"
KAFKA_SECURITY_PROTOCOL="PLAINTEXT"
# No SASL credentials needed for local development

# Local database connections
DATABASE_URL="postgresql://service_user:service_password@postgres:5432/service_db"
```

### Staging Environment
```bash
# Staging-specific configuration
GIN_MODE="release"
LOG_LEVEL="info"
MATCHING_MIN_CONFIDENCE="0.5"  # Lower threshold for testing
```

### Production Environment
```bash
# Production-specific configuration
GIN_MODE="release"
LOG_LEVEL="warn"
MATCHING_MIN_CONFIDENCE="0.7"  # Higher threshold for accuracy
REQUEST_TIMEOUT="10s"          # Stricter timeouts
```

## Validation and Troubleshooting

### Required Variable Validation
Each service validates its required environment variables on startup:

1. **fn-posts**: Validates DATABASE_URL, GCS_BUCKET_NAME, KAFKA_* variables
2. **fn-notifications**: Validates DATABASE_URL, KAFKA_* variables, notification provider keys
3. **fn-media-ai**: Validates KAFKA_* variables, GCS_BUCKET_NAME, model paths
4. **fn-matcher**: Validates DATABASE_URL, KAFKA_* variables, algorithm weights

### Common Configuration Issues

#### Kafka Connection Issues
```bash
# Verify Kafka connectivity
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties --list
```

#### Database Connection Issues
```bash
# Test database connectivity
psql $DATABASE_URL -c "SELECT 1;"
```

#### Google Cloud Storage Issues
```bash
# Verify GCS access
gsutil ls gs://$GCS_BUCKET_NAME
```

### Environment Variable Precedence
1. **Explicit Environment Variables**: Highest priority
2. **Container/Pod Environment**: Medium priority
3. **Service Defaults**: Lowest priority (fallback)

## Migration Guide

### Breaking Changes in Recent Updates

#### Kafka Configuration Standardization
- **All Services**: Changed `KAFKA_BROKERS` to `KAFKA_BOOTSTRAP_SERVERS`
- **fn-media-ai**: `KAFKA_CONSUMER_TOPICS` now requires JSON array format
- **fn-matcher**: Updated to use consolidated consumer topics configuration
- **All Services**: Standardized topic names to `posts.events`, `posts.enhancements`, `posts.matching`, `notifications.events`

#### Algorithm Configuration
- **fn-matcher**: Added configurable algorithm weights via `MATCHING_*_WEIGHT` variables
- **fn-matcher**: Default weights: Location=0.3, Visual=0.4, Text=0.2, Temporal=0.1

---

*For deployment-specific configurations, see individual service deployment guides. For troubleshooting, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).*