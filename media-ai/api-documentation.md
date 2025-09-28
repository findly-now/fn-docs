# fn-media-ai API Documentation

**REST API and event processing interface for AI-powered photo analysis and Lost & Found post enhancement.**

## Base Configuration

**Base URL**: `http://localhost:8000` (development) / `https://api.findlynow.com/media-ai` (production)

**API Version**: `v1`

**Content Type**: `application/json`

**Authentication**: Internal service-to-service communication (no external API access)

## Service Architecture

fn-media-ai is primarily an **event-driven service** that processes photos asynchronously through Kafka events rather than exposing REST endpoints for photo analysis. The REST API is limited to health checks and service information.

### Event Flow

```mermaid
graph LR
    A[fn-posts] -->|post.created| B[Kafka Topic]
    B --> C[fn-media-ai Consumer]
    C --> D[AI Processing Pipeline]
    D --> E[Enhanced Metadata]
    E -->|post.enhanced| F[Kafka Topic]
    F --> G[fn-posts]
```

## REST API Endpoints

### 1. Service Information

**Endpoint**: `GET /`

**Purpose**: Get basic service information and API discovery.

**Response**:
```json
{
  "service": "fn-media-ai",
  "version": "0.1.0",
  "description": "AI-powered photo analysis for Lost & Found enhancement",
  "docs_url": "/docs"
}
```

**Example Request**:
```bash
curl http://localhost:8000/
```

### 2. Health Check Endpoints

#### Basic Health Check

**Endpoint**: `GET /api/v1/health`

**Purpose**: Simple health check for load balancer probes.

**Response**:
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T14:30:00Z",
  "service": "fn-media-ai",
  "version": "0.1.0",
  "checks": {
    "api": "healthy"
  }
}
```

**HTTP Status Codes**:
- `200 OK`: Service is healthy
- `503 Service Unavailable`: Service is unhealthy

#### Detailed Health Check

**Endpoint**: `GET /api/v1/health/detailed`

**Purpose**: Comprehensive health check including all dependencies.

**Response**:
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T14:30:00Z",
  "service": "fn-media-ai",
  "version": "0.1.0",
  "uptime_seconds": 3600.5,
  "checks": {
    "kafka": {
      "status": "healthy",
      "response_time_ms": 45,
      "broker_count": 3,
      "topics": ["posts.events", "posts.enhancements"]
    },
    "gcs": {
      "status": "healthy",
      "bucket_name": "findly-now-photos-prod",
      "location": "US",
      "storage_class": "STANDARD"
    },
    "openai": {
      "status": "healthy",
      "model_count": 127,
      "configured_model": "gpt-4-vision-preview"
    },
    "redis": {
      "status": "healthy",
      "ping_response": true
    }
  }
}
```

**Component Health Status**:
- `healthy`: Component is fully operational
- `degraded`: Component has issues but service can continue
- `unhealthy`: Component is down, affecting service functionality

**HTTP Status Codes**:
- `200 OK`: All components healthy or degraded
- `503 Service Unavailable`: Critical components unhealthy

#### Kubernetes Readiness Probe

**Endpoint**: `GET /api/v1/health/ready`

**Purpose**: Kubernetes readiness check to determine if service can receive traffic.

**Response**:
```json
{
  "status": "ready",
  "timestamp": "2024-01-15T14:30:00Z",
  "checks": {
    "kafka": {
      "status": "healthy",
      "response_time_ms": 23,
      "broker_count": 3
    }
  }
}
```

**HTTP Status Codes**:
- `200 OK`: Service is ready to receive traffic
- `503 Service Unavailable`: Service is not ready

#### Kubernetes Liveness Probe

**Endpoint**: `GET /api/v1/health/live`

**Purpose**: Kubernetes liveness check to determine if service should be restarted.

**Response**:
```json
{
  "status": "alive",
  "timestamp": "2024-01-15T14:30:00Z",
  "service": "fn-media-ai"
}
```

**HTTP Status Codes**:
- `200 OK`: Service is running normally
- `503 Service Unavailable`: Service should be restarted

### 3. API Documentation

#### Interactive API Documentation

**Endpoint**: `GET /docs` (development only)

**Purpose**: Swagger/OpenAPI interactive documentation.

**Access**: Only available when `DEBUG=true` in development environment.

#### ReDoc Documentation

**Endpoint**: `GET /redoc` (development only)

**Purpose**: Alternative API documentation interface.

**Access**: Only available when `DEBUG=true` in development environment.

## Event Processing Interface

### Consumed Events

#### PostCreated Event

**Topic**: `posts.events`

**Event Type**: `post.created`

**Purpose**: Triggers AI analysis when a new post with photos is created.

**Event Schema**:
```json
{
  "event_id": "evt-123e4567-e89b-12d3-a456-426614174000",
  "event_type": "post.created",
  "timestamp": "2024-01-15T14:30:00Z",
  "version": "1.0",
  "data": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "user_id": "user-123a4567-b89c-12d3-e456-789012345678",
    "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
    "title": "Lost iPhone 15 Pro",
    "description": "Black iPhone with blue MagSafe case",
    "post_type": "lost",
    "location": {
      "lat": 40.7831,
      "lng": -73.9665,
      "address": "Central Park, New York, NY"
    },
    "photos": [
      {
        "photo_id": "photo-789a0123-b45c-67d8-e901-234567890abc",
        "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
        "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
        "content_type": "image/jpeg",
        "size_bytes": 2048576,
        "display_order": 1
      }
    ],
    "created_at": "2024-01-15T14:30:00Z"
  }
}
```

**Processing Pipeline**:
1. Download photos from Google Cloud Storage
2. Run parallel AI model inference:
   - Object detection (YOLO)
   - Scene classification (ResNet)
   - OCR text extraction (Tesseract)
   - Location inference (GPT-4 Vision)
3. Combine results with confidence scoring
4. Publish enhancement event

### Published Events

#### PostEnhanced Event

**Topic**: `posts.enhancements`

**Event Type**: `post.enhanced`

**Purpose**: Notifies fn-posts service of AI-generated metadata for post enhancement.

**Event Schema**:
```json
{
  "event_id": "evt-abc12345-def6-7890-1234-567890abcdef",
  "event_type": "post.enhanced",
  "timestamp": "2024-01-15T14:30:15Z",
  "version": "1.0",
  "correlation_id": "evt-123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "analysis_id": "analysis-fedcba09-8765-4321-0fed-cba098765432",
    "processing_duration_ms": 3420,
    "confidence_score": 0.87,
    "enhancement_level": "auto_enhance",
    "generated_metadata": {
      "auto_tags": [
        {
          "tag": "electronics",
          "confidence": 0.95,
          "source": "object_detection"
        },
        {
          "tag": "smartphone",
          "confidence": 0.92,
          "source": "object_detection"
        },
        {
          "tag": "apple_iphone",
          "confidence": 0.89,
          "source": "object_detection"
        },
        {
          "tag": "blue_case",
          "confidence": 0.81,
          "source": "color_analysis"
        }
      ],
      "detected_objects": [
        {
          "object": "smartphone",
          "confidence": 0.92,
          "bounding_box": {
            "x": 120,
            "y": 80,
            "width": 180,
            "height": 320
          },
          "attributes": {
            "brand": "Apple",
            "model": "iPhone",
            "color": "black"
          }
        },
        {
          "object": "phone_case",
          "confidence": 0.85,
          "bounding_box": {
            "x": 115,
            "y": 75,
            "width": 190,
            "height": 330
          },
          "attributes": {
            "color": "blue",
            "material": "silicone"
          }
        }
      ],
      "scene_classification": {
        "primary_scene": "indoor",
        "confidence": 0.78,
        "sub_scenes": ["office", "desk"],
        "lighting": "natural",
        "background": "neutral"
      },
      "extracted_text": [
        {
          "text": "iPhone",
          "confidence": 0.94,
          "language": "en",
          "location": {
            "x": 140,
            "y": 300,
            "width": 80,
            "height": 20
          }
        }
      ],
      "location_context": {
        "inferred_location_type": "office_building",
        "confidence": 0.72,
        "landmarks": [],
        "geographic_features": []
      },
      "enhanced_description": {
        "generated_text": "Black Apple iPhone smartphone with blue silicone case, photographed in an indoor office setting. The device appears to be a recent iPhone model based on its design characteristics.",
        "confidence": 0.83,
        "improvement_areas": [
          "Added specific brand identification",
          "Enhanced color description",
          "Included material identification",
          "Added environmental context"
        ]
      }
    },
    "ai_models_used": [
      {
        "model": "yolov8n",
        "version": "8.0.0",
        "task": "object_detection",
        "processing_time_ms": 1200
      },
      {
        "model": "resnet50",
        "version": "1.0",
        "task": "scene_classification",
        "processing_time_ms": 800
      },
      {
        "model": "tesseract",
        "version": "5.0",
        "task": "ocr",
        "processing_time_ms": 600
      },
      {
        "model": "gpt-4-vision-preview",
        "version": "gpt-4-1106-vision-preview",
        "task": "location_inference",
        "processing_time_ms": 820
      }
    ],
    "analysis_metadata": {
      "photo_count": 1,
      "total_size_bytes": 2048576,
      "average_resolution": "1920x1080",
      "quality_score": 0.88,
      "processing_started_at": "2024-01-15T14:30:02Z",
      "processing_completed_at": "2024-01-15T14:30:15Z"
    }
  }
}
```

### Enhancement Levels

Based on confidence scores, different enhancement levels are applied:

| Confidence Range | Enhancement Level | Action |
|-----------------|-------------------|---------|
| 0.85 - 1.00 | `auto_enhance` | Automatically update post with high-confidence metadata |
| 0.70 - 0.84 | `suggest_tags` | Suggest tags for user approval |
| 0.50 - 0.69 | `human_review` | Flag for manual verification |
| 0.00 - 0.49 | `discard` | Discard low-confidence results |

### Error Events

#### ProcessingError Event

**Topic**: `posts.enhancements`

**Event Type**: `processing.error`

**Purpose**: Notifies of failures during AI processing.

**Event Schema**:
```json
{
  "event_id": "evt-error123-4567-8901-2345-678901234567",
  "event_type": "processing.error",
  "timestamp": "2024-01-15T14:30:05Z",
  "version": "1.0",
  "correlation_id": "evt-123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "error_type": "photo_download_failed",
    "error_message": "Failed to download photo from GCS: Access denied",
    "retry_count": 2,
    "max_retries": 3,
    "will_retry": true,
    "next_retry_at": "2024-01-15T14:35:05Z",
    "failed_photos": [
      {
        "photo_id": "photo-789a0123-b45c-67d8-e901-234567890abc",
        "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
        "error": "403 Forbidden"
      }
    ]
  }
}
```

## AI Processing Capabilities

### Object Detection

**Models Used**: YOLOv8n, DETR
**Capabilities**:
- Electronics (phones, laptops, tablets, headphones)
- Accessories (cases, bags, jewelry, watches)
- Clothing items (jackets, shoes, bags)
- Personal items (keys, wallets, documents)
- Sports equipment
- Household items

**Response Format**:
```json
{
  "object": "smartphone",
  "confidence": 0.92,
  "bounding_box": {"x": 120, "y": 80, "width": 180, "height": 320},
  "attributes": {
    "brand": "Apple",
    "model": "iPhone",
    "color": "black",
    "condition": "good"
  }
}
```

### Scene Classification

**Models Used**: ResNet50, EfficientNet
**Capabilities**:
- Indoor/outdoor classification
- Location types (office, park, street, store, transport)
- Lighting conditions
- Background analysis

**Response Format**:
```json
{
  "primary_scene": "indoor",
  "confidence": 0.78,
  "sub_scenes": ["office", "desk"],
  "lighting": "natural",
  "background": "neutral"
}
```

### OCR Text Extraction

**Models Used**: Tesseract, EasyOCR
**Capabilities**:
- Serial numbers and model numbers
- Brand names and labels
- License plates (for vehicles)
- Street signs and addresses
- Multiple language support

**Response Format**:
```json
{
  "text": "iPhone",
  "confidence": 0.94,
  "language": "en",
  "location": {"x": 140, "y": 300, "width": 80, "height": 20}
}
```

### Location Inference

**Models Used**: GPT-4 Vision
**Capabilities**:
- Landmark recognition
- Geographic feature identification
- Building and location type inference
- Environmental context analysis

**Response Format**:
```json
{
  "inferred_location_type": "university_campus",
  "confidence": 0.85,
  "landmarks": ["clock_tower", "university_building"],
  "geographic_features": ["courtyard", "walkway"]
}
```

## Performance Metrics

### Processing Performance

- **Target Latency**: <5 seconds per photo (goal: <3 seconds)
- **Throughput**: 100+ concurrent photo analyses
- **Accuracy**: >85% object detection accuracy for common Lost & Found items
- **Availability**: 99.5% uptime

### Model Performance

| Model | Average Latency | Accuracy | Resource Usage |
|-------|----------------|----------|----------------|
| YOLO Object Detection | 1.2s | 92% | High GPU |
| ResNet Scene Classification | 0.8s | 88% | Medium GPU |
| Tesseract OCR | 0.6s | 94% | Low CPU |
| GPT-4 Vision | 0.8s | 89% | High API |

## Error Handling

### Common Error Responses

**Photo Download Errors**:
```json
{
  "error_type": "photo_download_failed",
  "error_code": "GCS_ACCESS_DENIED",
  "message": "Failed to download photo from Google Cloud Storage",
  "retry_count": 1,
  "max_retries": 3
}
```

**AI Model Errors**:
```json
{
  "error_type": "model_inference_failed",
  "error_code": "MODEL_TIMEOUT",
  "message": "Object detection model timed out after 30 seconds",
  "model": "yolov8n",
  "fallback_applied": true
}
```

**API Rate Limit Errors**:
```json
{
  "error_type": "api_rate_limit",
  "error_code": "OPENAI_RATE_LIMIT",
  "message": "OpenAI API rate limit exceeded",
  "retry_after_seconds": 60
}
```

### Retry Logic

- **Transient Failures**: Exponential backoff (1s, 2s, 4s, 8s)
- **Rate Limits**: Respect API-provided retry-after headers
- **Photo Download**: 3 retry attempts with different endpoints
- **Model Inference**: Circuit breaker pattern with fallback models

## Security & Access Control

### Internal Service Communication

- Service-to-service authentication via Kafka SASL/SSL
- Google Cloud Storage access via service account keys
- OpenAI API access via secure API keys
- Redis access with authentication when configured

### Data Protection

- Photos accessed read-only from Google Cloud Storage
- No photo data stored locally beyond processing
- AI analysis results cached temporarily in Redis
- All external API calls use TLS encryption

### Privacy Compliance

- No personal data stored beyond processing duration
- Photos automatically purged from local cache after analysis
- AI analysis results contain no personally identifiable information
- Audit logs for all photo access and processing activities

## Rate Limiting

### External API Limits

- **OpenAI GPT-4 Vision**: 1000 requests/minute (configurable)
- **Google Cloud Storage**: No specific limits (pay-per-use)
- **Photo Processing**: 100 concurrent analyses (configurable)

### Internal Processing Limits

- **Kafka Consumer**: 100 messages per batch
- **Concurrent Photo Analysis**: 50 photos simultaneously
- **Memory Usage**: 512MB per photo analysis worker
- **Timeout Limits**: 30 seconds per AI model inference

## Monitoring & Observability

### Key Metrics

```json
{
  "processing_metrics": {
    "photos_processed_total": 15420,
    "average_processing_time_ms": 3200,
    "confidence_score_distribution": {
      "high_confidence_0.8+": 0.65,
      "medium_confidence_0.5-0.8": 0.25,
      "low_confidence_<0.5": 0.10
    }
  },
  "model_metrics": {
    "yolo_detection_latency_ms": 1200,
    "resnet_classification_latency_ms": 800,
    "ocr_extraction_latency_ms": 600,
    "gpt4_inference_latency_ms": 820
  },
  "error_metrics": {
    "photo_download_errors": 12,
    "model_inference_errors": 5,
    "api_rate_limit_errors": 2
  }
}
```

### Health Monitoring

- **Kafka Consumer Lag**: Monitor event processing delay
- **Model Response Times**: Track AI model performance
- **External API Health**: Monitor OpenAI and GCS availability
- **Memory and CPU Usage**: Track resource consumption

---

*This API documentation covers the complete interface for the fn-media-ai service. For architectural details, see [domain-architecture.md](domain-architecture.md). For deployment instructions, see [deployment-guide.md](deployment-guide.md).*