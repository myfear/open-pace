# ActivityPub Client-to-Server (C2S) Pattern

This document explains the correct pattern for implementing ActivityPub Client-to-Server interactions, where clients post Activity objects and servers process them based on the activity type.

## Core Principle

**Clients POST Activity objects to the outbox. The server processes based on the `type` field in the JSON.**

According to the [ActivityPub C2S Specification](https://www.w3.org/TR/activitypub/#client-to-server-interactions):

> Clients send an HTTP POST to the actor's outbox URL. The request body may be either:
> - A single **Activity** object (which may include embedded other objects), or
> - A single non-Activity AS2 object — in which case the server *must* wrap it inside a `Create` activity.

## The Pattern

### Single Endpoint: POST to Outbox

Instead of separate endpoints for different activity types, there should be **one endpoint**:

```
POST /users/{username}/outbox
Content-Type: application/activity+json
```

The server then **determines what to do based on the `type` field** in the JSON payload.

**Note**: This is the **ActivityPub endpoint** (required by spec). For UI convenience, you can also provide `/api/*` endpoints, but the ActivityPub endpoint is the standard. See [API Design](API_DESIGN.md) for endpoint organization strategy.

### Activity Type Determination

The server reads the `type` field from the JSON to determine how to process it:

```json
{
  "type": "Create",
  "object": {
    "type": "Note",
    "content": "Hello world"
  }
}
```

```json
{
  "type": "Create",
  "object": {
    "type": "https://fedisports.example/ns#Run",
    "distance": 5000,
    "duration": "PT25M30S"
  }
}
```

```json
{
  "type": "Follow",
  "object": "https://other-instance.com/users/bob"
}
```

```json
{
  "type": "Like",
  "object": "https://other-instance.com/users/bob/activities/123"
}
```

## Implementation Pattern

### Server-Side Processing

```java
@POST
@Path("/users/{username}/outbox")
@Consumes("application/activity+json")
@Produces("application/activity+json")
public Response postToOutbox(
        @PathParam("username") String username,
        JsonNode activityJson) {
    
    // 1. Authenticate and authorize the actor
    Actor actor = authenticateAndAuthorize(username);
    
    // 2. Extract activity type from JSON
    String activityType = activityJson.get("type").asText();
    
    // 3. Process based on activity type
    switch (activityType) {
        case "Create":
            return handleCreate(actor, activityJson);
        case "Update":
            return handleUpdate(actor, activityJson);
        case "Delete":
            return handleDelete(actor, activityJson);
        case "Follow":
            return handleFollow(actor, activityJson);
        case "Like":
            return handleLike(actor, activityJson);
        case "Announce":
            return handleAnnounce(actor, activityJson);
        case "Undo":
            return handleUndo(actor, activityJson);
        default:
            return Response.status(400)
                .entity("Unsupported activity type: " + activityType)
                .build();
    }
}
```

### Handling Create Activities

For `Create` activities, the server must also inspect the **object type**:

```java
private Response handleCreate(Actor actor, JsonNode activityJson) {
    JsonNode objectNode = activityJson.get("object");
    if (objectNode == null) {
        return Response.status(400).entity("Create activity missing object").build();
    }
    
    // Determine object type from JSON
    String objectType = objectNode.get("type").asText();
    
    // Process based on object type
    if ("Note".equals(objectType)) {
        return handleCreateNote(actor, activityJson, objectNode);
    } else if (objectType.contains("Run")) {
        return handleCreateRun(actor, activityJson, objectNode);
    } else if (objectType.contains("Ride")) {
        return handleCreateRide(actor, activityJson, objectNode);
    } else if (objectType.contains("Swim")) {
        return handleCreateSwim(actor, activityJson, objectNode);
    } else if (objectType.contains("Walk")) {
        return handleCreateWalk(actor, activityJson, objectNode);
    } else {
        return Response.status(400)
            .entity("Unsupported object type: " + objectType)
            .build();
    }
}
```

## Examples

### Example 1: Create Note Activity

**Client POST:**
```json
POST /users/alice/outbox
Content-Type: application/activity+json

{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "object": {
    "type": "Note",
    "content": "Just finished a great run!"
  }
}
```

**Server Processing:**
1. Receives JSON at outbox endpoint
2. Reads `type: "Create"`
3. Reads `object.type: "Note"`
4. Processes as Create Note activity
5. Stores activity and object
6. Delivers to followers

### Example 2: Create Run Activity

**Client POST:**
```json
POST /users/alice/outbox
Content-Type: application/activity+json

{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://fedisports.example/ns"
  ],
  "type": "Create",
  "object": {
    "type": "https://fedisports.example/ns#Run",
    "name": "Morning 5K",
    "distance": 5000,
    "duration": "PT25M30S",
    "startTime": "2026-01-17T07:00:00Z"
  }
}
```

**Server Processing:**
1. Receives JSON at outbox endpoint
2. Reads `type: "Create"`
3. Reads `object.type: "https://fedisports.example/ns#Run"`
4. Processes as Create Run activity
5. Stores activity and custom Run object
6. Delivers to followers

### Example 3: Follow Activity

**Client POST:**
```json
POST /users/alice/outbox
Content-Type: application/activity+json

{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Follow",
  "object": "https://other-instance.com/users/bob"
}
```

**Server Processing:**
1. Receives JSON at outbox endpoint
2. Reads `type: "Follow"`
3. Processes as Follow activity
4. Sends Follow to target actor's inbox
5. Waits for Accept response

## Activity Type Side Effects

According to the ActivityPub spec, different activity types have specific side effects:

| Activity Type | Server Must Do |
|--------------|----------------|
| **Create** | Create the object; set `attributedTo` if missing; deliver to recipients |
| **Update** | Update the object (partial update); check permissions |
| **Delete** | Mark object as deleted (Tombstone); return 410 Gone |
| **Follow** | Add to following collection (pending until Accept) |
| **Like** | Add to actor's liked collection |
| **Announce** | Share/boost the object |
| **Undo** | Reverse the effects of the original activity |
| **Block** | Prevent interactions from blocked actor |

## Object Type Handling

For `Create` activities, the server must handle different object types:

- **Standard Types**: `Note`, `Article`, `Image`, `Video`
- **Custom Types**: `Run`, `Ride`, `Swim`, `Walk` (with custom vocabulary)

The server determines the object type from:
```json
{
  "type": "Create",
  "object": {
    "type": "..."  // <-- Object type determined here
  }
}
```

## Why This Pattern?

### ✅ Advantages

1. **Spec Compliant**: Follows ActivityPub C2S specification exactly
2. **Extensible**: Easy to add new activity types without new endpoints
3. **Consistent**: All activities go through the same endpoint
4. **Flexible**: Clients can send any valid ActivityPub activity
5. **Standard**: Works with any ActivityPub client

### ❌ Anti-Pattern (ActivityPub Endpoints)

**Don't do this for ActivityPub endpoints:**
```
POST /users/{username}/post          # For Note
POST /users/{username}/post/run       # For Run
POST /users/{username}/post/ride      # For Ride
```

**Why it's wrong:**
- Not ActivityPub compliant
- Requires separate endpoints for each type
- Doesn't work with standard ActivityPub clients
- Breaks federation expectations

**Note**: You **can** provide these as `/api/*` endpoints for UI convenience, but the ActivityPub standard endpoint (`POST /users/{username}/outbox`) must also exist. See [API Design](API_DESIGN.md) for details.

## Implementation Checklist

When implementing the outbox POST endpoint:

- [ ] **Single endpoint**: `POST /users/{username}/outbox`
- [ ] **Content-Type**: Accept `application/activity+json`
- [ ] **Authentication**: Verify actor owns the outbox
- [ ] **Type extraction**: Read `type` field from JSON
- [ ] **Type-based routing**: Process based on activity type
- [ ] **Object type handling**: For Create, also check `object.type`
- [ ] **Side effects**: Implement required side effects per activity type
- [ ] **Response**: Return `201 Created` with `Location` header
- [ ] **Storage**: Add activity to outbox collection
- [ ] **Delivery**: Deliver to recipients (for Create, Update, etc.)

## Migration from Current Implementation

To migrate from separate endpoints to the C2S pattern:

1. **Add POST to OutboxResource** (ActivityPub endpoint):
   ```java
   @POST
   @Path("/users/{username}/outbox")
   @Consumes("application/activity+json")
   public Response postToOutbox(...)
   ```

2. **Extract activity type** from JSON
3. **Route to appropriate handler** based on type
4. **For Create activities**, extract object type and route accordingly
5. **Optionally move convenience endpoints** to `/api/*`:
   ```java
   @POST
   @Path("/api/users/{username}/posts/run")
   @Consumes(MediaType.APPLICATION_JSON)
   public Response createRun(...)
   ```
6. **Keep both**: ActivityPub endpoint for federation, `/api/*` endpoints for UI convenience

**See**: [API Design](API_DESIGN.md) for endpoint organization strategy.

## References

- [ActivityPub C2S Specification](https://www.w3.org/TR/activitypub/#client-to-server-interactions)
- [Activity Types](https://www.w3.org/TR/activitystreams-vocabulary/#activity-types)
- [Object Types](https://www.w3.org/TR/activitystreams-vocabulary/#object-types)

## Summary

**Key Points:**
1. ✅ Clients POST Activity objects to `/users/{username}/outbox`
2. ✅ Server processes based on `type` field in JSON
3. ✅ For Create activities, also check `object.type`
4. ✅ Single endpoint handles all activity types
5. ✅ Follows ActivityPub C2S specification

**Pattern:**
```
Client → POST Activity JSON → Outbox Endpoint → Extract type → Process accordingly
```
