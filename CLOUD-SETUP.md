# Cloud Services Setup

**Document Ownership**: This document OWNS all cloud service configurations and environment setup across all services.

## Prerequisites

- **Google Cloud Account** with billing enabled
- **Confluent Cloud Account** for managed Kafka
- **Twilio Account** for SMS/WhatsApp delivery
- **Email Provider** (SendGrid, Mailgun, or SMTP)

## Platform Compatibility

### Apple Silicon (M1/M2/M3) Support

All docker-compose files include `platform: linux/amd64` specifications to prevent architecture warnings on Apple Silicon Macs. Services run under emulation but maintain full functionality.

**Docker Compose Configuration:**
```yaml
services:
  postgres:
    image: postgis/postgis:15-3.4
    platform: linux/amd64  # Prevents AMD warnings on Apple Silicon
    # ... rest of configuration
```

**Local Development Notes:**
- PostgreSQL, Kafka, and Zookeeper containers use explicit platform specification
- No performance impact for development workloads
- Production deployments use native architecture optimizations

### Cross-Platform Docker Builds

For production deployments, use multi-platform builds:
```bash
# Build for both AMD64 and ARM64
docker buildx build --platform linux/amd64,linux/arm64 -t service:latest .

# Push to registry with multi-platform support
docker buildx build --platform linux/amd64,linux/arm64 --push -t registry/service:latest .
```

## Service Dependencies by Domain

### fn-posts Service
- **Supabase**: PostgreSQL + PostGIS for geospatial queries
- **Google Cloud Storage**: Photo storage with global CDN
- **Confluent Cloud Kafka**: Publishing post lifecycle events

### fn-notifications Service
- **Cloud PostgreSQL**: User preferences and notification tracking
- **Confluent Cloud Kafka**: Consuming events via Broadway
- **Twilio**: SMS and WhatsApp delivery
- **Email Provider**: Email delivery via Swoosh

### fn-media-ai Service
- **Confluent Cloud Kafka**: Consuming PostCreated events
- **Google Cloud Storage**: Photo download and access
- **OpenAI API**: Computer vision and NLP services
- **Hugging Face**: Alternative AI model providers

### fn-contract Service
- **Confluent Schema Registry**: API/event schema validation

## Environment Configuration

### fn-posts (.env)
**Note**: fn-posts has two environment templates:
- `.env.cloud.example` - For cloud-based development (recommended)
- `.env.local.example` - For local Docker development

**Cloud Development (.env.cloud.example):**
```bash
# Database - Supabase
DATABASE_URL=postgresql://user:pass@db.supabase.co:5432/db

# Google Cloud Storage
GOOGLE_CLOUD_PROJECT=findly-now-dev
GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcs-service-account.json
GCS_BUCKET_NAME=posts-photos-dev

# Confluent Cloud Kafka
KAFKA_BROKERS=pkc-xxx.confluent.cloud:9092
KAFKA_API_KEY=your-api-key
KAFKA_API_SECRET=your-api-secret
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN

# Service Configuration
PORT=8080
GIN_MODE=release
```

**Local Development (.env.local.example):**
Uses Docker services instead of cloud services for local development.

### fn-notifications (.env)
```bash
# Database - Cloud PostgreSQL
DATABASE_URL=postgresql://user:pass@your-postgres.com:5432/notifications

# Confluent Cloud Kafka
KAFKA_BROKERS=pkc-xxx.confluent.cloud:9092
KAFKA_API_KEY=your-api-key
KAFKA_API_SECRET=your-api-secret
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN

# Twilio (SMS/WhatsApp)
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE_NUMBER=+1234567890
TWILIO_WHATSAPP_NUMBER=whatsapp:+1234567890

# Email Provider (choose one)
# SendGrid
SENDGRID_API_KEY=your-sendgrid-key
# OR Mailgun
MAILGUN_API_KEY=your-mailgun-key
MAILGUN_DOMAIN=your-domain.mailgun.org
# OR SMTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# Service Configuration
PORT=4000
MIX_ENV=prod
SECRET_KEY_BASE=your-64-char-secret-key
```

### fn-media-ai (.env)
```bash
# Confluent Cloud Kafka
KAFKA_BROKERS=pkc-xxx.confluent.cloud:9092
KAFKA_API_KEY=your-api-key
KAFKA_API_SECRET=your-api-secret
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN

# Google Cloud Storage
GOOGLE_CLOUD_PROJECT=findly-now-dev
GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcs-service-account.json

# AI Services
OPENAI_API_KEY=your-openai-api-key
HUGGINGFACE_API_KEY=your-huggingface-key

# Optional AI Providers
GOOGLE_CLOUD_VISION_API_KEY=your-vision-api-key
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret

# Service Configuration
PORT=8000
UVICORN_LOG_LEVEL=info
```

### fn-contract (.env)
```bash
# Confluent Schema Registry
SCHEMA_REGISTRY_URL=https://psrc-xxx.confluent.cloud
SCHEMA_REGISTRY_KEY=your-schema-key
SCHEMA_REGISTRY_SECRET=your-schema-secret
```

## Cloud Service Setup Steps

### 1. Google Cloud Platform Setup

```bash
# Install Google Cloud CLI
curl https://sdk.cloud.google.com | bash

# Authenticate
gcloud auth login
gcloud config set project findly-now-dev

# Create service account for GCS
gcloud iam service-accounts create fn-storage-sa \
  --display-name="Findly Now Storage Service Account"

# Grant storage permissions
gcloud projects add-iam-policy-binding findly-now-dev \
  --member="serviceAccount:fn-storage-sa@findly-now-dev.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

# Create and download key
gcloud iam service-accounts keys create gcs-key.json \
  --iam-account=fn-storage-sa@findly-now-dev.iam.gserviceaccount.com

# Create GCS bucket
gsutil mb gs://posts-photos-dev
gsutil cors set cors.json gs://posts-photos-dev
```

### 2. Confluent Cloud Setup

```bash
# Install Confluent CLI
curl -sL --http1.1 https://cnfl.io/cli | sh -s -- latest

# Login to Confluent Cloud
confluent login

# Create environment and cluster
confluent environment create findly-now-dev
confluent kafka cluster create posts-cluster --cloud gcp --region us-central1

# Create topics
confluent kafka topic create post.created --partitions 3
confluent kafka topic create post.matched --partitions 3
confluent kafka topic create post.claimed --partitions 3
confluent kafka topic create post.resolved --partitions 3
confluent kafka topic create post.enhanced --partitions 3

# Create API keys
confluent api-key create --resource <cluster-id>
```

### 3. Supabase Setup

1. Go to [supabase.com](https://supabase.com)
2. Create new project: `findly-now-dev`
3. Enable PostGIS extension:
   ```sql
   CREATE EXTENSION IF NOT EXISTS postgis;
   ```
4. Get connection string from Settings → Database

### 4. Twilio Setup

1. Go to [twilio.com](https://twilio.com)
2. Create account and verify phone number
3. Get Account SID and Auth Token from Console
4. Purchase phone number for SMS
5. Set up WhatsApp Business API sandbox

### 5. Database Schema Deployment

```bash
# fn-posts (migrations)
cd fn-posts
make migrate-up

# fn-notifications (schema.sql)
cd fn-notifications
psql "$DATABASE_URL" -f schema.sql

# fn-media-ai (no database)
# Uses event processing only
```

## Environment Files Setup

```bash
# Copy templates for each service
cd fn-posts && cp .env.cloud.example .env        # For cloud development
# OR
cd fn-posts && cp .env.local.example .env        # For local development

cd fn-notifications && cp .env.example .env
cd fn-media-ai && cp .env.example .env
cd fn-matcher && cp .env.example .env
cd fn-contract && cp .env.example .env

# Edit each .env file with your cloud credentials
# NEVER commit .env files to version control
```

## Verification Commands

```bash
# Test fn-posts
curl http://localhost:8080/health

# Test fn-notifications
curl http://localhost:4000/api/health

# Test fn-media-ai
curl http://localhost:8000/health

# Test fn-matcher
curl http://localhost:3000/health

# Test Kafka connectivity
confluent kafka topic list
```

## Security Notes

- **Never commit** `.env` files or credentials to version control
- Use **Google Secret Manager** for production environments
- Rotate **API keys** regularly
- Enable **VPC firewall rules** for database access
- Use **IAM roles** with principle of least privilege

## Cost Optimization

- **Supabase**: Use pooling for database connections
- **GCS**: Set lifecycle policies for old photos
- **Confluent Cloud**: Monitor topic retention and partitions
- **Twilio**: Use least expensive channels based on urgency

---

*For service-specific implementation details, see individual service DEVELOPMENT.md files.*