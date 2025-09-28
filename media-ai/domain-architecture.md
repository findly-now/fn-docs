# Media AI Domain Architecture

**The intelligent enhancement engine that transforms photo-based Lost & Found posts into rich, searchable metadata through computer vision and machine learning.**

## Domain Purpose

Transform Lost & Found from manual text descriptions to **AI-powered intelligent discovery** through automated photo analysis and metadata extraction.

**Core Value**: AI analysis converts visual information into structured, searchable metadata, enabling intelligent matching, enhanced discovery, and improved reunification success rates.

## Domain Boundaries

**Bounded Context**: AI-powered photo analysis and post enrichment
- Event-driven processing of PostCreated events with photos
- Computer vision analysis for object detection and scene classification
- Automated tag generation and attribute extraction
- Enhanced description creation through AI models
- PostEnhanced event publishing for intelligent discovery

## Key Architecture Decisions

### 1. Event-Driven AI Processing
**Decision**: Process photos asynchronously via Kafka events rather than synchronous API calls.

**Why**: AI processing can take 2-10 seconds per photo set:
- **Non-blocking**: Users can continue using the platform while AI processes
- **Resilience**: Failed processing doesn't impact core post creation
- **Scalability**: Can queue and batch process during high load
- **Reliability**: Built-in retry mechanisms for transient failures

**Event Flow**:
```
fn-posts → PostCreated → fn-media-ai → PostEnhanced → fn-posts
```

### 2. Python + FastAPI Stack
**Decision**: Use Python with FastAPI for the AI processing service.

**Why**:
- **ML Ecosystem**: Rich Python ecosystem for computer vision (OpenCV, PIL, scikit-image)
- **AI Frameworks**: Direct access to Hugging Face, OpenAI, Google Vision APIs
- **Performance**: FastAPI provides async capabilities for concurrent processing
- **Development Speed**: Rapid prototyping and model experimentation

**Technology Stack**:
```python
# Core Framework
FastAPI + Uvicorn (async web server)

# AI/ML Libraries
transformers     # Hugging Face models
torch           # PyTorch for deep learning
opencv-python   # Computer vision
pillow          # Image processing
openai          # OpenAI Vision API

# Infrastructure
confluent-kafka # Event consumption
google-cloud-storage # Photo access
redis           # Model caching
```

### 3. Multi-Model AI Pipeline
**Decision**: Use multiple specialized AI models rather than a single general-purpose model.

**Why**: Different models excel at different tasks:

| Task | Model Type | Purpose |
|------|------------|---------|
| Object Detection | YOLO/DETR | Identify items, brands, accessories |
| Scene Classification | ResNet/EfficientNet | Detect location context (park, street, indoor) |
| OCR Text Extraction | Tesseract/PaddleOCR | Extract visible text, serial numbers |
| Location Inference | GPT-4 Vision | Landmark detection, location context |

**Pipeline Architecture**:
```python
async def process_photo(photo_url: str) -> EnhancedMetadata:
    # Download and preprocess
    image = await download_photo(photo_url)

    # Parallel AI processing
    objects = await detect_objects(image)      # YOLO
    scene = await classify_scene(image)        # ResNet
    text = await extract_text(image)          # OCR
    location = await infer_location(image)     # GPT-4V

    # Combine results
    return combine_analysis(objects, scene, text, location)
```

### 4. Confidence-Based Enhancement
**Decision**: Include confidence scores with all AI-generated metadata.

**Why**: AI predictions vary in accuracy:
- **Transparency**: Users and systems know prediction reliability
- **Threshold-based**: Only use high-confidence predictions for auto-enhancement
- **Human Review**: Flag low-confidence results for manual verification
- **Model Improvement**: Track confidence vs accuracy for model training

**Confidence Thresholds**:
```python
CONFIDENCE_THRESHOLDS = {
    "auto_enhance": 0.85,      # Automatically enhance post
    "suggest_tags": 0.70,      # Suggest tags to user
    "human_review": 0.50,      # Flag for manual review
    "discard": 0.30           # Discard low-confidence results
}
```

### 5. Google Cloud Storage Integration
**Decision**: Download photos directly from GCS for processing.

**Why**:
- **Performance**: Direct access without API overhead
- **Security**: Service account access to photo storage
- **Caching**: Can cache processed results alongside photos
- **Bandwidth**: Optimized for large image processing

**Processing Flow**:
```python
# Efficient photo processing
async def process_post_photos(post_id: str, photo_urls: List[str]):
    for url in photo_urls:
        # Download from GCS
        image_data = await gcs_client.download_blob(url)

        # Process with AI models
        metadata = await ai_pipeline.process(image_data)

        # Cache results
        await redis_client.set(f"ai:{post_id}:{url}", metadata)
```

### 6. PostEnhanced Event Publishing
**Decision**: Publish enriched metadata as domain events rather than direct API updates.

**Why**: Maintains loose coupling and enables multiple consumers:
- **fn-posts**: Updates post with enhanced tags and description
- **fn-search**: Indexes enhanced metadata for intelligent search
- **fn-analytics**: Tracks AI performance and user engagement
- **Future services**: Recommendations, matching algorithms

## Domain Model

**Aggregate Root**: `PhotoAnalysis`
- Contains AI-generated metadata for a set of photos
- Manages confidence scores and model versions
- Controls enhancement quality thresholds

**Value Objects**:
- `ObjectDetection` (name, confidence, bounding box)
- `SceneClassification` (indoor/outdoor, specific location)
- `TextExtraction` (OCR results with confidence)
- `LocationInference` (coordinates, source, confidence)

**Domain Services**:
- `AIModelPipeline` (coordinates multiple AI models)
- `ConfidenceEvaluator` (determines enhancement quality)
- `MetadataCombiner` (merges results from different models)

## Performance Targets

- **Processing Latency**: <5 seconds per photo (target: <3s)
- **Throughput**: Handle 100+ concurrent photo analyses
- **Accuracy**: >85% object detection accuracy for common items
- **Availability**: 99.5% uptime (AI failures don't impact core platform)

## Integration Points

**Consumes Events**:
- `post.created` → Triggers AI analysis for posts with photos

**Publishes Events**:
- `post.enhanced` → Enriched metadata for intelligent discovery

**External Services**:
- **Google Cloud Storage**: Photo download and access
- **OpenAI Vision API**: Advanced visual analysis and location inference
- **Hugging Face Hub**: Pre-trained computer vision models
- **Confluent Cloud**: Event consumption and publishing

## AI/ML Pipeline

**Model Selection Strategy**:
- **Local Models**: Fast inference for common tasks (object detection, scene classification)
- **Cloud APIs**: Advanced analysis requiring large models (complex reasoning, rare objects)
- **Model Caching**: Cache model outputs to reduce latency and costs

**Processing Stages**:
1. **Image Preprocessing**: Resize, normalize, format conversion
2. **Parallel Inference**: Run multiple models concurrently
3. **Result Fusion**: Combine and validate outputs across models
4. **Confidence Filtering**: Apply thresholds based on model reliability
5. **Metadata Generation**: Create structured output for post enhancement

## Scalability & Performance

**Horizontal Scaling**:
- Stateless service design enables easy replication
- Model inference parallelized across GPU/CPU resources
- Kafka consumer groups for distributed event processing

**Model Optimization**:
- **Quantization**: Reduce model size for faster inference
- **Batching**: Process multiple images together when possible
- **Model Serving**: Use optimized inference servers (TensorRT, ONNX)

**Caching Strategy**:
- Cache AI results to avoid reprocessing identical images
- Model weight caching for faster startup
- Feature extraction caching for incremental analysis

## Key Constraints

- Photos must be accessible via Google Cloud Storage URLs
- AI processing is asynchronous (eventual consistency)
- Confidence scores must accompany all AI-generated metadata
- Failed AI processing must not impact core post functionality
- Model versions tracked for result traceability and debugging

---

*This domain focuses purely on AI-powered enhancement of Lost & Found posts through computer vision and machine learning. For system-wide architecture decisions, see [../ARCHITECTURE-DECISIONS.md](../ARCHITECTURE-DECISIONS.md). For detailed technical implementation, see the fn-media-ai repository.*