# Posts Domain Architecture

**Photo-first lost & found item lifecycle management with geospatial intelligence and event-driven design.**

## Domain Purpose

Transform lost & found item reporting from text-heavy, manual processes to **photo-first, sub-15 second workflows** with intelligent geospatial discovery and automated event publishing for ecosystem-wide integration.

**Core Value**: Photo-centric approach eliminates description barriers while PostGIS-powered spatial search enables rapid discovery within customizable radius zones.

## Domain Boundaries

**Bounded Context**: Lost & Found item lifecycle management
- Photo-first post creation with 1-10 photo requirement and Google Cloud Storage integration
- Geospatial search with PostGIS radius-based queries and proximity scoring
- Post status lifecycle management (active → resolved/expired/deleted)
- Event publishing for all state changes to enable cross-domain integration
- Organization-based multi-tenancy with complete data isolation
- User-driven post management with ownership validation

## Key Architecture Decisions

### 1. Go + Gin + PostGIS for Performance-Critical Operations

**Decision**: Use Go with Gin framework and PostGIS for the posts service.

**Why**: Posts are the foundation of the Lost & Found ecosystem requiring high performance:
- **Performance**: Go's concurrency model handles thousands of simultaneous post creations and searches
- **Spatial Queries**: PostGIS enables sub-second radius searches on millions of posts
- **Simplicity**: Gin provides lightweight HTTP handling without framework overhead
- **Reliability**: Go's error handling and type safety prevent data corruption

**Technology Stack**:
```go
// Core Framework
gin-gonic/gin     // Lightweight HTTP framework
lib/pq           // PostgreSQL driver with PostGIS support

// Cloud Integration
cloud.google.com/go/storage  // Google Cloud Storage for photos
google.golang.org/api        // Google APIs

// Event Publishing
segmentio/kafka-go          // Kafka event publishing

// Testing & Quality
testcontainers/testcontainers-go  // E2E testing with real infrastructure
stretchr/testify                  // Assertions and test utilities
```

### 2. Photo-First Architecture with Google Cloud Storage

**Decision**: Require 1-10 photos per post with direct upload to Google Cloud Storage.

**Why**: Photos are more effective than text descriptions for Lost & Found:
- **Visual Recognition**: Photos enable immediate visual matching vs. ambiguous text
- **Language Barriers**: Photos transcend language differences in international environments
- **User Experience**: Mobile-first photo upload is faster than typing descriptions
- **AI Integration**: Photos enable future computer vision and matching algorithms

**Photo Upload Flow**:
```go
type PhotoUploadRequest struct {
    PostID      uuid.UUID `json:"post_id"`
    PhotoData   []byte    `json:"photo_data"`
    ContentType string    `json:"content_type"`
}

func (s *PhotoService) UploadPhoto(req PhotoUploadRequest) (*Photo, error) {
    // Validate image format and size
    if err := s.validateImage(req.PhotoData, req.ContentType); err != nil {
        return nil, err
    }

    // Generate unique filename with organization isolation
    filename := s.generateFilename(req.PostID, req.ContentType)

    // Upload to GCS with proper metadata
    objectHandle := s.gcsBucket.Object(filename)
    writer := objectHandle.NewWriter(context.Background())
    writer.Metadata = map[string]string{
        "post-id": req.PostID.String(),
        "content-type": req.ContentType,
        "uploaded-at": time.Now().UTC().Format(time.RFC3339),
    }

    if _, err := writer.Write(req.PhotoData); err != nil {
        return nil, fmt.Errorf("failed to upload to GCS: %w", err)
    }

    if err := writer.Close(); err != nil {
        return nil, fmt.Errorf("failed to finalize upload: %w", err)
    }

    // Generate public URL and thumbnail
    publicURL := fmt.Sprintf("https://storage.googleapis.com/%s/%s", s.bucketName, filename)
    thumbnailURL, err := s.generateThumbnail(publicURL)
    if err != nil {
        log.Printf("Failed to generate thumbnail: %v", err)
        thumbnailURL = publicURL // Fallback to original
    }

    // Create domain photo entity
    photo := domain.NewPhoto(req.PostID, publicURL, thumbnailURL, req.ContentType)

    // Publish photo uploaded event
    s.eventPublisher.Publish("photo.uploaded", PhotoUploadedEvent{
        PostID:       req.PostID,
        PhotoURL:     publicURL,
        ThumbnailURL: thumbnailURL,
        UploadedAt:   time.Now(),
    })

    return photo, nil
}
```

### 3. PostGIS for Geospatial Intelligence

**Decision**: Use PostgreSQL with PostGIS extension for location-based search and proximity calculation.

**Why**: Location is critical for Lost & Found item recovery:
- **Accurate Distance**: Haversine distance calculations for true geographic proximity
- **Performance**: Spatial indexes enable sub-second searches on millions of posts
- **Radius Search**: ST_DWithin function efficiently finds posts within custom radius
- **Proximity Scoring**: Distance-based ranking helps users prioritize nearby items

**Geospatial Schema Design**:
```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Posts table with spatial column
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(200) NOT NULL,
    description TEXT,
    location GEOMETRY(POINT, 4326),  -- WGS84 coordinate system
    radius_meters INTEGER DEFAULT 1000 CHECK (radius_meters >= 100 AND radius_meters <= 50000),
    status post_status DEFAULT 'active',
    type post_type NOT NULL,
    user_id UUID NOT NULL,
    organization_id UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Critical spatial index for performance
CREATE INDEX idx_posts_location_gist ON posts USING GIST (location);
```

**Optimized Spatial Queries**:
```go
func (r *PostgresPostRepository) FindByRadius(
    center Location,
    radiusMeters int,
    postType PostType,
) ([]*Post, error) {
    query := `
        SELECT
            id, title, description, type, status,
            ST_X(location::geometry) as lng,
            ST_Y(location::geometry) as lat,
            radius_meters, organization_id, user_id,
            created_at, updated_at,
            ST_Distance(location, ST_SetSRID(ST_MakePoint($1, $2), 4326)) as distance_meters
        FROM posts
        WHERE status = 'active'
          AND type != $3  -- Lost posts search Found posts and vice versa
          AND ST_DWithin(
              location,
              ST_SetSRID(ST_MakePoint($1, $2), 4326),
              $4
          )
        ORDER BY distance_meters ASC
        LIMIT 100`

    rows, err := r.db.Query(query, center.Lng, center.Lat, postType, radiusMeters)
    if err != nil {
        return nil, fmt.Errorf("spatial query failed: %w", err)
    }
    defer rows.Close()

    var posts []*Post
    for rows.Next() {
        post, err := r.scanPostWithDistance(rows)
        if err != nil {
            return nil, err
        }
        posts = append(posts, post)
    }

    return posts, nil
}
```

### 4. Event-Driven Integration Architecture

**Decision**: Publish domain events for all state changes using Kafka with Avro schemas.

**Why**: Posts are the central entity requiring ecosystem-wide integration:
- **Loose Coupling**: Other domains react to post events without direct dependencies
- **Scalability**: Asynchronous processing prevents blocking on external services
- **Auditability**: Complete event log provides business intelligence and debugging
- **Extensibility**: New domains can subscribe to existing events without code changes

**Event Publishing Pattern**:
```go
type PostEventPublisher struct {
    kafkaWriter *kafka.Writer
    translator  *OutboundEventTranslator
}

func (p *PostEventPublisher) PublishPostCreated(post *domain.Post) error {
    event := &PostCreatedEvent{
        PostID:       post.ID().String(),
        Title:        post.Title(),
        Description:  post.Description(),
        PostType:     string(post.PostType()),
        Location:     LocationDTO{
            Lat: post.Location().Lat,
            Lng: post.Location().Lng,
        },
        RadiusMeters: post.RadiusMeters(),
        UserID:       post.CreatedBy().String(),
        OrgID:        post.OrganizationID().String(),
        PhotoURLs:    p.extractPhotoURLs(post.Photos()),
        CreatedAt:    post.CreatedAt(),
        Timestamp:    time.Now().UTC(),
    }

    // Translate to external schema format
    externalEvent, err := p.translator.TranslatePostCreated(event)
    if err != nil {
        return fmt.Errorf("event translation failed: %w", err)
    }

    // Publish to Kafka topic
    message := kafka.Message{
        Key:   []byte(event.PostID),
        Value: externalEvent,
        Time:  event.Timestamp,
    }

    if err := p.kafkaWriter.WriteMessages(context.Background(), message); err != nil {
        return fmt.Errorf("kafka publish failed: %w", err)
    }

    log.Printf("Published post.created event: post_id=%s", event.PostID)
    return nil
}
```

**Published Events**:
```go
// Core post lifecycle events
type PostCreatedEvent struct {
    PostID       string      `json:"post_id"`
    Title        string      `json:"title"`
    Description  string      `json:"description"`
    PostType     string      `json:"post_type"`     // "lost" | "found"
    Location     LocationDTO `json:"location"`
    RadiusMeters int         `json:"radius_meters"`
    UserID       string      `json:"user_id"`
    OrgID        string      `json:"organization_id"`
    PhotoURLs    []string    `json:"photo_urls"`
    CreatedAt    time.Time   `json:"created_at"`
    Timestamp    time.Time   `json:"timestamp"`
}

type PostStatusUpdatedEvent struct {
    PostID       string    `json:"post_id"`
    OldStatus    string    `json:"old_status"`
    NewStatus    string    `json:"new_status"`
    UpdatedBy    string    `json:"updated_by"`
    Reason       string    `json:"reason,omitempty"`
    Timestamp    time.Time `json:"timestamp"`
}

type PostResolvedEvent struct {
    PostID       string    `json:"post_id"`
    Title        string    `json:"title"`
    PostType     string    `json:"post_type"`
    UserID       string    `json:"user_id"`
    ResolvedAt   time.Time `json:"resolved_at"`
    ResolutionType string  `json:"resolution_type"` // "found" | "claimed" | "expired"
    Timestamp    time.Time `json:"timestamp"`
}
```

### 5. Domain-Driven Design with Aggregate Boundaries

**Decision**: Implement strict DDD patterns with Post as aggregate root and explicit boundaries.

**Why**: Posts manage complex business rules requiring consistency:
- **Business Rules**: Photo count limits, status transitions, location validation
- **Consistency**: Aggregate boundaries ensure invariants are maintained
- **Testability**: Domain logic isolated from infrastructure concerns
- **Maintainability**: Clear separation between business logic and technical implementation

**Aggregate Root Design**:
```go
// Post aggregate root manages all business rules
type Post struct {
    id             PostID
    title          string
    description    string
    photos         []Photo          // Managed collection with business rules
    location       Location         // Value object with validation
    radiusMeters   int
    status         PostStatus       // Controlled state transitions
    postType       PostType
    createdBy      UserID
    organizationID *OrganizationID
    createdAt      time.Time
    updatedAt      time.Time
}

// Business rule: Photo count limits
func (p *Post) AddPhoto(photo Photo) error {
    if len(p.photos) >= 10 {
        return ErrTooManyPhotos
    }

    if len(p.photos) == 0 && p.status == PostStatusActive {
        return ErrCannotAddFirstPhotoToActivePost
    }

    p.photos = append(p.photos, photo)
    p.updatedAt = time.Now()
    return nil
}

// Business rule: Status transition validation
func (p *Post) UpdateStatus(newStatus PostStatus) error {
    if err := p.validateStatusTransition(newStatus); err != nil {
        return err
    }

    oldStatus := p.status
    p.status = newStatus
    p.updatedAt = time.Now()

    // Domain event for status changes
    return p.publishStatusUpdated(oldStatus, newStatus)
}

func (p *Post) validateStatusTransition(newStatus PostStatus) error {
    validTransitions := map[PostStatus][]PostStatus{
        PostStatusActive:   {PostStatusResolved, PostStatusExpired, PostStatusDeleted},
        PostStatusResolved: {PostStatusActive, PostStatusDeleted},
        PostStatusExpired:  {PostStatusActive, PostStatusDeleted},
        PostStatusDeleted:  {}, // Terminal state
    }

    allowedStatuses, exists := validTransitions[p.status]
    if !exists {
        return ErrInvalidStatus
    }

    for _, allowed := range allowedStatuses {
        if allowed == newStatus {
            return nil
        }
    }

    return ErrInvalidStatusTransition
}
```

### 6. Multi-Tenant Organization Isolation

**Decision**: Implement organization-based data isolation with optional tenant boundaries.

**Why**: Posts service must support enterprise deployments with data isolation:
- **Security**: Organizations cannot access each other's posts
- **Compliance**: Data residency and privacy requirements
- **Performance**: Query optimization through tenant-aware indexes
- **Scalability**: Horizontal partitioning by organization

**Multi-Tenancy Implementation**:
```go
type PostRepository interface {
    CreateWithOrg(post *Post, orgID OrganizationID) error
    FindByOrg(orgID OrganizationID, filters PostFilters) ([]*Post, error)
    FindByRadiusWithinOrg(location Location, radius int, orgID OrganizationID) ([]*Post, error)
}

// All queries are organization-scoped
func (r *PostgresPostRepository) FindByRadiusWithinOrg(
    location Location,
    radius int,
    orgID OrganizationID,
) ([]*Post, error) {
    query := `
        SELECT id, title, description, type, status,
               ST_X(location::geometry) as lng, ST_Y(location::geometry) as lat,
               radius_meters, organization_id, user_id, created_at, updated_at
        FROM posts
        WHERE organization_id = $1
          AND status = 'active'
          AND ST_DWithin(
              location,
              ST_SetSRID(ST_MakePoint($2, $3), 4326),
              $4
          )
        ORDER BY ST_Distance(location, ST_SetSRID(ST_MakePoint($2, $3), 4326))
        LIMIT 100`

    return r.queryPostsWithLocation(query, orgID.String(), location.Lng, location.Lat, radius)
}
```

## Domain Model

**Aggregate Root**: `Post`
- Manages photo collection with business rule enforcement (1-10 photos)
- Controls status transitions with validation (active → resolved/expired/deleted)
- Enforces location validation and radius constraints
- Publishes domain events for all state changes
- Maintains organization-based data isolation

**Entities**:
- `Photo` - Individual photo with GCS URL, thumbnail, and metadata
- `Location` - Geographic coordinates with validation and distance calculations

**Value Objects**:
- `PostID`, `UserID`, `OrganizationID`, `PhotoID` - Strongly-typed identifiers
- `PostStatus` - Enumerated status with transition validation
- `PostType` - Lost or Found classification
- `Coordinates` - Lat/lng pair with validation

**Domain Services**:
- `PhotoUploadService` - Manages GCS upload and thumbnail generation
- `GeospatialSearchService` - Optimized PostGIS query coordination
- `PostEventPublisher` - Domain event publishing with external translation
- `PostExpirationService` - Automatic status transitions based on business rules

## Performance Targets

- **Post Creation**: <15 seconds end-to-end including photo upload and event publishing
- **Geospatial Search**: <200ms for radius queries within 50km on 1M+ posts
- **Photo Upload**: <5 seconds for images up to 10MB
- **Event Publishing**: <100ms for Kafka message delivery

## Integration Points

**Publishes Events**:
- `post.created` → Notifications (confirmation), Media AI (photo analysis), Matcher (initial matching)
- `post.status_updated` → Notifications (status alerts), Matcher (availability changes)
- `post.resolved` → Notifications (success stories), Analytics (completion tracking)
- `photo.uploaded` → Media AI (computer vision analysis)

**External Dependencies**:
- **Google Cloud Storage**: Photo storage with global CDN distribution
- **PostgreSQL + PostGIS**: Spatial data storage and geospatial query processing
- **Confluent Cloud Kafka**: Event publishing with guaranteed delivery
- **Supabase**: Managed PostgreSQL with automatic backups and scaling

## API Surface

**Core Endpoints**:
```
POST   /api/posts                     # Create new post with photos
GET    /api/posts                     # List posts with filtering
GET    /api/posts/nearby              # Geospatial search within radius
GET    /api/posts/{id}                # Get specific post with photos
PUT    /api/posts/{id}                # Update post content
PATCH  /api/posts/{id}/status         # Update post status
DELETE /api/posts/{id}                # Delete post (soft delete)

POST   /api/posts/{id}/photos         # Upload additional photo
DELETE /api/posts/{id}/photos/{photo} # Remove photo from post

GET    /api/users/{user}/posts        # Get user's posts across organizations
```

**Request/Response Examples**:
```json
POST /api/posts
{
  "title": "Lost iPhone 15 Pro",
  "description": "Black iPhone with blue case, lost near Central Park fountain",
  "post_type": "lost",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "organization_id": "org-123",
  "photos": [
    {
      "data": "base64-encoded-image-data",
      "content_type": "image/jpeg"
    }
  ]
}

Response:
{
  "id": "post-456",
  "title": "Lost iPhone 15 Pro",
  "description": "Black iPhone with blue case, lost near Central Park fountain",
  "post_type": "lost",
  "status": "active",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "photos": [
    {
      "id": "photo-789",
      "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
      "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
      "content_type": "image/jpeg",
      "display_order": 1
    }
  ],
  "user_id": "user-123",
  "organization_id": "org-123",
  "created_at": "2024-01-15T14:30:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

## Business Rules Enforced

- **Photo Requirements**: Every post must have 1-10 photos; text-only posts are not allowed
- **Location Validation**: GPS coordinates must be valid and within reasonable bounds
- **Radius Constraints**: Search radius must be between 100m and 50km
- **Status Transitions**: Only valid state transitions allowed (active → resolved/expired/deleted)
- **Organization Isolation**: Posts are completely isolated by organization ID
- **User Ownership**: Only post creators or organization admins can modify posts
- **Content Validation**: Title required (1-200 chars), description optional (max 500 chars)

## Scalability Patterns

**Horizontal Scaling**:
- Stateless service design with shared PostgreSQL state
- Photo storage distributed across GCS regions for global performance
- Organization-based data partitioning for query optimization

**Performance Optimization**:
- **PostGIS Spatial Indexes**: Sub-second geospatial queries on millions of posts
- **Photo CDN**: Global content distribution via Google Cloud Storage
- **Database Connection Pooling**: Managed connection pools prevent resource exhaustion
- **Async Event Publishing**: Non-blocking Kafka publishing maintains response times

**Caching Strategy**:
- **Photo URLs**: CDN caching with 30-day TTL for static content
- **Frequent Searches**: Redis caching for popular geospatial queries
- **User Sessions**: JWT tokens minimize database lookups for authentication

## Error Handling

**Domain-Specific Errors**:
```go
var (
    ErrPostNotFound      = errors.New("post not found")
    ErrTooManyPhotos     = errors.New("maximum 10 photos allowed per post")
    ErrNoPhotosProvided  = errors.New("at least 1 photo required")
    ErrInvalidLocation   = errors.New("invalid GPS coordinates")
    ErrInvalidRadius     = errors.New("radius must be between 100m and 50km")
    ErrUnauthorizedAccess = errors.New("user not authorized for this organization")
    ErrInvalidStatusTransition = errors.New("invalid post status transition")
)
```

**HTTP Error Mapping**:
- `400 Bad Request`: Validation errors, invalid input
- `401 Unauthorized`: Missing or invalid authentication
- `403 Forbidden`: User lacks organization permissions
- `404 Not Found`: Post or photo not found
- `409 Conflict`: Status transition conflicts
- `413 Payload Too Large`: Photo size exceeds limits
- `500 Internal Server Error`: Infrastructure failures

---

*This domain focuses purely on photo-first lost & found post management with geospatial intelligence. For system-wide architecture decisions, see [../ARCHITECTURE.md](../ARCHITECTURE.md). For detailed API specifications, see [api-documentation.md](api-documentation.md).*