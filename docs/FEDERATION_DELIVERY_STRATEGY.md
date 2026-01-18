# Federation Delivery Strategy

This document defines the strategy for reliably delivering ActivityPub activities to remote servers' inboxes, including retry mechanisms, queue management, and optimizations.

## Overview

**Goal**: Reliably deliver activities to followers' inboxes with retry mechanisms, error handling, and performance optimizations.

**Current State**: Fire-and-forget delivery using Vert.x WebClient
- Activities delivered asynchronously
- Errors logged but not retried
- No delivery status tracking
- No queue management

**Note**: The current implementation uses Vert.x Mutiny WebClient for async delivery. See "Vert.x WebClient Implementation" section below for details.

**Target State**: Production-ready federation delivery with:
- Asynchronous job queue (Redis)
- Retry mechanism with exponential backoff
- Shared inbox optimization (batch delivery)
- Server reputation tracking
- Delivery status tracking

## Core Principles

### 1. Asynchronous Delivery

**Pattern**: Immediate response, background processing

```
POST /users/alice/outbox
  ↓
201 Created (immediate response)
  ↓
Activity queued for background delivery
  ↓
Worker processes delivery asynchronously
```

**Benefits**:
- Fast API response (user doesn't wait for federation)
- Non-blocking request handling
- Can handle high-volume delivery

### 2. Job Queue Architecture

**Technology**: Redis (using `quarkus-redis-client`)

**Queue Structure**:
- **Pending Queue**: Activities waiting to be delivered
- **Retry Queue**: Failed deliveries waiting for retry
- **Dead Letter Queue**: Permanently failed deliveries

**Workers**: Scheduled jobs using `quarkus-scheduler` process the queue

### 3. Shared Inbox Optimization

**Problem**: Sending to multiple users on the same server individually:
```
POST https://mastodon.social/users/bob/inbox
POST https://mastodon.social/users/carol/inbox
POST https://mastodon.social/users/dave/inbox
```

**Solution**: Batch delivery to shared inbox (if supported):
```
POST https://mastodon.social/inbox
  (with all recipients in `to` field)
```

**Benefits**:
- Reduces server load
- Faster delivery
- Respects rate limits better
- More efficient network usage

**Note**: Not all servers support shared inbox. Fall back to individual delivery if shared inbox unavailable.

### 4. Server Reputation Tracking

**Purpose**: Track delivery success/failure per remote server

**Metrics**:
- Success rate
- Failure rate
- Average response time
- Last successful delivery
- Last failure time
- Consecutive failures

**Actions Based on Reputation**:
- **Healthy**: Normal delivery
- **Degraded**: Reduce delivery rate, increase retry delays
- **Suspended**: Stop delivery, manual review required

### 5. Retry Strategy

**Exponential Backoff**:
```
Attempt 1: Immediate
Attempt 2: 5 minutes later
Attempt 3: 25 minutes later (5 * 5)
Attempt 4: 2 hours later (25 * 5)
Attempt 5: 10 hours later (2 * 5)
Then: Give up or suspend server
```

**Max Attempts**: 5 attempts before moving to dead letter queue

## Architecture

### Components

```
┌─────────────────┐
│  Outbox POST    │
│  (Resource)     │
└────────┬────────┘
         │
         │ Queue activity
         ↓
┌─────────────────┐
│  Redis Queue    │
│  (Pending)      │
└────────┬────────┘
         │
         │ Worker picks up
         ↓
┌─────────────────┐
│  Delivery       │
│  Service        │
└────────┬────────┘
         │
         ├─→ Success → Mark delivered
         │
         └─→ Failure → Retry Queue
                        │
                        └─→ Retry with backoff
```

### Redis Data Structures

#### 1. Pending Delivery Queue
**Key**: `federation:delivery:pending`
**Type**: List (FIFO)
**Value**: JSON string with delivery job data

```json
{
  "activityId": "123",
  "actorUsername": "alice",
  "recipientInbox": "https://mastodon.social/users/bob/inbox",
  "recipientServer": "mastodon.social",
  "activityJson": "{...}",
  "createdAt": "2026-01-17T10:00:00Z",
  "attempt": 1
}
```

#### 2. Retry Queue
**Key**: `federation:delivery:retry:{server}`
**Type**: Sorted Set (by retry time)
**Score**: Unix timestamp (when to retry)
**Value**: Same JSON as pending queue

#### 3. Delivery Status
**Key**: `federation:delivery:status:{activityId}:{recipientInbox}`
**Type**: Hash
**Fields**:
- `status`: `pending`, `delivered`, `failed`, `dead_letter`
- `attempts`: Number of attempts
- `lastAttempt`: Timestamp
- `nextRetry`: Timestamp (if retrying)
- `error`: Error message (if failed)

#### 4. Server Reputation
**Key**: `federation:server:reputation:{server}`
**Type**: Hash
**Fields**:
- `successCount`: Number of successful deliveries
- `failureCount`: Number of failed deliveries
- `lastSuccess`: Timestamp
- `lastFailure`: Timestamp
- `consecutiveFailures`: Current streak
- `status`: `healthy`, `degraded`, `suspended`
- `avgResponseTime`: Average response time in ms

## Implementation

### Dependencies

Add to `pom.xml`:

```xml
<!-- Redis Client -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-redis-client</artifactId>
</dependency>

<!-- Scheduler -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-scheduler</artifactId>
</dependency>
```

**Quarkus Dev Services**: Automatically starts Redis in dev/test mode.

### Configuration

`application.properties`:

```properties
# Redis Configuration (Dev Services handles this automatically)
# quarkus.redis.hosts=redis://localhost:6379

# Federation Delivery Configuration
federation.delivery.max-attempts=5
federation.delivery.retry-backoff-base=5
federation.delivery.retry-backoff-max-hours=10
federation.delivery.workers=3
federation.delivery.batch-size=10

# Server Reputation
federation.reputation.suspend-threshold=10
federation.reputation.degraded-threshold=5
federation.reputation.success-rate-threshold=0.8
```

### Service Layer

#### FederationDeliveryService

```java
@ApplicationScoped
public class FederationDeliveryService {
    
    @Inject
    RedisDataSource redis;
    
    @Inject
    ReactiveRedisDataSource reactiveRedis;
    
    private StringCommands<String, String> stringCommands;
    private ListCommands<String, String> listCommands;
    private SortedSetCommands<String, String> sortedSetCommands;
    private HashCommands<String, String, String> hashCommands;
    
    @PostConstruct
    void init() {
        stringCommands = redis.string(String.class);
        listCommands = redis.list(String.class);
        sortedSetCommands = redis.sortedSet(String.class);
        hashCommands = redis.hash(String.class);
    }
    
    /**
     * Queue activity for delivery to a recipient.
     */
    public void queueDelivery(Activity activity, Actor actor, String recipientInbox) {
        String server = extractServer(recipientInbox);
        
        DeliveryJob job = new DeliveryJob(
            activity.id,
            actor.username,
            recipientInbox,
            server,
            serializeActivity(activity),
            Instant.now(),
            1
        );
        
        // Add to pending queue
        String jobJson = objectMapper.writeValueAsString(job);
        listCommands.lpush("federation:delivery:pending", jobJson);
        
        // Track status
        updateDeliveryStatus(activity.id, recipientInbox, "pending", 1);
    }
    
    /**
     * Queue activity for multiple recipients (batch delivery).
     */
    public void queueDeliveryBatch(Activity activity, Actor actor, List<String> recipientInboxes) {
        // Group by server for shared inbox optimization
        Map<String, List<String>> byServer = recipientInboxes.stream()
            .collect(Collectors.groupingBy(this::extractServer));
        
        for (Map.Entry<String, List<String>> entry : byServer.entrySet()) {
            String server = entry.getKey();
            List<String> inboxes = entry.getValue();
            
            // Try shared inbox first
            String sharedInbox = getSharedInbox(server);
            if (sharedInbox != null && inboxes.size() > 1) {
                queueSharedInboxDelivery(activity, actor, sharedInbox, inboxes);
            } else {
                // Individual delivery
                for (String inbox : inboxes) {
                    queueDelivery(activity, actor, inbox);
                }
            }
        }
    }
    
    /**
     * Process pending deliveries (called by scheduler).
     */
    @Scheduled(every = "10s")
    void processPendingDeliveries() {
        for (int i = 0; i < batchSize; i++) {
            String jobJson = listCommands.rpop("federation:delivery:pending");
            if (jobJson == null) {
                break; // Queue empty
            }
            
            try {
                DeliveryJob job = objectMapper.readValue(jobJson, DeliveryJob.class);
                deliver(job);
            } catch (Exception e) {
                Log.errorf(e, "Failed to process delivery job: %s", jobJson);
            }
        }
    }
    
    /**
     * Process retry queue (called by scheduler).
     */
    @Scheduled(every = "1m")
    void processRetryQueue() {
        long now = Instant.now().getEpochSecond();
        
        // Get all servers with retry queues
        Set<String> servers = getServersWithRetries();
        
        for (String server : servers) {
            String retryKey = "federation:delivery:retry:" + server;
            
            // Get jobs ready to retry (score <= now)
            List<String> readyJobs = sortedSetCommands.zrangebyscore(
                retryKey, 
                0, 
                now
            );
            
            for (String jobJson : readyJobs) {
                try {
                    DeliveryJob job = objectMapper.readValue(jobJson, DeliveryJob.class);
                    sortedSetCommands.zrem(retryKey, jobJson);
                    deliver(job);
                } catch (Exception e) {
                    Log.errorf(e, "Failed to retry delivery: %s", jobJson);
                }
            }
        }
    }
    
    /**
     * Deliver activity to recipient inbox.
     */
    private void deliver(DeliveryJob job) {
        // Check server reputation
        ServerReputation reputation = getServerReputation(job.server);
        if (reputation.status == ServerStatus.SUSPENDED) {
            Log.warnf("Skipping delivery to suspended server: %s", job.server);
            moveToDeadLetter(job, "Server suspended");
            return;
        }
        
        try {
            // Deliver using Vert.x WebClient
            webClient.postAbs(job.recipientInbox)
                .putHeader("Content-Type", "application/activity+json")
                .putHeader("Accept", "application/activity+json")
                .sendJson(new JsonObject(job.activityJson))
                .await()
                .atMost(Duration.ofSeconds(30));
            
            // Success
            updateDeliveryStatus(job.activityId, job.recipientInbox, "delivered", job.attempt);
            updateServerReputation(job.server, true);
            
        } catch (Exception e) {
            // Failure - schedule retry
            handleDeliveryFailure(job, e);
        }
    }
    
    /**
     * Handle delivery failure with retry logic.
     */
    private void handleDeliveryFailure(DeliveryJob job, Exception error) {
        job.attempt++;
        
        if (job.attempt > maxAttempts) {
            // Max attempts reached - move to dead letter
            moveToDeadLetter(job, error.getMessage());
            updateServerReputation(job.server, false);
            return;
        }
        
        // Calculate retry delay (exponential backoff)
        long delaySeconds = calculateRetryDelay(job.attempt);
        long retryTime = Instant.now().getEpochSecond() + delaySeconds;
        
        // Add to retry queue
        String retryKey = "federation:delivery:retry:" + job.server;
        String jobJson = objectMapper.writeValueAsString(job);
        sortedSetCommands.zadd(retryKey, retryTime, jobJson);
        
        // Update status
        updateDeliveryStatus(
            job.activityId, 
            job.recipientInbox, 
            "retrying", 
            job.attempt,
            Instant.now().plusSeconds(delaySeconds)
        );
        
        updateServerReputation(job.server, false);
        
        Log.warnf("Delivery failed (attempt %d), will retry in %d seconds: %s", 
            job.attempt, delaySeconds, error.getMessage());
    }
    
    /**
     * Calculate retry delay using exponential backoff.
     */
    private long calculateRetryDelay(int attempt) {
        // Base: 5 minutes = 300 seconds
        // Attempt 1: 5 minutes
        // Attempt 2: 25 minutes (5 * 5)
        // Attempt 3: 2 hours (25 * 5 = 125 minutes)
        // Attempt 4: 10 hours (125 * 5 = 625 minutes)
        // Cap at max hours
        long baseSeconds = 300; // 5 minutes
        long delaySeconds = baseSeconds * (long) Math.pow(5, attempt - 1);
        long maxSeconds = maxRetryHours * 3600;
        
        return Math.min(delaySeconds, maxSeconds);
    }
    
    /**
     * Update server reputation based on delivery result.
     */
    private void updateServerReputation(String server, boolean success) {
        String key = "federation:server:reputation:" + server;
        
        if (success) {
            hashCommands.hincrby(key, "successCount", 1);
            hashCommands.hset(key, "lastSuccess", Instant.now().toString());
            hashCommands.hset(key, "consecutiveFailures", "0");
        } else {
            hashCommands.hincrby(key, "failureCount", 1);
            hashCommands.hset(key, "lastFailure", Instant.now().toString());
            
            long consecutiveFailures = Long.parseLong(
                hashCommands.hget(key, "consecutiveFailures").orElse("0")
            ) + 1;
            hashCommands.hset(key, "consecutiveFailures", String.valueOf(consecutiveFailures));
            
            // Check if should suspend
            if (consecutiveFailures >= suspendThreshold) {
                hashCommands.hset(key, "status", "suspended");
                Log.warnf("Server %s suspended due to %d consecutive failures", server, consecutiveFailures);
            } else if (consecutiveFailures >= degradedThreshold) {
                hashCommands.hset(key, "status", "degraded");
            }
        }
        
        // Update success rate
        long successCount = Long.parseLong(
            hashCommands.hget(key, "successCount").orElse("0")
        );
        long failureCount = Long.parseLong(
            hashCommands.hget(key, "failureCount").orElse("0")
        );
        long total = successCount + failureCount;
        
        if (total > 0) {
            double successRate = (double) successCount / total;
            hashCommands.hset(key, "successRate", String.valueOf(successRate));
            
            if (successRate < successRateThreshold && total > 10) {
                hashCommands.hset(key, "status", "degraded");
            }
        }
    }
}
```

### Integration with ActivityPubService

Update `ActivityPubService.deliverToFollowers()`:

```java
@Inject
FederationDeliveryService deliveryService;

private void deliverToFollowers(Actor actor, Activity activity) {
    List<Follower> followers = Follower.findByActor(actor);
    Log.infof("Queueing activity for delivery to %d followers of %s", 
        followers.size(), actor.username);
    
    List<String> recipientInboxes = followers.stream()
        .map(f -> f.followerInbox)
        .collect(Collectors.toList());
    
    // Queue for background delivery
    deliveryService.queueDeliveryBatch(activity, actor, recipientInboxes);
}
```

### Shared Inbox Detection

```java
/**
 * Get shared inbox URL for a server (if available).
 */
private String getSharedInbox(String server) {
    // Check cache first
    String cacheKey = "federation:server:shared-inbox:" + server;
    String cached = stringCommands.get(cacheKey);
    if (cached != null) {
        return cached.equals("null") ? null : cached;
    }
    
    // Try to fetch actor profile to discover shared inbox
    // This is a simplified example - in practice, you'd need
    // to know a user on that server to fetch their actor profile
    // For now, return null (no shared inbox)
    stringCommands.setex(cacheKey, 3600, "null"); // Cache for 1 hour
    return null;
}
```

## Retry Strategy Details

### Exponential Backoff Formula

```
delay = base * (multiplier ^ (attempt - 1))
```

**Configuration**:
- `base`: 5 minutes (300 seconds)
- `multiplier`: 5
- `max`: 10 hours

**Attempt Timeline**:
1. **Attempt 1**: Immediate (0 seconds)
2. **Attempt 2**: 5 minutes (300 seconds)
3. **Attempt 3**: 25 minutes (1,500 seconds)
4. **Attempt 4**: 2 hours 5 minutes (7,500 seconds)
5. **Attempt 5**: 10 hours 25 minutes (37,500 seconds, capped at 10 hours)

### Max Attempts

After 5 attempts, the delivery is moved to the **dead letter queue** for manual review.

## Server Reputation System

### Status Levels

1. **Healthy**: Normal delivery, no issues
2. **Degraded**: Some failures, reduce delivery rate
3. **Suspended**: Too many failures, stop delivery

### Thresholds

- **Degraded**: 5 consecutive failures OR success rate < 80% (with >10 total attempts)
- **Suspended**: 10 consecutive failures

### Actions

- **Healthy**: Normal delivery queue
- **Degraded**: 
  - Increase retry delays
  - Reduce delivery rate
  - Monitor closely
- **Suspended**: 
  - Stop all delivery to this server
  - Require manual review to resume
  - Log for investigation

## Monitoring & Observability

### Metrics to Track

- **Pending deliveries**: Queue size
- **Retry queue size**: Per server
- **Delivery success rate**: Overall and per server
- **Average delivery time**: Per server
- **Dead letter queue size**: Failed deliveries
- **Server reputation**: Status distribution

### Logging

- **Info**: Successful deliveries, queue processing
- **Warn**: Retries, degraded servers
- **Error**: Failures, suspended servers, dead letter moves

### Health Checks

- **Queue health**: Pending queue not growing too large
- **Worker health**: Scheduler jobs running
- **Redis health**: Connection status

## Configuration Reference

```properties
# Federation Delivery
federation.delivery.max-attempts=5
federation.delivery.retry-backoff-base=300
federation.delivery.retry-backoff-multiplier=5
federation.delivery.retry-backoff-max-hours=10
federation.delivery.workers=3
federation.delivery.batch-size=10
federation.delivery.process-interval=10s

# Server Reputation
federation.reputation.suspend-threshold=10
federation.reputation.degraded-threshold=5
federation.reputation.success-rate-threshold=0.8
federation.reputation.min-attempts-for-rate=10
```

## Migration from Current Implementation

### Step 1: Add Dependencies
- Add `quarkus-redis-client`
- Add `quarkus-scheduler`

### Step 2: Create Service
- Create `FederationDeliveryService`
- Implement queue operations
- Implement retry logic

### Step 3: Update ActivityPubService
- Replace direct `deliverToInbox()` calls
- Use `deliveryService.queueDeliveryBatch()`

### Step 4: Add Schedulers
- Add `@Scheduled` methods for queue processing
- Add retry queue processing

### Step 5: Test
- Test with single recipient
- Test with multiple recipients
- Test retry mechanism
- Test server suspension

## Future Enhancements

### HTTP Signatures
- Sign outgoing requests (S2S authentication)
- Verify incoming requests
- Key management

### Priority Queue
- High priority: Follow/Undo activities
- Normal priority: Create activities
- Low priority: Like/Announce activities

### Delivery Batching
- Batch multiple activities to same server
- Reduce HTTP overhead

### Analytics
- Delivery success rates
- Server performance metrics
- Queue depth monitoring

## Summary

**Key Components**:
- ✅ Redis queue for pending/retry jobs
- ✅ Scheduler for background processing
- ✅ Exponential backoff retry strategy
- ✅ Server reputation tracking
- ✅ Shared inbox optimization
- ✅ Dead letter queue for failures

**Benefits**:
- ✅ Reliable delivery with retries
- ✅ Non-blocking API responses
- ✅ Automatic server health management
- ✅ Efficient batch delivery
- ✅ Production-ready federation

## Vert.x WebClient Implementation

### Current Implementation (Part 1-4)

**Technology**: Vert.x Mutiny WebClient (`smallrye-mutiny-vertx-web-client`)

**Pattern**: Fire-and-forget async delivery

```java
@PostConstruct
void init() {
    webClient = WebClient.create(vertx);
}

private void deliverToInbox(String inboxUrl, Activity activity) {
    JsonObject json = JsonObject.mapFrom(activity);
    webClient.postAbs(inboxUrl)
        .putHeader("Content-Type", "application/activity+json")
        .putHeader("Accept", "application/activity+json")
        .sendJson(json)
        .subscribe()
        .with(
            response -> Log.infof("Delivered: %d", response.statusCode()),
            error -> Log.errorf(error, "Delivery failed")
        );
}
```

**Important**: In Quarkus 3.30+, use `smallrye-mutiny-vertx-web-client` (not `quarkus-vertx-web`).

**Characteristics**:
- Non-blocking I/O: doesn't block request threads
- Async delivery: activities delivered in background
- Efficient: handles many concurrent deliveries
- Quarkus-native: integrates well with Quarkus reactive stack
- Fire-and-forget: errors logged but don't block

**Migration to Queue-Based (Part 5+)**:
- Replace direct `deliverToInbox()` calls with `queueDelivery()`
- Workers process queue asynchronously
- Retry mechanism handles failures
- See "Service Layer" section above for queue-based implementation

**See Also**:
- [Quarkus Redis Client Guide](https://quarkus.io/guides/redis)
- [Quarkus Scheduler Guide](https://quarkus.io/guides/scheduler)
- [Architectural Gaps](ARCHITECTURAL_GAPS.md)
