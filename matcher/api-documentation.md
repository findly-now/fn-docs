# fn-matcher API Documentation

**REST API for managing intelligent matches between lost and found posts in the Findly Now ecosystem.**

## Base Configuration

**Base URL**: `http://localhost:8003` (development) / `https://api.findlynow.com/matcher` (production)

**API Version**: `v1`

**Content Type**: `application/json`

**Authentication**: Bearer token (JWT) in Authorization header

## Core Endpoints

### 1. List Matches

**Endpoint**: `GET /api/v1/matches`

**Purpose**: Retrieve matches with filtering and pagination support.

**Query Parameters**:
- `user_id` (optional): Filter matches for specific user
- `post_id` (optional): Filter matches for specific post
- `status` (optional): Filter by match status (`pending`, `confirmed`, `rejected`, `claimed`, `expired`)
- `confidence_min` (optional): Minimum confidence threshold (0.0-1.0)
- `confidence_max` (optional): Maximum confidence threshold (0.0-1.0)
- `limit` (optional): Number of results per page (default: 20, max: 100)
- `offset` (optional): Number of results to skip (default: 0)
- `sort` (optional): Sort order (`confidence_desc`, `confidence_asc`, `created_desc`, `created_asc`)

**Response**:
```json
{
  "matches": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "lost_post_id": "123e4567-e89b-12d3-a456-426614174000",
      "found_post_id": "789e0123-e45b-67c8-d901-234567890123",
      "confidence_score": 0.87,
      "match_quality": "High",
      "status": "pending",
      "created_at": "2024-01-15T10:30:00Z",
      "expires_at": "2024-01-22T10:30:00Z",
      "confirmed_at": null,
      "confirmed_by": null,
      "reasons": [
        {
          "factor": "location",
          "score": 0.92,
          "weight": 0.40,
          "details": {
            "distance_meters": 150.5,
            "lost_location": {"lat": 40.7128, "lng": -74.0060},
            "found_location": {"lat": 40.7140, "lng": -74.0050}
          }
        },
        {
          "factor": "visual",
          "score": 0.85,
          "weight": 0.35,
          "details": {
            "common_objects": ["smartphone", "black_case"],
            "color_similarity": 0.90,
            "brand_match": "Apple iPhone"
          }
        },
        {
          "factor": "text",
          "score": 0.78,
          "weight": 0.15,
          "details": {
            "title_similarity": 0.80,
            "description_similarity": 0.75
          }
        },
        {
          "factor": "temporal",
          "score": 0.95,
          "weight": 0.10,
          "details": {
            "time_difference_hours": 2.5,
            "lost_at": "2024-01-15T08:00:00Z",
            "found_at": "2024-01-15T10:30:00Z"
          }
        }
      ],
      "lost_post": {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "title": "Lost iPhone 15 Pro in Central Park",
        "post_type": "lost",
        "location": {"lat": 40.7128, "lng": -74.0060},
        "photos": ["https://storage.googleapis.com/fn-photos/lost_iphone_1.jpg"],
        "created_at": "2024-01-15T08:00:00Z"
      },
      "found_post": {
        "id": "789e0123-e45b-67c8-d901-234567890123",
        "title": "Found black iPhone near Bethesda Fountain",
        "post_type": "found",
        "location": {"lat": 40.7140, "lng": -74.0050},
        "photos": ["https://storage.googleapis.com/fn-photos/found_iphone_1.jpg"],
        "created_at": "2024-01-15T10:30:00Z"
      }
    }
  ],
  "pagination": {
    "total": 42,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

**Example Requests**:
```bash
# Get all high-confidence matches for user
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/matches?user_id=user123&confidence_min=0.80"

# Get pending matches for specific post
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/matches?post_id=123e4567-e89b-12d3-a456-426614174000&status=pending"

# Get recent matches sorted by confidence
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/matches?sort=confidence_desc&limit=10"
```

### 2. Get Specific Match

**Endpoint**: `GET /api/v1/matches/{match_id}`

**Purpose**: Retrieve detailed information about a specific match.

**Path Parameters**:
- `match_id`: UUID of the match

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "lost_post_id": "123e4567-e89b-12d3-a456-426614174000",
  "found_post_id": "789e0123-e45b-67c8-d901-234567890123",
  "confidence_score": 0.87,
  "match_quality": "High",
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2024-01-22T10:30:00Z",
  "confirmed_at": null,
  "confirmed_by": null,
  "algorithm_version": "v1.2.1",
  "reasons": [
    // ... detailed scoring breakdown as above
  ],
  "lost_post": {
    // ... full post details
  },
  "found_post": {
    // ... full post details
  },
  "claims": [
    {
      "id": "claim-456",
      "user_id": "user789",
      "created_at": "2024-01-16T14:20:00Z",
      "status": "pending_verification",
      "verification_code": "ABC123"
    }
  ]
}
```

**Example Request**:
```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/matches/550e8400-e29b-41d4-a716-446655440000"
```

### 3. Confirm Match

**Endpoint**: `PUT /api/v1/matches/{match_id}/confirm`

**Purpose**: Mark a match as confirmed by the user.

**Path Parameters**:
- `match_id`: UUID of the match

**Request Body**:
```json
{
  "user_id": "user123",
  "notes": "Yes, this is definitely my phone! The scratch on the back matches perfectly."
}
```

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "confirmed",
  "confirmed_at": "2024-01-16T09:15:00Z",
  "confirmed_by": "user123",
  "message": "Match confirmed successfully. Both parties will be notified."
}
```

**Example Request**:
```bash
curl -X PUT -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "user123", "notes": "This is my phone"}' \
  "http://localhost:8003/api/v1/matches/550e8400-e29b-41d4-a716-446655440000/confirm"
```

### 4. Reject Match

**Endpoint**: `PUT /api/v1/matches/{match_id}/reject`

**Purpose**: Mark a match as rejected by the user.

**Path Parameters**:
- `match_id`: UUID of the match

**Request Body**:
```json
{
  "user_id": "user123",
  "reason": "wrong_item",
  "notes": "This is not my phone - mine has a blue case, not black."
}
```

**Response**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "rejected",
  "rejected_at": "2024-01-16T09:15:00Z",
  "rejected_by": "user123",
  "rejection_reason": "wrong_item",
  "message": "Match rejected. We'll continue looking for better matches."
}
```

**Rejection Reasons**:
- `wrong_item`: Not the same item
- `wrong_location`: Found in different area
- `wrong_time`: Timeline doesn't match
- `already_found`: Item was found through other means
- `other`: Custom reason with notes

**Example Request**:
```bash
curl -X PUT -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "user123", "reason": "wrong_item", "notes": "Different phone model"}' \
  "http://localhost:8003/api/v1/matches/550e8400-e29b-41d4-a716-446655440000/reject"
```

### 5. Claim Item

**Endpoint**: `POST /api/v1/matches/{match_id}/claim`

**Purpose**: Start the claiming process for a confirmed match.

**Path Parameters**:
- `match_id`: UUID of the match

**Request Body**:
```json
{
  "user_id": "user123",
  "contact_method": "phone",
  "contact_value": "+1-555-123-4567",
  "preferred_meeting_location": "Starbucks on 5th Ave near Central Park",
  "notes": "Available weekdays after 5pm, weekends anytime"
}
```

**Response**:
```json
{
  "claim_id": "claim-789",
  "match_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending_verification",
  "verification_code": "ABC123",
  "created_at": "2024-01-16T14:20:00Z",
  "expires_at": "2024-01-17T14:20:00Z",
  "next_steps": [
    "Share verification code 'ABC123' with the person who found your item",
    "Coordinate meeting location and time",
    "Bring photo ID for verification if requested"
  ],
  "message": "Claim initiated successfully. The finder has been notified via SMS."
}
```

**Example Request**:
```bash
curl -X POST -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123",
    "contact_method": "phone",
    "contact_value": "+1-555-123-4567",
    "notes": "Available after 5pm weekdays"
  }' \
  "http://localhost:8003/api/v1/matches/550e8400-e29b-41d4-a716-446655440000/claim"
```

### 6. Get Claims for Match

**Endpoint**: `GET /api/v1/matches/{match_id}/claims`

**Purpose**: Retrieve all claims for a specific match.

**Path Parameters**:
- `match_id`: UUID of the match

**Response**:
```json
{
  "claims": [
    {
      "id": "claim-789",
      "user_id": "user123",
      "status": "verified",
      "contact_method": "phone",
      "contact_value": "+1-555-***-4567",
      "verification_code": "ABC***",
      "created_at": "2024-01-16T14:20:00Z",
      "verified_at": "2024-01-16T15:30:00Z",
      "notes": "Available after 5pm weekdays"
    }
  ],
  "total_claims": 1,
  "verified_claims": 1,
  "pending_claims": 0
}
```

### 7. Match Statistics

**Endpoint**: `GET /api/v1/stats`

**Purpose**: Get system-wide matching statistics and performance metrics.

**Query Parameters**:
- `start_date` (optional): Start date for statistics (ISO 8601)
- `end_date` (optional): End date for statistics (ISO 8601)
- `user_id` (optional): Filter statistics for specific user

**Response**:
```json
{
  "time_period": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-31T23:59:59Z"
  },
  "matching_performance": {
    "total_matches_created": 1247,
    "high_confidence_matches": 432,
    "medium_confidence_matches": 581,
    "low_confidence_matches": 234,
    "confirmed_matches": 156,
    "rejected_matches": 89,
    "expired_matches": 67,
    "claimed_items": 142,
    "success_rate": 0.124,
    "average_confidence": 0.73
  },
  "timing_metrics": {
    "average_match_detection_time_ms": 145,
    "average_time_to_confirmation_hours": 8.5,
    "average_time_to_claim_hours": 12.3
  },
  "algorithm_breakdown": {
    "location_factor_avg": 0.68,
    "visual_factor_avg": 0.71,
    "text_factor_avg": 0.45,
    "temporal_factor_avg": 0.82
  },
  "user_engagement": {
    "matches_viewed": 2341,
    "matches_confirmed": 156,
    "matches_rejected": 89,
    "user_response_rate": 0.67
  }
}
```

**Example Request**:
```bash
# Get overall statistics
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/stats"

# Get statistics for specific time period
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8003/api/v1/stats?start_date=2024-01-01&end_date=2024-01-31"
```

## Error Responses

All endpoints use consistent error response format:

```json
{
  "error": {
    "code": "MATCH_NOT_FOUND",
    "message": "Match with ID 550e8400-e29b-41d4-a716-446655440000 not found",
    "details": {
      "match_id": "550e8400-e29b-41d4-a716-446655440000",
      "user_id": "user123"
    },
    "timestamp": "2024-01-16T09:15:00Z"
  }
}
```

**Common Error Codes**:
- `MATCH_NOT_FOUND` (404): Match does not exist
- `UNAUTHORIZED` (401): Invalid or missing authentication token
- `FORBIDDEN` (403): User doesn't have permission to access match
- `INVALID_REQUEST` (400): Request body validation failed
- `INVALID_STATE_TRANSITION` (409): Cannot perform action in current match state
- `RATE_LIMIT_EXCEEDED` (429): Too many requests
- `INTERNAL_ERROR` (500): Unexpected server error

## Rate Limiting

**Rate Limits**:
- List matches: 100 requests per minute per user
- Get specific match: 200 requests per minute per user
- Confirm/reject match: 20 requests per minute per user
- Claim item: 10 requests per minute per user
- Statistics: 30 requests per minute per user

**Rate Limit Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642334400
```

## Webhooks (Future)

Planned webhook support for real-time match notifications:

```json
{
  "event": "match.created",
  "data": {
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "confidence_score": 0.87,
    "match_quality": "High",
    "lost_post_id": "123e4567-e89b-12d3-a456-426614174000",
    "found_post_id": "789e0123-e45b-67c8-d901-234567890123"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Integration Examples

### Frontend Integration

```javascript
// Fetch matches for current user
async function getUserMatches(userId, confidenceMin = 0.7) {
  const response = await fetch(
    `/api/v1/matches?user_id=${userId}&confidence_min=${confidenceMin}&sort=confidence_desc`,
    {
      headers: {
        'Authorization': `Bearer ${getAuthToken()}`,
        'Content-Type': 'application/json'
      }
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to fetch matches: ${response.status}`);
  }

  return await response.json();
}

// Confirm a match
async function confirmMatch(matchId, userId, notes) {
  const response = await fetch(`/api/v1/matches/${matchId}/confirm`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${getAuthToken()}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      notes: notes
    })
  });

  if (!response.ok) {
    throw new Error(`Failed to confirm match: ${response.status}`);
  }

  return await response.json();
}
```

### Backend Integration

```python
# Python integration example
import requests
from typing import List, Dict, Optional

class MatcherAPIClient:
    def __init__(self, base_url: str, auth_token: str):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Bearer {auth_token}',
            'Content-Type': 'application/json'
        }

    def get_matches(
        self,
        user_id: Optional[str] = None,
        confidence_min: Optional[float] = None,
        status: Optional[str] = None,
        limit: int = 20
    ) -> Dict:
        params = {'limit': limit}
        if user_id:
            params['user_id'] = user_id
        if confidence_min:
            params['confidence_min'] = confidence_min
        if status:
            params['status'] = status

        response = requests.get(
            f'{self.base_url}/api/v1/matches',
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()

    def confirm_match(self, match_id: str, user_id: str, notes: str = '') -> Dict:
        response = requests.put(
            f'{self.base_url}/api/v1/matches/{match_id}/confirm',
            headers=self.headers,
            json={'user_id': user_id, 'notes': notes}
        )
        response.raise_for_status()
        return response.json()
```

---

*For domain architecture and algorithm details, see [domain-architecture.md](domain-architecture.md). For system-wide API patterns, see [../STANDARDS.md](../STANDARDS.md).*