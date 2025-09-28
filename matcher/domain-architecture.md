# Matcher Domain Architecture

**The intelligent matching engine that connects lost and found posts through multi-factor analysis, enabling rapid reunification in the Findly Now ecosystem.**

## Domain Purpose

Transform Lost & Found from manual browsing to **intelligent, automated matching** through sophisticated algorithms that analyze location proximity, visual similarity, textual content, and temporal patterns.

**Core Value**: Intelligent matching dramatically increases reunification rates by proactively identifying potential matches with confidence scoring, reducing search time from hours to seconds.

## Domain Boundaries

**Bounded Context**: Intelligent matching between lost and found posts
- Event-driven processing of PostCreated and PostEnhanced events
- Multi-factor matching algorithm with configurable weights and thresholds
- Match confidence scoring and quality assessment using domain expertise
- Match lifecycle management from detection through resolution
- Item claiming workflow with verification and fraud prevention
- Analytics and insights on matching performance and success rates

## Key Architecture Decisions

### 1. Rust + Axum for Performance-Critical Matching
**Decision**: Use Rust with Axum framework for the matching service.

**Why**: Matching algorithms are computationally intensive and latency-sensitive:
- **Performance**: Rust's zero-cost abstractions enable high-throughput matching
- **Memory Safety**: Prevents crashes during intensive computation
- **Concurrency**: Tokio async runtime handles thousands of concurrent matching operations
- **Type Safety**: Domain modeling with strong types prevents matching logic errors

**Technology Stack**:
```rust
// Core Framework
axum          // High-performance web framework
tokio         // Async runtime
serde         // Serialization/deserialization

// Domain Logic
uuid          // Match identifiers
chrono        // Temporal matching
geo           // Geospatial calculations

// Infrastructure
sqlx          // PostgreSQL with PostGIS
rdkafka       // Kafka event processing
redis         // Caching and rate limiting
```

### 2. Multi-Factor Matching Algorithm
**Decision**: Use weighted, multi-factor analysis rather than single-criteria matching.

**Why**: Single-factor matching produces too many false positives/negatives:

| Factor | Weight | Purpose | Method |
|--------|--------|---------|--------|
| Location Proximity | 40% | Geographic nearness | PostGIS distance queries with decay |
| Visual Similarity | 35% | Photo-based matching | AI tags, colors, objects from Media AI |
| Text Similarity | 15% | Description matching | TF-IDF + semantic similarity |
| Temporal Proximity | 10% | Time-based scoring | Exponential decay from report time |

**Algorithm Architecture**:
```rust
pub struct MatchingEngine {
    location_scorer: LocationProximityScorer,
    visual_scorer: VisualSimilarityScorer,
    text_scorer: TextSimilarityScorer,
    temporal_scorer: TemporalProximityScorer,
}

impl MatchingEngine {
    pub async fn calculate_match_score(
        &self,
        lost_post: &Post,
        found_post: &Post,
    ) -> Result<MatchScore> {
        let location_score = self.location_scorer.score(lost_post, found_post).await?;
        let visual_score = self.visual_scorer.score(lost_post, found_post).await?;
        let text_score = self.text_scorer.score(lost_post, found_post).await?;
        let temporal_score = self.temporal_scorer.score(lost_post, found_post).await?;

        let weighted_score = (location_score * 0.40) +
                           (visual_score * 0.35) +
                           (text_score * 0.15) +
                           (temporal_score * 0.10);

        Ok(MatchScore::new(weighted_score, vec![
            MatchReason::location(location_score),
            MatchReason::visual(visual_score),
            MatchReason::text(text_score),
            MatchReason::temporal(temporal_score),
        ]))
    }
}
```

### 3. PostgreSQL + PostGIS for Geospatial Matching
**Decision**: Use PostgreSQL with PostGIS extension for location-based matching.

**Why**: Location is the primary matching factor (40% weight):
- **Spatial Queries**: Efficient radius-based searches with geographic indexing
- **Distance Calculations**: Accurate haversine distance calculations
- **Performance**: Spatial indexes enable sub-second queries on millions of posts
- **Scalability**: PostGIS handles complex geometric operations at database level

**Geospatial Matching Logic**:
```sql
-- Find potential matches within radius with distance scoring
SELECT
    p.id,
    p.title,
    p.created_at,
    ST_Distance(p.location, $1) as distance_meters,
    -- Exponential decay: 1.0 at 0m, 0.5 at 500m, 0.1 at 1km
    EXP(-ST_Distance(p.location, $1) / 500.0) as location_score
FROM posts p
WHERE p.post_type != $2  -- Lost posts match Found posts and vice versa
  AND p.status = 'active'
  AND ST_DWithin(p.location, $1, $3)  -- Within max search radius
  AND p.created_at > $4  -- Within temporal window
ORDER BY location_score DESC
LIMIT 100;
```

### 4. Event-Driven Matching Triggers
**Decision**: Trigger matching on both PostCreated and PostEnhanced events.

**Why**: Different events provide different matching opportunities:

**PostCreated Events**:
- **Immediate Matching**: Fast matching using basic metadata (location, title, time)
- **User Experience**: Quick feedback that system is actively looking for matches
- **Simple Algorithm**: Lightweight matching using available data

**PostEnhanced Events**:
- **Improved Matching**: Rich AI metadata enables more accurate visual matching
- **Re-evaluation**: Previously low-confidence matches may become high-confidence
- **Better Quality**: Enhanced tags and attributes improve match precision

**Event Processing Flow**:
```rust
#[async_trait]
impl EventHandler<PostCreatedEvent> for MatchingService {
    async fn handle(&self, event: PostCreatedEvent) -> Result<()> {
        // Fast initial matching with basic data
        let matches = self.engine.detect_initial_matches(&event.post).await?;

        for match_candidate in matches {
            if match_candidate.confidence > INITIAL_THRESHOLD {
                self.create_match(match_candidate).await?;
            }
        }
        Ok(())
    }
}

#[async_trait]
impl EventHandler<PostEnhancedEvent> for MatchingService {
    async fn handle(&self, event: PostEnhancedEvent) -> Result<()> {
        // Enhanced matching with AI metadata
        let enhanced_matches = self.engine.detect_enhanced_matches(&event.post).await?;

        // Re-evaluate existing matches
        self.reevaluate_existing_matches(&event.post).await?;

        for match_candidate in enhanced_matches {
            if match_candidate.confidence > ENHANCED_THRESHOLD {
                self.create_or_update_match(match_candidate).await?;
            }
        }
        Ok(())
    }
}
```

### 5. Confidence-Based Match Quality
**Decision**: Use confidence scoring to determine match quality and automatic actions.

**Why**: Match quality varies significantly based on available data:
- **User Experience**: Only surface high-confidence matches to reduce noise
- **Automatic Actions**: Enable auto-notifications for very high confidence matches
- **Human Review**: Flag medium-confidence matches for manual verification
- **Learning**: Track confidence vs actual success rates for algorithm improvement

**Confidence Thresholds**:
```rust
pub struct MatchThresholds {
    pub auto_notify: f64,      // 0.85 - Automatically notify both parties
    pub surface_match: f64,    // 0.70 - Show match in UI
    pub human_review: f64,     // 0.50 - Flag for manual review
    pub discard: f64,          // 0.30 - Don't store low-confidence matches
}

pub enum MatchQuality {
    High,    // 0.85+ - Auto-notify, high priority
    Medium,  // 0.70+ - Surface in UI, manual review
    Low,     // 0.50+ - Store but don't surface
}
```

### 6. Match Lifecycle Management
**Decision**: Implement explicit match states with automatic transitions and expiration.

**Why**: Matches require active lifecycle management:
- **User Clarity**: Clear states help users understand match status
- **Automatic Cleanup**: Expired matches don't clutter the system
- **Fraud Prevention**: Claiming workflow prevents false claims
- **Analytics**: State transitions provide insights into match quality

**Match State Machine**:
```rust
#[derive(Debug, Clone, PartialEq)]
pub enum MatchStatus {
    Pending,      // Initial state, waiting for user action
    Confirmed,    // Both parties confirmed it's a match
    Rejected,     // At least one party rejected the match
    Claimed,      // Item has been claimed
    Expired,      // Match expired without action (7 days)
}

impl Match {
    pub fn confirm(&mut self, user_id: Uuid) -> Result<Vec<DomainEvent>> {
        match self.status {
            MatchStatus::Pending => {
                self.status = MatchStatus::Confirmed;
                self.confirmed_at = Some(Utc::now());
                self.confirmed_by = Some(user_id);

                Ok(vec![DomainEvent::MatchConfirmed {
                    match_id: self.id,
                    confirmed_by: user_id,
                    lost_post_id: self.lost_post_id,
                    found_post_id: self.found_post_id,
                }])
            }
            _ => Err(DomainError::InvalidStateTransition)
        }
    }
}
```

## Domain Model

**Aggregate Root**: `Match`
- Represents a potential connection between lost and found posts
- Contains confidence score with detailed reasoning
- Manages state transitions and lifecycle events
- Enforces business rules around claiming and verification

**Entities**:
- `Claim` - Record of someone claiming an item from a match
- `MatchRule` - Configurable rules for matching algorithm weights

**Value Objects**:
- `ConfidenceScore` (0.0-1.0 with quality level)
- `MatchReason` (factor type, individual score, weight)
- `LocationProximity` (distance, decay score)
- `MatchStatus` (enum with validation)

**Domain Services**:
- `MatchingEngine` - Core matching algorithm coordination
- `SimilarityScorer` - Individual scoring algorithms
- `MatchLifecycleManager` - State transitions and expiration
- `ClaimVerificationService` - Fraud prevention and verification

## Performance Targets

- **Matching Latency**: <500ms for initial matching, <2s for enhanced matching
- **Throughput**: Handle 1000+ concurrent matching operations
- **Accuracy**: >90% precision at high confidence threshold (0.85+)
- **Recall**: >80% of actual matches detected at medium confidence (0.70+)

## Integration Points

**Consumes Events**:
- `post.created` → Trigger initial matching with basic metadata
- `post.enhanced` → Trigger enhanced matching with AI metadata

**Publishes Events**:
- `post.matched` → Notify users of high-confidence matches
- `post.claimed` → Urgent notification for item claiming
- `match.expired` → Cleanup notification for expired matches

**External Services**:
- **PostgreSQL + PostGIS**: Geospatial queries and match storage
- **Redis**: Caching post metadata and rate limiting
- **Confluent Cloud**: Event consumption and publishing

## Matching Algorithm Details

**Location Proximity Scoring**:
```rust
pub fn score_location_proximity(
    lost_location: Point,
    found_location: Point,
    max_radius_km: f64,
) -> f64 {
    let distance_km = haversine_distance(lost_location, found_location);

    if distance_km > max_radius_km {
        return 0.0;
    }

    // Exponential decay: perfect score at 0km, 50% at 1km, 10% at 3km
    (-distance_km / 1.44).exp()
}
```

**Visual Similarity Scoring**:
```rust
pub fn score_visual_similarity(
    lost_post_ai_tags: &[AiTag],
    found_post_ai_tags: &[AiTag],
) -> f64 {
    let lost_objects: HashSet<_> = lost_post_ai_tags.iter()
        .filter(|tag| tag.confidence > 0.8)
        .map(|tag| &tag.name)
        .collect();

    let found_objects: HashSet<_> = found_post_ai_tags.iter()
        .filter(|tag| tag.confidence > 0.8)
        .map(|tag| &tag.name)
        .collect();

    if lost_objects.is_empty() || found_objects.is_empty() {
        return 0.0;
    }

    let intersection = lost_objects.intersection(&found_objects).count();
    let union = lost_objects.union(&found_objects).count();

    // Jaccard similarity
    intersection as f64 / union as f64
}
```

**Text Similarity Scoring**:
```rust
pub fn score_text_similarity(lost_text: &str, found_text: &str) -> f64 {
    // TF-IDF vectorization
    let lost_vector = tfidf_vectorize(lost_text);
    let found_vector = tfidf_vectorize(found_text);

    // Cosine similarity
    cosine_similarity(&lost_vector, &found_vector)
}
```

**Temporal Proximity Scoring**:
```rust
pub fn score_temporal_proximity(
    lost_time: DateTime<Utc>,
    found_time: DateTime<Utc>,
) -> f64 {
    let time_diff_hours = (found_time - lost_time).num_hours().abs() as f64;

    // Items found immediately after being lost score highest
    // Decay over 24-48 hours, minimal score after 1 week
    (-time_diff_hours / 24.0).exp()
}
```

## Scalability & Performance

**Horizontal Scaling**:
- Stateless service design with shared PostgreSQL state
- Kafka consumer groups for distributed event processing
- Redis cluster for distributed caching and rate limiting

**Algorithm Optimization**:
- **Spatial Indexing**: PostGIS R-tree indexes for fast location queries
- **Candidate Pre-filtering**: Narrow search space before expensive similarity calculations
- **Batch Processing**: Process multiple posts in single database transactions
- **Caching Strategy**: Cache post metadata for 30 days to avoid repeated database hits

**Performance Monitoring**:
- Match detection latency per algorithm factor
- False positive/negative rates by confidence threshold
- Database query performance and spatial index effectiveness
- Memory usage during intensive matching operations

## Key Constraints

- Posts must contain valid geographic coordinates for location matching
- AI metadata from Media AI domain required for visual similarity matching
- Match confidence must be calculated transparently with detailed reasoning
- All matches expire after 7 days without user action
- System must handle graceful degradation when external services are unavailable

---

*This domain focuses purely on intelligent matching between lost and found posts. For system-wide architecture decisions, see [../ARCHITECTURE.md](../ARCHITECTURE.md). For detailed API specifications, see [api-documentation.md](api-documentation.md).*