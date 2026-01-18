# Background Job Processing Strategy

This document defines the strategy for background job processing in Open Pace, using Redis queues and Quarkus Scheduler for asynchronous task execution.

## Overview

**Goal**: Process time-consuming or non-critical tasks asynchronously to improve API response times and system scalability.

**Technology**: 
- **Redis**: Job queues (using `quarkus-redis-client`)
- **Quarkus Scheduler**: Scheduled workers (using `quarkus-scheduler`)

**Approach**: Consistent with [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md) - Redis queues with scheduled workers.

## Core Principles

### 1. Asynchronous Processing

**Pattern**: Immediate API response, background job processing

```
POST /api/users/alice/posts
  ↓
201 Created (immediate response)
  ↓
Job queued for background processing
  ↓
Worker processes job asynchronously
```

**Benefits**:
- Fast API responses (user doesn't wait)
- Non-blocking request handling
- Can handle high-volume processing
- Better user experience

### 2. Redis Queue Architecture

**Technology**: Redis (using `quarkus-redis-client`)

**Queue Types**:
- **Pending Queue**: Jobs waiting to be processed
- **Retry Queue**: Failed jobs waiting for retry
- **Dead Letter Queue**: Permanently failed jobs

**Pattern**: Same as federation delivery queues (see [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md))

### 3. Scheduled Workers

**Technology**: `quarkus-scheduler` with `@Scheduled` annotations

**Pattern**: Workers poll queues at regular intervals and process jobs in batches

## Job Types

### 1. Federation Delivery (Part 5+)

**Queue**: `federation:delivery:pending`, `federation:delivery:retry:{server}`

**Purpose**: Deliver activities to remote servers' inboxes

**See**: [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md) for complete implementation

**Workers**:
- `processPendingDeliveries()` - Every 10 seconds
- `processRetryQueue()` - Every 1 minute

### 2. Map Image Generation (Part 7+)

**Queue**: `jobs:map-generation:pending`

**Purpose**: Generate static map images from GPX data

**Trigger**: When activity with GPX data is created

**Processing**:
- Parse GPX file
- Simplify track using Douglas-Peucker algorithm
- Fetch OSM tiles
- Composite map image
- Store in disk cache (L3)

**Worker**: `processMapGenerationJobs()` - Every 30 seconds

### 3. Activity Aggregation (Part 4+)

**Queue**: `jobs:aggregation:pending`

**Purpose**: Calculate statistics, leaderboards, segments

**Trigger**: Scheduled (daily/hourly) or on-demand

**Processing**:
- Calculate user statistics
- Update leaderboards
- Process segment times
- Aggregate activity data

**Worker**: `processAggregationJobs()` - Every 5 minutes

### 4. Cleanup Tasks

**Queue**: `jobs:cleanup:pending`

**Purpose**: Periodic maintenance tasks

**Tasks**:
- Clean old cache files
- Remove expired sessions
- Archive old activities
- Clean up temporary files

**Worker**: `processCleanupJobs()` - Every 24 hours

### 5. Email Notifications (Future)

**Queue**: `jobs:email:pending`

**Purpose**: Send email notifications (verification, password reset, etc.)

**Worker**: `processEmailJobs()` - Every 1 minute

## Implementation

### Dependencies

Add to `pom.xml`:

```xml
<!-- Redis Client (already included for federation delivery) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-redis-client</artifactId>
</dependency>

<!-- Scheduler (already included for federation delivery) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-scheduler</artifactId>
</dependency>
```

**Note**: These dependencies are already included for federation delivery (Part 5+). Background jobs use the same infrastructure.

### Configuration

`application.properties`:

```properties
# Background Jobs Configuration

# Map Generation
jobs.map-generation.enabled=true
jobs.map-generation.batch-size=5
jobs.map-generation.max-retries=3

# Aggregation
jobs.aggregation.enabled=true
jobs.aggregation.batch-size=10
jobs.aggregation.schedule=0 0 2 * * ?  # Daily at 2 AM

# Cleanup
jobs.cleanup.enabled=true
jobs.cleanup.batch-size=20
jobs.cleanup.schedule=0 0 3 * * ?  # Daily at 3 AM

# Email (future)
jobs.email.enabled=false
jobs.email.batch-size=50
```

### Base Job Service

Create a base service for common job queue operations:

```java
@ApplicationScoped
public class JobQueueService {
    
    @Inject
    RedisDataSource redis;
    
    private ListCommands<String, String> listCommands;
    private SortedSetCommands<String, String> sortedSetCommands;
    private HashCommands<String, String, String> hashCommands;
    
    @PostConstruct
    void init() {
        listCommands = redis.list(String.class);
        sortedSetCommands = redis.sortedSet(String.class);
        hashCommands = redis.hash(String.class);
    }
    
    /**
     * Queue a job for processing.
     */
    public void queueJob(String queueName, Job job) {
        try {
            String jobJson = objectMapper.writeValueAsString(job);
            listCommands.lpush(queueName, jobJson);
            updateJobStatus(job.id, "pending");
        } catch (Exception e) {
            Log.errorf(e, "Failed to queue job: %s", job.id);
        }
    }
    
    /**
     * Get next job from queue (FIFO).
     */
    public Optional<Job> getNextJob(String queueName) {
        String jobJson = listCommands.rpop(queueName);
        if (jobJson == null) {
            return Optional.empty();
        }
        
        try {
            Job job = objectMapper.readValue(jobJson, Job.class);
            return Optional.of(job);
        } catch (Exception e) {
            Log.errorf(e, "Failed to deserialize job: %s", jobJson);
            return Optional.empty();
        }
    }
    
    /**
     * Schedule job for retry.
     */
    public void scheduleRetry(String retryQueueName, Job job, long retryDelaySeconds) {
        long retryTime = Instant.now().getEpochSecond() + retryDelaySeconds;
        try {
            String jobJson = objectMapper.writeValueAsString(job);
            sortedSetCommands.zadd(retryQueueName, retryTime, jobJson);
            updateJobStatus(job.id, "retrying", retryTime);
        } catch (Exception e) {
            Log.errorf(e, "Failed to schedule retry for job: %s", job.id);
        }
    }
    
    /**
     * Move job to dead letter queue.
     */
    public void moveToDeadLetter(String deadLetterQueueName, Job job, String reason) {
        try {
            String jobJson = objectMapper.writeValueAsString(job);
            listCommands.lpush(deadLetterQueueName, jobJson);
            updateJobStatus(job.id, "dead_letter", reason);
            Log.warnf("Job moved to dead letter queue: %s, reason: %s", job.id, reason);
        } catch (Exception e) {
            Log.errorf(e, "Failed to move job to dead letter: %s", job.id);
        }
    }
    
    /**
     * Update job status.
     */
    private void updateJobStatus(String jobId, String status, Object... metadata) {
        String key = "job:status:" + jobId;
        hashCommands.hset(key, "status", status);
        hashCommands.hset(key, "updatedAt", Instant.now().toString());
        if (metadata.length > 0) {
            hashCommands.hset(key, "metadata", objectMapper.writeValueAsString(metadata));
        }
        hashCommands.expire(key, Duration.ofDays(7)); // Keep status for 7 days
    }
}
```

### Base Job Class

```java
public abstract class Job {
    public String id;
    public String type;
    public Instant createdAt;
    public int attempt;
    public Map<String, Object> metadata;
    
    public Job(String type) {
        this.id = UUID.randomUUID().toString();
        this.type = type;
        this.createdAt = Instant.now();
        this.attempt = 1;
        this.metadata = new HashMap<>();
    }
}
```

### Map Generation Job

```java
public class MapGenerationJob extends Job {
    public Long activityId;
    public String gpxData; // Base64 encoded or file path
    
    public MapGenerationJob(Long activityId, String gpxData) {
        super("map-generation");
        this.activityId = activityId;
        this.gpxData = gpxData;
    }
}
```

### Map Generation Service

```java
@ApplicationScoped
public class MapGenerationService {
    
    @Inject
    JobQueueService jobQueue;
    
    @ConfigProperty(name = "jobs.map-generation.enabled", defaultValue = "true")
    boolean enabled;
    
    @ConfigProperty(name = "jobs.map-generation.batch-size", defaultValue = "5")
    int batchSize;
    
    /**
     * Queue map generation job.
     */
    public void queueMapGeneration(Long activityId, String gpxData) {
        if (!enabled) {
            return;
        }
        
        MapGenerationJob job = new MapGenerationJob(activityId, gpxData);
        jobQueue.queueJob("jobs:map-generation:pending", job);
        Log.debugf("Queued map generation job for activity: %d", activityId);
    }
    
    /**
     * Process map generation jobs (called by scheduler).
     */
    @Scheduled(every = "30s")
    void processMapGenerationJobs() {
        if (!enabled) {
            return;
        }
        
        for (int i = 0; i < batchSize; i++) {
            Optional<Job> jobOpt = jobQueue.getNextJob("jobs:map-generation:pending");
            if (jobOpt.isEmpty()) {
                break; // Queue empty
            }
            
            Job job = jobOpt.get();
            if (!(job instanceof MapGenerationJob)) {
                Log.warnf("Invalid job type in map generation queue: %s", job.type);
                continue;
            }
            
            MapGenerationJob mapJob = (MapGenerationJob) job;
            
            try {
                generateMapImage(mapJob);
                jobQueue.updateJobStatus(mapJob.id, "completed");
                Log.infof("Generated map image for activity: %d", mapJob.activityId);
            } catch (Exception e) {
                handleJobFailure(mapJob, e);
            }
        }
    }
    
    private void generateMapImage(MapGenerationJob job) throws Exception {
        // 1. Parse GPX data
        GPXData gpx = parseGPX(job.gpxData);
        
        // 2. Simplify track
        LineString simplifiedTrack = simplifyTrack(gpx.track);
        
        // 3. Fetch OSM tiles
        List<Tile> tiles = fetchOsmTiles(simplifiedTrack.getBounds());
        
        // 4. Generate map image
        BufferedImage mapImage = compositeMap(tiles, simplifiedTrack);
        
        // 5. Store in disk cache
        diskCacheService.storeMapImage(job.activityId.toString(), mapImage, "png");
        
        // 6. Update activity with map attachment
        updateActivityWithMap(job.activityId, mapImage);
    }
    
    private void handleJobFailure(MapGenerationJob job, Exception error) {
        job.attempt++;
        
        if (job.attempt > maxRetries) {
            jobQueue.moveToDeadLetter("jobs:map-generation:dead-letter", job, error.getMessage());
            return;
        }
        
        // Exponential backoff: 1min, 5min, 25min
        long delaySeconds = (long) Math.pow(5, job.attempt - 1) * 60;
        jobQueue.scheduleRetry("jobs:map-generation:retry", job, delaySeconds);
        
        Log.warnf("Map generation failed (attempt %d), will retry in %d seconds: %s", 
            job.attempt, delaySeconds, error.getMessage());
    }
}
```

### Aggregation Service

```java
@ApplicationScoped
public class AggregationService {
    
    @Inject
    JobQueueService jobQueue;
    
    @ConfigProperty(name = "jobs.aggregation.enabled", defaultValue = "true")
    boolean enabled;
    
    /**
     * Queue aggregation job.
     */
    public void queueAggregation(String aggregationType, Map<String, Object> params) {
        if (!enabled) {
            return;
        }
        
        AggregationJob job = new AggregationJob(aggregationType, params);
        jobQueue.queueJob("jobs:aggregation:pending", job);
    }
    
    /**
     * Process aggregation jobs (called by scheduler).
     */
    @Scheduled(every = "5m")
    void processAggregationJobs() {
        if (!enabled) {
            return;
        }
        
        int processed = 0;
        while (processed < batchSize) {
            Optional<Job> jobOpt = jobQueue.getNextJob("jobs:aggregation:pending");
            if (jobOpt.isEmpty()) {
                break;
            }
            
            AggregationJob job = (AggregationJob) jobOpt.get();
            
            try {
                processAggregation(job);
                jobQueue.updateJobStatus(job.id, "completed");
                processed++;
            } catch (Exception e) {
                handleJobFailure(job, e);
            }
        }
    }
    
    /**
     * Scheduled daily aggregation (leaderboards, statistics).
     */
    @Scheduled(cron = "{jobs.aggregation.schedule}")
    void scheduledDailyAggregation() {
        if (!enabled) {
            return;
        }
        
        // Queue daily aggregation jobs
        queueAggregation("leaderboards", Map.of());
        queueAggregation("user-stats", Map.of());
        queueAggregation("segment-times", Map.of());
    }
    
    private void processAggregation(AggregationJob job) {
        switch (job.aggregationType) {
            case "leaderboards":
                updateLeaderboards();
                break;
            case "user-stats":
                updateUserStatistics();
                break;
            case "segment-times":
                updateSegmentTimes();
                break;
        }
    }
}
```

### Cleanup Service

```java
@ApplicationScoped
public class CleanupService {
    
    @Inject
    JobQueueService jobQueue;
    
    @Inject
    DiskCacheService diskCache;
    
    @ConfigProperty(name = "jobs.cleanup.enabled", defaultValue = "true")
    boolean enabled;
    
    /**
     * Scheduled cleanup tasks (daily).
     */
    @Scheduled(cron = "{jobs.cleanup.schedule}")
    void scheduledCleanup() {
        if (!enabled) {
            return;
        }
        
        // Queue cleanup jobs
        queueCleanup("cache-files", Map.of("maxAge", "7d"));
        queueCleanup("expired-sessions", Map.of("maxAge", "30d"));
        queueCleanup("temp-files", Map.of("maxAge", "1d"));
    }
    
    /**
     * Process cleanup jobs.
     */
    @Scheduled(every = "1h")
    void processCleanupJobs() {
        if (!enabled) {
            return;
        }
        
        Optional<Job> jobOpt = jobQueue.getNextJob("jobs:cleanup:pending");
        if (jobOpt.isEmpty()) {
            return;
        }
        
        CleanupJob job = (CleanupJob) jobOpt.get();
        
        try {
            performCleanup(job);
            jobQueue.updateJobStatus(job.id, "completed");
        } catch (Exception e) {
            Log.errorf(e, "Cleanup job failed: %s", job.id);
            // Cleanup jobs don't retry - just log and continue
        }
    }
    
    private void performCleanup(CleanupJob job) {
        switch (job.cleanupType) {
            case "cache-files":
                diskCache.cleanupOldFiles(Duration.parse("P7D"));
                break;
            case "expired-sessions":
                // Clean up expired sessions
                break;
            case "temp-files":
                // Clean up temporary files
                break;
        }
    }
}
```

## Integration with Federation Delivery

### Shared Infrastructure

Background jobs use the same Redis and Scheduler infrastructure as federation delivery:

**Redis**:
- Same Redis instance
- Different queue names (namespaced)
- Same Redis client (`quarkus-redis-client`)

**Scheduler**:
- Same scheduler (`quarkus-scheduler`)
- Different `@Scheduled` methods
- Can run concurrently

### Queue Naming Convention

**Federation Delivery** (from Federation Delivery Strategy):
- `federation:delivery:pending`
- `federation:delivery:retry:{server}`
- `federation:delivery:dead-letter`

**Background Jobs**:
- `jobs:{job-type}:pending`
- `jobs:{job-type}:retry`
- `jobs:{job-type}:dead-letter`

**Examples**:
- `jobs:map-generation:pending`
- `jobs:aggregation:pending`
- `jobs:cleanup:pending`
- `jobs:email:pending`

### Worker Coordination

**Multiple Workers**: Different `@Scheduled` methods can run concurrently:

```java
@ApplicationScoped
public class JobWorkers {
    
    // Federation delivery workers (from FederationDeliveryService)
    @Scheduled(every = "10s")
    void processPendingDeliveries() { ... }
    
    @Scheduled(every = "1m")
    void processRetryQueue() { ... }
    
    // Map generation worker
    @Scheduled(every = "30s")
    void processMapGenerationJobs() { ... }
    
    // Aggregation worker
    @Scheduled(every = "5m")
    void processAggregationJobs() { ... }
    
    // Cleanup worker
    @Scheduled(cron = "0 0 3 * * ?")  // Daily at 3 AM
    void processCleanupJobs() { ... }
}
```

**No Conflicts**: Each worker processes its own queue, no coordination needed.

## Job Status Tracking

### Status States

- **pending**: Queued, waiting to be processed
- **processing**: Currently being processed
- **completed**: Successfully completed
- **retrying**: Failed, scheduled for retry
- **dead_letter**: Permanently failed

### Status Storage

**Redis Hash**: `job:status:{jobId}`

```java
// Store status
hashCommands.hset("job:status:" + jobId, "status", "completed");
hashCommands.hset("job:status:" + jobId, "updatedAt", Instant.now().toString());
hashCommands.expire("job:status:" + jobId, Duration.ofDays(7));
```

### Status API (Optional)

```java
@Path("/api/jobs/{jobId}/status")
public class JobStatusResource {
    
    @GET
    public Response getJobStatus(@PathParam("jobId") String jobId) {
        String status = jobQueue.getJobStatus(jobId);
        return Response.ok(new JobStatusResponse(jobId, status)).build();
    }
}
```

## Retry Strategy

### Exponential Backoff

**Pattern**: Same as federation delivery

```
Attempt 1: Immediate
Attempt 2: 1 minute later
Attempt 3: 5 minutes later
Attempt 4: 25 minutes later
Then: Dead letter queue
```

**Implementation**:
```java
private long calculateRetryDelay(int attempt) {
    // Base: 1 minute
    // Attempt 2: 1 minute
    // Attempt 3: 5 minutes (1 * 5)
    // Attempt 4: 25 minutes (5 * 5)
    long baseSeconds = 60;
    long delaySeconds = baseSeconds * (long) Math.pow(5, attempt - 1);
    return delaySeconds;
}
```

### Max Retries

**Default**: 3 attempts (configurable per job type)

**After Max Retries**: Move to dead letter queue

## Error Handling

### Job Processing Errors

**Pattern**: Catch exceptions, log, and handle retry:

```java
try {
    processJob(job);
    jobQueue.updateJobStatus(job.id, "completed");
} catch (Exception e) {
    Log.errorf(e, "Job processing failed: %s", job.id);
    handleJobFailure(job, e);
}
```

### Dead Letter Queue

**Purpose**: Store permanently failed jobs for manual review

**Access**: Admin interface or manual inspection

**Action**: Investigate and either:
- Fix and requeue
- Delete if not needed
- Update job parameters and retry

## Monitoring

### Metrics to Track

- **Queue Depth**: Number of pending jobs per queue
- **Processing Rate**: Jobs processed per minute
- **Success Rate**: Percentage of successful jobs
- **Retry Rate**: Number of jobs requiring retry
- **Dead Letter Count**: Permanently failed jobs
- **Processing Time**: Average time to process jobs

### Logging

```java
// Log job queued
Log.debugf("Queued job: type=%s, id=%s", job.type, job.id);

// Log job processing
Log.infof("Processing job: type=%s, id=%s, attempt=%d", job.type, job.id, job.attempt);

// Log job completion
Log.infof("Job completed: type=%s, id=%s, duration=%dms", job.type, job.id, duration);

// Log job failure
Log.warnf("Job failed: type=%s, id=%s, attempt=%d, error=%s", job.type, job.id, job.attempt, error);
```

## Configuration Reference

```properties
# Background Jobs - General
jobs.enabled=true

# Map Generation
jobs.map-generation.enabled=true
jobs.map-generation.batch-size=5
jobs.map-generation.max-retries=3
jobs.map-generation.retry-delay-base=60

# Aggregation
jobs.aggregation.enabled=true
jobs.aggregation.batch-size=10
jobs.aggregation.schedule=0 0 2 * * ?  # Daily at 2 AM
jobs.aggregation.max-retries=2

# Cleanup
jobs.cleanup.enabled=true
jobs.cleanup.batch-size=20
jobs.cleanup.schedule=0 0 3 * * ?  # Daily at 3 AM
jobs.cleanup.max-retries=1  # Cleanup jobs typically don't retry

# Email (future)
jobs.email.enabled=false
jobs.email.batch-size=50
jobs.email.max-retries=5
```

## Best Practices

### 1. Idempotent Jobs

**Principle**: Jobs should be idempotent (safe to retry)

**Example**: Map generation
- Check if map already exists before generating
- If exists, skip generation
- Safe to retry if previous attempt failed

### 2. Batch Processing

**Principle**: Process multiple jobs per scheduler invocation

**Benefits**:
- More efficient
- Better throughput
- Reduces scheduler overhead

**Configuration**: `batch-size` property per job type

### 3. Error Recovery

**Principle**: Always handle errors gracefully

**Pattern**:
- Log error with context
- Schedule retry if appropriate
- Move to dead letter if max retries exceeded
- Never crash the worker

### 4. Resource Limits

**Principle**: Limit resource usage per job

**Considerations**:
- Memory usage (large GPX files)
- CPU usage (image processing)
- Network usage (OSM tile fetching)
- Disk I/O (cache writes)

### 5. Job Priority

**Future Enhancement**: Priority queues for urgent jobs

**Pattern**: Use separate queues or priority field:
- `jobs:map-generation:high-priority`
- `jobs:map-generation:normal`
- `jobs:map-generation:low-priority`

## Integration Points

### Activity Creation

```java
@Path("/api/users/{username}/posts")
public class PostResource {
    
    @Inject
    MapGenerationService mapGenerationService;
    
    @POST
    public Response createPost(CreatePostRequest request) {
        // Create activity
        Activity activity = activityService.createActivity(actor, request);
        
        // Queue map generation if GPX provided
        if (request.gpxData != null) {
            mapGenerationService.queueMapGeneration(activity.id, request.gpxData);
        }
        
        return Response.ok(activity).build();
    }
}
```

### Scheduled Tasks

```java
@ApplicationScoped
public class ScheduledJobs {
    
    @Inject
    AggregationService aggregationService;
    
    @Inject
    CleanupService cleanupService;
    
    // Daily aggregation at 2 AM
    @Scheduled(cron = "0 0 2 * * ?")
    void dailyAggregation() {
        aggregationService.queueAggregation("leaderboards", Map.of());
        aggregationService.queueAggregation("user-stats", Map.of());
    }
    
    // Daily cleanup at 3 AM
    @Scheduled(cron = "0 0 3 * * ?")
    void dailyCleanup() {
        cleanupService.scheduledCleanup();
    }
}
```

## Migration from Synchronous to Asynchronous

### Step 1: Identify Synchronous Operations

**Candidates**:
- Map image generation (Part 7)
- Activity aggregation (Part 4)
- Email sending (future)

### Step 2: Create Job Classes

```java
public class MapGenerationJob extends Job {
    public Long activityId;
    public String gpxData;
}
```

### Step 3: Create Job Service

```java
@ApplicationScoped
public class MapGenerationService {
    public void queueMapGeneration(Long activityId, String gpxData) { ... }
    
    @Scheduled(every = "30s")
    void processMapGenerationJobs() { ... }
}
```

### Step 4: Update API Endpoints

**Before**:
```java
@POST
public Response createPost(CreatePostRequest request) {
    Activity activity = createActivity(request);
    generateMapImage(activity); // Synchronous - blocks response
    return Response.ok(activity).build();
}
```

**After**:
```java
@POST
public Response createPost(CreatePostRequest request) {
    Activity activity = createActivity(request);
    mapGenerationService.queueMapGeneration(activity.id, request.gpxData); // Async
    return Response.ok(activity).build(); // Fast response
}
```

## Summary

**Technology**:
- ✅ Redis queues (same as federation delivery)
- ✅ Quarkus Scheduler (`@Scheduled` annotations)
- ✅ Shared infrastructure with federation delivery

**Job Types**:
- ✅ Federation delivery (Part 5+)
- ✅ Map generation (Part 7+)
- ✅ Aggregation (Part 4+)
- ✅ Cleanup (Part 3+)
- ✅ Email (future)

**Patterns**:
- ✅ Immediate API response, background processing
- ✅ Redis queues (pending, retry, dead letter)
- ✅ Scheduled workers with batch processing
- ✅ Exponential backoff retry strategy
- ✅ Job status tracking

**Benefits**:
- ✅ Fast API responses
- ✅ Scalable processing
- ✅ Non-blocking operations
- ✅ Consistent with federation delivery approach

**See Also**:
- [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md) - Shared queue infrastructure
- [Quarkus Scheduler Guide](https://quarkus.io/guides/scheduler)
- [Quarkus Redis Client Guide](https://quarkus.io/guides/redis)
- [Caching Strategy](CACHING_STRATEGY.md) - Disk cache for map images
- [Architectural Gaps](ARCHITECTURAL_GAPS.md)
