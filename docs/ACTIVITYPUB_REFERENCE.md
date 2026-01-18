# ActivityPub Quick Reference

This document provides a quick reference for ActivityPub concepts used throughout the tutorial series. Use consistent terminology and examples from this guide.

**See also**: [ActivityPub C2S Pattern](ACTIVITYPUB_C2S_PATTERN.md) for the correct client-to-server implementation pattern.

## Core Concepts

### Actor
An ActivityPub entity that can perform activities. In Open Pace, each user is an Actor.

**Example**:
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Person",
  "id": "https://openpace.example/users/alice",
  "preferredUsername": "alice",
  "name": "Alice Runner",
  "inbox": "https://openpace.example/users/alice/inbox",
  "outbox": "https://openpace.example/users/alice/outbox",
  "followers": "https://openpace.example/users/alice/followers",
  "following": "https://openpace.example/users/alice/following"
}
```

### Activity
An action performed by an Actor. Common types: `Create`, `Follow`, `Like`, `Announce`, `Undo`.

**Example**:
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "actor": "https://openpace.example/users/alice",
  "object": {
    "type": "Note",
    "content": "Just finished a 5K run!"
  }
}
```

### Object
The target of an Activity. Can be a `Note`, `Article`, or custom types like `Run`, `Ride`, `Swim`.

### Inbox
The endpoint where an Actor receives activities from other Actors. POST requests are sent here.

**Endpoint**: `POST /users/{username}/inbox`
**Content-Type**: `application/activity+json`

### Outbox
The endpoint that serves activities created by an Actor. 

**GET**: Retrieves the activity stream
- **Endpoint**: `GET /users/{username}/outbox`
- **Response**: OrderedCollection of activities

**POST**: Clients POST Activity objects here (C2S pattern)
- **Endpoint**: `POST /users/{username}/outbox`
- **Content-Type**: `application/activity+json`
- **Server processes based on activity `type` field**

See [ActivityPub C2S Pattern](ACTIVITYPUB_C2S_PATTERN.md) for implementation details.

### WebFinger
Protocol for discovering Actor information from a user identifier.

**Endpoint**: `GET /.well-known/webfinger?resource=acct:username@domain`
**Response**: JSON with links to Actor profile

## Common Activity Types

### Create
Publishes a new object (like posting a run).

```json
{
  "type": "Create",
  "actor": "https://openpace.example/users/alice",
  "object": {
    "type": "https://fedisports.example/ns#Run",
    "name": "Morning 5K",
    "distance": 5000
  }
}
```

### Follow
Requests to follow another Actor.

```json
{
  "type": "Follow",
  "actor": "https://mastodon.example/users/bob",
  "object": "https://openpace.example/users/alice"
}
```

### Accept
Accepts a Follow request.

```json
{
  "type": "Accept",
  "actor": "https://openpace.example/users/alice",
  "object": {
    "type": "Follow",
    "actor": "https://mastodon.example/users/bob",
    "object": "https://openpace.example/users/alice"
  }
}
```

### Like
Likes an object.

```json
{
  "type": "Like",
  "actor": "https://mastodon.example/users/bob",
  "object": "https://openpace.example/users/alice/activities/123"
}
```

### Announce
Shares/boosts an object (like retweeting).

```json
{
  "type": "Announce",
  "actor": "https://openpace.example/users/alice",
  "object": "https://openpace.example/users/bob/activities/456"
}
```

## HTTP Headers

### Required Headers for Inbox POST
```
Content-Type: application/activity+json
Accept: application/activity+json
```

### Required Headers for Actor/Outbox GET
```
Accept: application/activity+json
```

## Status Codes

- `200 OK`: Successful GET request
- `201 Created`: Successful POST to inbox
- `202 Accepted`: Activity accepted for processing
- `400 Bad Request`: Invalid activity JSON
- `401 Unauthorized`: Missing/invalid authentication
- `404 Not Found`: Actor or object not found

## JSON-LD Context

Always include the ActivityStreams context:

```json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://fedisports.example/ns"  // For custom types (Part 2+)
  ]
}
```

## Custom Object Types (Part 2+)

### Run Object
```json
{
  "type": "https://fedisports.example/ns#Run",
  "name": "Morning 5K",
  "distance": 5000,
  "duration": "PT25M30S",
  "startTime": "2026-01-17T07:00:00Z",
  "averagePace": "5:06/km",
  "elevation": 120,
  "gpxData": "...",
  "attachment": [
    {
      "type": "Image",
      "url": "https://openpace.example/activities/123/map.png"
    }
  ]
}
```

## Federation Flow

1. **Discovery**: Bob uses WebFinger to find Alice's Actor URL
2. **Follow**: Bob sends a `Follow` activity to Alice's inbox
3. **Accept**: Alice's server sends an `Accept` activity to Bob's inbox
4. **Create**: Alice posts a run, creating a `Create` activity
5. **Delivery**: Alice's server POSTs the activity to all followers' inboxes (including Bob's)
6. **Display**: Bob's server (Mastodon) displays the activity in Bob's feed

## Common Patterns

### Sending to Followers
```java
// Pseudo-code
List<Actor> followers = getFollowers(actor);
for (Actor follower : followers) {
    postToInbox(follower.getInbox(), activity);
}
```

### Validating Activities
1. Check `@context` includes ActivityStreams
2. Verify `type` is a known Activity type
3. Validate `actor` and `object` are valid URLs
4. Check signature (if using HTTP signatures)

## Terminology Consistency

Throughout the tutorials, use:
- ✅ **Actor** (not "user" in ActivityPub context)
- ✅ **Activity** (not "action" or "event")
- ✅ **Object** (not "item" or "content")
- ✅ **Inbox/Outbox** (not "feed" or "stream")
- ✅ **Follow** (not "subscribe" or "friend")
- ✅ **Federation** (not "network" or "protocol")

## Testing with Mastodon

### Test Account Setup
1. Create account on a Mastodon instance
2. Note the instance URL (e.g., `https://mastodon.social`)

### Following from Mastodon
1. Search for: `@alice@openpace.example`
2. Mastodon will use WebFinger to discover the Actor
3. Click "Follow"
4. Mastodon sends `Follow` activity to your inbox

### Verifying Activities Appear
1. Post an activity on Open Pace
2. Check Mastodon feed
3. Activity should appear (may be simplified for Mastodon)

## JSON-LD Context

### ActivityStreams Context

**Requirement**: All ActivityPub JSON responses must include `@context` field.

**Standard Context**:
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Note",
  "content": "Hello world"
}
```

**Implementation**:
```java
@JsonProperty("@context")
public Object context = List.of("https://www.w3.org/ns/activitystreams");
```

### Custom Vocabulary Context (Part 2+)

**Pattern**: Include both ActivityStreams and custom vocabulary:

```json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://fedisports.example/ns"
  ],
  "type": "https://fedisports.example/ns#Run",
  "distance": 5000
}
```

**Implementation**:
```java
@JsonProperty("@context")
public Object context = List.of(
    "https://www.w3.org/ns/activitystreams",
    "https://fedisports.example/ns"
);
```

**Important**: Always include both contexts - ActivityStreams for standard ActivityPub compatibility, custom vocabulary for sports-specific types.

### Context Enrichment

**Pattern**: Automatically add `@context` if missing when processing activities:

```java
private JsonNode enrichCustomObject(Actor actor, JsonNode objectJson, String objectType) {
    ObjectNode enriched = objectJson.deepCopy();
    
    // Ensure @context is present
    if (!enriched.has("@context")) {
        enriched.set("@context", objectMapper.createArrayNode()
            .add("https://www.w3.org/ns/activitystreams")
            .add("https://fedisports.example/ns"));
    }
    
    return enriched;
}
```

## Activity Type vs Object Type

### Two-Level Processing

ActivityPub has two levels of types:

1. **Activity Type**: The action (`Create`, `Follow`, `Like`, `Announce`, `Undo`)
2. **Object Type**: The target (`Note`, `Run`, `Ride`, `Swim`, `Walk`)

### Processing Pattern

```java
// First level: Activity type
String activityType = activityJson.get("type").asText(); // "Create", "Follow", etc.

// Second level: Object type (for Create activities)
if ("Create".equals(activityType)) {
    JsonNode objectNode = activityJson.get("object");
    String objectType = objectNode.get("type").asText(); // "Note", "Run", etc.
    // Process based on object type
}
```

**Key Points**:
- `Create` activity can create different object types (Note, Run, etc.)
- Server must handle both levels
- Same `Create` activity type, different object types
- Server routes based on both levels

## Resources

- [ActivityPub Spec](https://www.w3.org/TR/activitypub/)
- [ActivityStreams Vocabulary](https://www.w3.org/TR/activitystreams-vocabulary/)
- [WebFinger Spec](https://tools.ietf.org/html/rfc7033)
- [JSON-LD Spec](https://www.w3.org/TR/json-ld/)
