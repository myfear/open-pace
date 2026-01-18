# Error Handling & Validation Strategy

This document defines the error handling and validation strategy for the Open Pace tutorial series, ensuring consistent error responses and proper validation across all parts.

## Core Principles

1. **Never expose exceptions to users**: All exceptions are caught and converted to user-friendly error responses
2. **Consistent error format**: All errors use the same JSON structure
3. **Validation first**: Use Hibernate Validator for input validation
4. **Clear error messages**: Help users understand what went wrong and how to fix it
5. **Appropriate HTTP status codes**: Use correct status codes per HTTP/ActivityPub standards

## Error Response Format

All error responses follow this consistent format:

```json
{
  "error": "<ERROR_CODE>",
  "message": "<USER_FRIENDLY_MESSAGE>"
}
```

### Error Response Structure

```java
public class ErrorResponse {
    @JsonProperty("error")
    public String error;
    
    @JsonProperty("message")
    public String message;
    
    public ErrorResponse(String error, String message) {
        this.error = error;
        this.message = message;
    }
}
```

### Example Error Responses

```json
// Validation error
{
  "error": "VALIDATION_ERROR",
  "message": "Username is required"
}

// Not found
{
  "error": "ACTOR_NOT_FOUND",
  "message": "Actor 'alice' not found"
}

// Authorization error
{
  "error": "UNAUTHORIZED",
  "message": "You can only post to your own outbox"
}

// ActivityPub error
{
  "error": "INVALID_ACTIVITY",
  "message": "Activity must have a 'type' field"
}
```

## Validation Strategy

### Hibernate Validator Integration

**Dependency**:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-validator</artifactId>
</dependency>
```

**Configuration**:
```properties
# Enable validation
quarkus.hibernate-validator.enabled=true
```

### Validation Annotations

Use standard Bean Validation annotations:

```java
public class RegistrationRequest {
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Pattern(regexp = "^[a-z0-9_]+$", message = "Username can only contain lowercase letters, numbers, and underscores")
    public String username;
    
    @NotBlank(message = "Password is required")
    @Size(min = 12, message = "Password must be at least 12 characters")
    public String password;
    
    @Email(message = "Email must be valid")
    public String email;
}
```

### Validation in REST Endpoints

```java
@POST
@Path("/api/auth/register")
@Consumes(MediaType.APPLICATION_JSON)
public Response register(@Valid RegistrationRequest request) {
    // Validation happens automatically before method is called
    // If validation fails, ConstraintViolationException is thrown
    // Exception mapper converts it to ErrorResponse
}
```

## Exception Mapping

### Global Exception Mapper

Create a global exception mapper to catch all exceptions and convert them to consistent error responses:

```java
@Provider
public class GlobalExceptionMapper implements ExceptionMapper<Exception> {
    
    private static final Logger LOG = Logger.getLogger(GlobalExceptionMapper.class);
    
    @Override
    public Response toResponse(Exception exception) {
        // Log the exception for debugging (never expose to user)
        LOG.error("Unhandled exception", exception);
        
        // Convert to user-friendly error response
        ErrorResponse error = mapExceptionToError(exception);
        
        return Response.status(error.getStatusCode())
            .type(MediaType.APPLICATION_JSON)
            .entity(error)
            .build();
    }
    
    private ErrorResponse mapExceptionToError(Exception exception) {
        if (exception instanceof ConstraintViolationException) {
            return handleValidationError((ConstraintViolationException) exception);
        } else if (exception instanceof WebApplicationException) {
            return handleWebApplicationException((WebApplicationException) exception);
        } else if (exception instanceof IllegalArgumentException) {
            return new ErrorResponse("INVALID_INPUT", exception.getMessage());
        } else {
            // Generic error - never expose exception details
            return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
        }
    }
}
```

### Specific Exception Mappers

#### Validation Exception Mapper

```java
@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {
    
    @Override
    public Response toResponse(ConstraintViolationException exception) {
        // Collect all validation errors
        List<String> errors = exception.getConstraintViolations().stream()
            .map(violation -> violation.getMessage())
            .collect(Collectors.toList());
        
        String message = errors.size() == 1 
            ? errors.get(0)
            : "Validation failed: " + String.join(", ", errors);
        
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", message);
        
        return Response.status(Response.Status.BAD_REQUEST)
            .type(MediaType.APPLICATION_JSON)
            .entity(error)
            .build();
    }
}
```

#### ActivityPub Exception Mapper

```java
@Provider
public class ActivityPubExceptionMapper implements ExceptionMapper<ActivityPubException> {
    
    @Override
    public Response toResponse(ActivityPubException exception) {
        ErrorResponse error = new ErrorResponse(
            exception.getErrorCode(),
            exception.getUserMessage()
        );
        
        return Response.status(exception.getStatusCode())
            .type("application/activity+json")
            .entity(error)
            .build();
    }
}
```

### Custom Exception Class

```java
public class ActivityPubException extends RuntimeException {
    private final String errorCode;
    private final String userMessage;
    private final Response.Status statusCode;
    
    public ActivityPubException(String errorCode, String userMessage, Response.Status statusCode) {
        super(userMessage); // For logging
        this.errorCode = errorCode;
        this.userMessage = userMessage;
        this.statusCode = statusCode;
    }
    
    // Getters...
}
```

## Error Codes

### Standard Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Input validation failed |
| `INVALID_INPUT` | 400 | Invalid input format or value |
| `ACTOR_NOT_FOUND` | 404 | Actor does not exist |
| `ACTIVITY_NOT_FOUND` | 404 | Activity does not exist |
| `UNAUTHORIZED` | 401 | Authentication required |
| `FORBIDDEN` | 403 | Not authorized for this action |
| `INVALID_ACTIVITY` | 400 | ActivityPub activity is invalid |
| `MISSING_FIELD` | 400 | Required field is missing |
| `DUPLICATE_USERNAME` | 409 | Username already exists |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### ActivityPub-Specific Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `INVALID_ACTIVITY_TYPE` | 400 | Activity type not supported |
| `INVALID_OBJECT_TYPE` | 400 | Object type not supported |
| `MISSING_ACTIVITY_TYPE` | 400 | Activity missing 'type' field |
| `INVALID_ACTOR` | 400 | Actor URL is invalid |
| `INBOX_ERROR` | 400 | Error processing inbox activity |

## Implementation Pattern

### Resource Endpoint Pattern

```java
@Path("/users/{username}/outbox")
public class OutboxResource {
    
    @Inject
    ActivityPubService activityPubService;
    
    @POST
    @Consumes("application/activity+json")
    @Produces("application/activity+json")
    public Response postToOutbox(
            @PathParam("username") @NotBlank String username,
            @Valid @NotNull JsonNode activityJson) {
        
        try {
            // Validate actor exists
            Actor actor = Actor.findByUsername(username);
            if (actor == null) {
                throw new ActivityPubException(
                    "ACTOR_NOT_FOUND",
                    "Actor '" + username + "' not found",
                    Response.Status.NOT_FOUND
                );
            }
            
            // Validate activity structure
            if (!activityJson.has("type")) {
                throw new ActivityPubException(
                    "MISSING_ACTIVITY_TYPE",
                    "Activity must have a 'type' field",
                    Response.Status.BAD_REQUEST
                );
            }
            
            // Process activity
            ActivityPubModels.Activity activity = activityPubService.processActivity(actor, activityJson);
            
            return Response.status(Response.Status.CREATED)
                .header("Location", activity.id)
                .entity(activity)
                .build();
                
        } catch (ActivityPubException e) {
            // Re-throw - exception mapper handles it
            throw e;
        } catch (Exception e) {
            // Log and convert to generic error
            throw new ActivityPubException(
                "INTERNAL_ERROR",
                "An error occurred processing the activity",
                Response.Status.INTERNAL_SERVER_ERROR
            );
        }
    }
}
```

### Service Layer Pattern

```java
@ApplicationScoped
public class ActivityPubService {
    
    public ActivityPubModels.Activity processActivity(Actor actor, JsonNode activityJson) {
        // Validate activity
        validateActivity(activityJson);
        
        // Process based on type
        String activityType = activityJson.get("type").asText();
        
        switch (activityType) {
            case "Create":
                return handleCreate(actor, activityJson);
            default:
                throw new ActivityPubException(
                    "INVALID_ACTIVITY_TYPE",
                    "Activity type '" + activityType + "' is not supported",
                    Response.Status.BAD_REQUEST
                );
        }
    }
    
    private void validateActivity(JsonNode activityJson) {
        if (!activityJson.has("type")) {
            throw new ActivityPubException(
                "MISSING_ACTIVITY_TYPE",
                "Activity must have a 'type' field",
                Response.Status.BAD_REQUEST
            );
        }
        
        // Additional validation...
    }
}
```

## Validation Examples

### Registration Request

```java
public class RegistrationRequest {
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Pattern(regexp = "^[a-z0-9_]+$", message = "Username can only contain lowercase letters, numbers, and underscores")
    public String username;
    
    @NotBlank(message = "Password is required")
    @Size(min = 12, message = "Password must be at least 12 characters")
    public String password;
    
    @Email(message = "Email must be valid")
    public String email;
}
```

### Activity Creation Request

```java
public class CreateActivityRequest {
    @NotNull(message = "Activity type is required")
    @NotBlank(message = "Activity type cannot be empty")
    public String type;
    
    @NotNull(message = "Object is required")
    public JsonNode object;
    
    @Valid
    @NotNull(message = "Object must be valid")
    public ActivityObject object;
}
```

### Path Parameter Validation

```java
@GET
@Path("/users/{username}")
public Response getActor(
        @PathParam("username") 
        @NotBlank(message = "Username is required")
        @Pattern(regexp = "^[a-z0-9_]+$", message = "Invalid username format")
        String username) {
    // ...
}
```

## Error Response Enhancement

### Enhanced Error Response (Optional)

For more detailed errors, you can extend the error response:

```java
public class ErrorResponse {
    @JsonProperty("error")
    public String error;
    
    @JsonProperty("message")
    public String message;
    
    @JsonProperty("field")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public String field; // For field-specific errors
    
    @JsonProperty("details")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public Map<String, Object> details; // For additional context
    
    public ErrorResponse(String error, String message) {
        this.error = error;
        this.message = message;
    }
    
    public ErrorResponse(String error, String message, String field) {
        this.error = error;
        this.message = message;
        this.field = field;
    }
}
```

### Example Enhanced Response

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Username must be between 3 and 50 characters",
  "field": "username"
}
```

## HTTP Status Codes

### Standard Status Codes

| Status Code | Use Case |
|-------------|----------|
| `200 OK` | Successful GET request |
| `201 Created` | Successful POST (with Location header) |
| `202 Accepted` | Activity accepted (inbox POSTs) |
| `400 Bad Request` | Invalid input, validation errors |
| `401 Unauthorized` | Authentication required |
| `403 Forbidden` | Not authorized for action |
| `404 Not Found` | Resource not found |
| `409 Conflict` | Duplicate resource (e.g., username exists) |
| `500 Internal Server Error` | Unexpected server error |

### ActivityPub-Specific Status Codes

- `202 Accepted`: Always returned for inbox POSTs (ActivityPub spec requirement)
- `201 Created`: Returned for outbox POSTs with Location header

## Logging Strategy

### What to Log

**Always log** (for debugging):
- Full exception stack traces
- Request details (URL, method, parameters)
- User context (actor, username)
- Error codes and messages

**Never log** (security):
- Passwords or sensitive data
- Full request bodies with credentials

### Logging Pattern

```java
@Provider
public class GlobalExceptionMapper implements ExceptionMapper<Exception> {
    
    private static final Logger LOG = Logger.getLogger(GlobalExceptionMapper.class);
    
    @Override
    public Response toResponse(Exception exception) {
        // Log full exception with context
        LOG.errorf(exception, 
            "Error processing request: %s - %s", 
            exception.getClass().getSimpleName(),
            exception.getMessage()
        );
        
        // Return user-friendly error (no exception details)
        ErrorResponse error = mapExceptionToError(exception);
        return Response.status(error.getStatusCode())
            .entity(error)
            .build();
    }
}
```

## Testing Error Handling

### Unit Tests

```java
@Test
public void testValidationError() {
    RegistrationRequest request = new RegistrationRequest();
    request.username = ""; // Invalid: blank
    
    Response response = RestAssured.given()
        .contentType(ContentType.JSON)
        .body(request)
        .when()
        .post("/api/auth/register")
        .then()
        .statusCode(400)
        .extract()
        .response();
    
    ErrorResponse error = response.as(ErrorResponse.class);
    assertEquals("VALIDATION_ERROR", error.error);
    assertTrue(error.message.contains("Username is required"));
}
```

### Integration Tests

```java
@Test
public void testActorNotFound() {
    Response response = RestAssured.given()
        .when()
        .get("/users/nonexistent")
        .then()
        .statusCode(404)
        .extract()
        .response();
    
    ErrorResponse error = response.as(ErrorResponse.class);
    assertEquals("ACTOR_NOT_FOUND", error.error);
    assertEquals("Actor 'nonexistent' not found", error.message);
}
```

## Best Practices

### ✅ Do

- **Validate early**: Validate input as soon as it enters the system
- **Use specific error codes**: Help clients handle errors programmatically
- **Provide helpful messages**: Tell users what went wrong and how to fix it
- **Log everything**: Log exceptions with full context for debugging
- **Return appropriate status codes**: Follow HTTP and ActivityPub standards
- **Be consistent**: Use the same error format everywhere

### ❌ Don't

- **Don't expose exceptions**: Never return exception stack traces to users
- **Don't leak information**: Don't reveal internal details (database errors, file paths)
- **Don't use generic messages**: "Error occurred" is not helpful
- **Don't skip validation**: Always validate input, even if it seems safe
- **Don't ignore errors**: Always handle exceptions, never let them bubble up uncaught

## Part-Specific Considerations

### Part 1: Basic ActivityPub
- Validate WebFinger resource format
- Validate actor username format
- Handle missing actors gracefully
- Validate activity JSON structure

### Part 2: Custom Activities
- Validate custom object types
- Validate required fields for Run/Ride/Swim/Walk
- Handle invalid GPX data
- Validate JSON-LD context

### Part 3: Rich Interop
- Validate comment structure (inReplyTo)
- Validate attachment formats
- Handle missing activities for comments/likes

### Part 6: Security Integration
- Validate registration requests
- Validate password strength
- Handle duplicate usernames
- Validate authentication credentials

### Part 7: Mapping Integration
- Validate GPX file format
- Handle invalid coordinates
- Validate OSM tile requests
- Handle PostGIS errors gracefully

## Implementation Checklist

For each part, ensure:

- [ ] **Hibernate Validator dependency** added
- [ ] **Global exception mapper** implemented
- [ ] **Validation exception mapper** implemented
- [ ] **ErrorResponse class** created
- [ ] **Custom exceptions** created (ActivityPubException, etc.)
- [ ] **Validation annotations** on request DTOs
- [ ] **Error codes** defined and documented
- [ ] **Tests** for error scenarios
- [ ] **No exceptions exposed** to users

## Example: Complete Error Handling Setup

### 1. Add Dependency

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-validator</artifactId>
</dependency>
```

### 2. Create ErrorResponse

```java
package org.openpace.core;

import com.fasterxml.jackson.annotation.JsonProperty;

public class ErrorResponse {
    @JsonProperty("error")
    public String error;
    
    @JsonProperty("message")
    public String message;
    
    public ErrorResponse(String error, String message) {
        this.error = error;
        this.message = message;
    }
}
```

### 3. Create Custom Exception

```java
package org.openpace.core;

import jakarta.ws.rs.core.Response;

public class ActivityPubException extends RuntimeException {
    private final String errorCode;
    private final String userMessage;
    private final Response.Status statusCode;
    
    public ActivityPubException(String errorCode, String userMessage, Response.Status statusCode) {
        super(userMessage);
        this.errorCode = errorCode;
        this.userMessage = userMessage;
        this.statusCode = statusCode;
    }
    
    // Getters...
}
```

### 4. Create Exception Mappers

```java
// GlobalExceptionMapper.java
// ValidationExceptionMapper.java
// ActivityPubExceptionMapper.java
```

### 5. Use in Resources

```java
@POST
@Path("/users/{username}/outbox")
public Response postToOutbox(
        @PathParam("username") @NotBlank String username,
        @Valid JsonNode activityJson) {
    // Validation happens automatically
    // Exceptions are caught and converted to ErrorResponse
}
```

## References

- [Quarkus Validation Guide](https://quarkus.io/guides/validation)
- [Bean Validation Specification](https://beanvalidation.org/)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [ActivityPub Error Handling](https://www.w3.org/TR/activitypub/#security-considerations)

## ActivityPub-Specific Status Codes

### ActivityPub Endpoint Status Codes

**Decision**: Return appropriate HTTP status codes per ActivityPub specification.

**Rationale**:
- ActivityPub spec defines expected status codes
- Helps clients understand what went wrong
- Enables proper error recovery

**Status Codes**:
- `200 OK`: Successful GET requests
- `201 Created`: Successful POST to outbox (with `Location` header)
- `202 Accepted`: Successful POST to inbox (ActivityPub spec requirement)
- `400 Bad Request`: Invalid activity JSON or missing required fields
- `404 Not Found`: Actor or resource not found
- `501 Not Implemented`: Activity type not yet supported

**Implementation**:
```java
// Inbox always returns 202 Accepted (ActivityPub spec)
@POST
@Path("/users/{username}/inbox")
public Response receiveActivity(@PathParam("username") String username, JsonNode activityJson) {
    try {
        // Process activity
        activityPubService.handleInboxActivity(actor, activityJson);
    } catch (Exception e) {
        // Log error but still return 202
        Log.errorf(e, "Error processing inbox activity");
    }
    
    // Always return 202 Accepted (ActivityPub spec)
    return Response.status(Response.Status.ACCEPTED).build();
}

// Outbox returns 201 Created with Location header
@POST
@Path("/users/{username}/outbox")
public Response postToOutbox(@PathParam("username") String username, JsonNode activityJson) {
    ActivityPubModels.Activity activity = activityPubService.processActivity(actor, activityJson);
    
    return Response.status(Response.Status.CREATED)
        .header("Location", activity.id)
        .entity(activity)
        .build();
}
```

**Key Points**:
- Inbox always returns 202 (even on errors) - ActivityPub requirement
- Outbox returns 201 with Location header for created activities
- Errors are logged but don't prevent 202 response for inbox

## Summary

**Key Points**:
1. ✅ Use Hibernate Validator for input validation
2. ✅ All errors return `{"error": "...", "message": "..."}` format
3. ✅ Never expose exceptions to users
4. ✅ Use appropriate HTTP status codes (including ActivityPub-specific codes)
5. ✅ Log exceptions for debugging
6. ✅ Provide helpful, user-friendly error messages
7. ✅ Use consistent error codes across all parts
8. ✅ Inbox always returns 202 Accepted (ActivityPub spec requirement)

This ensures a professional, user-friendly error handling experience throughout the tutorial series.
