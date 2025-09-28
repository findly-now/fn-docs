# Contract Domain Architecture

**The schema governance and contract management system that ensures API and event compatibility across the Findly Now microservices ecosystem.**

## Domain Purpose

Transform service integration from ad-hoc, breaking changes to **centralized, versioned contract governance** with automated compatibility validation and schema evolution management.

**Core Value**: Centralized contract management prevents breaking changes, enables safe service evolution, and provides a single source of truth for all API and event schemas across the microservices ecosystem.

## Domain Boundaries

**Bounded Context**: Schema governance and contract management
- OpenAPI specification management for REST API contracts
- AsyncAPI schema definition for event-driven communication
- Avro schema registry integration for Kafka event serialization
- Contract versioning and backward compatibility validation
- Cross-service dependency mapping and impact analysis
- Automated schema validation in CI/CD pipelines

## Key Architecture Decisions

### 1. Contract-First Development Approach

**Decision**: Require all API and event schema changes to be defined in fn-contract before implementation.

**Why**: Contract-first development prevents integration issues and breaking changes:
- **Prevention**: Schema changes are validated before implementation
- **Coordination**: Teams coordinate through shared contract definitions
- **Documentation**: APIs are documented before they're built
- **Testing**: Contract tests can be written before implementation

**Contract Types Managed**:
```
fn-contract/
├── api/                    # OpenAPI specifications
│   ├── posts.yaml         # fn-posts REST API
│   ├── matcher.yaml       # fn-matcher REST API
│   ├── media-ai.yaml      # fn-media-ai REST API
│   └── notifications.yaml # fn-notifications REST API
├── events/                # AsyncAPI event schemas
│   ├── posts-events.yaml  # Post lifecycle events
│   ├── match-events.yaml  # Matching events
│   └── ai-events.yaml     # AI processing events
└── avro/                  # Avro schemas for Kafka
    ├── post-created.avsc
    ├── post-enhanced.avsc
    └── match-detected.avsc
```

### 2. Multi-Format Schema Support

**Decision**: Support OpenAPI, AsyncAPI, and Avro schemas for different integration patterns.

**Why**: Different communication patterns require different schema formats:
- **OpenAPI**: RESTful API contracts with HTTP semantics
- **AsyncAPI**: Event-driven communication documentation
- **Avro**: Efficient binary serialization for high-throughput events

**Schema Format Mapping**:
| Communication Type | Schema Format | Purpose |
|-------------------|---------------|----------|
| REST APIs | OpenAPI 3.1 | HTTP endpoint contracts |
| Event Documentation | AsyncAPI 2.6 | Event flow documentation |
| Kafka Serialization | Avro | Binary event serialization |
| GraphQL APIs | GraphQL SDL | Query contract definitions |

### 3. Confluent Schema Registry Integration

**Decision**: Use Confluent Schema Registry for runtime schema validation and evolution.

**Why**: Schema Registry provides production-grade schema management:
- **Runtime Validation**: Ensures producers and consumers use compatible schemas
- **Schema Evolution**: Manages forward/backward compatibility automatically
- **Performance**: Cached schemas reduce serialization overhead
- **Governance**: Centralized control over schema changes

**Schema Registry Architecture**:
```yaml
# Schema Registry Configuration
compatibility_level: BACKWARD  # Default compatibility mode
schema_subjects:
  posts.created:
    compatibility: BACKWARD_TRANSITIVE
    schema_type: AVRO
    version_strategy: SUBJECT_VERSION

  posts.enhanced:
    compatibility: FORWARD
    schema_type: AVRO
    version_strategy: SUBJECT_VERSION

  match.detected:
    compatibility: FULL
    schema_type: AVRO
    version_strategy: SUBJECT_VERSION
```

### 4. Automated Contract Validation Pipeline

**Decision**: Implement automated validation in CI/CD to prevent breaking changes.

**Why**: Manual schema validation is error-prone and doesn't scale:
- **Early Detection**: Catch breaking changes before deployment
- **Consistency**: Enforce schema standards across all services
- **Safety**: Prevent production outages from incompatible changes
- **Documentation**: Generate up-to-date API documentation automatically

**Validation Pipeline**:
```yaml
# Contract Validation Workflow
validation_stages:
  1_schema_syntax:
    - openapi_validation
    - asyncapi_validation
    - avro_schema_validation

  2_compatibility_check:
    - backward_compatibility
    - forward_compatibility
    - full_compatibility_when_required

  3_cross_service_impact:
    - dependency_analysis
    - breaking_change_detection
    - consumer_impact_assessment

  4_documentation_generation:
    - api_docs_update
    - event_catalog_refresh
    - schema_registry_publication
```

### 5. Contract Testing Framework

**Decision**: Implement consumer-driven contract testing with Pact or similar framework.

**Why**: Schema compatibility alone doesn't guarantee working integrations:
- **Consumer-Driven**: Consumers define what they actually need
- **Provider Verification**: Providers verify they meet consumer expectations
- **Safe Evolution**: Changes are validated against real usage patterns
- **Deployment Safety**: Prevents deployments that would break consumers

**Contract Testing Architecture**:
```javascript
// Consumer Contract Definition
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');

const provider = new PactV3({
  consumer: 'fn-notifications',
  provider: 'fn-posts',
  port: 8080,
});

// Define expected interaction
provider
  .given('a post exists with ID 123')
  .uponReceiving('a request for post details')
  .withRequest({
    method: 'GET',
    path: '/api/posts/123',
    headers: {
      'Authorization': MatchersV3.like('Bearer token123'),
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
      status: MatchersV3.term({
        matcher: '^(active|resolved|expired)$',
        generate: 'active',
      }),
    },
  });
```

### 6. Schema Evolution Governance

**Decision**: Implement formal schema evolution rules with compatibility levels.

**Why**: Uncontrolled schema changes break microservice integrations:
- **Backward Compatibility**: New schema versions can read old data
- **Forward Compatibility**: Old schema versions can read new data
- **Full Compatibility**: Both backward and forward compatibility
- **Breaking Changes**: Coordinated major version upgrades

**Evolution Rules by Domain**:
```yaml
# Schema Evolution Policies
post_lifecycle_events:
  compatibility: BACKWARD_TRANSITIVE
  rules:
    - required_fields_cannot_be_removed
    - field_types_cannot_change
    - new_fields_must_have_defaults
    - enums_can_only_add_values

match_events:
  compatibility: FORWARD
  rules:
    - consumers_must_handle_new_fields
    - field_removal_requires_major_version
    - type_changes_require_major_version

ai_enhancement_events:
  compatibility: FULL
  rules:
    - all_changes_must_be_additive
    - no_field_removal_allowed
    - no_type_changes_allowed
    - strict_compatibility_required
```

## Domain Model

**Aggregate Root**: `ContractDefinition`
- Manages schema versions and compatibility rules
- Enforces evolution policies and validation
- Tracks consumer dependencies and impact analysis
- Publishes schema change events for coordination

**Entities**:
- `APIContract` - OpenAPI specification with versioning
- `EventContract` - AsyncAPI schema with event flows
- `AvroSchema` - Binary serialization schema with evolution rules
- `ConsumerRegistration` - Service dependency tracking

**Value Objects**:
- `SchemaVersion` - Semantic versioning with compatibility metadata
- `CompatibilityLevel` - Backward, forward, full, or none
- `ValidationResult` - Schema validation outcome with errors
- `ImpactAssessment` - Breaking change analysis across consumers

**Domain Services**:
- `SchemaValidator` - Multi-format schema validation engine
- `CompatibilityChecker` - Cross-version compatibility analysis
- `ImpactAnalyzer` - Consumer dependency and breaking change detection
- `SchemaRegistryPublisher` - Confluent Schema Registry integration

## Technology Stack

### Core Framework
- **Node.js + TypeScript**: Contract validation and tooling
- **OpenAPI Tools**: Schema validation and documentation generation
- **AsyncAPI Tools**: Event documentation and validation
- **Avro Tools**: Schema compilation and compatibility checking

### Schema Management
- **Confluent Schema Registry**: Production schema storage and validation
- **JSON Schema**: OpenAPI and AsyncAPI validation
- **Avro Schema**: Binary serialization and evolution
- **GraphQL Tools**: GraphQL schema validation and federation

### Validation Pipeline
- **GitHub Actions**: Automated validation on pull requests
- **Spectral**: OpenAPI linting and style validation
- **AsyncAPI CLI**: Event schema validation
- **Avro Tools**: Compatibility checking and code generation

### Documentation Generation
- **Redoc**: Interactive API documentation
- **AsyncAPI Generator**: Event catalog generation
- **Schema Registry UI**: Schema browsing and management
- **Confluence**: Generated documentation publishing

## Contract Types and Formats

### OpenAPI REST API Contracts

**Posts API Contract Example**:
```yaml
openapi: 3.1.0
info:
  title: fn-posts API
  version: 1.2.0
  description: Photo-first lost & found post management API

paths:
  /api/posts:
    post:
      operationId: createPost
      summary: Create a new lost or found post
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePostRequest'
      responses:
        '201':
          description: Post created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'

components:
  schemas:
    CreatePostRequest:
      type: object
      required: [title, post_type, location, photos]
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 200
        post_type:
          type: string
          enum: [lost, found]
        location:
          $ref: '#/components/schemas/Location'
        photos:
          type: array
          minItems: 1
          maxItems: 10
          items:
            $ref: '#/components/schemas/PhotoUpload'
```

### AsyncAPI Event Contracts

**Post Events Contract Example**:
```yaml
asyncapi: 2.6.0
info:
  title: Posts Domain Events
  version: 1.1.0
  description: Event schema for post lifecycle management

channels:
  posts.created:
    description: Published when a new post is created
    publish:
      operationId: publishPostCreated
      message:
        $ref: '#/components/messages/PostCreated'

  posts.enhanced:
    description: Published when AI enhancement completes
    subscribe:
      operationId: handlePostEnhanced
      message:
        $ref: '#/components/messages/PostEnhanced'

components:
  messages:
    PostCreated:
      name: PostCreated
      title: Post Created Event
      summary: A new post has been created
      contentType: application/json
      payload:
        $ref: '#/components/schemas/PostCreatedPayload'

  schemas:
    PostCreatedPayload:
      type: object
      required: [event_id, post_id, user_id, title, post_type, location, photos]
      properties:
        event_id:
          type: string
          format: uuid
        post_id:
          type: string
          format: uuid
        user_id:
          type: string
          format: uuid
        title:
          type: string
        post_type:
          type: string
          enum: [lost, found]
        location:
          $ref: '#/components/schemas/Location'
        photos:
          type: array
          items:
            $ref: '#/components/schemas/PhotoMetadata'
        created_at:
          type: string
          format: date-time
```

### Avro Schema Definitions

**Post Created Avro Schema**:
```json
{
  "type": "record",
  "name": "PostCreatedEvent",
  "namespace": "com.findlynow.posts.events",
  "doc": "Event published when a new post is created",
  "fields": [
    {
      "name": "event_id",
      "type": "string",
      "doc": "Unique identifier for this event"
    },
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
      "name": "title",
      "type": "string",
      "doc": "Post title"
    },
    {
      "name": "post_type",
      "type": {
        "type": "enum",
        "name": "PostType",
        "symbols": ["LOST", "FOUND"]
      },
      "doc": "Type of post - lost or found"
    },
    {
      "name": "location",
      "type": {
        "type": "record",
        "name": "Location",
        "fields": [
          {"name": "lat", "type": "double"},
          {"name": "lng", "type": "double"},
          {"name": "address", "type": ["null", "string"], "default": null}
        ]
      },
      "doc": "Geographic location where item was lost/found"
    },
    {
      "name": "photos",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "PhotoMetadata",
          "fields": [
            {"name": "photo_id", "type": "string"},
            {"name": "url", "type": "string"},
            {"name": "thumbnail_url", "type": "string"},
            {"name": "content_type", "type": "string"},
            {"name": "size_bytes", "type": "long"}
          ]
        }
      },
      "doc": "Array of photo metadata"
    },
    {
      "name": "created_at",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "Timestamp when post was created"
    }
  ]
}
```

## Validation and Governance Processes

### Schema Validation Pipeline

**Automated Validation Steps**:
1. **Syntax Validation**: Ensure schema is well-formed JSON/YAML
2. **Semantic Validation**: Validate against OpenAPI/AsyncAPI specifications
3. **Style Validation**: Enforce organizational style guidelines
4. **Compatibility Check**: Verify backward/forward compatibility
5. **Impact Analysis**: Assess breaking changes across consumers
6. **Documentation Generation**: Update API documentation automatically

**Validation Configuration**:
```yaml
# .spectral.yml - OpenAPI Linting Rules
extends: ["@stoplight/spectral/rulesets/oas"]
rules:
  operation-operationId: error
  operation-description: error
  operation-summary: error
  info-description: error
  tag-description: error
  no-unresolved-refs: error
  findly-naming-convention:
    description: "Use camelCase for properties"
    given: "$.components.schemas..properties.*~"
    then:
      function: casing
      functionOptions:
        type: camel
```

### Breaking Change Detection

**Automated Breaking Change Analysis**:
```typescript
interface BreakingChangeDetector {
  detectApiChanges(oldSchema: OpenAPISchema, newSchema: OpenAPISchema): BreakingChange[];
  detectEventChanges(oldSchema: AsyncAPISchema, newSchema: AsyncAPISchema): BreakingChange[];
  detectAvroChanges(oldSchema: AvroSchema, newSchema: AvroSchema): BreakingChange[];
}

interface BreakingChange {
  type: 'FIELD_REMOVED' | 'TYPE_CHANGED' | 'REQUIRED_ADDED' | 'ENUM_VALUE_REMOVED';
  path: string;
  description: string;
  severity: 'MAJOR' | 'MINOR' | 'PATCH';
  affectedConsumers: string[];
  migrationGuide?: string;
}

// Example breaking change detection
const changes = breakingChangeDetector.detectApiChanges(oldPostsApi, newPostsApi);
// Result:
// [
//   {
//     type: 'FIELD_REMOVED',
//     path: '/components/schemas/Post/properties/legacy_field',
//     description: 'Field legacy_field was removed from Post schema',
//     severity: 'MAJOR',
//     affectedConsumers: ['fn-notifications', 'fn-matcher'],
//     migrationGuide: 'Use new_field instead of legacy_field'
//   }
// ]
```

### Cross-Service Impact Analysis

**Consumer Dependency Tracking**:
```yaml
# consumer-contracts.yml
consumer_dependencies:
  fn-notifications:
    consumes:
      apis:
        - service: fn-posts
          version: ">=1.0.0 <2.0.0"
          endpoints: ["/api/posts/{id}", "/api/posts/search"]
      events:
        - service: fn-posts
          topic: posts.created
          schema_version: ">=1.1.0"
        - service: fn-matcher
          topic: match.detected
          schema_version: ">=1.0.0"

  fn-matcher:
    consumes:
      apis:
        - service: fn-posts
          version: ">=1.0.0"
          endpoints: ["/api/posts/nearby"]
      events:
        - service: fn-posts
          topic: posts.created
          schema_version: ">=1.0.0"
        - service: fn-media-ai
          topic: posts.enhanced
          schema_version: ">=1.0.0"
```

## Performance and Scalability

### Schema Registry Performance

**Performance Targets**:
- **Schema Retrieval**: <10ms from cache, <50ms from registry
- **Validation Latency**: <5ms for cached schemas
- **Throughput**: 10,000+ schema validations per second
- **Availability**: 99.9% uptime for schema validation

**Caching Strategy**:
```yaml
# Schema caching configuration
cache_layers:
  local_cache:
    ttl: 300s  # 5 minutes
    max_entries: 1000
    eviction_policy: LRU

  redis_cache:
    ttl: 3600s  # 1 hour
    cluster_nodes: 3
    replication_factor: 2

  schema_registry:
    connection_pool: 10
    timeout: 5s
    retry_policy: exponential_backoff
```

### CI/CD Pipeline Optimization

**Fast Validation Pipeline**:
- **Parallel Validation**: Run syntax, semantic, and style checks concurrently
- **Incremental Validation**: Only validate changed schemas
- **Cached Dependencies**: Cache validation tools and dependencies
- **Early Termination**: Fail fast on critical validation errors

## Integration Points

### Schema Registry Integration

**Publishes Schemas To**:
- Confluent Schema Registry for runtime validation
- API Gateway for request/response validation
- Service meshes for contract enforcement
- Documentation sites for developer reference

**Consumes From**:
- Git repositories for schema source control
- CI/CD pipelines for automated validation
- Developer tools for schema generation
- Service deployments for compatibility verification

### Developer Tooling Integration

**IDE Plugins**:
```json
{
  "vscode_extension": {
    "name": "findly-contract-tools",
    "features": [
      "schema_validation",
      "auto_completion",
      "breaking_change_detection",
      "documentation_preview"
    ]
  },
  "cli_tools": {
    "contract_validator": "npx @findly/contract-validator validate --file api/posts.yaml",
    "schema_diff": "npx @findly/contract-validator diff --old v1.0.0 --new v1.1.0",
    "impact_analysis": "npx @findly/contract-validator impact --schema posts.yaml"
  }
}
```

## Security and Compliance

### Schema Security

**Access Control**:
- Schema modifications require code review and approval
- Production schema changes require security team review
- API keys for schema registry access with rotation
- Audit logging for all schema modifications

**Compliance Validation**:
```yaml
# compliance-rules.yml
data_protection:
  pii_detection:
    - personal_email
    - phone_number
    - government_id
    - payment_info
  classification_required: true
  retention_policy_defined: true

security_requirements:
  authentication_required: true
  authorization_documented: true
  rate_limiting_defined: true
  input_validation_specified: true
```

## Monitoring and Observability

### Contract Usage Metrics

**Key Metrics**:
- Schema validation success/failure rates
- Contract test execution results
- Breaking change detection frequency
- Consumer adoption of new schema versions
- API usage patterns by endpoint and consumer

**Alerting Rules**:
```yaml
# monitoring-alerts.yml
schema_validation_alerts:
  high_failure_rate:
    condition: "validation_failure_rate > 5%"
    severity: warning
    notification: slack

  breaking_change_detected:
    condition: "breaking_changes_count > 0"
    severity: critical
    notification: pagerduty

contract_drift_alerts:
  schema_version_lag:
    condition: "consumer_version_lag > 30_days"
    severity: warning
    notification: email
```

---

*This domain focuses purely on schema governance and contract management across the Findly Now microservices ecosystem. For system-wide architecture decisions, see [../ARCHITECTURE.md](../ARCHITECTURE.md). For detailed schema specifications, see [api-documentation.md](api-documentation.md).*