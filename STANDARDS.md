# Code Standards & Patterns

**Document Ownership**: This document OWNS cross-service code standards, DDD patterns, and development guidelines.

## Domain-Driven Design Patterns

### Universal DDD Principles
- **Entities** contain business logic and invariants
- **Aggregates** maintain consistency boundaries
- **Repository interfaces** in domain, concrete implementations in infrastructure
- **Domain events** published for all state changes
- **Anti-Corruption Layers** protect domain from external schemas

### Aggregate Root Pattern
**Aggregate Roots** encapsulate business logic and maintain invariants within their bounded context:

- **Encapsulate business rules**: All domain logic resides within the aggregate
- **Maintain invariants**: Ensure consistency within the aggregate boundary
- **Control access**: External systems can only modify aggregates through their public methods
- **Validate state changes**: All mutations must pass through domain validation
- **Generate domain events**: Publish events when significant state changes occur

**Key Characteristics**:
- Single entry point for modifications
- Internal state is private and protected
- Business rules are enforced at the domain level
- State transitions are explicit and validated

```rust
// Rust example (fn-matcher)
pub struct Match {
    id: MatchId,
    lost_post_id: PostId,
    found_post_id: PostId,
    confidence_score: ConfidenceScore,
    status: MatchStatus,
}

impl Match {
    pub fn claim(&mut self, user_id: UserId) -> Result<(), MatchError> {
        if self.status != MatchStatus::Pending {
            return Err(MatchError::AlreadyClaimed);
        }
        self.status = MatchStatus::Claimed;
        Ok(())
    }
}
```

### Repository Pattern
```go
// Interface in domain layer
type PostRepository interface {
    Create(post *Post) error
    FindByRadius(location Location, radius Distance) ([]*Post, error)
    FindByID(id PostID) (*Post, error)
}

// Implementation in infrastructure layer
type PostgresPostRepository struct {
    db *sql.DB
}
```

```elixir
# Behavior in domain layer
defmodule FnNotifications.Domain.Repositories.NotificationRepositoryBehavior do
  @callback create(notification :: Notification.t()) :: {:ok, Notification.t()} | {:error, term()}
  @callback find_by_user_id(user_id :: String.t()) :: {:ok, [Notification.t()]} | {:error, term()}
end
```

## Event-Driven Patterns

### Domain Events
```go
// Go event publishing
type PostCreatedEvent struct {
    PostID    string    `json:"post_id"`
    UserID    string    `json:"user_id"`
    Location  Location  `json:"location"`
    PhotoURLs []string  `json:"photo_urls"`
    Timestamp time.Time `json:"timestamp"`
}

func (s *PostService) CreatePost(cmd CreatePostCommand) (*Post, error) {
    post := domain.NewPost(cmd)

    if err := s.repo.Create(post); err != nil {
        return nil, err
    }

    s.events.Publish("post.created", PostCreatedEvent{
        PostID:    post.ID(),
        Location:  post.Location(),
        PhotoURLs: post.PhotoURLs(),
    })

    return post, nil
}
```

```elixir
# Elixir event consumption
defmodule FnNotifications.Application.EventHandlers.PostsEventProcessor do
  use Broadway

  def handle_message(_processor, message, _context) do
    event = decode_event(message.data)

    case EventTranslator.translate(event) do
      {:ok, command} ->
        NotificationService.send_notification(command)
      {:error, reason} ->
        Logger.error("Failed to translate event: #{reason}")
    end

    message
  end
end
```

### Anti-Corruption Layer
```elixir
# Protects domain from external event schemas
defmodule FnNotifications.Application.AntiCorruption.EventTranslator do
  def translate(%{"type" => "post.created"} = external_event) do
    {:ok, %SendNotificationCommand{
      user_id: external_event["user_id"],
      notification_type: :post_confirmation,
      metadata: %{
        post_id: external_event["post_id"],
        location: external_event["location"]
      }
    }}
  end
end
```

## Testing Strategy

### E2E Tests Only
All services use **End-to-End tests only** - no unit tests or mocks.

**Key Principles**:
- **Real infrastructure**: Tests use actual databases, message queues, and external services
- **Complete workflows**: Test entire business processes from start to finish
- **No mocking**: All dependencies are real implementations or test-equivalent services
- **Environment isolation**: Each test suite runs in a clean, isolated environment
- **Data cleanup**: Tests manage their own test data lifecycle

**Benefits**:
- **High confidence**: Tests validate real system behavior
- **Integration verification**: Catches integration issues between components
- **Deployment validation**: Tests mirror production environment closely
- **Reduced maintenance**: No complex mocking setup to maintain

## Code Quality Standards

### Error Handling

**Structured Error Handling**: All services implement language-appropriate error patterns

**Key Principles**:
- **Explicit error types**: Define custom error types with meaningful codes and messages
- **Consistent patterns**: Use language-idiomatic error handling consistently
- **Error context**: Include relevant context and details for debugging
- **Error boundaries**: Handle errors at appropriate service boundaries
- **Graceful degradation**: Services should handle errors gracefully and maintain stability

**Error Classification**:
- **Domain errors**: Business rule violations (validation, state transition errors)
- **Infrastructure errors**: Database, network, external service failures
- **System errors**: Unexpected errors that require monitoring and alerting

**Example Pattern**: Result types with custom error enums
```rust
#[derive(Debug, thiserror::Error)]
pub enum MatchingError {
    #[error("Post not found: {id}")]
    PostNotFound { id: String },
    #[error("Invalid confidence score: {score}")]
    InvalidConfidenceScore { score: f64 },
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
}

fn calculate_match_score(post: &Post) -> Result<ConfidenceScore, MatchingError> {
    let score = process_matching_algorithm(post)?;
    ConfidenceScore::new(score)
        .map_err(|_| MatchingError::InvalidConfidenceScore { score })
}
```

### Minimal Comments
- **Only comment complex business rules**
- **Never comment obvious code**
- **Use descriptive function/variable names instead**

```go
// Good: Comments explain business logic
func CalculateSearchRadius(itemType ItemType) Distance {
    // Lost electronics need wider search radius due to mobility
    if itemType == Electronics {
        return Distance{Meters: 5000}
    }
    // Personal items typically stay closer to loss location
    return Distance{Meters: 1000}
}

// Bad: Comments state the obvious
func GetUserID() string {
    // Return the user ID
    return user.ID
}
```

### Language-Specific Standards

**Go Standards:**
```bash
# Required tools
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Commands
go fmt ./...           # Format code
golangci-lint run      # Lint code
go mod tidy           # Clean dependencies
```

**Elixir Standards:**
```bash
# Commands
mix format            # Format code
mix credo            # Static analysis
mix dialyzer         # Type checking
```

**Python Standards:**
```bash
# Commands
black .              # Format code
flake8 .            # Lint code
mypy .              # Type checking
```

**Rust Standards:**
```bash
# Commands
cargo fmt            # Format code
cargo clippy -- -D warnings  # Lint code
cargo check          # Type checking
cargo test           # Run tests
```

## Git Workflow

### Branch Strategy
- **main**: Production-ready code
- **feature/***: New features (`feature/post-photo-upload`)
- **fix/***: Bug fixes (`fix/notification-retry-logic`)

### Commit Convention
```bash
feat(posts): add photo upload validation
fix(notifications): resolve delivery retry loop
docs(architecture): update domain boundaries
```

### Pull Request Requirements
- [ ] Code follows domain-specific standards
- [ ] E2E tests pass
- [ ] Linting passes
- [ ] No secrets in code
- [ ] Documentation updated if needed

## Security Standards

### Secrets Management
- **Never commit** secrets or API keys
- **Use environment variables** for all credentials
- **Rotate keys** regularly
- **Principle of least privilege** for all access

### Data Protection
- **Encrypt** all data at rest and in transit
- **Validate** all inputs at domain boundaries
- **Sanitize** all outputs to prevent injection
- **Log security events** but never log secrets

### Contact Information Rules
- Contact info (email/phone) stored **only** in `user_preferences` table
- **Never duplicate** contact info in other tables
- **Cache** preferences with TTL for performance
- **Invalidate cache** on preference updates

## Performance Standards

### Database
- **Manual SQL** for complex queries (especially PostGIS)
- **Proper indexing** on frequently queried fields
- **Connection pooling** for all database connections
- **Query timeouts** to prevent hanging

### API Response Times
- **Posts creation**: <15 seconds end-to-end
- **Geospatial queries**: <200ms
- **Notification delivery**: <5 seconds
- **Health checks**: <100ms

### Event Processing
- **Backpressure handling** in all consumers
- **Batch processing** where applicable
- **Dead letter queues** for failed events
- **Event replay** capability for recovery

---

*For service-specific implementation details, see individual service DEVELOPMENT.md files.*