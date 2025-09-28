# fn-contract API Documentation

**Schema management APIs and tools for contract governance across the Findly Now microservices ecosystem.**

## Base Configuration

**Repository URL**: `https://github.com/findly-now/fn-contract`

**Schema Registry URL**: `https://schema-registry.findlynow.com` (production)

**API Documentation**: `https://contracts.findlynow.com/docs` (generated)

**CLI Tool**: `@findly/contract-validator`

## Service Architecture

fn-contract is primarily a **schema governance repository** with supporting APIs and tools for validation, compatibility checking, and documentation generation. It operates through:

1. **Git Repository**: Source of truth for all schema definitions
2. **CI/CD Pipeline**: Automated validation and publishing
3. **Schema Registry**: Runtime schema storage and validation
4. **CLI Tools**: Developer utilities for validation and testing
5. **Documentation Portal**: Generated API and event documentation

## Repository Structure

### Schema Organization

```
fn-contract/
├── api/                          # OpenAPI specifications
│   ├── posts/
│   │   ├── v1.yaml              # fn-posts API v1
│   │   ├── v1.1.yaml            # fn-posts API v1.1
│   │   └── v2.yaml              # fn-posts API v2 (draft)
│   ├── matcher/
│   │   └── v1.yaml              # fn-matcher API
│   ├── media-ai/
│   │   └── v1.yaml              # fn-media-ai API (internal)
│   └── notifications/
│       └── v1.yaml              # fn-notifications API
├── events/                       # AsyncAPI specifications
│   ├── posts-events/
│   │   ├── v1.yaml              # Post lifecycle events
│   │   └── v1.1.yaml            # Enhanced post events
│   ├── match-events/
│   │   └── v1.yaml              # Matching events
│   └── ai-events/
│       └── v1.yaml              # AI processing events
├── avro/                        # Avro schemas for Kafka
│   ├── posts/
│   │   ├── post-created.avsc
│   │   ├── post-enhanced.avsc
│   │   └── post-status-updated.avsc
│   ├── matcher/
│   │   ├── match-detected.avsc
│   │   └── match-claimed.avsc
│   └── media-ai/
│       ├── photo-analyzed.avsc
│       └── processing-error.avsc
├── docs/                        # Generated documentation
├── tools/                       # Validation and publishing tools
├── tests/                       # Contract tests
└── scripts/                     # CI/CD and utility scripts
```

## CLI Tools and APIs

### Contract Validator CLI

**Installation**:
```bash
# Install globally
npm install -g @findly/contract-validator

# Or use with npx
npx @findly/contract-validator --help
```

#### Schema Validation

**Validate Single Schema**:
```bash
# Validate OpenAPI specification
contract-validator validate api/posts/v1.yaml

# Validate AsyncAPI specification
contract-validator validate events/posts-events/v1.yaml

# Validate Avro schema
contract-validator validate avro/posts/post-created.avsc

# Validate all schemas
contract-validator validate-all
```

**Response Format**:
```json
{
  "validation_result": "success",
  "schema_file": "api/posts/v1.yaml",
  "schema_type": "openapi",
  "version": "1.0.0",
  "errors": [],
  "warnings": [
    {
      "rule": "operation-description",
      "path": "paths./api/posts.post",
      "message": "Operation should have a description",
      "severity": "warning"
    }
  ],
  "validation_time_ms": 245
}
```

#### Compatibility Checking

**Compare Schema Versions**:
```bash
# Check backward compatibility
contract-validator compatibility \
  --old api/posts/v1.yaml \
  --new api/posts/v1.1.yaml \
  --type backward

# Check all compatibility types
contract-validator compatibility \
  --old api/posts/v1.yaml \
  --new api/posts/v2.yaml \
  --type full
```

**Compatibility Response**:
```json
{
  "compatibility_check": "backward",
  "result": "breaking_changes_detected",
  "old_version": "1.0.0",
  "new_version": "1.1.0",
  "breaking_changes": [
    {
      "type": "field_removed",
      "path": "/components/schemas/Post/properties/deprecated_field",
      "description": "Required field 'deprecated_field' was removed",
      "severity": "major",
      "impact": "Existing consumers will fail when accessing this field"
    }
  ],
  "compatible_changes": [
    {
      "type": "field_added",
      "path": "/components/schemas/Post/properties/new_optional_field",
      "description": "Optional field 'new_optional_field' was added",
      "severity": "minor"
    }
  ],
  "recommendations": [
    "Add deprecation notice before removing fields",
    "Provide migration guide for affected consumers"
  ]
}
```

#### Impact Analysis

**Analyze Consumer Impact**:
```bash
# Analyze impact of schema changes
contract-validator impact \
  --schema api/posts/v1.1.yaml \
  --consumers fn-notifications,fn-matcher

# Generate impact report
contract-validator impact \
  --schema api/posts/v1.1.yaml \
  --output impact-report.json
```

**Impact Analysis Response**:
```json
{
  "impact_analysis": {
    "schema": "api/posts/v1.1.yaml",
    "version": "1.1.0",
    "affected_consumers": [
      {
        "service": "fn-notifications",
        "endpoints_used": [
          "/api/posts/{id}",
          "/api/posts/search"
        ],
        "impact_level": "high",
        "breaking_changes": [
          {
            "endpoint": "/api/posts/{id}",
            "field": "deprecated_field",
            "impact": "Consumer code will fail accessing removed field"
          }
        ],
        "migration_required": true,
        "estimated_effort": "2-4 hours"
      },
      {
        "service": "fn-matcher",
        "endpoints_used": [
          "/api/posts/nearby"
        ],
        "impact_level": "none",
        "breaking_changes": [],
        "migration_required": false
      }
    ],
    "recommendations": [
      "Update fn-notifications to handle removed field",
      "Deploy fn-notifications before deploying fn-posts v1.1.0",
      "Consider gradual rollout with feature flags"
    ]
  }
}
```

### Schema Registry API

#### Schema Management Endpoints

**Get Schema by Subject**:
```bash
curl -X GET \
  https://schema-registry.findlynow.com/subjects/posts.created/versions/latest \
  -H "Authorization: Bearer $SCHEMA_REGISTRY_TOKEN"
```

**Response**:
```json
{
  "subject": "posts.created",
  "version": 3,
  "id": 12345,
  "schema": "{\"type\":\"record\",\"name\":\"PostCreatedEvent\"...}",
  "schemaType": "AVRO",
  "references": []
}
```

**Register New Schema Version**:
```bash
curl -X POST \
  https://schema-registry.findlynow.com/subjects/posts.created/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -H "Authorization: Bearer $SCHEMA_REGISTRY_TOKEN" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"PostCreatedEvent\",\"fields\":[...]}"
  }'
```

**Response**:
```json
{
  "id": 12346
}
```

**Check Compatibility**:
```bash
curl -X POST \
  https://schema-registry.findlynow.com/compatibility/subjects/posts.created/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -H "Authorization: Bearer $SCHEMA_REGISTRY_TOKEN" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"PostCreatedEvent\",\"fields\":[...]}"
  }'
```

**Response**:
```json
{
  "is_compatible": true
}
```

#### Configuration Management

**Get Compatibility Level**:
```bash
curl -X GET \
  https://schema-registry.findlynow.com/config/posts.created \
  -H "Authorization: Bearer $SCHEMA_REGISTRY_TOKEN"
```

**Response**:
```json
{
  "compatibilityLevel": "BACKWARD"
}
```

**Update Compatibility Level**:
```bash
curl -X PUT \
  https://schema-registry.findlynow.com/config/posts.created \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -H "Authorization: Bearer $SCHEMA_REGISTRY_TOKEN" \
  -d '{
    "compatibility": "BACKWARD_TRANSITIVE"
  }'
```

## OpenAPI Contract Examples

### Posts API Contract

**Base Specification**:
```yaml
openapi: 3.1.0
info:
  title: fn-posts API
  version: 1.1.0
  description: Photo-first lost & found post management API
  contact:
    name: Findly Now API Team
    url: https://findlynow.com/support
    email: api-support@findlynow.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.findlynow.com/posts
    description: Production server
  - url: https://staging-api.findlynow.com/posts
    description: Staging server

security:
  - bearerAuth: []

paths:
  /api/posts:
    post:
      operationId: createPost
      summary: Create a new lost or found post
      description: Creates a new post with photos and location data
      tags:
        - Posts
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePostRequest'
            examples:
              lost_phone:
                summary: Lost phone example
                value:
                  title: "Lost iPhone 15 Pro"
                  description: "Black iPhone with blue case"
                  post_type: "lost"
                  location:
                    lat: 40.7831
                    lng: -73.9665
                    address: "Central Park, New York, NY"
                  photos: [
                    {
                      data: "base64-encoded-image...",
                      content_type: "image/jpeg"
                    }
                  ]
      responses:
        '201':
          description: Post created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '422':
          $ref: '#/components/responses/ValidationError'

  /api/posts/{id}:
    get:
      operationId: getPost
      summary: Get post by ID
      description: Retrieves a specific post with all details
      tags:
        - Posts
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
          description: Unique identifier for the post
      responses:
        '200':
          description: Post retrieved successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    CreatePostRequest:
      type: object
      required: [title, post_type, location, photos]
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 200
          description: Brief title describing the lost/found item
          example: "Lost iPhone 15 Pro"
        description:
          type: string
          maxLength: 500
          description: Detailed description of the item
          example: "Black iPhone with blue MagSafe case, lost near fountain"
        post_type:
          type: string
          enum: [lost, found]
          description: Type of post - whether item is lost or found
        location:
          $ref: '#/components/schemas/Location'
        radius_meters:
          type: integer
          minimum: 100
          maximum: 50000
          default: 1000
          description: Search radius in meters
        organization_id:
          type: string
          format: uuid
          description: Organization ID for multi-tenant isolation
        photos:
          type: array
          minItems: 1
          maxItems: 10
          items:
            $ref: '#/components/schemas/PhotoUpload'

    PostResponse:
      type: object
      required: [id, title, post_type, status, location, photos, user_id, created_at]
      properties:
        id:
          type: string
          format: uuid
          description: Unique identifier for the post
        title:
          type: string
          description: Post title
        description:
          type: string
          description: Post description
        post_type:
          type: string
          enum: [lost, found]
        status:
          type: string
          enum: [active, resolved, expired, deleted]
          description: Current status of the post
        location:
          $ref: '#/components/schemas/Location'
        radius_meters:
          type: integer
          description: Search radius in meters
        photos:
          type: array
          items:
            $ref: '#/components/schemas/PhotoMetadata'
        user_id:
          type: string
          format: uuid
          description: ID of the user who created the post
        organization_id:
          type: string
          format: uuid
          description: Organization ID for multi-tenant isolation
        created_at:
          type: string
          format: date-time
          description: Timestamp when post was created
        updated_at:
          type: string
          format: date-time
          description: Timestamp when post was last updated

    Location:
      type: object
      required: [lat, lng]
      properties:
        lat:
          type: number
          format: double
          minimum: -90
          maximum: 90
          description: Latitude coordinate
        lng:
          type: number
          format: double
          minimum: -180
          maximum: 180
          description: Longitude coordinate
        address:
          type: string
          maxLength: 200
          description: Human-readable address
          example: "Central Park, New York, NY"

  responses:
    BadRequest:
      description: Invalid request data
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    ValidationError:
      description: Request validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationErrorResponse'

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

## AsyncAPI Event Contracts

### Post Events Specification

**Event Documentation**:
```yaml
asyncapi: 2.6.0
info:
  title: Posts Domain Events
  version: 1.1.0
  description: Event schema for post lifecycle management in the Findly Now ecosystem
  contact:
    name: Findly Now Events Team
    email: events@findlynow.com
  license:
    name: MIT

servers:
  production:
    url: pkc-lzvrd.us-west4.gcp.confluent.cloud:9092
    protocol: kafka-secure
    description: Production Kafka cluster
    security:
      - saslScram: []

  staging:
    url: staging-kafka.findlynow.com:9092
    protocol: kafka
    description: Staging Kafka cluster

channels:
  'fn-posts.post.created':
    description: Published when a new lost or found post is created
    publish:
      operationId: publishPostCreated
      summary: Publish post created event
      message:
        $ref: '#/components/messages/PostCreated'

  'fn-posts.post.status_updated':
    description: Published when a post status changes
    publish:
      operationId: publishPostStatusUpdated
      summary: Publish post status update event
      message:
        $ref: '#/components/messages/PostStatusUpdated'

  'fn-media-ai.post.enhanced':
    description: Consumed when AI enhancement completes
    subscribe:
      operationId: handlePostEnhanced
      summary: Handle post enhancement completion
      message:
        $ref: '#/components/messages/PostEnhanced'

components:
  securitySchemes:
    saslScram:
      type: scramSha256
      description: SASL/SCRAM authentication for Kafka

  messages:
    PostCreated:
      name: PostCreated
      title: Post Created Event
      summary: A new lost or found post has been created
      contentType: application/json
      tags:
        - name: post-lifecycle
        - name: high-priority
      payload:
        $ref: '#/components/schemas/PostCreatedPayload'
      examples:
        - name: lost_phone_created
          summary: Lost phone post creation
          payload:
            event_id: "evt-123e4567-e89b-12d3-a456-426614174000"
            event_type: "post.created"
            timestamp: "2024-01-15T14:30:00Z"
            version: "1.1"
            data:
              post_id: "post-456e7890-e12b-34c5-a678-901234567890"
              user_id: "user-123a4567-b89c-12d3-e456-789012345678"
              title: "Lost iPhone 15 Pro"
              post_type: "lost"
              location:
                lat: 40.7831
                lng: -73.9665
                address: "Central Park, New York, NY"

    PostStatusUpdated:
      name: PostStatusUpdated
      title: Post Status Updated Event
      summary: A post status has been changed
      contentType: application/json
      payload:
        $ref: '#/components/schemas/PostStatusUpdatedPayload'

  schemas:
    PostCreatedPayload:
      type: object
      required: [event_id, event_type, timestamp, version, data]
      properties:
        event_id:
          type: string
          format: uuid
          description: Unique identifier for this event
        event_type:
          type: string
          const: "post.created"
          description: Type of event
        timestamp:
          type: string
          format: date-time
          description: When the event occurred
        version:
          type: string
          pattern: '^\d+\.\d+$'
          description: Event schema version
        correlation_id:
          type: string
          format: uuid
          description: Optional correlation ID for request tracking
        data:
          type: object
          required: [post_id, user_id, title, post_type, location, photos]
          properties:
            post_id:
              type: string
              format: uuid
              description: Unique identifier for the post
            user_id:
              type: string
              format: uuid
              description: ID of the user who created the post
            organization_id:
              type: string
              format: uuid
              description: Organization ID for multi-tenant isolation
            title:
              type: string
              description: Post title
            description:
              type: string
              description: Post description
            post_type:
              type: string
              enum: [lost, found]
              description: Type of post
            location:
              $ref: '#/components/schemas/Location'
            radius_meters:
              type: integer
              minimum: 100
              maximum: 50000
              description: Search radius in meters
            photos:
              type: array
              minItems: 1
              maxItems: 10
              items:
                $ref: '#/components/schemas/PhotoMetadata'
            created_at:
              type: string
              format: date-time
              description: When the post was created
```

## Avro Schema Definitions

### Post Created Event Schema

**Binary Event Schema**:
```json
{
  "type": "record",
  "name": "PostCreatedEvent",
  "namespace": "com.findlynow.posts.events.v1",
  "doc": "Event published when a new lost or found post is created",
  "fields": [
    {
      "name": "event_id",
      "type": "string",
      "doc": "Unique identifier for this event instance"
    },
    {
      "name": "event_type",
      "type": "string",
      "default": "post.created",
      "doc": "Type of event, always 'post.created' for this schema"
    },
    {
      "name": "timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "Timestamp when the event occurred (milliseconds since epoch)"
    },
    {
      "name": "version",
      "type": "string",
      "default": "1.1",
      "doc": "Schema version for this event"
    },
    {
      "name": "correlation_id",
      "type": ["null", "string"],
      "default": null,
      "doc": "Optional correlation ID for request tracking"
    },
    {
      "name": "data",
      "type": {
        "type": "record",
        "name": "PostCreatedData",
        "fields": [
          {
            "name": "post_id",
            "type": "string",
            "doc": "Unique identifier for the post"
          },
          {
            "name": "user_id",
            "type": "string",
            "doc": "ID of the user who created the post"
          },
          {
            "name": "organization_id",
            "type": ["null", "string"],
            "default": null,
            "doc": "Organization ID for multi-tenant isolation"
          },
          {
            "name": "title",
            "type": "string",
            "doc": "Post title describing the lost/found item"
          },
          {
            "name": "description",
            "type": ["null", "string"],
            "default": null,
            "doc": "Optional detailed description of the item"
          },
          {
            "name": "post_type",
            "type": {
              "type": "enum",
              "name": "PostType",
              "symbols": ["LOST", "FOUND"],
              "doc": "Type of post - whether item is lost or found"
            }
          },
          {
            "name": "location",
            "type": {
              "type": "record",
              "name": "Location",
              "fields": [
                {
                  "name": "lat",
                  "type": "double",
                  "doc": "Latitude coordinate"
                },
                {
                  "name": "lng",
                  "type": "double",
                  "doc": "Longitude coordinate"
                },
                {
                  "name": "address",
                  "type": ["null", "string"],
                  "default": null,
                  "doc": "Human-readable address"
                }
              ]
            },
            "doc": "Geographic location where item was lost/found"
          },
          {
            "name": "radius_meters",
            "type": "int",
            "default": 1000,
            "doc": "Search radius in meters"
          },
          {
            "name": "photos",
            "type": {
              "type": "array",
              "items": {
                "type": "record",
                "name": "PhotoMetadata",
                "fields": [
                  {
                    "name": "photo_id",
                    "type": "string",
                    "doc": "Unique identifier for the photo"
                  },
                  {
                    "name": "url",
                    "type": "string",
                    "doc": "Public URL to access the photo"
                  },
                  {
                    "name": "thumbnail_url",
                    "type": "string",
                    "doc": "URL to thumbnail version of the photo"
                  },
                  {
                    "name": "content_type",
                    "type": "string",
                    "doc": "MIME type of the photo (image/jpeg, image/png, etc.)"
                  },
                  {
                    "name": "size_bytes",
                    "type": "long",
                    "doc": "Size of the photo file in bytes"
                  },
                  {
                    "name": "display_order",
                    "type": "int",
                    "doc": "Order for displaying multiple photos"
                  },
                  {
                    "name": "created_at",
                    "type": {
                      "type": "long",
                      "logicalType": "timestamp-millis"
                    },
                    "doc": "When the photo was uploaded"
                  }
                ]
              }
            },
            "doc": "Array of photo metadata for the post"
          },
          {
            "name": "created_at",
            "type": {
              "type": "long",
              "logicalType": "timestamp-millis"
            },
            "doc": "Timestamp when the post was created"
          }
        ]
      },
      "doc": "Post data payload"
    }
  ]
}
```

## Contract Testing Framework

### Consumer Contract Tests

**Example Contract Test (JavaScript/Pact)**:
```javascript
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { PostsApiClient } = require('../src/clients/posts-api');

describe('Posts API Consumer Contract', () => {
  const provider = new PactV3({
    consumer: 'fn-notifications',
    provider: 'fn-posts',
    port: 8080,
  });

  describe('when requesting post details', () => {
    it('should return post with required fields', async () => {
      // Arrange
      provider
        .given('a post exists with ID 123')
        .uponReceiving('a request for post details')
        .withRequest({
          method: 'GET',
          path: '/api/posts/123',
          headers: {
            'Authorization': MatchersV3.like('Bearer token123'),
            'Accept': 'application/json',
          },
        })
        .willRespondWith({
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: MatchersV3.like('123'),
            title: MatchersV3.like('Lost iPhone'),
            post_type: MatchersV3.term({
              matcher: '^(lost|found)$',
              generate: 'lost',
            }),
            status: MatchersV3.term({
              matcher: '^(active|resolved|expired|deleted)$',
              generate: 'active',
            }),
            location: {
              lat: MatchersV3.decimal(40.7831),
              lng: MatchersV3.decimal(-73.9665),
              address: MatchersV3.like('Central Park, New York, NY'),
            },
            photos: MatchersV3.eachLike({
              photo_id: MatchersV3.like('photo-123'),
              url: MatchersV3.like('https://storage.googleapis.com/photos/photo-123.jpg'),
              thumbnail_url: MatchersV3.like('https://storage.googleapis.com/photos/thumb-123.jpg'),
              content_type: MatchersV3.term({
                matcher: '^image\\/(jpeg|png|webp)$',
                generate: 'image/jpeg',
              }),
            }),
            user_id: MatchersV3.like('user-456'),
            created_at: MatchersV3.iso8601DateTime(),
            updated_at: MatchersV3.iso8601DateTime(),
          },
        });

      // Act
      const postsClient = new PostsApiClient('http://localhost:8080');
      const result = await postsClient.getPost('123');

      // Assert
      expect(result.id).toBe('123');
      expect(result.title).toBeDefined();
      expect(['lost', 'found']).toContain(result.post_type);
      expect(['active', 'resolved', 'expired', 'deleted']).toContain(result.status);
      expect(result.photos).toBeInstanceOf(Array);
      expect(result.photos.length).toBeGreaterThan(0);
    });
  });

  describe('when creating a new post', () => {
    it('should accept valid post data and return created post', async () => {
      // Arrange
      const newPost = {
        title: 'Lost iPhone 15 Pro',
        description: 'Black iPhone with blue case',
        post_type: 'lost',
        location: {
          lat: 40.7831,
          lng: -73.9665,
          address: 'Central Park, New York, NY',
        },
        photos: [
          {
            data: 'base64-encoded-image-data',
            content_type: 'image/jpeg',
          },
        ],
      };

      provider
        .given('user is authenticated')
        .uponReceiving('a request to create a new post')
        .withRequest({
          method: 'POST',
          path: '/api/posts',
          headers: {
            'Authorization': MatchersV3.like('Bearer token123'),
            'Content-Type': 'application/json',
          },
          body: MatchersV3.like(newPost),
        })
        .willRespondWith({
          status: 201,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: MatchersV3.uuid(),
            title: MatchersV3.like(newPost.title),
            post_type: MatchersV3.like(newPost.post_type),
            status: 'active',
            location: MatchersV3.like(newPost.location),
            photos: MatchersV3.eachLike({
              photo_id: MatchersV3.uuid(),
              url: MatchersV3.like('https://storage.googleapis.com/photos/photo.jpg'),
              thumbnail_url: MatchersV3.like('https://storage.googleapis.com/photos/thumb.jpg'),
              content_type: 'image/jpeg',
            }),
            user_id: MatchersV3.uuid(),
            created_at: MatchersV3.iso8601DateTime(),
            updated_at: MatchersV3.iso8601DateTime(),
          },
        });

      // Act
      const postsClient = new PostsApiClient('http://localhost:8080');
      const result = await postsClient.createPost(newPost);

      // Assert
      expect(result.id).toBeDefined();
      expect(result.title).toBe(newPost.title);
      expect(result.post_type).toBe(newPost.post_type);
      expect(result.status).toBe('active');
    });
  });
});
```

### Provider Contract Verification

**Example Provider Test (Go)**:
```go
package contracts

import (
    "testing"
    "net/http/httptest"
    "github.com/pact-foundation/pact-go/dsl"
    "github.com/pact-foundation/pact-go/types"
    "github.com/your-org/fn-posts/internal/handlers"
)

func TestPostsProviderContract(t *testing.T) {
    // Setup
    pact := &dsl.Pact{
        Provider: "fn-posts",
        LogDir:   "../../logs",
        PactDir:  "../../pacts",
    }
    defer pact.Teardown()

    // Provider states
    pact.AddProviderState("a post exists with ID 123", func() error {
        // Setup test data - create a post with ID 123
        return setupTestPost("123", "Lost iPhone", "lost", "active")
    })

    pact.AddProviderState("user is authenticated", func() error {
        // Setup authentication - create valid JWT token
        return setupAuthToken("Bearer valid-token")
    })

    // Start provider server
    server := httptest.NewServer(handlers.NewPostsHandler())
    defer server.Close()

    // Verify contracts
    _, err := pact.VerifyProvider(t, types.VerifyRequest{
        ProviderBaseURL:        server.URL,
        PactURLs:               []string{"../../pacts/fn-notifications-fn-posts.json"},
        ProviderVersion:        "1.0.0",
        BrokerURL:              "https://pact-broker.findlynow.com",
        BrokerUsername:         "pact-user",
        BrokerPassword:         "pact-password",
        PublishVerificationResults: true,
        ProviderStatesSetupURL:     server.URL + "/_pact/provider-states",
    })

    if err != nil {
        t.Fatalf("Provider contract verification failed: %v", err)
    }
}

func setupTestPost(id, title, postType, status string) error {
    // Implementation to create test post in database
    testPost := &models.Post{
        ID:        id,
        Title:     title,
        PostType:  postType,
        Status:    status,
        Location:  models.Location{Lat: 40.7831, Lng: -73.9665},
        Photos:    []models.Photo{},
        UserID:    "user-456",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    return testDatabase.CreatePost(testPost)
}
```

## CI/CD Integration

### GitHub Actions Workflow

**Contract Validation Pipeline**:
```yaml
name: Contract Validation

on:
  pull_request:
    paths:
      - 'api/**'
      - 'events/**'
      - 'avro/**'

jobs:
  validate-contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm install

      - name: Validate OpenAPI schemas
        run: |
          for file in api/**/*.yaml; do
            echo "Validating $file..."
            npx @apidevtools/swagger-parser validate "$file"
          done

      - name: Validate AsyncAPI schemas
        run: |
          for file in events/**/*.yaml; do
            echo "Validating $file..."
            npx @asyncapi/cli validate "$file"
          done

      - name: Validate Avro schemas
        run: |
          for file in avro/**/*.avsc; do
            echo "Validating $file..."
            java -jar avro-tools.jar compile schema "$file" /tmp/compiled
          done

      - name: Check compatibility
        run: |
          # Compare with previous version
          git fetch origin main
          npx @findly/contract-validator compatibility \
            --old origin/main \
            --new HEAD \
            --output compatibility-report.json

      - name: Upload compatibility report
        uses: actions/upload-artifact@v3
        with:
          name: compatibility-report
          path: compatibility-report.json

  publish-schemas:
    needs: validate-contracts
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Publish to Schema Registry
        env:
          SCHEMA_REGISTRY_URL: ${{ secrets.SCHEMA_REGISTRY_URL }}
          SCHEMA_REGISTRY_AUTH: ${{ secrets.SCHEMA_REGISTRY_AUTH }}
        run: |
          # Publish Avro schemas to Confluent Schema Registry
          for file in avro/**/*.avsc; do
            subject=$(basename "$file" .avsc)
            echo "Publishing $subject..."
            curl -X POST \
              -H "Content-Type: application/vnd.schemaregistry.v1+json" \
              -H "Authorization: Basic $SCHEMA_REGISTRY_AUTH" \
              --data "{\"schema\":\"$(cat $file | jq -c . | sed 's/"/\\"/g')\"}" \
              "$SCHEMA_REGISTRY_URL/subjects/$subject/versions"
          done

      - name: Generate documentation
        run: |
          # Generate API documentation from OpenAPI specs
          npx redoc-cli build api/posts/v1.yaml --output docs/posts-api.html
          npx redoc-cli build api/matcher/v1.yaml --output docs/matcher-api.html

          # Generate event catalog from AsyncAPI specs
          npx @asyncapi/generator events/ @asyncapi/html-template \
            --output docs/events/ --force-write

      - name: Deploy documentation
        run: |
          # Deploy to documentation site
          aws s3 sync docs/ s3://findly-contracts-docs/ --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

## Error Handling and Validation

### Schema Validation Errors

**Common Validation Error Types**:
```json
{
  "validation_errors": [
    {
      "error_type": "syntax_error",
      "file": "api/posts/v1.yaml",
      "line": 45,
      "column": 12,
      "message": "Invalid YAML syntax: unexpected character",
      "severity": "error"
    },
    {
      "error_type": "schema_violation",
      "file": "api/posts/v1.yaml",
      "path": "paths./api/posts.post.requestBody",
      "message": "requestBody is required for POST operations",
      "rule": "operation-requestBody",
      "severity": "error"
    },
    {
      "error_type": "compatibility_issue",
      "file": "api/posts/v1.1.yaml",
      "path": "components.schemas.Post.properties.required_field",
      "message": "Removing required field breaks backward compatibility",
      "severity": "error",
      "impact": "breaking_change"
    },
    {
      "error_type": "style_violation",
      "file": "api/posts/v1.yaml",
      "path": "components.schemas.PostResponse.properties.user_id",
      "message": "Property names should use camelCase",
      "rule": "property-naming-convention",
      "severity": "warning"
    }
  ]
}
```

### Contract Test Failures

**Contract Test Error Response**:
```json
{
  "contract_test_result": "failed",
  "provider": "fn-posts",
  "consumer": "fn-notifications",
  "test_suite": "posts-api-contract",
  "failures": [
    {
      "interaction": "get post by ID",
      "expected": {
        "status": 200,
        "body": {
          "id": "123",
          "title": "Lost iPhone",
          "status": "active"
        }
      },
      "actual": {
        "status": 200,
        "body": {
          "id": "123",
          "title": "Lost iPhone",
          "state": "active"
        }
      },
      "error": "Expected field 'status' but received 'state'",
      "suggestion": "Update consumer to use 'state' field or provider to use 'status'"
    }
  ],
  "summary": {
    "total_interactions": 5,
    "passed": 4,
    "failed": 1,
    "success_rate": 0.8
  }
}
```

## Performance and Monitoring

### Schema Registry Performance

**Performance Metrics**:
- **Schema Retrieval Latency**: <10ms (cached), <50ms (registry)
- **Validation Throughput**: 10,000+ validations/second
- **Schema Storage**: 99.9% availability
- **Cache Hit Rate**: >95% for active schemas

**Monitoring Dashboards**:
```yaml
# Grafana dashboard configuration
dashboard:
  title: "Contract Management Metrics"
  panels:
    - title: "Schema Validation Success Rate"
      type: "stat"
      targets:
        - expr: "rate(schema_validations_total{result='success'}[5m]) / rate(schema_validations_total[5m])"

    - title: "Contract Test Success Rate"
      type: "graph"
      targets:
        - expr: "rate(contract_tests_total{result='passed'}[5m])"
        - expr: "rate(contract_tests_total{result='failed'}[5m])"

    - title: "Schema Registry Response Time"
      type: "histogram"
      targets:
        - expr: "histogram_quantile(0.95, schema_registry_request_duration_seconds_bucket)"

    - title: "Breaking Changes Detected"
      type: "table"
      targets:
        - expr: "increase(breaking_changes_detected_total[24h])"
```

---

*This API documentation provides comprehensive guidance for managing contracts and schemas across the Findly Now microservices ecosystem. For architectural details, see [domain-architecture.md](domain-architecture.md). For deployment instructions, see [deployment-guide.md](deployment-guide.md).*