# Cloud Services Setup

**Document Ownership**: This document OWNS all cloud service configurations and environment setup across all services.

## Prerequisites

- **Google Cloud Account** with billing enabled
- **Confluent Cloud Account** for managed Kafka
- **Twilio Account** for SMS/WhatsApp delivery
- **Email Provider** (SendGrid, Mailgun, or SMTP)
- **GitHub Account** for repository hosting and CI/CD
- **Terraform** >= 1.5.0 for infrastructure as code
- **Helm** >= 3.12.0 for Kubernetes deployments
- **kubectl** >= 1.28.0 for cluster management

## GKE Autopilot Infrastructure

### Architecture Overview

**Google Kubernetes Engine (GKE) Autopilot** provides a fully-managed, cost-optimized Kubernetes platform:

- **Serverless Kubernetes**: No node management required
- **Pay-per-Pod**: Only pay for actual pod resources (~$100/month)
- **Auto-scaling**: Automatic horizontal and vertical scaling
- **Built-in Security**: Workload Identity, Binary Authorization, encrypted secrets
- **Regional Availability**: Multi-zone deployment for high availability

### Cost Breakdown (~$100/month)

| Component | Monthly Cost | Details |
|-----------|-------------|----------|
| GKE Autopilot Control Plane | $0 | Free managed control plane |
| Pod Compute (4 services) | ~$60 | 0.5 vCPU, 1GB RAM per service |
| Load Balancer | ~$18 | Single HTTPS ingress |
| Persistent Storage | ~$5 | 20GB for PostgreSQL |
| Network Egress | ~$10 | API and user traffic |
| Google Secret Manager | ~$5 | Secure secret storage |
| Container Registry | ~$2 | Docker image storage |

### Infrastructure Components

#### Terraform Configuration
Infrastructure as Code for reproducible deployments:

```hcl
# terraform/main.tf
resource "google_container_cluster" "autopilot" {
  name             = "findly-${var.environment}-cluster"
  location         = var.region
  enable_autopilot = true

  # Workload Identity for secure pod authentication
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Private cluster for enhanced security
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block = "172.16.0.0/28"
  }
}
```

#### Helm Charts Structure
Standardized deployment templates for all services:

```yaml
# helm/<service>/values.yaml
replicaCount: 2  # High availability
resources:
  requests:
    cpu: 500m     # Autopilot minimum
    memory: 1Gi
  limits:
    cpu: 1000m
    memory: 2Gi
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

#### CI/CD Pipeline with GitHub Actions
Automated deployment workflow:

```yaml
# .github/workflows/deploy.yml
- Build & test code
- Build Docker image
- Push to Google Artifact Registry
- Deploy with Helm to GKE
- Run smoke tests
- Monitor deployment
```

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

## GKE Deployment Setup

### 1. Initial GKE Cluster Setup

```bash
# Clone infrastructure repository
cd fn-infra

# Initialize Terraform
cd terraform
terraform init

# Create GKE Autopilot cluster (one-time setup)
terraform plan -var="environment=dev"
terraform apply -var="environment=dev" -auto-approve

# Configure kubectl
gcloud container clusters get-credentials findly-dev-cluster --region=us-central1

# Verify cluster access
kubectl get nodes
kubectl get namespaces
```

### 2. Workload Identity Setup

Enable secure pod authentication without service account keys:

```bash
# Create Google service account
gcloud iam service-accounts create findly-workload-sa \
  --display-name="Findly Workload Identity Service Account"

# Grant necessary permissions
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:findly-workload-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:findly-workload-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Bind to Kubernetes service account
kubectl annotate serviceaccount findly-sa \
  iam.gke.io/gcp-service-account=findly-workload-sa@${PROJECT_ID}.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding \
  findly-workload-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[findly/findly-sa]"
```

### 3. Google Secret Manager Configuration

Store sensitive configuration securely:

```bash
# Create secrets for each service
gcloud secrets create kafka-credentials --data-file=kafka-creds.json
gcloud secrets create database-url --data-file=db-url.txt
gcloud secrets create twilio-credentials --data-file=twilio.json
gcloud secrets create openai-api-key --data-file=openai-key.txt

# Grant access to workload identity
gcloud secrets add-iam-policy-binding kafka-credentials \
  --member="serviceAccount:findly-workload-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### 4. Helm Deployment

```bash
# Add Helm repositories
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager for TLS
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Install NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Deploy services with Helm
cd fn-infra/helm

# Deploy each service
helm install fn-posts ./fn-posts -f ./fn-posts/values-dev.yaml
helm install fn-notifications ./fn-notifications -f ./fn-notifications/values-dev.yaml
helm install fn-media-ai ./fn-media-ai -f ./fn-media-ai/values-dev.yaml
helm install fn-matcher ./fn-matcher -f ./fn-matcher/values-dev.yaml

# Verify deployments
kubectl get deployments -n findly
kubectl get pods -n findly
kubectl get services -n findly
```

### 5. Monitoring and Observability

```bash
# Enable Google Cloud Monitoring
gcloud services enable monitoring.googleapis.com

# Deploy monitoring stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Access Grafana dashboard
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Open http://localhost:3000 (admin/prom-operator)
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