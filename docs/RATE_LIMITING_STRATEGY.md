# Rate Limiting Strategy

This document defines the rate limiting strategy for Open Pace to prevent abuse, protect resources, and ensure fair usage.

## Overview

**Goal**: Prevent abuse, protect against brute force attacks, and ensure fair resource usage.

**Technology**: `quarkus-smallrye-fault-tolerance` with `@RateLimit` annotation ([SmallRye Fault Tolerance Guide](https://quarkus.io/guides/smallrye-fault-tolerance))

**Approach**: Method-level rate limiting using annotations, with per-endpoint configuration.

## Core Principles

### 1. Different Limits for Different Operations

- **Registration**: Very strict (prevent spam accounts)
- **Login**: Strict (prevent brute force)
- **API Endpoints**: Moderate (per-user limits)
- **Federation**: Per-server limits (prevent abuse)

### 2. Per-User vs Per-IP

- **Per-User**: For authenticated endpoints (user-specific limits)
- **Per-IP**: For public endpoints (registration, login, public feeds)

### 3. Graceful Degradation

- Return `429 Too Many Requests` with `Retry-After` header
- Provide clear error messages
- Don't block legitimate users

## Rate Limiting Scenarios

### 1. User Registration

**Endpoint**: `POST /api/auth/register`
**Limit**: 3 registrations per hour per IP
**Rationale**: Prevent spam account creation

### 2. Login Attempts

**Endpoint**: `POST /api/auth/login`
**Limit**: 5 attempts per 15 minutes per IP
**Rationale**: Prevent brute force attacks

### 3. API Endpoints (Authenticated)

**Endpoints**: All `/api/*` endpoints (except auth)
**Limit**: 100 requests per minute per user
**Rationale**: Prevent abuse, ensure fair usage

### 4. Activity Creation

**Endpoint**: `POST /api/users/{username}/posts`
**Limit**: 10 posts per minute per user
**Rationale**: Prevent spam, ensure quality

### 5. Federation Delivery

**Endpoint**: Internal federation delivery service
**Limit**: 50 deliveries per minute per remote server
**Rationale**: Respect remote server capacity, prevent abuse

### 6. Public Feeds

**Endpoint**: `GET /api/feed/public`
**Limit**: 60 requests per minute per IP
**Rationale**: Prevent scraping, ensure fair access

## Implementation

### Dependencies

Add to `pom.xml`:

```xml
<!-- SmallRye Fault Tolerance (includes @RateLimit) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
</dependency>
```

### Configuration

`application.properties`:

```properties
# Rate Limiting Configuration

# Registration: 3 per hour per IP
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/register".rate-limit.value=3
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/register".rate-limit.window=1
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/register".rate-limit.window-unit=hours

# Login: 5 per 15 minutes per IP
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/login".rate-limit.value=5
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/login".rate-limit.window=15
quarkus.fault-tolerance."org.openpace.security.resources.api.AuthResource/login".rate-limit.window-unit=minutes

# Activity Creation: 10 per minute per user
quarkus.fault-tolerance."org.openpace.core.resources.api.PostResource/createPost".rate-limit.value=10
quarkus.fault-tolerance."org.openpace.core.resources.api.PostResource/createPost".rate-limit.window=1
quarkus.fault-tolerance."org.openpace.core.resources.api.PostResource/createPost".rate-limit.window-unit=minutes

# Federation Delivery: 50 per minute per server
quarkus.fault-tolerance."org.openpace.federation.FederationDeliveryService/deliver".rate-limit.value=50
quarkus.fault-tolerance."org.openpace.federation.FederationDeliveryService/deliver".rate-limit.window=1
quarkus.fault-tolerance."org.openpace.federation.FederationDeliveryService/deliver".rate-limit.window-unit=minutes

# Public Feed: 60 per minute per IP
quarkus.fault-tolerance."org.openpace.core.resources.api.FeedResource/getPublicFeed".rate-limit.value=60
quarkus.fault-tolerance."org.openpace.core.resources.api.FeedResource/getPublicFeed".rate-limit.window=1
quarkus.fault-tolerance."org.openpace.core.resources.api.FeedResource/getPublicFeed".rate-limit.window-unit=minutes
```

### Implementation Examples

#### 1. Registration Rate Limiting (Per-IP)

```java
package org.openpace.security.resources.api;

import io.smallrye.faulttolerance.api.RateLimit;
import io.smallrye.faulttolerance.api.RateLimitException;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.Response;
import java.time.temporal.ChronoUnit;

@Path("/api/auth")
public class AuthResource {
    
    @POST
    @Path("/register")
    @RateLimit(value = 3, window = 1, windowUnit = ChronoUnit.HOURS)
    public Response register(RegisterRequest request) {
        // Registration logic
        return Response.ok().build();
    }
}
```

**Exception Handling**:

```java
@ApplicationScoped
public class RateLimitExceptionMapper implements ExceptionMapper<RateLimitException> {
    
    @Override
    public Response toResponse(RateLimitException exception) {
        // Calculate retry-after (time until next window)
        long retryAfter = calculateRetryAfter(exception);
        
        return Response.status(429) // Too Many Requests
            .entity(new ErrorResponse(
                "RateLimitExceeded",
                "Rate limit exceeded. Please try again later."
            ))
            .header("Retry-After", retryAfter)
            .type(MediaType.APPLICATION_JSON)
            .build();
    }
    
    private long calculateRetryAfter(RateLimitException exception) {
        // Calculate seconds until rate limit window resets
        // This is a simplified example - actual implementation
        // would need to track window timing
        return 60; // Default to 60 seconds
    }
}
```

#### 2. Login Rate Limiting (Per-IP)

```java
@POST
@Path("/login")
@RateLimit(value = 5, window = 15, windowUnit = ChronoUnit.MINUTES)
public Response login(LoginRequest request) {
    // Login logic
    return Response.ok().build();
}
```

#### 3. Activity Creation Rate Limiting (Per-User)

```java
package org.openpace.core.resources.api;

import io.smallrye.faulttolerance.api.RateLimit;
import jakarta.annotation.security.RolesAllowed;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import java.time.temporal.ChronoUnit;

@Path("/api/users/{username}/posts")
@RolesAllowed("user")
public class PostResource {
    
    @Inject
    SecurityIdentity securityIdentity;
    
    @POST
    @RateLimit(value = 10, window = 1, windowUnit = ChronoUnit.MINUTES)
    public Response createPost(
        @PathParam("username") String username,
        CreatePostRequest request
    ) {
        // Verify user owns this account
        if (!securityIdentity.getPrincipal().getName().equals(username)) {
            return Response.status(403).build();
        }
        
        // Create post logic
        return Response.ok().build();
    }
}
```

**Note**: `@RateLimit` state is per-method, not per-user. For per-user rate limiting, you'll need custom logic (see "Per-User Rate Limiting" section below).

#### 4. Federation Delivery Rate Limiting (Per-Server)

```java
package org.openpace.federation;

import io.smallrye.faulttolerance.api.RateLimit;
import java.time.temporal.ChronoUnit;

@ApplicationScoped
public class FederationDeliveryService {
    
    /**
     * Deliver activity to remote server inbox.
     * Rate limited per server to respect remote server capacity.
     */
    @RateLimit(value = 50, window = 1, windowUnit = ChronoUnit.MINUTES)
    public void deliver(String server, String inboxUrl, Activity activity) {
        // Delivery logic
        // Note: @RateLimit state is shared across all calls to this method
        // For per-server limiting, use custom implementation (see below)
    }
}
```

**Note**: `@RateLimit` on a method applies globally to that method. For per-server or per-user limiting, you need custom logic.

## Per-User Rate Limiting

Since `@RateLimit` applies globally to a method (not per-user), you need custom logic for per-user limits:

### Option 1: Custom Rate Limiter Service

```java
@ApplicationScoped
public class PerUserRateLimiter {
    
    @Inject
    RedisDataSource redis;
    
    private HashCommands<String, String, String> hashCommands;
    
    @PostConstruct
    void init() {
        hashCommands = redis.hash(String.class);
    }
    
    /**
     * Check if user has exceeded rate limit.
     * @return true if allowed, false if rate limited
     */
    public boolean checkRateLimit(String userId, String operation, int limit, Duration window) {
        String key = String.format("ratelimit:%s:%s", operation, userId);
        
        // Get current count
        String countStr = hashCommands.hget(key, "count").orElse("0");
        String windowStartStr = hashCommands.hget(key, "windowStart").orElse("0");
        
        long count = Long.parseLong(countStr);
        long windowStart = Long.parseLong(windowStartStr);
        long now = Instant.now().getEpochSecond();
        
        // Check if window has expired
        if (now - windowStart >= window.getSeconds()) {
            // Reset window
            hashCommands.hset(key, "count", "1");
            hashCommands.hset(key, "windowStart", String.valueOf(now));
            hashCommands.expire(key, window);
            return true;
        }
        
        // Check if limit exceeded
        if (count >= limit) {
            return false;
        }
        
        // Increment count
        hashCommands.hincrby(key, "count", 1);
        return true;
    }
    
    /**
     * Get time until rate limit resets (in seconds).
     */
    public long getRetryAfter(String userId, String operation, Duration window) {
        String key = String.format("ratelimit:%s:%s", operation, userId);
        String windowStartStr = hashCommands.hget(key, "windowStart").orElse("0");
        
        long windowStart = Long.parseLong(windowStartStr);
        long now = Instant.now().getEpochSecond();
        long elapsed = now - windowStart;
        long remaining = window.getSeconds() - elapsed;
        
        return Math.max(0, remaining);
    }
}
```

### Option 2: Use @RateLimit with Custom Key

You can't directly use `@RateLimit` for per-user limiting, but you can create separate methods per user (not practical) or use a custom interceptor.

### Recommended: Hybrid Approach

Use `@RateLimit` for per-IP/public endpoints, and custom rate limiter for per-user endpoints:

```java
@Path("/api/users/{username}/posts")
@RolesAllowed("user")
public class PostResource {
    
    @Inject
    PerUserRateLimiter rateLimiter;
    
    @Inject
    SecurityIdentity securityIdentity;
    
    @POST
    public Response createPost(
        @PathParam("username") String username,
        CreatePostRequest request
    ) {
        // Verify ownership
        String userId = securityIdentity.getPrincipal().getName();
        if (!userId.equals(username)) {
            return Response.status(403).build();
        }
        
        // Check per-user rate limit
        if (!rateLimiter.checkRateLimit(userId, "createPost", 10, Duration.ofMinutes(1))) {
            long retryAfter = rateLimiter.getRetryAfter(userId, "createPost", Duration.ofMinutes(1));
            return Response.status(429)
                .entity(new ErrorResponse(
                    "RateLimitExceeded",
                    "Rate limit exceeded. Maximum 10 posts per minute."
                ))
                .header("Retry-After", retryAfter)
                .build();
        }
        
        // Create post
        return Response.ok().build();
    }
}
```

## Per-IP Rate Limiting

For per-IP rate limiting, extract IP from request:

```java
@ApplicationScoped
public class IpRateLimiter {
    
    @Inject
    RedisDataSource redis;
    
    private HashCommands<String, String, String> hashCommands;
    
    @PostConstruct
    void init() {
        hashCommands = redis.hash(String.class);
    }
    
    /**
     * Get client IP from request.
     */
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            // Take first IP (original client)
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        
        return request.getRemoteAddr();
    }
    
    /**
     * Check rate limit for IP.
     */
    public boolean checkRateLimit(String ip, String operation, int limit, Duration window) {
        String key = String.format("ratelimit:ip:%s:%s", operation, ip);
        
        // Same logic as per-user rate limiter
        // ...
    }
}
```

## Federation Rate Limiting (Per-Server)

For federation delivery, limit per remote server:

```java
@ApplicationScoped
public class FederationRateLimiter {
    
    @Inject
    RedisDataSource redis;
    
    /**
     * Check rate limit for remote server.
     */
    public boolean checkServerRateLimit(String server, int limit, Duration window) {
        String key = String.format("ratelimit:federation:%s", server);
        
        // Same Redis-based rate limiting logic
        // ...
    }
}
```

## Error Response Format

Consistent error response for rate limiting:

```java
public class RateLimitErrorResponse {
    public String error = "RateLimitExceeded";
    public String message;
    public long retryAfter; // seconds
    
    public RateLimitErrorResponse(String message, long retryAfter) {
        this.message = message;
        this.retryAfter = retryAfter;
    }
}
```

**Example Response**:
```json
{
  "error": "RateLimitExceeded",
  "message": "Rate limit exceeded. Maximum 10 posts per minute.",
  "retryAfter": 45
}
```

## Configuration Reference

### Annotation-Based Configuration

```java
@RateLimit(
    value = 10,                    // Max invocations
    window = 1,                    // Time window
    windowUnit = ChronoUnit.MINUTES, // Window unit
    minSpacing = 0,                // Minimum time between calls (optional)
    minSpacingUnit = ChronoUnit.SECONDS
)
```

### Property-Based Configuration

```properties
# Per-method configuration
quarkus.fault-tolerance."ClassName/methodName".rate-limit.value=10
quarkus.fault-tolerance."ClassName/methodName".rate-limit.window=1
quarkus.fault-tolerance."ClassName/methodName".rate-limit.window-unit=minutes
quarkus.fault-tolerance."ClassName/methodName".rate-limit.min-spacing=1
quarkus.fault-tolerance."ClassName/methodName".rate-limit.min-spacing-unit=seconds
quarkus.fault-tolerance."ClassName/methodName".rate-limit.type=fixed

# Per-class configuration
quarkus.fault-tolerance."ClassName".rate-limit.value=100
quarkus.fault-tolerance."ClassName".rate-limit.window=1
quarkus.fault-tolerance."ClassName".rate-limit.window-unit=minutes

# Global configuration
quarkus.fault-tolerance.global.rate-limit.value=1000
quarkus.fault-tolerance.global.rate-limit.window=1
quarkus.fault-tolerance.global.rate-limit.window-unit=minutes
```

### Rate Limit Types

- **fixed**: Fixed time window (default)
- **rolling**: Rolling time window
- **smooth**: Smooth rate limiting

## Rate Limit Recommendations

### Registration
- **Limit**: 3 per hour per IP
- **Rationale**: Prevent spam accounts
- **Implementation**: `@RateLimit` on registration endpoint

### Login
- **Limit**: 5 per 15 minutes per IP
- **Rationale**: Prevent brute force attacks
- **Implementation**: `@RateLimit` on login endpoint

### API Endpoints
- **Limit**: 100 per minute per user
- **Rationale**: Prevent abuse, ensure fair usage
- **Implementation**: Custom per-user rate limiter

### Activity Creation
- **Limit**: 10 per minute per user
- **Rationale**: Prevent spam, ensure quality
- **Implementation**: Custom per-user rate limiter

### Federation Delivery
- **Limit**: 50 per minute per remote server
- **Rationale**: Respect remote server capacity
- **Implementation**: Custom per-server rate limiter

### Public Feeds
- **Limit**: 60 per minute per IP
- **Rationale**: Prevent scraping
- **Implementation**: `@RateLimit` on public feed endpoint

## Monitoring

### Metrics to Track

- **Rate Limit Hits**: Number of 429 responses
- **Rate Limit by Endpoint**: Which endpoints are rate limited most
- **Rate Limit by User/IP**: Which users/IPs are rate limited
- **Average Retry-After**: How long users wait

### Logging

```java
@RateLimit(value = 10, window = 1, windowUnit = ChronoUnit.MINUTES)
public Response createPost(...) {
    // Log rate limit hits
    if (rateLimited) {
        Log.warnf("Rate limit exceeded for user: %s, operation: createPost", userId);
    }
}
```

## Best Practices

### 1. Set Appropriate Limits

- **Too strict**: Blocks legitimate users
- **Too lenient**: Doesn't prevent abuse
- **Balance**: Based on expected usage patterns

### 2. Provide Clear Error Messages

```json
{
  "error": "RateLimitExceeded",
  "message": "Rate limit exceeded. Maximum 10 posts per minute. Please try again in 45 seconds.",
  "retryAfter": 45
}
```

### 3. Include Retry-After Header

Always include `Retry-After` header in 429 responses:
```
Retry-After: 45
```

### 4. Different Limits for Different Users

Consider:
- **Verified users**: Higher limits
- **New users**: Lower limits
- **VIP users**: Even higher limits

### 5. Monitor and Adjust

- Monitor rate limit hits
- Adjust limits based on usage patterns
- Consider time-of-day variations

## Integration with Security

Rate limiting should work with authentication:

```java
@Path("/api/users/{username}/posts")
@RolesAllowed("user")
public class PostResource {
    
    @Inject
    SecurityIdentity securityIdentity;
    
    @Inject
    PerUserRateLimiter rateLimiter;
    
    @POST
    public Response createPost(...) {
        // Authentication already verified by @RolesAllowed
        String userId = securityIdentity.getPrincipal().getName();
        
        // Check rate limit
        if (!rateLimiter.checkRateLimit(userId, "createPost", 10, Duration.ofMinutes(1))) {
            // Return 429
        }
        
        // Proceed with operation
    }
}
```

## Summary

**Technology**: `quarkus-smallrye-fault-tolerance` with `@RateLimit`

**Approach**:
- **Per-IP**: Use `@RateLimit` annotation (registration, login, public feeds)
- **Per-User**: Custom Redis-based rate limiter (API endpoints, activity creation)
- **Per-Server**: Custom Redis-based rate limiter (federation delivery)

**Key Limits**:
- Registration: 3/hour/IP
- Login: 5/15min/IP
- API: 100/min/user
- Activity Creation: 10/min/user
- Federation: 50/min/server
- Public Feed: 60/min/IP

**Error Response**: 429 Too Many Requests with `Retry-After` header

**See Also**:
- [SmallRye Fault Tolerance Guide](https://quarkus.io/guides/smallrye-fault-tolerance)
- [Security Integration](SECURITY_INTEGRATION.md)
- [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md)
- [Architectural Gaps](ARCHITECTURAL_GAPS.md)
