# Database Design & Modeling

This document defines the database schema design, entity relationships, indexing strategy, and query patterns for Open Pace.

## Overview

**Database**: PostgreSQL with PostGIS (Part 7+)
**ORM**: Hibernate ORM with Panache
**Migrations**: Flyway
**Dev Services**: Automatic PostgreSQL container in dev/test mode

## Core Schema Design

### Entity Relationships

```
actors (1) ──< (many) activities
  │
  └──< (many) followers (many) >── actors (followers)
  
activities (1) ──< (many) comments
activities (1) ──< (many) likes
activities (1) ──< (many) attachments
```

### Part 1: Basic Schema

#### Actors Table

```sql
CREATE TABLE actors (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_actors_username ON actors(username);
```

**Design Points**:
- `username` is unique and indexed for fast lookups
- Supports both local and remote actors (remote actors have full URL in username or separate field)

#### Activities Table

```sql
CREATE TABLE activities (
    id BIGSERIAL PRIMARY KEY,
    actor_id BIGINT REFERENCES actors(id) NOT NULL,
    activity_type VARCHAR(50) NOT NULL, -- "Create", "Follow", "Like", etc.
    activity_id VARCHAR(500) UNIQUE NOT NULL, -- ActivityPub ID (URL)
    object_type VARCHAR(100), -- "Note", "Run", "Ride", etc.
    object_content TEXT, -- For simple Note objects (Part 1)
    object_json JSONB, -- For custom types (Part 2+)
    object_id VARCHAR(500), -- ActivityPub object ID (URL)
    published_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_activities_actor_id ON activities(actor_id);
CREATE INDEX idx_activities_published_at ON activities(published_at DESC);
CREATE INDEX idx_activities_activity_id ON activities(activity_id);
CREATE INDEX idx_activities_object_id ON activities(object_id);
CREATE INDEX idx_activities_object_json ON activities USING GIN (object_json); -- For JSONB queries
```

**Design Points**:
- Foreign key to `actors` for efficient joins
- `published_at` indexed for chronological ordering
- `activity_id` and `object_id` indexed for ActivityPub lookups
- `object_json` has GIN index for JSONB property queries
- Dual storage: `object_content` (TEXT) for Notes, `object_json` (JSONB) for custom types

#### Followers Table

```sql
CREATE TABLE followers (
    id BIGSERIAL PRIMARY KEY,
    actor_id BIGINT REFERENCES actors(id) NOT NULL,
    follower_actor_id BIGINT REFERENCES actors(id), -- NULL for remote followers
    follower_actor_url VARCHAR(500) NOT NULL,
    follower_inbox VARCHAR(500) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(actor_id, follower_actor_url) -- Prevent duplicate follows
);

CREATE INDEX idx_followers_actor_id ON followers(actor_id);
CREATE INDEX idx_followers_follower_url ON followers(follower_actor_url);
```

**Design Points**:
- `follower_actor_id` can be NULL for remote followers (Part 1 simplification)
- Unique constraint on `(actor_id, follower_actor_url)` prevents duplicate follows
- Indexed for efficient "who follows this actor" queries

### Part 2: Custom Types (JSONB Storage)

#### JSONB Storage Strategy

**Decision**: Store custom activity objects as JSONB to preserve all properties.

**Rationale**:
- Custom objects have variable schemas (Run has different fields than Ride)
- Preserves all data without schema changes
- Allows querying object properties directly via PostgreSQL JSONB operators
- Flexible for future schema evolution
- No data loss when storing/retrieving

**Implementation**:
```sql
-- Add JSONB column (if not already present)
ALTER TABLE activities ADD COLUMN object_json JSONB;

-- GIN index for JSONB queries
CREATE INDEX idx_activities_object_json ON activities USING GIN (object_json);
```

**Entity Field**:
```java
@Column(name = "object_json", columnDefinition = "jsonb")
@JdbcTypeCode(SqlTypes.JSON)
public com.fasterxml.jackson.databind.JsonNode objectJson;
```

**Key Benefits**:
- Can query: `SELECT * FROM activities WHERE object_json->>'distance' > '5000'`
- Preserves JSON-LD context (`@context` field)
- Maintains all custom properties exactly as received
- No need to update schema when adding new activity types

**Trade-offs**:
- Less type safety (JsonNode vs strongly-typed classes)
- Requires careful handling when updating (must preserve all fields)
- JSONB queries are slower than indexed columns (but acceptable for this use case)

#### Dual Storage Strategy

**Decision**: Maintain both `object_content` (TEXT) for Notes and `object_json` (JSONB) for custom types.

**Rationale**:
- Backward compatibility with Part 1's simple Note storage
- Notes are simple (just text content), don't need JSONB overhead
- Custom types need full JSON preservation
- Clear separation: simple vs complex objects

**Retrieval Logic**:
```java
if (dbActivity.objectJson != null) {
    // Use stored JSON (custom type)
    activity.object = dbActivity.objectJson;
} else {
    // Reconstruct Note from objectContent
    Note note = new Note();
    note.content = dbActivity.objectContent;
    activity.object = note;
}
```

### Part 3: Comments, Likes, Attachments

#### Comments Table

```sql
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    activity_id BIGINT REFERENCES activities(id) NOT NULL,
    comment_activity_id BIGINT REFERENCES activities(id) NOT NULL,
    commenter_actor_url VARCHAR(500) NOT NULL,
    in_reply_to VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_activity_id ON comments(activity_id);
CREATE INDEX idx_comments_commenter ON comments(commenter_actor_url);
```

**Design Points**:
- Links comment activity to target activity
- `comment_activity_id` references the Create activity that is the comment
- `in_reply_to` stored for ActivityPub compliance
- Indexed for efficient "comments for activity" queries

#### Likes Table

```sql
CREATE TABLE likes (
    id BIGSERIAL PRIMARY KEY,
    activity_id BIGINT REFERENCES activities(id) NOT NULL,
    like_activity_id BIGINT REFERENCES activities(id) NOT NULL,
    liker_actor_url VARCHAR(500) NOT NULL,
    object_id VARCHAR(500) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(activity_id, liker_actor_url) -- One like per actor per activity
);

CREATE INDEX idx_likes_activity_id ON likes(activity_id);
CREATE INDEX idx_likes_liker ON likes(liker_actor_url);
```

**Design Points**:
- Unique constraint prevents duplicate likes from same actor
- Indexed for efficient "who liked this" queries
- `like_activity_id` references the Like activity record

#### Attachments Table

```sql
CREATE TABLE attachments (
    id BIGSERIAL PRIMARY KEY,
    activity_id BIGINT REFERENCES activities(id) NOT NULL,
    attachment_type VARCHAR(50), -- 'map', 'preview', etc.
    url VARCHAR(500) NOT NULL,
    media_type VARCHAR(100), -- 'image/svg+xml', etc.
    width INTEGER,
    height INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_attachments_activity_id ON attachments(activity_id);
```

**Design Points**:
- Stores attachment metadata (URL, media type, dimensions)
- Actual content generated on-demand (not stored)
- Can extend to support file storage in future

### Part 7: Geospatial Data (PostGIS)

#### Geospatial Columns

```sql
-- Add PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Add geospatial columns to activities
ALTER TABLE activities ADD COLUMN track_line LINESTRING;
ALTER TABLE activities ADD COLUMN start_point POINT;
ALTER TABLE activities ADD COLUMN end_point POINT;

-- Spatial indexes
CREATE INDEX idx_activities_track_line ON activities USING GIST (track_line);
CREATE INDEX idx_activities_start_point ON activities USING GIST (start_point);
CREATE INDEX idx_activities_end_point ON activities USING GIST (end_point);
```

**Design Points**:
- `track_line`: Simplified GPX track as LineString
- `start_point` and `end_point`: Start and end coordinates
- GIST indexes for efficient spatial queries
- See [Mapping Integration](MAPPING_INTEGRATION.md) for details

### Part 8: HTTP Signatures (Actor Keys)

#### Actor Keys Table

```sql
CREATE TABLE actor_keys (
    id BIGSERIAL PRIMARY KEY,
    actor_id BIGINT REFERENCES actors(id) NOT NULL,
    key_id VARCHAR(500) UNIQUE NOT NULL,  -- Full URL: https://domain/users/username#main-key
    private_key_pem VARCHAR(500) NOT NULL,  -- Vault path reference (e.g., "actor-keys/{actorId}/private-key")
    public_key_pem TEXT NOT NULL,   -- Public key (stored in database, can be public)
    algorithm VARCHAR(50) NOT NULL DEFAULT 'rsa-sha256',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_actor_keys_actor_id ON actor_keys(actor_id);
CREATE INDEX idx_actor_keys_key_id ON actor_keys(key_id);
```

**Design Points**:
- **One-to-one relationship**: Each actor has one key pair (via `actor_id`)
- **Private key storage**: `private_key_pem` stores a Vault path reference, NOT the actual key
  - Example: `"actor-keys/123/private-key"` points to Vault secret
  - Actual private key stored in HashiCorp Vault (encrypted at rest)
- **Public key storage**: `public_key_pem` stores the actual public key (safe to store in database)
- **Key ID**: Full ActivityPub URL format (`https://domain/users/username#main-key`)
- **Algorithm**: Default RSA-SHA256, supports Ed25519 for future
- **Indexes**: Fast lookups by actor and key ID

**Security**:
- ✅ Private keys never stored in database
- ✅ Vault handles encryption at rest
- ✅ Access control via Vault policies
- ✅ See [HTTP Signatures Strategy](HTTP_SIGNATURES_STRATEGY.md) for details

**Migration**:
```sql
-- Part 8 migration: Add actor_keys table
-- V8__add_actor_keys.sql
```

## Indexing Strategy

### Primary Indexes

**Actors**:
- `username` (UNIQUE) - Fast actor lookups

**Actor Keys** (Part 8+):
- `actor_id` - Join to actors
- `key_id` (UNIQUE) - ActivityPub key ID lookups

**Activities**:
- `actor_id` - Join to actors
- `published_at DESC` - Chronological ordering
- `activity_id` (UNIQUE) - ActivityPub ID lookups
- `object_id` - Object ID lookups
- `object_json` (GIN) - JSONB property queries

**Followers**:
- `actor_id` - "Who follows this actor"
- `follower_actor_url` - "Who does this actor follow"

**Comments**:
- `activity_id` - "Comments for activity"

**Likes**:
- `activity_id` - "Likes for activity"
- `(activity_id, liker_actor_url)` (UNIQUE) - Prevent duplicates

### Query Patterns

#### Get Actor's Activities (Chronological)

```java
List<Activity> activities = Activity.find(
    "actor = ?1 ORDER BY published_at DESC",
    actor
).list();
```

**Index Used**: `idx_activities_actor_id`, `idx_activities_published_at`

#### Get Comments for Activity

```java
List<Comment> comments = Comment.find(
    "activity = ?1 ORDER BY created_at ASC",
    activity
).list();
```

**Index Used**: `idx_comments_activity_id`

#### Query Custom Type Properties

```sql
SELECT * FROM activities 
WHERE object_json->>'distance' > '5000' 
AND object_type = 'Run';
```

**Index Used**: `idx_activities_object_json` (GIN index)

#### Get Followers

```java
List<Follower> followers = Follower.find(
    "actor = ?1",
    actor
).list();
```

**Index Used**: `idx_followers_actor_id`

## Migration Strategy

### Flyway Migrations

**Pattern**: `V{version}__{description}.sql`

**Part 1**:
- `V1__initial_schema.sql` - Actors, activities, followers

**Part 2**:
- `V2__add_jsonb_column.sql` - Add `object_json` column and index

**Part 3**:
- `V3__add_comments_likes_attachments.sql` - Comments, likes, attachments tables

**Part 7**:
- `V4__add_postgis_columns.sql` - PostGIS extension and geospatial columns

**Part 8**:
- `V8__add_actor_keys.sql` - Actor keys table for HTTP Signatures

### Migration Best Practices

1. **Always Additive**: Never drop columns in migrations (add new ones)
2. **Backward Compatible**: New columns should be nullable or have defaults
3. **Index After Data**: Add indexes after data migration (if migrating existing data)
4. **Test Migrations**: Test migrations on copy of production data

## Connection Pooling

### Configuration

```properties
# Connection pool settings
quarkus.datasource.jdbc.min-size=5
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.idle-removal-interval=5M
```

**Recommendations**:
- **Min size**: 5 connections (always available)
- **Max size**: 20 connections (adjust based on load)
- **Idle removal**: 5 minutes (clean up unused connections)

## Query Optimization

### N+1 Problem Prevention

**Problem**: Loading activities and their actors separately:

```java
// BAD: N+1 queries
List<Activity> activities = Activity.listAll();
for (Activity activity : activities) {
    Actor actor = activity.actor; // Separate query per activity
}
```

**Solution**: Use JOIN FETCH:

```java
// GOOD: Single query with join
List<Activity> activities = Activity.find(
    "SELECT a FROM Activity a JOIN FETCH a.actor"
).list();
```

### Pagination

**Pattern**: Use Panache's built-in pagination:

```java
List<Activity> activities = Activity.find(
    "actor = ?1 ORDER BY published_at DESC",
    actor
)
.page(page, limit)
.list();
```

**Benefits**:
- Efficient LIMIT/OFFSET queries
- Built-in count queries
- Prevents loading entire result set

## Hibernate JSONB Configuration

### Configuration Issue

**Problem**: Hibernate detects JSONB columns and warns about Jackson configuration conflicts when `quarkus.jackson.write-dates-as-timestamps=false`.

**Error**:
```
Persistence unit uses Quarkus' main formatting facilities for JSON columns...
Detected 'quarkus.jackson.write-dates-as-timestamps' property set to 'false'
```

**Solution**: Set `quarkus.hibernate-orm.mapping.format.global=ignore`:

```properties
# application.properties
quarkus.hibernate-orm.mapping.format.global=ignore
```

**Rationale**:
- Tells Hibernate to use its own JSON handling for database columns
- Prevents conflicts with Jackson's REST serialization configuration
- Prevents startup errors and potential data loss

**Reference**: This is a known issue in Quarkus 3.30+ when using JSONB columns with custom Jackson configuration.

## Type Safety: Activity vs Activity

### Naming Conflict

**Discovery**: Java naming conflict between database entity `Activity` and JSON model `Activity`.

**Problem**:
```java
import org.openpace.core.ActivityPubModels.Activity; // Shadows database entity

// This fails - trying to call findById on model class
Activity activity = Activity.findById(activityId); // ERROR
```

**Solution**: Remove conflicting import, use fully qualified names for model class:

```java
// DON'T import ActivityPubModels.Activity
// Use fully qualified name instead

public Response getActivity(@PathParam("activityId") Long activityId) {
    // Database entity (no import needed - same package)
    Activity activity = Activity.findById(activityId);
    
    // JSON model (fully qualified)
    ActivityPubModels.Activity activityModel = activityPubService.toActivity(activity);
    
    return Response.ok(activityModel).build();
}
```

**Lesson Learned**: When classes have same name in different packages, avoid importing the one that shadows the commonly-used one. Use fully qualified names for the less-commonly-used class.

## Entity Pattern

### Panache EntityBase

All database entities extend `PanacheEntityBase`:

```java
@Entity
@Table(name = "actors")
public class Actor extends PanacheEntityBase {
    @Id
    @GeneratedValue
    public Long id;
    
    @Column(unique = true, nullable = false)
    public String username;
    
    public String name;
    
    // Static finder methods
    public static Actor findByUsername(String username) {
        return find("username", username).firstResult();
    }
}
```

**Benefits**:
- Static finder methods: `Actor.findByUsername()`
- Public fields for simplicity (Part 1)
- Helper methods for URL generation
- Active Record pattern (optional)

## Summary

**Key Design Principles**:
- ✅ Separate tables with foreign key relationships
- ✅ Index frequently queried columns
- ✅ Use JSONB for flexible custom types
- ✅ Dual storage: TEXT for Notes, JSONB for custom types
- ✅ Always additive migrations (never drop columns)
- ✅ Efficient pagination for large result sets
- ✅ Prevent N+1 queries with JOIN FETCH

**See Also**:
- [Quarkus Hibernate ORM Panache Guide](https://quarkus.io/guides/hibernate-orm-panache)
- [PostgreSQL JSONB Documentation](https://www.postgresql.org/docs/current/datatype-json.html)
- [PostGIS Documentation](https://postgis.net/documentation/)
- [Mapping Integration](MAPPING_INTEGRATION.md) (Part 7)
