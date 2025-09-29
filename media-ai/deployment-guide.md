# fn-media-ai Deployment Guide

**Complete deployment guide for the Media AI domain service across development, staging, and production environments.**

## Prerequisites

### Cloud Infrastructure
- **Confluent Cloud** Kafka cluster for event processing
- **Google Cloud Storage** bucket for photo access
- **OpenAI API** account with GPT-4 Vision access
- **Redis** cluster for AI result caching (optional)
- **Kubernetes cluster** (GKE, EKS, or AKS) for container orchestration

### Required Tools
- **Docker** 20.10+ for containerization
- **kubectl** for Kubernetes management
- **Helm** 3.0+ for Kubernetes deployments
- **Python** 3.11+ for local development
- **pip** and **pipenv** for dependency management
- **Make** for build automation

### Access & Credentials
- Confluent Cloud Kafka API keys and bootstrap servers
- Google Cloud Storage service account with read permissions
- OpenAI API key with GPT-4 Vision API access
- Redis connection string (if using external Redis)
- Container registry access (Google Container Registry or Docker Hub)

### AI/ML Prerequisites
- **CUDA support** for GPU acceleration (optional but recommended)
- Sufficient storage for AI models (minimum 5GB)
- Memory requirements: 2GB+ for model loading
- CPU requirements: 4+ cores recommended for production

## Environment Configuration

### Environment Variables

**Core Service Configuration**:
```bash
# Application Settings
APP_NAME="fn-media-ai"
APP_VERSION="0.1.0"
ENVIRONMENT=production              # development | staging | production
DEBUG=false
LOG_LEVEL="INFO"                   # DEBUG | INFO | WARNING | ERROR | CRITICAL

# Server Configuration
HOST="0.0.0.0"
PORT=8000
WORKERS=4                          # Number of uvicorn workers (production: 4-8)

# Security Settings
ALLOWED_HOSTS="api.findlynow.com,localhost"
CORS_ORIGINS="https://app.findlynow.com,https://admin.findlynow.com"
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOW_METHODS="GET,POST,OPTIONS"
CORS_ALLOW_HEADERS="*"
```

**Kafka Event Streaming Configuration (Standardized)**:
```bash
# Confluent Cloud Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS="your-kafka-cluster.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-kafka-api-key"
KAFKA_SASL_PASSWORD="your-kafka-api-secret"
KAFKA_SECURITY_PROTOCOL="SASL_SSL"
KAFKA_SASL_MECHANISM="PLAIN"

# Topic Configuration (Standardized Names)
KAFKA_CONSUMER_GROUP_ID="fn-media-ai-production"
KAFKA_CONSUMER_TOPICS="posts.events"           # Input: Posts events for photo analysis
KAFKA_POST_ENHANCED_TOPIC="media-ai.enrichment" # Output: Enhanced posts with AI metadata

# Consumer Performance Settings
KAFKA_AUTO_OFFSET_RESET="latest"
KAFKA_MAX_POLL_RECORDS=50
KAFKA_SESSION_TIMEOUT_MS=30000
KAFKA_HEARTBEAT_INTERVAL_MS=10000
KAFKA_FETCH_MIN_BYTES=1024
KAFKA_FETCH_MAX_WAIT_MS=500
KAFKA_MAX_PARTITION_FETCH_BYTES=1048576

# Consumer Resilience
KAFKA_RETRY_BACKOFF_MS=1000
KAFKA_REQUEST_TIMEOUT_MS=30000
KAFKA_RECONNECT_BACKOFF_MS=50
KAFKA_RECONNECT_BACKOFF_MAX_MS=1000
```

**Google Cloud Storage Configuration**:
```bash
# Photo Storage Access
GCS_BUCKET_NAME="findly-now-photos-production"
GCS_PROJECT_ID="findly-now-production"

# Authentication (choose one method)
# Method 1: Service Account Key File
GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"

# Method 2: Service Account Key as Environment Variable
# GOOGLE_APPLICATION_CREDENTIALS_JSON='{...service account JSON...}'

# Storage Performance Settings
GCS_DOWNLOAD_TIMEOUT=30
GCS_CHUNK_SIZE=8192
GCS_MAX_CONCURRENT_DOWNLOADS=10
```

**OpenAI API Configuration**:
```bash
# OpenAI GPT-4 Vision API
OPENAI_API_KEY="sk-..."
OPENAI_MODEL="gpt-4-vision-preview"
OPENAI_MAX_TOKENS=1000
OPENAI_TEMPERATURE=0.1

# Rate Limiting and Timeout
OPENAI_REQUEST_TIMEOUT=60
OPENAI_MAX_RETRIES=3
OPENAI_RETRY_DELAY=1.0
OPENAI_MAX_REQUESTS_PER_MINUTE=1000
OPENAI_MAX_TOKENS_PER_MINUTE=50000
```

**Redis Caching Configuration** (Optional):
```bash
# Redis for AI Result Caching
REDIS_URL="redis://username:password@redis-host:6379/0"
REDIS_MAX_CONNECTIONS=20
REDIS_CONNECTION_TIMEOUT=5
REDIS_SOCKET_TIMEOUT=5
REDIS_SOCKET_KEEPALIVE=true
REDIS_HEALTH_CHECK_INTERVAL=30

# Cache TTL Settings
REDIS_AI_RESULTS_TTL=3600          # 1 hour
REDIS_MODEL_CACHE_TTL=86400        # 24 hours
REDIS_PHOTO_METADATA_TTL=1800      # 30 minutes
```

**AI Model Configuration**:
```bash
# Model Settings
AI_MODEL_CACHE_DIR="/app/models/cache"
AI_MODEL_DOWNLOAD_TIMEOUT=300
AI_ENABLE_GPU=false                # Set to true if CUDA available
AI_BATCH_SIZE=1
AI_MAX_IMAGE_SIZE=2048
AI_IMAGE_QUALITY=85

# Confidence Thresholds
AI_CONFIDENCE_AUTO_ENHANCE=0.85    # Auto-enhance posts (85%+)
AI_CONFIDENCE_SUGGEST_TAGS=0.70    # Suggest tags for user approval (70%+)
AI_CONFIDENCE_HUMAN_REVIEW=0.50    # Flag for manual verification (50%+)
AI_CONFIDENCE_DISCARD_BELOW=0.30   # Discard low-confidence results (<30%)

# Model-Specific Settings
YOLO_MODEL_SIZE="n"                # n | s | m | l | x (nano to extra-large)
RESNET_PRETRAINED=true
OCR_LANGUAGE="eng"                 # Language for OCR processing
OCR_ENGINE_MODE=3                  # Tesseract engine mode
```

**Development Environment**:
```bash
# Development overrides
ENVIRONMENT="development"
DEBUG=true
LOG_LEVEL="DEBUG"
WORKERS=1
AI_ENABLE_GPU=false
OPENAI_MODEL="gpt-4-vision-preview"  # Use cheaper model for dev
```

**Staging Environment**:
```bash
# Staging configuration
ENVIRONMENT="staging"
DEBUG=false
LOG_LEVEL="INFO"
WORKERS=2
KAFKA_CONSUMER_GROUP_ID="fn-media-ai-staging"
GCS_BUCKET_NAME="findly-now-photos-staging"
```

## AI Models Setup

### Model Download and Initialization

**Download Required Models**:
```bash
# Download all AI models
make models-download-all

# Or download individually
python -c "
from transformers import AutoModel, AutoTokenizer
from ultralytics import YOLO

# Download YOLO models
YOLO('yolov8n.pt')  # Nano - fastest, lowest accuracy
YOLO('yolov8s.pt')  # Small - balanced
YOLO('yolov8m.pt')  # Medium - higher accuracy

# Download ResNet for scene classification
AutoModel.from_pretrained('microsoft/resnet-50')

# Download OCR model
AutoModel.from_pretrained('microsoft/trocr-base-printed')
"
```

**Model Storage Structure**:
```
models/
├── cache/                 # HuggingFace model cache
├── local/                 # Custom model files
├── yolov8n.pt            # YOLO object detection model
├── yolov8s.pt            # Alternative YOLO model
└── weights/              # Additional model weights
```

**Model Validation**:
```bash
# Validate all models work correctly
make models-validate

# Check model information
make models-info-detailed
```

### GPU Configuration (Optional)

**CUDA Setup for GPU Acceleration**:
```bash
# Check CUDA availability
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}')"

# Enable GPU in environment
AI_ENABLE_GPU=true
CUDA_VISIBLE_DEVICES=0
```

**Docker GPU Support**:
```dockerfile
# Add to Dockerfile for GPU support
FROM nvidia/cuda:11.8-runtime-ubuntu22.04

# Install CUDA-compatible PyTorch
RUN pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
```

## Docker Configuration

### Multi-Stage Dockerfile

The service uses an optimized multi-stage Dockerfile:

```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    libopencv-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install dependencies
COPY pyproject.toml requirements.txt ./
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    libopencv-imgproc4.5 \
    libopencv-imgcodecs4.5 \
    tesseract-ocr \
    tesseract-ocr-eng \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy wheels and install
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache /wheels/*

# Copy application code
COPY src/ ./src/
COPY models/ ./models/

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/api/v1/health || exit 1

# Start command
CMD ["uvicorn", "fn_media_ai.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose Development

**Complete Development Stack**:
```yaml
version: '3.8'

services:
  fn-media-ai:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=development
      - DEBUG=true
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./src:/app/src
      - ./models:/app/models
      - ./logs:/app/logs
    depends_on:
      - kafka
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  redis_data:
  grafana_data:
```

### Container Build Commands

```bash
# Build development image
docker build -t fn-media-ai:dev .

# Build production image with optimizations
docker build -t fn-media-ai:latest --target production .

# Build with GPU support
docker build -t fn-media-ai:gpu --build-arg ENABLE_GPU=true .

# Multi-platform build for deployment
docker buildx build --platform linux/amd64,linux/arm64 -t fn-media-ai:latest .

# Build with specific Python version
docker build -t fn-media-ai:python311 --build-arg PYTHON_VERSION=3.11 .
```

## Kubernetes Deployment

### Kubernetes Manifests

**Deployment Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fn-media-ai
  namespace: findly-now
  labels:
    app: fn-media-ai
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fn-media-ai
  template:
    metadata:
      labels:
        app: fn-media-ai
        version: v1
    spec:
      containers:
      - name: fn-media-ai
        image: gcr.io/findly-now/fn-media-ai:latest
        ports:
        - containerPort: 8000
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: KAFKA_BOOTSTRAP_SERVERS
          valueFrom:
            configMapKeyRef:
              name: kafka-config
              key: bootstrap-servers
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: fn-media-ai-secrets
              key: openai-api-key
        - name: GOOGLE_APPLICATION_CREDENTIALS_JSON
          valueFrom:
            secretKeyRef:
              name: fn-media-ai-secrets
              key: gcs-service-account
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /api/v1/health/live
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v1/health/ready
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: model-cache
          mountPath: /app/models/cache
        - name: tmp-storage
          mountPath: /tmp
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache-pvc
      - name: tmp-storage
        emptyDir:
          sizeLimit: 5Gi
      nodeSelector:
        workload-type: ai-compute
      tolerations:
      - key: ai-compute
        operator: Equal
        value: "true"
        effect: NoSchedule
```

**Service Configuration**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fn-media-ai-service
  namespace: findly-now
spec:
  selector:
    app: fn-media-ai
  ports:
  - port: 8000
    targetPort: 8000
    protocol: TCP
  type: ClusterIP
```

**Persistent Volume for Model Cache**:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache-pvc
  namespace: findly-now
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd
```

**HorizontalPodAutoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fn-media-ai-hpa
  namespace: findly-now
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fn-media-ai
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
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "10"
```

### ConfigMaps and Secrets

**Create Secrets**:
```bash
# Create secrets for sensitive data
kubectl create secret generic fn-media-ai-secrets \
  --from-literal=openai-api-key="sk-..." \
  --from-literal=kafka-sasl-username="api-key" \
  --from-literal=kafka-sasl-password="api-secret" \
  --from-file=gcs-service-account=service-account.json \
  --namespace findly-now
```

**ConfigMap for Non-Sensitive Configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fn-media-ai-config
  namespace: findly-now
data:
  LOG_LEVEL: "INFO"
  WORKERS: "4"
  AI_CONFIDENCE_AUTO_ENHANCE: "0.85"
  AI_CONFIDENCE_SUGGEST_TAGS: "0.70"
  KAFKA_MAX_POLL_RECORDS: "50"
  REDIS_AI_RESULTS_TTL: "3600"
```

### Helm Chart Deployment

**Install via Helm**:
```bash
# Add Findly Now Helm repository
helm repo add findly-now https://charts.findlynow.com
helm repo update

# Install fn-media-ai
helm install fn-media-ai findly-now/fn-media-ai \
  --namespace findly-now \
  --create-namespace \
  --set image.tag=latest \
  --set openai.apiKey=$OPENAI_API_KEY \
  --set kafka.bootstrapServers=$KAFKA_BOOTSTRAP_SERVERS \
  --set gcs.bucketName=$GCS_BUCKET_NAME

# Upgrade deployment
helm upgrade fn-media-ai findly-now/fn-media-ai \
  --namespace findly-now \
  --set image.tag=v1.2.0

# Check deployment status
helm status fn-media-ai -n findly-now
```

**Custom Values for Production**:
```yaml
# values-production.yaml
image:
  repository: gcr.io/findly-now/fn-media-ai
  tag: "1.0.0"
  pullPolicy: IfNotPresent

replicas: 5

resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "4000m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

kafka:
  bootstrapServers: "production-kafka.confluent.cloud:9092"
  securityProtocol: "SASL_SSL"
  consumerGroupId: "fn-media-ai-production"

storage:
  modelCache:
    size: "20Gi"
    storageClass: "fast-ssd"

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s

nodeSelector:
  workload-type: ai-compute

tolerations:
- key: ai-compute
  operator: Equal
  value: "true"
  effect: NoSchedule
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Deploy fn-media-ai

on:
  push:
    branches: [main]
    paths: ['fn-media-ai/**']
  workflow_dispatch:

env:
  REGISTRY: gcr.io
  PROJECT_ID: findly-now-production
  SERVICE_NAME: fn-media-ai

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v4

    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      working-directory: fn-media-ai
      run: |
        pip install -e ".[dev]"
        make download-models

    - name: Run code quality checks
      working-directory: fn-media-ai
      run: |
        make ci-quality

    - name: Run tests
      working-directory: fn-media-ai
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY_TEST }}
        TEST_MOCK_EXTERNAL_APIS: "true"
      run: |
        make ci-test

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: |
          fn-media-ai/test-results.xml
          fn-media-ai/coverage.xml

  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Run security scans
      working-directory: fn-media-ai
      run: |
        make ci-security

    - name: Upload security reports
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: security-reports
        path: |
          fn-media-ai/safety-report.json
          fn-media-ai/bandit-report.json

  build-and-deploy:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Setup Google Cloud
      uses: google-github-actions/setup-gcloud@v1
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ env.PROJECT_ID }}

    - name: Configure Docker for GCR
      run: gcloud auth configure-docker

    - name: Build Docker image
      working-directory: fn-media-ai
      run: |
        docker build \
          --tag ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          --tag ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:latest \
          .

    - name: Push Docker image
      run: |
        docker push ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:latest

    - name: Deploy to Kubernetes
      run: |
        gcloud container clusters get-credentials production-cluster \
          --region us-central1 \
          --project ${{ env.PROJECT_ID }}

        kubectl set image deployment/${{ env.SERVICE_NAME }} \
          ${{ env.SERVICE_NAME }}=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          -n findly-now

        kubectl rollout status deployment/${{ env.SERVICE_NAME }} -n findly-now

    - name: Run post-deployment tests
      run: |
        sleep 60  # Wait for deployment to stabilize
        kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never \
          -- curl -f http://fn-media-ai-service.findly-now:8000/api/v1/health/detailed

  notify:
    needs: [build-and-deploy]
    runs-on: ubuntu-latest
    if: always()
    steps:
    - name: Notify deployment status
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Performance Optimization

### AI Model Optimization

**Model Selection for Performance**:
```bash
# Production model configuration
YOLO_MODEL_SIZE="s"               # Small model for balanced speed/accuracy
AI_BATCH_SIZE=4                   # Process multiple images in batch
AI_MAX_IMAGE_SIZE=1024            # Resize large images for faster processing
AI_ENABLE_GPU=true                # Use GPU acceleration if available
```

**Model Caching Strategy**:
```python
# Redis caching configuration
REDIS_AI_RESULTS_TTL=3600         # Cache results for 1 hour
REDIS_MODEL_CACHE_TTL=86400       # Cache model weights for 24 hours
REDIS_PHOTO_METADATA_TTL=1800     # Cache photo metadata for 30 minutes
```

### Resource Allocation

**CPU and Memory Tuning**:
```yaml
# Kubernetes resource configuration
resources:
  requests:
    memory: "1Gi"      # Minimum for model loading
    cpu: "1000m"       # 1 full CPU core
  limits:
    memory: "4Gi"      # Maximum for large batch processing
    cpu: "4000m"       # 4 CPU cores for AI inference
```

**Storage Optimization**:
```yaml
# Persistent storage for models
volumes:
- name: model-cache
  persistentVolumeClaim:
    claimName: model-cache-pvc
- name: tmp-storage
  emptyDir:
    sizeLimit: 5Gi    # Temporary storage for image processing
```

### Kafka Consumer Optimization

**Consumer Performance Tuning**:
```bash
# Kafka consumer settings for high throughput
KAFKA_MAX_POLL_RECORDS=20         # Batch size for processing
KAFKA_FETCH_MIN_BYTES=16384       # Minimum fetch size
KAFKA_FETCH_MAX_WAIT_MS=500       # Maximum wait time
KAFKA_SESSION_TIMEOUT_MS=30000    # Session timeout
KAFKA_HEARTBEAT_INTERVAL_MS=10000 # Heartbeat interval
```

## Monitoring and Observability

### Health Checks and Metrics

**Health Check Endpoints**:
```bash
# Basic health check
curl http://localhost:8000/api/v1/health

# Detailed health with dependencies
curl http://localhost:8000/api/v1/health/detailed

# Kubernetes probes
curl http://localhost:8000/api/v1/health/ready
curl http://localhost:8000/api/v1/health/live
```

**Prometheus Metrics**:
```python
# Key metrics to monitor
ai_processing_duration_seconds      # Time spent processing photos
ai_confidence_score_distribution    # Distribution of confidence scores
kafka_consumer_lag                  # Consumer lag in processing events
model_inference_duration_seconds    # Time for each AI model
openai_api_requests_total          # OpenAI API usage
gcs_download_duration_seconds      # Photo download performance
redis_cache_hit_ratio              # Cache effectiveness
```

**Grafana Dashboard Configuration**:
```yaml
# Example Grafana dashboard panels
panels:
- title: "AI Processing Latency"
  type: "graph"
  targets:
  - expr: "histogram_quantile(0.95, ai_processing_duration_seconds_bucket)"
  - expr: "histogram_quantile(0.50, ai_processing_duration_seconds_bucket)"

- title: "Confidence Score Distribution"
  type: "histogram"
  targets:
  - expr: "ai_confidence_score_distribution"

- title: "Kafka Consumer Lag"
  type: "singlestat"
  targets:
  - expr: "kafka_consumer_lag"
  thresholds: [10, 50]

- title: "Model Inference Performance"
  type: "table"
  targets:
  - expr: "avg by (model_name) (model_inference_duration_seconds)"
```

### Logging Configuration

**Structured Logging**:
```python
# Example log format
{
  "timestamp": "2024-01-15T14:30:00Z",
  "level": "info",
  "service": "fn-media-ai",
  "correlation_id": "abc-123",
  "user_id": "user-456",
  "post_id": "post-789",
  "message": "AI processing completed",
  "duration_ms": 3420,
  "confidence_score": 0.87,
  "models_used": ["yolo", "resnet", "gpt4v"],
  "photo_count": 3
}
```

**Log Aggregation**:
```yaml
# Fluent Bit configuration for log forwarding
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [INPUT]
        Name tail
        Path /var/log/containers/fn-media-ai*.log
        Parser docker
        Tag kube.*

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On

    [OUTPUT]
        Name elasticsearch
        Match *
        Host elasticsearch.logging.svc.cluster.local
        Port 9200
        Index fn-media-ai-logs
```

## Security Configuration

### API Security

**Authentication and Authorization**:
```yaml
# Network policy for service isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fn-media-ai-network-policy
  namespace: findly-now
spec:
  podSelector:
    matchLabels:
      app: fn-media-ai
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: kafka-consumer
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to: []  # Allow all outbound for API calls
```

**Secret Management**:
```bash
# Rotate secrets regularly
kubectl create secret generic fn-media-ai-secrets-new \
  --from-literal=openai-api-key="new-key" \
  --namespace findly-now

# Update deployment to use new secret
kubectl patch deployment fn-media-ai \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"fn-media-ai","env":[{"name":"OPENAI_API_KEY","valueFrom":{"secretKeyRef":{"name":"fn-media-ai-secrets-new","key":"openai-api-key"}}}]}]}}}}' \
  -n findly-now
```

### Data Protection

**Image Processing Security**:
```python
# Security measures for image processing
MAX_IMAGE_SIZE=10485760           # 10MB max image size
ALLOWED_IMAGE_TYPES=["image/jpeg", "image/png", "image/webp"]
SCAN_FOR_MALWARE=true            # Enable virus scanning
STRIP_EXIF_DATA=true             # Remove metadata from images
```

**API Rate Limiting**:
```yaml
# Rate limiting configuration
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fn-media-ai-rate-limit
spec:
  host: fn-media-ai-service
  trafficPolicy:
    rateLimits:
    - requests: 100
      perUnit: minute
    - requests: 1000
      perUnit: hour
```

## Troubleshooting

### Common Issues

**Model Download Failures**:
```bash
# Check model download status
make models-info-detailed

# Re-download models
rm -rf models/cache/*
make models-download-all

# Check disk space
df -h models/cache/
```

**Kafka Consumer Issues**:
```bash
# Check consumer lag
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  python -c "
  from confluent_kafka.admin import AdminClient
  admin = AdminClient({'bootstrap.servers': '$KAFKA_BOOTSTRAP_SERVERS'})
  metadata = admin.list_consumer_groups()
  print(metadata)
  "

# Reset consumer group
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --group fn-media-ai-production \
  --reset-offsets --to-latest \
  --all-topics --execute
```

**OpenAI API Issues**:
```bash
# Check API quota and usage
curl -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/usage

# Test API connectivity
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  python -c "
  import openai
  client = openai.OpenAI(api_key='$OPENAI_API_KEY')
  models = client.models.list()
  print(f'Available models: {len(models.data)}')
  "
```

**Memory and Performance Issues**:
```bash
# Check memory usage
kubectl top pods -l app=fn-media-ai -n findly-now

# View detailed resource usage
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  python -c "
  import psutil
  print(f'Memory: {psutil.virtual_memory().percent}%')
  print(f'CPU: {psutil.cpu_percent()}%')
  print(f'Disk: {psutil.disk_usage(\"/\").percent}%')
  "

# Profile AI model performance
make benchmark-models
```

### Emergency Procedures

**Service Recovery**:
```bash
# Restart deployment
kubectl rollout restart deployment/fn-media-ai -n findly-now

# Scale down temporarily
kubectl scale deployment fn-media-ai --replicas=0 -n findly-now

# Scale back up
kubectl scale deployment fn-media-ai --replicas=3 -n findly-now

# Rollback to previous version
kubectl rollout undo deployment/fn-media-ai -n findly-now
```

**Clear Caches and Reset**:
```bash
# Clear Redis cache
kubectl exec -it deployment/redis -n findly-now -- redis-cli FLUSHDB

# Clear local model cache
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  rm -rf /app/models/cache/*

# Reset Kafka consumer group
kubectl exec -it deployment/fn-media-ai -n findly-now -- \
  kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --group fn-media-ai-production \
  --reset-offsets --to-earliest \
  --all-topics --execute
```

## Performance Benchmarks

### Expected Performance Metrics

- **Photo Processing**: <5 seconds per photo (target: <3 seconds)
- **Batch Processing**: 20+ photos per minute
- **Memory Usage**: <2GB per replica under normal load
- **CPU Usage**: <70% per replica under normal load
- **Kafka Consumer Lag**: <10 messages during normal operation

### Load Testing

**Basic Load Test**:
```bash
# Test health endpoint
make load-test

# Test AI processing endpoint
make stress-test

# Custom load test with k6
k6 run - <<EOF
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 50 },
    { duration: '2m', target: 0 },
  ],
};

export default function() {
  let response = http.get('http://fn-media-ai-service.findly-now:8000/api/v1/health');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
}
EOF
```

---

*This deployment guide provides comprehensive instructions for deploying the fn-media-ai service across all environments. For architectural details, see [domain-architecture.md](domain-architecture.md). For API specifications, see [api-documentation.md](api-documentation.md).*