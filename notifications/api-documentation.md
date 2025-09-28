# fn-notifications API Documentation

**REST API and real-time interfaces for enterprise notification management with multi-channel delivery orchestration.**

## Base Configuration

**Base URL**: `http://localhost:4000` (development) / `https://api.findlynow.com/notifications` (production)

**API Version**: `v1`

**Content Type**: `application/json`

**Authentication**: Bearer token (JWT) in Authorization header

**Real-time Dashboard**: `http://localhost:4000` (Phoenix LiveView interface)

## Core API Endpoints

### 1. Send Notification

**Endpoint**: `POST /api/v1/notifications`

**Purpose**: Create and send a notification through specified channels with intelligent routing.

**Request Body**:
```json
{
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "notification_type": "match_detected",
  "title": "Possible match found for your item!",
  "body": "We found a potential match for your lost iPhone with 87% confidence. Please review the match details and confirm if this looks like your item.",
  "channels": ["email", "sms"],
  "metadata": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "match_id": "match-789a0123-b45c-67d8-e901-234567890abc",
    "confidence_score": 0.87,
    "deduplication_key": "match_detected_match-789a0123"
  },
  "scheduled_at": null
}
```

**Request Validation**:
- `user_id`: Required, valid UUID
- `notification_type`: Required, must be valid notification type
- `title`: Required, 1-255 characters
- `body`: Required, 1-2000 characters
- `channels`: Optional, defaults to user's preferred channels
- `metadata`: Optional, key-value pairs for template variables
- `scheduled_at`: Optional, future timestamp for scheduled delivery

**Valid Notification Types**:
- `post_confirmation`: Post creation confirmation
- `match_detected`: Potential item match found
- `urgent_claim`: Someone wants to claim item (high priority)
- `post_resolved`: Item successfully recovered
- `welcome`: New user welcome message
- `reminder`: General reminder notification

**Valid Channels**:
- `email`: Email delivery via Swoosh
- `sms`: SMS delivery via Twilio
- `whatsapp`: WhatsApp Business delivery via Twilio

**Response**:
```json
{
  "id": "notif-abc1234-d56e-78f9-0123-456789abcdef",
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "notification_type": "match_detected",
  "title": "Possible match found for your item!",
  "body": "We found a potential match for your lost iPhone with 87% confidence...",
  "channels": ["email", "sms"],
  "status": "sent",
  "metadata": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "match_id": "match-789a0123-b45c-67d8-e901-234567890abc",
    "confidence_score": 0.87,
    "deduplication_key": "match_detected_match-789a0123"
  },
  "scheduled_at": null,
  "sent_at": "2024-01-15T14:30:00Z",
  "delivered_at": null,
  "retry_count": 0,
  "max_retries": 3,
  "delivery_attempts": [
    {
      "channel": "email",
      "status": "delivered",
      "attempted_at": "2024-01-15T14:30:01Z",
      "delivered_at": "2024-01-15T14:30:02Z",
      "provider_id": "email-12345",
      "provider_response": {"message_id": "0100018c1234"}
    },
    {
      "channel": "sms",
      "status": "sent",
      "attempted_at": "2024-01-15T14:30:01Z",
      "provider_id": "SM1234567890abcdef",
      "provider_response": {"status": "queued"}
    }
  ],
  "created_at": "2024-01-15T14:30:00Z",
  "updated_at": "2024-01-15T14:30:02Z"
}
```

**Example Requests**:
```bash
# Send urgent claim notification
curl -X POST http://localhost:4000/api/v1/notifications \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-123",
    "notification_type": "urgent_claim",
    "title": "URGENT: Someone wants to claim your item!",
    "body": "A person has initiated the claiming process...",
    "channels": ["sms", "email", "whatsapp"],
    "metadata": {
      "post_id": "post-456",
      "claim_id": "claim-789",
      "deduplication_key": "urgent_claim_claim-789"
    }
  }'

# Schedule reminder notification
curl -X POST http://localhost:4000/api/v1/notifications \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-123",
    "notification_type": "reminder",
    "title": "Remember to check for updates",
    "body": "Don'\''t forget to check your Findly Now dashboard...",
    "channels": ["email"],
    "scheduled_at": "2024-01-16T09:00:00Z"
  }'
```

### 2. List Notifications

**Endpoint**: `GET /api/v1/notifications`

**Purpose**: Retrieve notifications with filtering, sorting, and pagination.

**Query Parameters**:
- `user_id` (optional): Filter by specific user
- `notification_type` (optional): Filter by notification type
- `channel` (optional): Filter by delivery channel
- `status` (optional): Filter by status (`pending`, `sent`, `delivered`, `failed`)
- `created_after` (optional): Filter by creation date (ISO 8601)
- `created_before` (optional): Filter by creation date (ISO 8601)
- `sort` (optional): Sort order (`created_desc`, `created_asc`, `status_asc`)
- `limit` (optional): Number of results (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response**:
```json
{
  "notifications": [
    {
      "id": "notif-abc1234-d56e-78f9-0123-456789abcdef",
      "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
      "notification_type": "match_detected",
      "title": "Possible match found for your item!",
      "body": "We found a potential match...",
      "channels": ["email", "sms"],
      "status": "delivered",
      "scheduled_at": null,
      "sent_at": "2024-01-15T14:30:00Z",
      "delivered_at": "2024-01-15T14:30:02Z",
      "retry_count": 0,
      "created_at": "2024-01-15T14:30:00Z"
    }
  ],
  "pagination": {
    "total": 156,
    "limit": 20,
    "offset": 0,
    "has_more": true
  },
  "summary": {
    "total_notifications": 156,
    "delivered_today": 23,
    "failed_today": 1,
    "pending_count": 5
  }
}
```

**Example Requests**:
```bash
# Get recent notifications for user
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/notifications?user_id=user-123&sort=created_desc&limit=10"

# Get failed notifications requiring attention
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/notifications?status=failed&sort=created_desc"

# Get urgent notifications from last 24 hours
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/notifications?notification_type=urgent_claim&created_after=2024-01-14T14:30:00Z"
```

### 3. Get Specific Notification

**Endpoint**: `GET /api/v1/notifications/{id}`

**Purpose**: Retrieve detailed information about a specific notification including all delivery attempts.

**Path Parameters**:
- `id`: UUID of the notification

**Response**:
```json
{
  "id": "notif-abc1234-d56e-78f9-0123-456789abcdef",
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "notification_type": "urgent_claim",
  "title": "URGENT: Someone wants to claim your item!",
  "body": "A person has initiated the claiming process for your lost iPhone...",
  "channels": ["sms", "email", "whatsapp"],
  "status": "delivered",
  "metadata": {
    "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
    "claim_id": "claim-789a0123-b45c-67d8-e901-234567890abc",
    "deduplication_key": "urgent_claim_claim-789a0123",
    "priority": "urgent"
  },
  "scheduled_at": null,
  "sent_at": "2024-01-15T16:45:00Z",
  "delivered_at": "2024-01-15T16:45:03Z",
  "failed_at": null,
  "retry_count": 0,
  "max_retries": 3,
  "delivery_attempts": [
    {
      "id": "attempt-111",
      "channel": "sms",
      "status": "delivered",
      "attempted_at": "2024-01-15T16:45:01Z",
      "delivered_at": "2024-01-15T16:45:02Z",
      "provider_id": "SM9876543210fedcba",
      "provider_response": {
        "status": "delivered",
        "message_id": "SM9876543210fedcba",
        "to": "+1234567890",
        "cost": "0.0075"
      },
      "failure_reason": null
    },
    {
      "id": "attempt-222",
      "channel": "email",
      "status": "delivered",
      "attempted_at": "2024-01-15T16:45:01Z",
      "delivered_at": "2024-01-15T16:45:03Z",
      "provider_id": "email-67890",
      "provider_response": {
        "message_id": "0100018c5678",
        "status": "sent"
      },
      "failure_reason": null
    },
    {
      "id": "attempt-333",
      "channel": "whatsapp",
      "status": "failed",
      "attempted_at": "2024-01-15T16:45:01Z",
      "delivered_at": null,
      "provider_id": null,
      "provider_response": null,
      "failure_reason": "User not opted in to WhatsApp Business"
    }
  ],
  "user_preferences": {
    "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
    "email": "user@example.com",
    "phone": "+1234567890",
    "global_enabled": true,
    "channel_preferences": {
      "email": {"enabled": true},
      "sms": {"enabled": true},
      "whatsapp": {"enabled": false}
    },
    "timezone": "America/New_York",
    "language": "en"
  },
  "created_at": "2024-01-15T16:45:00Z",
  "updated_at": "2024-01-15T16:45:03Z"
}
```

**Example Request**:
```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/notifications/notif-abc1234-d56e-78f9-0123-456789abcdef"
```

### 4. Retry Failed Notification

**Endpoint**: `POST /api/v1/notifications/{id}/retry`

**Purpose**: Manually retry a failed notification delivery.

**Path Parameters**:
- `id`: UUID of the notification

**Request Body**:
```json
{
  "channels": ["sms"],
  "reason": "Manual retry after Twilio service restoration"
}
```

**Response**:
```json
{
  "id": "notif-abc1234-d56e-78f9-0123-456789abcdef",
  "status": "pending",
  "retry_count": 1,
  "new_attempts": [
    {
      "channel": "sms",
      "status": "pending",
      "scheduled_at": "2024-01-15T17:00:00Z"
    }
  ],
  "retry_reason": "Manual retry after Twilio service restoration",
  "retried_at": "2024-01-15T17:00:00Z",
  "retried_by": "admin-user-456"
}
```

**Example Request**:
```bash
curl -X POST -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channels": ["sms"], "reason": "Manual retry"}' \
  "http://localhost:4000/api/v1/notifications/notif-abc1234/retry"
```

## User Preferences Management

### 5. Get User Preferences

**Endpoint**: `GET /api/v1/preferences/{user_id}`

**Purpose**: Retrieve notification preferences and contact information for a user.

**Path Parameters**:
- `user_id`: UUID of the user

**Response**:
```json
{
  "id": "pref-def4567-8901-23ab-cdef-456789012345",
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "global_enabled": true,
  "email": "user@example.com",
  "phone": "+1234567890",
  "timezone": "America/New_York",
  "language": "en",
  "channel_preferences": {
    "email": {
      "enabled": true,
      "quiet_hours_respected": true
    },
    "sms": {
      "enabled": true,
      "quiet_hours_respected": false
    },
    "whatsapp": {
      "enabled": false,
      "opt_in_required": true
    }
  },
  "quiet_hours": {
    "enabled": true,
    "start_hour": 22,
    "end_hour": 8,
    "timezone": "America/New_York"
  },
  "notification_type_preferences": {
    "post_confirmation": ["email"],
    "match_detected": ["email", "sms"],
    "urgent_claim": ["sms", "email"],
    "post_resolved": ["email"],
    "welcome": ["email"],
    "reminder": ["email"]
  },
  "created_at": "2024-01-10T09:00:00Z",
  "updated_at": "2024-01-14T15:30:00Z"
}
```

**Example Request**:
```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/preferences/user-123e4567-e89b-12d3-a456-426614174000"
```

### 6. Update User Preferences

**Endpoint**: `PUT /api/v1/preferences/{user_id}`

**Purpose**: Update notification preferences and contact information.

**Path Parameters**:
- `user_id`: UUID of the user

**Request Body**:
```json
{
  "global_enabled": true,
  "email": "newemail@example.com",
  "phone": "+1987654321",
  "timezone": "America/Los_Angeles",
  "language": "es",
  "channel_preferences": {
    "email": {"enabled": true},
    "sms": {"enabled": true},
    "whatsapp": {"enabled": true}
  },
  "quiet_hours": {
    "enabled": true,
    "start_hour": 23,
    "end_hour": 7,
    "timezone": "America/Los_Angeles"
  },
  "notification_type_preferences": {
    "post_confirmation": ["email"],
    "match_detected": ["sms", "email"],
    "urgent_claim": ["sms", "whatsapp", "email"],
    "post_resolved": ["email"],
    "welcome": ["email"],
    "reminder": ["email"]
  }
}
```

**Response**:
```json
{
  "id": "pref-def4567-8901-23ab-cdef-456789012345",
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "global_enabled": true,
  "email": "newemail@example.com",
  "phone": "+1987654321",
  "timezone": "America/Los_Angeles",
  "language": "es",
  "channel_preferences": {
    "email": {"enabled": true},
    "sms": {"enabled": true},
    "whatsapp": {"enabled": true}
  },
  "quiet_hours": {
    "enabled": true,
    "start_hour": 23,
    "end_hour": 7,
    "timezone": "America/Los_Angeles"
  },
  "updated_at": "2024-01-15T17:15:00Z",
  "cache_invalidated_at": "2024-01-15T17:15:00Z"
}
```

**Example Request**:
```bash
curl -X PUT -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "newemail@example.com",
    "channel_preferences": {
      "email": {"enabled": true},
      "sms": {"enabled": true},
      "whatsapp": {"enabled": true}
    }
  }' \
  "http://localhost:4000/api/v1/preferences/user-123"
```

### 7. Test Notification Delivery

**Endpoint**: `POST /api/v1/test-delivery`

**Purpose**: Send a test notification to verify channel configuration and delivery.

**Request Body**:
```json
{
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "channels": ["email", "sms"],
  "test_type": "configuration_test"
}
```

**Valid Test Types**:
- `configuration_test`: Test channel configuration
- `delivery_speed`: Test delivery speed and latency
- `template_render`: Test template rendering with sample data

**Response**:
```json
{
  "test_id": "test-ghi7890-1234-56cd-ef78-901234567890",
  "user_id": "user-123e4567-e89b-12d3-a456-426614174000",
  "test_type": "configuration_test",
  "channels_tested": ["email", "sms"],
  "results": [
    {
      "channel": "email",
      "status": "success",
      "delivery_time_ms": 1247,
      "provider_response": {
        "message_id": "test-email-12345",
        "status": "sent"
      },
      "notes": "Email delivered successfully to newemail@example.com"
    },
    {
      "channel": "sms",
      "status": "success",
      "delivery_time_ms": 3456,
      "provider_response": {
        "message_id": "test-sms-67890",
        "status": "delivered"
      },
      "notes": "SMS delivered successfully to +1987654321"
    }
  ],
  "overall_status": "success",
  "test_message": "This is a test notification from Findly Now. If you received this, your notification settings are working correctly!",
  "executed_at": "2024-01-15T17:30:00Z",
  "execution_time_ms": 3456
}
```

**Example Request**:
```bash
curl -X POST -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-123",
    "channels": ["email", "sms"],
    "test_type": "configuration_test"
  }' \
  "http://localhost:4000/api/v1/test-delivery"
```

## System Monitoring & Analytics

### 8. Delivery Statistics

**Endpoint**: `GET /api/v1/stats`

**Purpose**: Get system-wide notification delivery statistics and performance metrics.

**Query Parameters**:
- `start_date` (optional): Start date for statistics (ISO 8601)
- `end_date` (optional): End date for statistics (ISO 8601)
- `notification_type` (optional): Filter by notification type
- `channel` (optional): Filter by delivery channel
- `granularity` (optional): Time granularity (`hour`, `day`, `week`, `month`)

**Response**:
```json
{
  "time_period": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-31T23:59:59Z",
    "granularity": "day"
  },
  "delivery_metrics": {
    "total_notifications": 12547,
    "total_attempts": 18923,
    "successful_deliveries": 11834,
    "failed_deliveries": 713,
    "pending_deliveries": 45,
    "delivery_rate": 0.943,
    "average_delivery_time_ms": 2341,
    "total_retry_attempts": 1089
  },
  "channel_breakdown": {
    "email": {
      "total_attempts": 8923,
      "successful": 8654,
      "failed": 269,
      "success_rate": 0.970,
      "avg_delivery_time_ms": 1876
    },
    "sms": {
      "total_attempts": 6547,
      "successful": 6234,
      "failed": 313,
      "success_rate": 0.952,
      "avg_delivery_time_ms": 3245
    },
    "whatsapp": {
      "total_attempts": 3453,
      "successful": 2946,
      "failed": 131,
      "success_rate": 0.853,
      "avg_delivery_time_ms": 4123
    }
  },
  "notification_type_breakdown": {
    "post_confirmation": {
      "total": 3456,
      "success_rate": 0.987,
      "primary_channels": ["email"]
    },
    "match_detected": {
      "total": 2134,
      "success_rate": 0.934,
      "primary_channels": ["email", "sms"]
    },
    "urgent_claim": {
      "total": 876,
      "success_rate": 0.912,
      "primary_channels": ["sms", "email", "whatsapp"]
    }
  },
  "failure_analysis": {
    "top_failure_reasons": [
      {
        "reason": "Invalid phone number",
        "count": 234,
        "percentage": 32.8
      },
      {
        "reason": "Email bounce - invalid address",
        "count": 156,
        "percentage": 21.9
      },
      {
        "reason": "WhatsApp user not opted in",
        "count": 123,
        "percentage": 17.3
      }
    ]
  },
  "performance_trends": [
    {
      "date": "2024-01-15",
      "total_notifications": 456,
      "success_rate": 0.934,
      "avg_delivery_time_ms": 2234
    }
  ],
  "circuit_breaker_status": {
    "email_service": "closed",
    "sms_service": "closed",
    "whatsapp_service": "half_open",
    "last_updated": "2024-01-15T17:45:00Z"
  }
}
```

**Example Requests**:
```bash
# Get overall statistics for last 30 days
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/stats?start_date=2024-01-01&end_date=2024-01-31"

# Get SMS-specific statistics
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/stats?channel=sms&granularity=day"

# Get urgent notification performance
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:4000/api/v1/stats?notification_type=urgent_claim"
```

### 9. Health Check

**Endpoint**: `GET /api/health`

**Purpose**: Comprehensive health check including external service connectivity.

**Response**:
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T18:00:00Z",
  "version": "1.2.3",
  "environment": "production",
  "checks": {
    "database": {
      "status": "healthy",
      "response_time_ms": 23,
      "details": "PostgreSQL connection successful"
    },
    "kafka": {
      "status": "healthy",
      "response_time_ms": 45,
      "details": "Confluent Cloud connectivity verified"
    },
    "email_service": {
      "status": "healthy",
      "response_time_ms": 234,
      "details": "Swoosh/SendGrid API responding"
    },
    "sms_service": {
      "status": "degraded",
      "response_time_ms": 2345,
      "details": "Twilio API responding slowly"
    },
    "whatsapp_service": {
      "status": "unhealthy",
      "response_time_ms": null,
      "details": "Twilio WhatsApp API timeout"
    },
    "cache": {
      "status": "healthy",
      "response_time_ms": 12,
      "details": "Cachex operational"
    }
  },
  "circuit_breakers": {
    "email_circuit": "closed",
    "sms_circuit": "closed",
    "whatsapp_circuit": "open"
  },
  "resource_pools": {
    "email_pool": {
      "active_connections": 3,
      "max_connections": 10,
      "utilization": 0.30
    },
    "sms_pool": {
      "active_connections": 2,
      "max_connections": 5,
      "utilization": 0.40
    }
  }
}
```

**Example Request**:
```bash
curl "http://localhost:4000/api/health"
```

## Real-time Dashboard Interface

### Phoenix LiveView Dashboard

**URL**: `http://localhost:4000/dashboard`

**Features**:
- **Real-time Notifications**: Live stream of notification deliveries
- **Channel Status**: Visual indicators for each delivery channel
- **Performance Metrics**: Real-time charts and statistics
- **Failed Delivery Management**: Interface for retrying failed notifications
- **User Preference Management**: Admin interface for user settings

**Dashboard Sections**:

1. **Overview Panel**:
   - Total notifications sent today
   - Success rate by channel
   - Average delivery time
   - Active circuit breaker status

2. **Live Activity Feed**:
   - Real-time notification delivery events
   - Failed delivery alerts
   - Retry attempt notifications
   - System health changes

3. **Analytics Charts**:
   - Delivery volume over time
   - Success rate trends
   - Channel performance comparison
   - Response time distribution

4. **Administration Tools**:
   - Manual notification sending
   - Bulk retry operations
   - User preference bulk updates
   - System configuration management

**WebSocket Events** (for custom integrations):
```javascript
// Connect to LiveView socket
const socket = new Phoenix.Socket("/live", {
  params: {token: "your-auth-token"}
});

socket.connect();

// Subscribe to notification events
const channel = socket.channel("notifications:dashboard", {});

channel.on("notification_sent", (payload) => {
  console.log("Notification sent:", payload);
});

channel.on("delivery_failed", (payload) => {
  console.log("Delivery failed:", payload);
});

channel.on("circuit_breaker_opened", (payload) => {
  console.log("Circuit breaker opened:", payload);
});

channel.join();
```

## Error Responses

All endpoints use consistent error response format:

```json
{
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "Notification with ID notif-abc1234 not found",
    "details": {
      "notification_id": "notif-abc1234",
      "user_id": "user-123"
    },
    "timestamp": "2024-01-15T18:15:00Z",
    "request_id": "req-def4567-8901-23ab"
  }
}
```

**Common Error Codes**:
- `NOTIFICATION_NOT_FOUND` (404): Notification does not exist
- `USER_PREFERENCES_NOT_FOUND` (404): User preferences not found
- `INVALID_NOTIFICATION_TYPE` (400): Unknown notification type
- `INVALID_CHANNEL` (400): Unsupported delivery channel
- `MISSING_CONTACT_INFO` (400): No email/phone for selected channels
- `DEDUPLICATION_CONFLICT` (409): Notification with same deduplication key exists
- `CIRCUIT_BREAKER_OPEN` (503): Delivery channel temporarily unavailable
- `RATE_LIMIT_EXCEEDED` (429): Too many requests
- `DELIVERY_FAILED` (502): External service delivery failure
- `INTERNAL_ERROR` (500): Unexpected server error

## Rate Limiting

**Rate Limits**:
- Send notification: 100 requests per minute per user
- List notifications: 200 requests per minute per user
- Get specific notification: 300 requests per minute per user
- Update preferences: 20 requests per minute per user
- Test delivery: 5 requests per minute per user
- Statistics: 60 requests per minute per user

**Rate Limit Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1642334400
X-RateLimit-Window: 60
```

## Authentication & Authorization

**JWT Token Requirements**:
- Valid JWT token in Authorization header: `Bearer <token>`
- Token must contain `user_id` and `roles` claims
- Different endpoints require different permission levels

**Permission Levels**:
- `user`: Can send notifications and manage own preferences
- `admin`: Can view all notifications and manage user preferences
- `system`: Can access system statistics and perform bulk operations

**Example Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
X-Request-ID: req-123e4567-e89b-12d3-a456-426614174000
```

## Integration Examples

### JavaScript/TypeScript Client

```typescript
interface NotificationRequest {
  user_id: string;
  notification_type: string;
  title: string;
  body: string;
  channels?: string[];
  metadata?: Record<string, any>;
  scheduled_at?: string;
}

interface NotificationResponse {
  id: string;
  status: string;
  delivery_attempts: DeliveryAttempt[];
  created_at: string;
}

class NotificationsAPIClient {
  constructor(
    private baseURL: string,
    private authToken: string
  ) {}

  async sendNotification(request: NotificationRequest): Promise<NotificationResponse> {
    const response = await fetch(`${this.baseURL}/api/v1/notifications`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(request)
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`Failed to send notification: ${error.error.message}`);
    }

    return await response.json();
  }

  async getUserPreferences(userId: string) {
    const response = await fetch(`${this.baseURL}/api/v1/preferences/${userId}`, {
      headers: {
        'Authorization': `Bearer ${this.authToken}`
      }
    });

    if (!response.ok) {
      throw new Error(`Failed to get preferences: ${response.status}`);
    }

    return await response.json();
  }

  async updatePreferences(userId: string, preferences: Partial<UserPreferences>) {
    const response = await fetch(`${this.baseURL}/api/v1/preferences/${userId}`, {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(preferences)
    });

    return await response.json();
  }
}

// Usage example
const client = new NotificationsAPIClient(
  'http://localhost:4000',
  'your-jwt-token'
);

// Send urgent claim notification
const notification = await client.sendNotification({
  user_id: 'user-123',
  notification_type: 'urgent_claim',
  title: 'URGENT: Someone wants to claim your item!',
  body: 'A person has initiated the claiming process...',
  channels: ['sms', 'email'],
  metadata: {
    post_id: 'post-456',
    claim_id: 'claim-789'
  }
});

console.log('Notification sent:', notification.id);
```

### Python Integration

```python
import requests
from typing import Dict, List, Optional
from datetime import datetime

class NotificationsClient:
    def __init__(self, base_url: str, auth_token: str):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Bearer {auth_token}',
            'Content-Type': 'application/json'
        }

    def send_notification(
        self,
        user_id: str,
        notification_type: str,
        title: str,
        body: str,
        channels: Optional[List[str]] = None,
        metadata: Optional[Dict] = None,
        scheduled_at: Optional[datetime] = None
    ) -> Dict:
        """Send a notification to a user."""
        payload = {
            'user_id': user_id,
            'notification_type': notification_type,
            'title': title,
            'body': body
        }

        if channels:
            payload['channels'] = channels
        if metadata:
            payload['metadata'] = metadata
        if scheduled_at:
            payload['scheduled_at'] = scheduled_at.isoformat()

        response = requests.post(
            f'{self.base_url}/api/v1/notifications',
            headers=self.headers,
            json=payload
        )
        response.raise_for_status()
        return response.json()

    def get_delivery_stats(
        self,
        start_date: Optional[datetime] = None,
        end_date: Optional[datetime] = None,
        channel: Optional[str] = None
    ) -> Dict:
        """Get delivery statistics."""
        params = {}
        if start_date:
            params['start_date'] = start_date.isoformat()
        if end_date:
            params['end_date'] = end_date.isoformat()
        if channel:
            params['channel'] = channel

        response = requests.get(
            f'{self.base_url}/api/v1/stats',
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()

# Usage example
client = NotificationsClient('http://localhost:4000', 'your-jwt-token')

# Send match detection notification
notification = client.send_notification(
    user_id='user-123',
    notification_type='match_detected',
    title='Possible match found!',
    body='We found a potential match for your lost item...',
    channels=['email', 'sms'],
    metadata={
        'post_id': 'post-456',
        'match_id': 'match-789',
        'confidence_score': 0.87
    }
)

print(f"Notification sent: {notification['id']}")

# Get delivery statistics
stats = client.get_delivery_stats(
    start_date=datetime(2024, 1, 1),
    end_date=datetime(2024, 1, 31)
)

print(f"Success rate: {stats['delivery_metrics']['delivery_rate']:.2%}")
```

---

*For domain architecture and business rules, see [domain-architecture.md](domain-architecture.md). For system-wide API patterns, see [../STANDARDS.md](../STANDARDS.md).*