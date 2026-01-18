# API Design: ActivityPub vs Application Endpoints

This document defines the API design strategy for separating ActivityPub (federation) endpoints from application (UI/internal) endpoints.

## Core Principle

**Separate ActivityPub endpoints from application endpoints** to:
- ✅ Maintain ActivityPub spec compliance (federation endpoints must be at specific paths)
- ✅ Enable UI development without affecting federation
- ✅ Clear distinction between public (ActivityPub) and application APIs
- ✅ Easier to version application API independently
- ✅ Better security boundaries (different auth requirements)

## Endpoint Organization

### ActivityPub Endpoints (Federation)

**Location**: Root level (required by ActivityPub spec)

These endpoints are **public** and must follow ActivityPub specification exactly:

```
/.well-known/webfinger
/users/{username}
/users/{username}/inbox
/users/{username}/outbox
/users/{username}/followers
/users/{username}/following
/activities/{activityId}
```

**Characteristics**:
- **Public**: Accessible to any ActivityPub server
- **Spec-compliant**: Must follow ActivityPub specification
- **Content-Type**: `application/activity+json`
- **No versioning**: ActivityPub spec doesn't support versioning
- **Stable**: Paths cannot change (breaks federation)

### Application Endpoints (UI/Internal)

**Location**: `/api/*` prefix

These endpoints are for **application use** (UI, mobile apps, internal services):

```
/api/auth/register
/api/auth/login
/api/auth/me
/api/activities
/api/activities/{id}
/api/activities/{id}/comments
/api/activities/{id}/likes
/api/stats
/api/settings
/api/users/{username}/posts
```

**Characteristics**:
- **Application-specific**: Designed for UI/mobile apps
- **Flexible**: Can evolve without breaking federation
- **Versioned**: Can add `/api/v1/*`, `/api/v2/*` if needed
- **Authentication**: Typically requires authentication
- **Content-Type**: `application/json`

## Current Endpoint Analysis

### Current ActivityPub Endpoints ✅

These are correctly placed at root level:

- `GET /.well-known/webfinger` - WebFinger discovery
- `GET /users/{username}` - Actor profile
- `POST /users/{username}/inbox` - Server-to-server inbox
- `GET /users/{username}/outbox` - Activity stream
- `POST /users/{username}/outbox` - Client-to-server (C2S)
- `GET /users/{username}/followers` - Followers collection
- `GET /users/{username}/following` - Following collection
- `GET /activities/{activityId}` - Activity by ID

### Current Application Endpoints (Need Migration)

These should move to `/api/*`:

**Current** → **Should Be**:
- `POST /users/{username}/post` → `POST /api/users/{username}/posts` (or use C2S)
- `POST /users/{username}/post/run` → `POST /api/users/{username}/posts/run` (or use C2S)
- `POST /users/{username}/post/ride` → `POST /api/users/{username}/posts/ride` (or use C2S)
- `POST /users/{username}/post/swim` → `POST /api/users/{username}/posts/swim` (or use C2S)
- `POST /users/{username}/post/walk` → `POST /api/users/{username}/posts/walk` (or use C2S)
- `GET /activities/{activityId}/comments` → `GET /api/activities/{activityId}/comments`
- `GET /activities/{activityId}/likes` → `GET /api/activities/{activityId}/likes`
- `GET /activities/{activityId}/map.svg` → `GET /api/activities/{activityId}/map.svg`

**Note**: The C2S pattern (`POST /users/{username}/outbox`) is the standard ActivityPub way to create activities. The `/api/*` endpoints are convenience endpoints for UI clients.

## Recommended API Structure

### ActivityPub Endpoints (Federation)

```
# Discovery
GET  /.well-known/webfinger?resource=acct:username@domain

# Actor
GET  /users/{username}
GET  /users/{username}/inbox
GET  /users/{username}/outbox
POST /users/{username}/inbox      # S2S: Server-to-server
POST /users/{username}/outbox     # C2S: Client-to-server
GET  /users/{username}/followers
GET  /users/{username}/following

# Activities
GET  /activities/{activityId}
```

### Application Endpoints (UI/Internal)

```
# Authentication (Part 6)
POST   /api/auth/register
POST   /api/auth/login
GET    /api/auth/me
POST   /api/auth/logout

# Activities (UI-friendly)
GET    /api/activities                    # List activities (with pagination, filtering)
GET    /api/activities/{id}              # Activity details (UI format)
GET    /api/activities/{id}/comments     # Comments (UI format)
GET    /api/activities/{id}/likes        # Likes (UI format)
POST   /api/activities/{id}/comments   # Post comment
POST   /api/activities/{id}/likes        # Like activity
DELETE /api/activities/{id}/likes        # Unlike activity

# Posts (Convenience endpoints for UI)
POST   /api/users/{username}/posts       # Create post (Note)
POST   /api/users/{username}/posts/run   # Create Run activity
POST   /api/users/{username}/posts/ride  # Create Ride activity
POST   /api/users/{username}/posts/swim  # Create Swim activity
POST   /api/users/{username}/posts/walk  # Create Walk activity

# User Management
GET    /api/users/{username}             # User profile (UI format)
PUT    /api/users/{username}             # Update profile
GET    /api/users/{username}/activities  # User's activities (UI format)
GET    /api/users/{username}/stats       # User statistics

# Feed
GET    /api/feed                         # Home feed (following feed)
GET    /api/feed/public                  # Public feed

# Settings
GET    /api/settings                     # User settings
PUT    /api/settings                     # Update settings

# Media/Attachments
GET    /api/activities/{id}/map.svg      # Map image
GET    /api/activities/{id}/map.png      # Map image (PNG)
POST   /api/media/upload                 # Upload media

# Search
GET    /api/search/users                 # Search users
GET    /api/search/activities            # Search activities
```

## Migration Strategy

### Option A: Keep Both (Recommended)

**Keep ActivityPub endpoints** for federation compatibility:
- `POST /users/{username}/outbox` - Standard C2S pattern

**Add application endpoints** for UI convenience:
- `POST /api/users/{username}/posts` - UI-friendly endpoint
- `POST /api/users/{username}/posts/run` - Type-specific endpoint

**Rationale**:
- ActivityPub clients use standard endpoints
- UI clients can use convenient `/api/*` endpoints
- Both work, choose based on client type

### Option B: Deprecate Application Endpoints

**Use only ActivityPub endpoints**:
- Remove `/users/{username}/post/*` endpoints
- UI uses `POST /users/{username}/outbox` (C2S pattern)

**Rationale**:
- Simpler API
- All clients use ActivityPub standard
- Less code to maintain

**Trade-off**:
- UI needs to construct full ActivityPub JSON
- Less convenient for simple use cases

**Recommendation**: **Option A** - Keep both for flexibility.

## Implementation Pattern

### ActivityPub Resource

```java
/**
 * ActivityPub endpoints for federation.
 * These must remain at root level for ActivityPub spec compliance.
 */
@Path("/users/{username}")
public class ActorResource {
    
    @GET
    @Produces("application/activity+json")
    public Response getActor(@PathParam("username") String username) {
        // ActivityPub Actor JSON
    }
}

@Path("/users/{username}/outbox")
public class OutboxResource {
    
    @POST
    @Consumes("application/activity+json")
    @Produces("application/activity+json")
    public Response postToOutbox(
            @PathParam("username") String username,
            JsonNode activityJson) {
        // C2S: Process activity based on type
    }
}
```

### Application Resource

```java
/**
 * Application endpoints for UI/internal use.
 * These are under /api/* prefix and can evolve independently.
 */
@Path("/api/users/{username}/posts")
public class PostResource {
    
    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public Response createPost(
            @PathParam("username") String username,
            @Valid PostRequest request) {
        // Convert to ActivityPub format internally
        // Use ActivityPubService to create activity
        // Return UI-friendly response
    }
}

@Path("/api/activities")
public class ActivityResource {
    
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response listActivities(
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("limit") @DefaultValue("20") int limit,
            @QueryParam("type") String type) {
        // Return UI-friendly activity list
        // With pagination, filtering
    }
}
```

## Response Format Differences

### ActivityPub Endpoints

**Format**: ActivityPub JSON (ActivityStreams)
**Content-Type**: `application/activity+json`

```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "actor": "https://openpace.example/users/alice",
  "object": {
    "type": "Note",
    "content": "Hello world"
  }
}
```

### Application Endpoints

**Format**: Application JSON (simplified, UI-friendly)
**Content-Type**: `application/json`

```json
{
  "id": 123,
  "type": "Note",
  "content": "Hello world",
  "author": {
    "username": "alice",
    "name": "Alice Runner"
  },
  "createdAt": "2026-01-17T10:00:00Z",
  "commentsCount": 5,
  "likesCount": 12
}
```

## Authentication Requirements

### ActivityPub Endpoints

- **Public endpoints**: WebFinger, Actor profile, Outbox GET, Followers, Following
- **Authenticated endpoints**: Outbox POST (C2S), Inbox POST (S2S uses HTTP signatures)

### Application Endpoints

- **Public endpoints**: `/api/auth/register`, `/api/auth/login`
- **Authenticated endpoints**: All other `/api/*` endpoints require authentication

## Versioning Strategy

### ActivityPub Endpoints

**No versioning**: ActivityPub spec doesn't support versioning. Endpoints must remain stable.

### Application Endpoints

**Versioning supported**: Can add version prefix if needed:
- `/api/v1/activities`
- `/api/v2/activities`

**Current**: No version prefix (implicit v1). Can add versioning later if needed.

## Benefits of Separation

### 1. **Clear Boundaries**
- Easy to identify ActivityPub vs application endpoints
- Different security requirements
- Different response formats

### 2. **Independent Evolution**
- Application API can change without affecting federation
- Can add features for UI without breaking ActivityPub compatibility
- Can version application API independently

### 3. **Better Security**
- ActivityPub endpoints: Public or HTTP signatures
- Application endpoints: Authentication required
- Different CORS policies if needed

### 4. **UI Development**
- UI can use convenient `/api/*` endpoints
- Don't need to construct full ActivityPub JSON
- Can add UI-specific features (pagination, filtering, search)

## Migration Plan

### Part 1-2: Current State
- ActivityPub endpoints at root level ✅
- Application endpoints mixed with ActivityPub (e.g., `/users/{username}/post`)

### Part 3+: Recommended Structure
- **Keep ActivityPub endpoints** at root level
- **Move application endpoints** to `/api/*`
- **Keep both** for flexibility (ActivityPub standard + UI convenience)

### Implementation Steps

1. **Create `/api/*` endpoints** alongside existing endpoints
2. **Update UI clients** to use `/api/*` endpoints
3. **Keep ActivityPub endpoints** for federation compatibility
4. **Document** which endpoints are for which purpose
5. **Optionally deprecate** old application endpoints at root level

## Examples

### Creating an Activity

**Via ActivityPub (C2S)**:
```bash
POST /users/alice/outbox
Content-Type: application/activity+json

{
  "type": "Create",
  "object": {
    "type": "Note",
    "content": "Hello world"
  }
}
```

**Via Application API**:
```bash
POST /api/users/alice/posts
Content-Type: application/json

{
  "content": "Hello world"
}
```

Both create the same activity, but the application endpoint is simpler for UI clients.

### Getting Activities

**Via ActivityPub**:
```bash
GET /users/alice/outbox
Accept: application/activity+json

# Returns OrderedCollection with ActivityPub format
```

**Via Application API**:
```bash
GET /api/activities?user=alice&page=0&limit=20
Accept: application/json

# Returns simplified JSON with pagination
```

## Package Organization

### ActivityPub Resources

```
org.openpace.core.resources.activitypub.*
  ├── WebFingerResource.java
  ├── ActorResource.java
  ├── InboxResource.java
  ├── OutboxResource.java
  └── FollowersResource.java
```

### Application Resources

```
org.openpace.core.resources.api.*
  ├── AuthResource.java
  ├── ActivityResource.java
  ├── PostResource.java
  ├── UserResource.java
  └── FeedResource.java
```

## OpenAPI Documentation

### Overview

**Application endpoints** (`/api/*`) should be documented with OpenAPI (Swagger) using `quarkus-smallrye-openapi`. This provides:
- ✅ Interactive API documentation at `/q/swagger-ui`
- ✅ OpenAPI spec at `/q/openapi`
- ✅ Better developer experience for UI/mobile app developers
- ✅ API contract definition for frontend teams

**ActivityPub endpoints** are **not** documented with OpenAPI because:
- They follow the ActivityPub specification (external documentation)
- They use `application/activity+json` (not standard JSON)
- They are stable and don't need versioned documentation
- The ActivityPub spec is the authoritative documentation

### Configuration

Add `quarkus-smallrye-openapi` to `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-openapi</artifactId>
</dependency>
```

**Configuration** (`application.properties`):

```properties
# OpenAPI Configuration
quarkus.smallrye-openapi.info-title=Open Pace API
quarkus.smallrye-openapi.info-version=1.0.0
quarkus.smallrye-openapi.info-description=Application API for Open Pace - A federated Strava alternative

# Enable Swagger UI (dev/test only)
quarkus.swagger-ui.always-include=true

# Path filter: Only document /api/* endpoints
quarkus.smallrye-openapi.paths=/api/.*
```

**Note**: The path filter ensures only application endpoints are documented, excluding ActivityPub endpoints.

### Documenting Application Endpoints

Use OpenAPI annotations to document all `/api/*` endpoints:

#### Example: Activity Resource

```java
import org.eclipse.microprofile.openapi.annotations.Operation;
import org.eclipse.microprofile.openapi.annotations.media.Content;
import org.eclipse.microprofile.openapi.annotations.media.Schema;
import org.eclipse.microprofile.openapi.annotations.parameters.Parameter;
import org.eclipse.microprofile.openapi.annotations.responses.APIResponse;
import org.eclipse.microprofile.openapi.annotations.tags.Tag;

/**
 * Application endpoints for activities.
 * These are documented with OpenAPI for UI/mobile app developers.
 */
@Path("/api/activities")
@Tag(name = "Activities", description = "Activity management endpoints")
public class ActivityResource {
    
    @GET
    @Operation(
        summary = "List activities",
        description = "Get a paginated list of activities with optional filtering"
    )
    @APIResponse(
        responseCode = "200",
        description = "List of activities",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = ActivityListResponse.class)
        )
    )
    @APIResponse(
        responseCode = "401",
        description = "Unauthorized - authentication required"
    )
    public Response listActivities(
        @Parameter(description = "Page number (0-indexed)", example = "0")
        @QueryParam("page") @DefaultValue("0") int page,
        
        @Parameter(description = "Items per page", example = "20")
        @QueryParam("limit") @DefaultValue("20") @Min(1) @Max(100) int limit,
        
        @Parameter(description = "Filter by activity type", example = "Run")
        @QueryParam("type") String type,
        
        @Parameter(description = "Filter by username", example = "alice")
        @QueryParam("username") String username
    ) {
        // Implementation
    }
    
    @GET
    @Path("/{id}")
    @Operation(
        summary = "Get activity by ID",
        description = "Get detailed information about a specific activity"
    )
    @APIResponse(
        responseCode = "200",
        description = "Activity details",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = ActivityResponse.class)
        )
    )
    @APIResponse(
        responseCode = "404",
        description = "Activity not found"
    )
    public Response getActivity(
        @Parameter(description = "Activity ID", required = true, example = "123")
        @PathParam("id") Long id
    ) {
        // Implementation
    }
}
```

#### Example: Post Resource

```java
@Path("/api/users/{username}/posts")
@Tag(name = "Posts", description = "Create posts and activities")
public class PostResource {
    
    @POST
    @Operation(
        summary = "Create a post",
        description = "Create a new Note activity (simple text post)"
    )
    @APIResponse(
        responseCode = "201",
        description = "Post created successfully",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = PostResponse.class)
        )
    )
    @APIResponse(
        responseCode = "400",
        description = "Invalid request - validation errors"
    )
    @APIResponse(
        responseCode = "401",
        description = "Unauthorized - authentication required"
    )
    @APIResponse(
        responseCode = "403",
        description = "Forbidden - can only create posts for your own account"
    )
    public Response createPost(
        @Parameter(description = "Username", required = true, example = "alice")
        @PathParam("username") String username,
        
        @Parameter(description = "Post content", required = true)
        @Valid @Schema(implementation = CreatePostRequest.class) CreatePostRequest request
    ) {
        // Implementation
    }
}
```

#### Example: Request/Response DTOs

Document request and response DTOs with schema annotations:

```java
@Schema(description = "Request to create a new post")
public class CreatePostRequest {
    
    @Schema(description = "Post content (text)", required = true, example = "Just finished a great run!")
    @NotBlank(message = "Content cannot be blank")
    @Size(max = 5000, message = "Content cannot exceed 5000 characters")
    public String content;
    
    @Schema(description = "Visibility: public, followers, or private", example = "public")
    @Pattern(regexp = "public|followers|private", message = "Visibility must be public, followers, or private")
    public String visibility = "public";
}

@Schema(description = "Activity response (UI-friendly format)")
public class ActivityResponse {
    
    @Schema(description = "Activity ID", example = "123")
    public Long id;
    
    @Schema(description = "Activity type", example = "Note")
    public String type;
    
    @Schema(description = "Activity content", example = "Just finished a great run!")
    public String content;
    
    @Schema(description = "Author information")
    public UserSummary author;
    
    @Schema(description = "Creation timestamp", example = "2026-01-17T10:00:00Z")
    public Instant createdAt;
    
    @Schema(description = "Number of comments", example = "5")
    public Integer commentsCount;
    
    @Schema(description = "Number of likes", example = "12")
    public Integer likesCount;
}
```

### Documentation Guidelines

#### ✅ Document Application Endpoints

**Always document**:
- All `/api/*` endpoints
- Request DTOs with validation constraints
- Response DTOs with example values
- Query parameters with descriptions and examples
- Path parameters with descriptions
- Error responses (400, 401, 403, 404, 500)
- Authentication requirements
- Tags for grouping related endpoints

**Example tags**:
- `Activities` - Activity management
- `Posts` - Creating posts/activities
- `Auth` - Authentication endpoints
- `Users` - User management
- `Feed` - Feed endpoints
- `Search` - Search functionality

#### ❌ Don't Document ActivityPub Endpoints

**Exclude from OpenAPI**:
- `/.well-known/webfinger`
- `/users/{username}`
- `/users/{username}/inbox`
- `/users/{username}/outbox`
- `/users/{username}/followers`
- `/users/{username}/following`
- `/activities/{activityId}`

**Why**: These follow the ActivityPub specification, which is the authoritative documentation. Use path filtering to exclude them.

### Path Filtering

Configure OpenAPI to only document `/api/*` endpoints:

```properties
# Only document application endpoints
quarkus.smallrye-openapi.paths=/api/.*
```

This ensures:
- ✅ Only application endpoints appear in Swagger UI
- ✅ ActivityPub endpoints are excluded (they have their own spec)
- ✅ Cleaner, focused API documentation

### Accessing Documentation

**Development/Test**:
- **Swagger UI**: `http://localhost:8080/q/swagger-ui`
- **OpenAPI Spec**: `http://localhost:8080/q/openapi`

**Production**:
- Disable Swagger UI (security):
  ```properties
  quarkus.swagger-ui.always-include=false
  ```
- OpenAPI spec can remain available at `/q/openapi` for API consumers

### Best Practices

1. **Use Descriptive Summaries**: Clear, concise operation summaries
2. **Provide Examples**: Include example values for all parameters and responses
3. **Document Errors**: Document all possible error responses
4. **Group with Tags**: Use tags to organize related endpoints
5. **Schema Documentation**: Document all request/response DTOs
6. **Validation Constraints**: Include validation constraints in schema descriptions
7. **Authentication**: Clearly mark which endpoints require authentication

### Example: Complete Resource Documentation

```java
@Path("/api/activities/{id}/comments")
@Tag(name = "Activities", description = "Activity management endpoints")
public class CommentResource {
    
    @GET
    @Operation(
        summary = "Get comments for an activity",
        description = "Retrieve all comments for a specific activity, ordered by creation time"
    )
    @APIResponse(
        responseCode = "200",
        description = "List of comments",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = CommentListResponse.class)
        )
    )
    @APIResponse(
        responseCode = "404",
        description = "Activity not found"
    )
    public Response getComments(
        @Parameter(description = "Activity ID", required = true, example = "123")
        @PathParam("id") Long activityId
    ) {
        // Implementation
    }
    
    @POST
    @Operation(
        summary = "Post a comment",
        description = "Add a comment to an activity. Requires authentication."
    )
    @APIResponse(
        responseCode = "201",
        description = "Comment created successfully",
        content = @Content(
            mediaType = MediaType.APPLICATION_JSON,
            schema = @Schema(implementation = CommentResponse.class)
        )
    )
    @APIResponse(
        responseCode = "400",
        description = "Invalid request - validation errors"
    )
    @APIResponse(
        responseCode = "401",
        description = "Unauthorized - authentication required"
    )
    @APIResponse(
        responseCode = "404",
        description = "Activity not found"
    )
    public Response createComment(
        @Parameter(description = "Activity ID", required = true, example = "123")
        @PathParam("id") Long activityId,
        
        @Parameter(description = "Comment content", required = true)
        @Valid @Schema(implementation = CreateCommentRequest.class) CreateCommentRequest request
    ) {
        // Implementation
    }
}
```

### Integration with Error Handling

OpenAPI documentation should align with error handling strategy:

```java
@APIResponse(
    responseCode = "400",
    description = "Validation error",
    content = @Content(
        mediaType = MediaType.APPLICATION_JSON,
        schema = @Schema(implementation = ErrorResponse.class)
    )
)
```

Where `ErrorResponse` matches the error handling format:
```json
{
  "error": "ValidationError",
  "message": "Content cannot be blank"
}
```

### Summary

**OpenAPI Documentation Strategy**:
- ✅ Document all `/api/*` endpoints with OpenAPI annotations
- ✅ Use path filtering to exclude ActivityPub endpoints
- ✅ Provide examples, descriptions, and error responses
- ✅ Group endpoints with tags
- ✅ Document request/response DTOs
- ❌ Don't document ActivityPub endpoints (they follow ActivityPub spec)

**Access**:
- Swagger UI: `/q/swagger-ui` (dev/test)
- OpenAPI Spec: `/q/openapi`

**See**: [Quarkus Tech Stack](QUARKUS_TECH_STACK.md) for `quarkus-smallrye-openapi` version details.

## Summary

**ActivityPub Endpoints** (Root level):
- Required by ActivityPub spec
- Public, federation-compatible
- Stable, no versioning
- Content-Type: `application/activity+json`
- **Not documented with OpenAPI** (follow ActivityPub spec)

**Application Endpoints** (`/api/*`):
- For UI/internal use
- Can evolve independently
- Can be versioned
- Content-Type: `application/json`
- Typically require authentication
- **Documented with OpenAPI** (Swagger UI at `/q/swagger-ui`)

**Recommendation**: Keep both - ActivityPub for federation, `/api/*` for UI convenience. Document application endpoints with OpenAPI for better developer experience.
