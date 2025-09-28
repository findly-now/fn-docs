# fn-posts API Documentation

**REST API for photo-first lost & found post management with geospatial search capabilities.**

## Base Configuration

**Base URL**: `http://localhost:8080` (development) / `https://api.findlynow.com/posts` (production)

**API Version**: `v1`

**Content Type**: `application/json`

**Authentication**: Bearer token (JWT) in Authorization header

## Core Endpoints

### 1. Create Post

**Endpoint**: `POST /api/posts`

**Purpose**: Create a new lost or found post with photos and location data.

**Request Body**:
```json
{
  "title": "Lost iPhone 15 Pro",
  "description": "Black iPhone with blue MagSafe case, lost near Central Park fountain around 2pm",
  "post_type": "lost",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
  "photos": [
    {
      "data": "iVBORw0KGgoAAAANSUhEUgAA...",
      "content_type": "image/jpeg"
    },
    {
      "data": "iVBORw0KGgoAAAANSUhEUgAA...",
      "content_type": "image/png"
    }
  ]
}
```

**Request Validation**:
- `title`: Required, 1-200 characters
- `description`: Optional, max 500 characters
- `post_type`: Required, must be "lost" or "found"
- `location.lat`: Required, valid latitude (-90 to 90)
- `location.lng`: Required, valid longitude (-180 to 180)
- `radius_meters`: Optional, 100-50000 (defaults to 1000)
- `photos`: Required, 1-10 photos maximum
- `photos[].data`: Required, base64-encoded image data
- `photos[].content_type`: Required, must be image/jpeg, image/png, or image/webp

**Response**:
```json
{
  "id": "post-456e7890-e12b-34c5-a678-901234567890",
  "title": "Lost iPhone 15 Pro",
  "description": "Black iPhone with blue MagSafe case, lost near Central Park fountain around 2pm",
  "post_type": "lost",
  "status": "active",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "photos": [
    {
      "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
      "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
      "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
      "content_type": "image/jpeg",
      "display_order": 1,
      "size_bytes": 2048576,
      "created_at": "2024-01-15T14:30:00Z"
    },
    {
      "id": "photo-abc1234-d56e-78f9-0123-456789abcdef",
      "url": "https://storage.googleapis.com/fn-photos/org123/post456/photoabc.png",
      "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photoabc.png",
      "content_type": "image/png",
      "display_order": 2,
      "size_bytes": 1536000,
      "created_at": "2024-01-15T14:30:01Z"
    }
  ],
  "user_id": "user-123a4567-b89c-12d3-e456-789012345678",
  "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
  "created_at": "2024-01-15T14:30:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

**Example Request**:
```bash
curl -X POST http://localhost:8080/api/posts \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Lost iPhone 15 Pro",
    "description": "Black iPhone with blue case",
    "post_type": "lost",
    "location": {"lat": 40.7831, "lng": -73.9665},
    "radius_meters": 1000,
    "photos": [{"data": "base64...jpg", "content_type": "image/jpeg"}]
  }'
```

### 2. Search Nearby Posts

**Endpoint**: `GET /api/posts/nearby`

**Purpose**: Find posts within a specified radius using geospatial search.

**Query Parameters**:
- `lat` (required): Latitude of search center
- `lng` (required): Longitude of search center
- `radius` (optional): Search radius in meters (default: 5000, max: 50000)
- `type` (optional): Filter by post type ("lost" or "found")
- `status` (optional): Filter by status (default: "active")
- `organization_id` (optional): Filter by organization
- `limit` (optional): Number of results (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response**:
```json
{
  "posts": [
    {
      "id": "post-456e7890-e12b-34c5-a678-901234567890",
      "title": "Found black iPhone",
      "description": "Found iPhone with blue case near fountain",
      "post_type": "found",
      "status": "active",
      "location": {
        "lat": 40.7835,
        "lng": -73.9660
      },
      "distance_meters": 58.2,
      "radius_meters": 1000,
      "photos": [
        {
          "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
          "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
          "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
          "content_type": "image/jpeg",
          "display_order": 1
        }
      ],
      "user_id": "user-987f6543-c21e-10d9-b876-543210987654",
      "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2024-01-15T15:45:00Z",
      "updated_at": "2024-01-15T15:45:00Z"
    }
  ],
  "pagination": {
    "total": 1,
    "limit": 20,
    "offset": 0,
    "has_more": false
  },
  "search_metadata": {
    "center": {
      "lat": 40.7831,
      "lng": -73.9665
    },
    "radius_meters": 5000,
    "query_time_ms": 45
  }
}
```

**Example Requests**:
```bash
# Basic nearby search
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts/nearby?lat=40.7831&lng=-73.9665&radius=1000"

# Search for found items only within 500 meters
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts/nearby?lat=40.7831&lng=-73.9665&radius=500&type=found"

# Organization-specific search
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts/nearby?lat=40.7831&lng=-73.9665&organization_id=org-123"
```

### 3. List Posts

**Endpoint**: `GET /api/posts`

**Purpose**: List posts with filtering, sorting, and pagination.

**Query Parameters**:
- `type` (optional): Filter by post type ("lost" or "found")
- `status` (optional): Filter by status ("active", "resolved", "expired", "deleted")
- `user_id` (optional): Filter by post creator
- `organization_id` (optional): Filter by organization
- `created_after` (optional): Filter posts created after timestamp (ISO 8601)
- `created_before` (optional): Filter posts created before timestamp (ISO 8601)
- `sort` (optional): Sort order ("created_desc", "created_asc", "updated_desc", "updated_asc")
- `limit` (optional): Number of results (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response**:
```json
{
  "posts": [
    {
      "id": "post-456e7890-e12b-34c5-a678-901234567890",
      "title": "Lost iPhone 15 Pro",
      "description": "Black iPhone with blue case",
      "post_type": "lost",
      "status": "active",
      "location": {
        "lat": 40.7831,
        "lng": -73.9665
      },
      "radius_meters": 1000,
      "photos": [
        {
          "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
          "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
          "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
          "content_type": "image/jpeg",
          "display_order": 1
        }
      ],
      "user_id": "user-123a4567-b89c-12d3-e456-789012345678",
      "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2024-01-15T14:30:00Z",
      "updated_at": "2024-01-15T14:30:00Z"
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
# Get recent lost items
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts?type=lost&status=active&sort=created_desc&limit=10"

# Get posts from specific user
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts?user_id=user-123&sort=created_desc"

# Get posts created in last 24 hours
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts?created_after=2024-01-14T14:30:00Z"
```

### 4. Get Specific Post

**Endpoint**: `GET /api/posts/{id}`

**Purpose**: Retrieve detailed information about a specific post.

**Path Parameters**:
- `id`: UUID of the post

**Response**:
```json
{
  "id": "post-456e7890-e12b-34c5-a678-901234567890",
  "title": "Lost iPhone 15 Pro",
  "description": "Black iPhone with blue MagSafe case, lost near Central Park fountain around 2pm. Has distinctive scratch on back camera.",
  "post_type": "lost",
  "status": "active",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "photos": [
    {
      "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
      "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
      "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
      "content_type": "image/jpeg",
      "display_order": 1,
      "size_bytes": 2048576,
      "created_at": "2024-01-15T14:30:00Z"
    }
  ],
  "user_id": "user-123a4567-b89c-12d3-e456-789012345678",
  "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
  "created_at": "2024-01-15T14:30:00Z",
  "updated_at": "2024-01-15T14:30:00Z",
  "view_count": 23,
  "contact_requests": 2
}
```

**Example Request**:
```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts/post-456e7890-e12b-34c5-a678-901234567890"
```

### 5. Update Post

**Endpoint**: `PUT /api/posts/{id}`

**Purpose**: Update post title and description (location and photos cannot be changed).

**Path Parameters**:
- `id`: UUID of the post

**Request Body**:
```json
{
  "title": "Lost iPhone 15 Pro - REWARD OFFERED",
  "description": "Black iPhone with blue MagSafe case, lost near Central Park fountain around 2pm. Has distinctive scratch on back camera. $100 reward for safe return."
}
```

**Response**:
```json
{
  "id": "post-456e7890-e12b-34c5-a678-901234567890",
  "title": "Lost iPhone 15 Pro - REWARD OFFERED",
  "description": "Black iPhone with blue MagSafe case, lost near Central Park fountain around 2pm. Has distinctive scratch on back camera. $100 reward for safe return.",
  "post_type": "lost",
  "status": "active",
  "location": {
    "lat": 40.7831,
    "lng": -73.9665
  },
  "radius_meters": 1000,
  "photos": [
    {
      "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
      "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
      "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
      "content_type": "image/jpeg",
      "display_order": 1
    }
  ],
  "user_id": "user-123a4567-b89c-12d3-e456-789012345678",
  "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
  "created_at": "2024-01-15T14:30:00Z",
  "updated_at": "2024-01-15T16:45:00Z"
}
```

**Example Request**:
```bash
curl -X PUT -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "Lost iPhone 15 Pro - REWARD OFFERED", "description": "Updated description with reward info"}' \
  "http://localhost:8080/api/posts/post-456e7890-e12b-34c5-a678-901234567890"
```

### 6. Update Post Status

**Endpoint**: `PATCH /api/posts/{id}/status`

**Purpose**: Update the status of a post (active, resolved, expired, deleted).

**Path Parameters**:
- `id`: UUID of the post

**Request Body**:
```json
{
  "status": "resolved",
  "reason": "Item was successfully returned to owner"
}
```

**Valid Status Transitions**:
- `active` → `resolved`, `expired`, `deleted`
- `resolved` → `active`, `deleted`
- `expired` → `active`, `deleted`
- `deleted` → (terminal state, no transitions allowed)

**Response**:
```json
{
  "id": "post-456e7890-e12b-34c5-a678-901234567890",
  "status": "resolved",
  "previous_status": "active",
  "updated_at": "2024-01-16T09:15:00Z",
  "updated_by": "user-123a4567-b89c-12d3-e456-789012345678",
  "reason": "Item was successfully returned to owner"
}
```

**Example Requests**:
```bash
# Mark post as resolved
curl -X PATCH -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "resolved", "reason": "Item recovered"}' \
  "http://localhost:8080/api/posts/post-456/status"

# Mark post as expired
curl -X PATCH -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "expired", "reason": "30 days elapsed"}' \
  "http://localhost:8080/api/posts/post-456/status"
```

### 7. Delete Post

**Endpoint**: `DELETE /api/posts/{id}`

**Purpose**: Soft delete a post (sets status to 'deleted', doesn't remove data).

**Path Parameters**:
- `id`: UUID of the post

**Request Body**:
```json
{
  "reason": "Duplicate post created by mistake"
}
```

**Response**:
```json
{
  "id": "post-456e7890-e12b-34c5-a678-901234567890",
  "status": "deleted",
  "deleted_at": "2024-01-16T10:30:00Z",
  "deleted_by": "user-123a4567-b89c-12d3-e456-789012345678",
  "reason": "Duplicate post created by mistake"
}
```

**Example Request**:
```bash
curl -X DELETE -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason": "Duplicate post"}' \
  "http://localhost:8080/api/posts/post-456e7890-e12b-34c5-a678-901234567890"
```

### 8. Upload Additional Photo

**Endpoint**: `POST /api/posts/{postId}/photos`

**Purpose**: Add additional photos to an existing post (up to 10 total).

**Path Parameters**:
- `postId`: UUID of the post

**Request Body**:
```json
{
  "data": "iVBORw0KGgoAAAANSUhEUgAA...",
  "content_type": "image/jpeg",
  "caption": "Another angle showing the distinctive scratch"
}
```

**Response**:
```json
{
  "id": "photo-def4567-8901-23ab-cdef-456789012345",
  "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
  "url": "https://storage.googleapis.com/fn-photos/org123/post456/photodef.jpg",
  "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photodef.jpg",
  "content_type": "image/jpeg",
  "display_order": 2,
  "caption": "Another angle showing the distinctive scratch",
  "size_bytes": 1875456,
  "created_at": "2024-01-15T16:20:00Z"
}
```

**Example Request**:
```bash
curl -X POST -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": "base64...jpg", "content_type": "image/jpeg", "caption": "Detail shot"}' \
  "http://localhost:8080/api/posts/post-456/photos"
```

### 9. Delete Photo

**Endpoint**: `DELETE /api/posts/{postId}/photos/{photoId}`

**Purpose**: Remove a photo from a post (must have at least 1 photo remaining).

**Path Parameters**:
- `postId`: UUID of the post
- `photoId`: UUID of the photo

**Response**:
```json
{
  "id": "photo-def4567-8901-23ab-cdef-456789012345",
  "deleted_at": "2024-01-16T11:45:00Z",
  "deleted_by": "user-123a4567-b89c-12d3-e456-789012345678"
}
```

**Example Request**:
```bash
curl -X DELETE -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/posts/post-456/photos/photo-def4567-8901-23ab-cdef-456789012345"
```

### 10. Get User's Posts

**Endpoint**: `GET /api/users/{userId}/posts`

**Purpose**: Retrieve all posts created by a specific user across organizations.

**Path Parameters**:
- `userId`: UUID of the user

**Query Parameters**:
- `status` (optional): Filter by status
- `type` (optional): Filter by post type
- `organization_id` (optional): Filter by specific organization
- `limit` (optional): Number of results (default: 20, max: 100)
- `offset` (optional): Pagination offset

**Response**:
```json
{
  "posts": [
    {
      "id": "post-456e7890-e12b-34c5-a678-901234567890",
      "title": "Lost iPhone 15 Pro",
      "post_type": "lost",
      "status": "active",
      "location": {
        "lat": 40.7831,
        "lng": -73.9665
      },
      "photos": [
        {
          "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
          "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg"
        }
      ],
      "organization_id": "org-123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2024-01-15T14:30:00Z"
    }
  ],
  "pagination": {
    "total": 8,
    "limit": 20,
    "offset": 0,
    "has_more": false
  }
}
```

**Example Request**:
```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
  "http://localhost:8080/api/users/user-123a4567-b89c-12d3-e456-789012345678/posts?status=active"
```

## Error Responses

All endpoints use consistent error response format:

```json
{
  "error": {
    "code": "POST_NOT_FOUND",
    "message": "Post with ID post-456e7890-e12b-34c5-a678-901234567890 not found",
    "details": {
      "post_id": "post-456e7890-e12b-34c5-a678-901234567890",
      "user_id": "user-123a4567-b89c-12d3-e456-789012345678"
    },
    "timestamp": "2024-01-16T09:15:00Z"
  }
}
```

**Common Error Codes**:
- `POST_NOT_FOUND` (404): Post does not exist or user lacks access
- `INVALID_PHOTO_COUNT` (400): Too many photos (max 10) or no photos provided
- `INVALID_PHOTO_FORMAT` (400): Unsupported image format or corrupted data
- `INVALID_LOCATION` (400): Invalid GPS coordinates
- `INVALID_RADIUS` (400): Radius outside allowed range (100-50000m)
- `INVALID_STATUS_TRANSITION` (409): Cannot transition from current status to requested status
- `UNAUTHORIZED` (401): Invalid or missing authentication token
- `FORBIDDEN` (403): User doesn't have permission for this organization
- `PAYLOAD_TOO_LARGE` (413): Photo exceeds size limit (10MB per photo)
- `RATE_LIMIT_EXCEEDED` (429): Too many requests
- `INTERNAL_ERROR` (500): Unexpected server error

## Rate Limiting

**Rate Limits**:
- Create post: 10 requests per hour per user
- Search nearby: 100 requests per minute per user
- List posts: 60 requests per minute per user
- Get post: 200 requests per minute per user
- Update operations: 30 requests per minute per user
- Photo upload: 20 requests per hour per user

**Rate Limit Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642334400
X-RateLimit-Window: 60
```

## Authentication & Authorization

**JWT Token Requirements**:
- Valid JWT token in Authorization header: `Bearer <token>`
- Token must contain `user_id` and `organization_id` claims
- Token expiration validated on each request

**Authorization Rules**:
- Users can create posts within their organization
- Users can view all active posts within their organization
- Users can only modify their own posts
- Organization admins can view/modify all posts in their organization
- Super admins can access posts across all organizations

**Example Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
X-Organization-ID: org-123e4567-e89b-12d3-a456-426614174000
```

## Response Formats

**Success Response Envelope**:
```json
{
  "data": { /* response data */ },
  "metadata": {
    "timestamp": "2024-01-16T09:15:00Z",
    "request_id": "req-789a0123-b45c-67d8-e901-234567890abc",
    "version": "v1"
  }
}
```

**Pagination Format**:
```json
{
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 40,
    "has_more": true,
    "next_offset": 60,
    "prev_offset": 20
  }
}
```

**Photo Format**:
```json
{
  "id": "photo-789a0123-b45c-67d8-e901-234567890abc",
  "url": "https://storage.googleapis.com/fn-photos/org123/post456/photo789.jpg",
  "thumbnail_url": "https://storage.googleapis.com/fn-photos/org123/post456/thumb_photo789.jpg",
  "content_type": "image/jpeg",
  "display_order": 1,
  "caption": "Front view of the device",
  "size_bytes": 2048576,
  "width": 1920,
  "height": 1080,
  "created_at": "2024-01-15T14:30:00Z"
}
```

## Integration Examples

### Frontend JavaScript Integration

```javascript
class PostsAPIClient {
  constructor(baseURL, authToken) {
    this.baseURL = baseURL;
    this.authToken = authToken;
  }

  async createPost(postData) {
    const response = await fetch(`${this.baseURL}/api/posts`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(postData)
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`Failed to create post: ${error.error.message}`);
    }

    return await response.json();
  }

  async searchNearby(lat, lng, radius = 5000, filters = {}) {
    const params = new URLSearchParams({
      lat: lat.toString(),
      lng: lng.toString(),
      radius: radius.toString(),
      ...filters
    });

    const response = await fetch(`${this.baseURL}/api/posts/nearby?${params}`, {
      headers: {
        'Authorization': `Bearer ${this.authToken}`
      }
    });

    if (!response.ok) {
      throw new Error(`Search failed: ${response.status}`);
    }

    return await response.json();
  }

  async uploadPhoto(postId, photoFile) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = async () => {
        const base64Data = reader.result.split(',')[1]; // Remove data:image/jpeg;base64, prefix

        try {
          const response = await fetch(`${this.baseURL}/api/posts/${postId}/photos`, {
            method: 'POST',
            headers: {
              'Authorization': `Bearer ${this.authToken}`,
              'Content-Type': 'application/json'
            },
            body: JSON.stringify({
              data: base64Data,
              content_type: photoFile.type
            })
          });

          if (!response.ok) {
            const error = await response.json();
            reject(new Error(`Photo upload failed: ${error.error.message}`));
          } else {
            resolve(await response.json());
          }
        } catch (error) {
          reject(error);
        }
      };
      reader.readAsDataURL(photoFile);
    });
  }
}

// Usage example
const client = new PostsAPIClient('http://localhost:8080', 'your-jwt-token');

// Create a new lost item post
const newPost = await client.createPost({
  title: "Lost iPhone 15 Pro",
  description: "Black iPhone with blue case",
  post_type: "lost",
  location: { lat: 40.7831, lng: -73.9665 },
  radius_meters: 1000,
  photos: [{
    data: "base64-encoded-image-data",
    content_type: "image/jpeg"
  }]
});

// Search for nearby found items
const nearbyPosts = await client.searchNearby(40.7831, -73.9665, 2000, {
  type: 'found',
  status: 'active'
});
```

### Mobile App Integration (React Native)

```javascript
import { launchImageLibrary } from 'react-native-image-picker';

const PostCreationScreen = () => {
  const [photos, setPhotos] = useState([]);
  const [location, setLocation] = useState(null);

  const selectPhotos = () => {
    launchImageLibrary(
      {
        mediaType: 'photo',
        maxWidth: 1920,
        maxHeight: 1920,
        quality: 0.8,
        selectionLimit: 10
      },
      (response) => {
        if (response.assets) {
          const photosData = response.assets.map(asset => ({
            data: asset.base64,
            content_type: asset.type
          }));
          setPhotos(photosData);
        }
      }
    );
  };

  const getCurrentLocation = () => {
    navigator.geolocation.getCurrentPosition(
      (position) => {
        setLocation({
          lat: position.coords.latitude,
          lng: position.coords.longitude
        });
      },
      (error) => console.error('Location error:', error),
      { enableHighAccuracy: true, timeout: 20000, maximumAge: 1000 }
    );
  };

  const createPost = async (title, description, postType) => {
    if (!location) {
      alert('Please enable location services');
      return;
    }

    if (photos.length === 0) {
      alert('Please add at least one photo');
      return;
    }

    try {
      const postData = {
        title,
        description,
        post_type: postType,
        location,
        radius_meters: 1000,
        photos
      };

      const result = await client.createPost(postData);
      console.log('Post created:', result);
      // Navigate to success screen or post detail
    } catch (error) {
      console.error('Failed to create post:', error);
      alert('Failed to create post. Please try again.');
    }
  };

  return (
    // UI components for photo selection, location, and post creation
  );
};
```

---

*For domain architecture and business rules, see [domain-architecture.md](domain-architecture.md). For system-wide API patterns, see [../STANDARDS.md](../STANDARDS.md).*