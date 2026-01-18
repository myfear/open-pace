# Caching Strategy

This document defines the caching strategy for Open Pace, using a multi-level cache architecture to optimize performance and reduce database load.

## Overview

**Goal**: Improve performance by caching frequently accessed data at multiple levels.

**Architecture**: Three-level cache hierarchy
- **L1**: In-memory cache (Caffeine - Quarkus default)
- **L2**: Redis cache (distributed, shared across instances)
- **L3**: Disk cache (file system, for large files)

## Cache Levels

### L1: In-Memory Cache (Caffeine)

**Technology**: Quarkus default cache (Caffeine)
**Scope**: Application instance only
**Use Case**: Hot data, very fast access
**Size**: Limited by JVM heap

**Characteristics**:
- Fastest access (nanoseconds)
- Not shared across instances
- Lost on application restart
- Good for frequently accessed, small objects

### L2: Redis Cache (Distributed)

**Technology**: `quarkus-redis-cache` ([Redis Cache Guide](https://quarkus.io/guides/cache-redis-reference))
**Scope**: Shared across all application instances
**Use Case**: Shared data, distributed caching
**Size**: Limited by Redis memory

**Characteristics**:
- Fast access (milliseconds)
- Shared across instances
- Survives application restarts
- Good for actor profiles, feeds, shared data

### L3: Disk Cache (File System)

**Technology**: File system storage
**Scope**: Local to application instance
**Use Case**: Large files (map images, OSM tiles)
**Size**: Limited by disk space

**Characteristics**:
- Slower access (milliseconds to seconds)
- Persistent across restarts
- Good for large, infrequently changing files
- Configurable storage path

## What to Cache

### 1. Actor Profiles

**Cache Levels**: L1 (in-memory), L2 (Redis)
**TTL**: 1 hour
**Invalidation**: On actor update

**Rationale**: 
- Frequently accessed (every activity fetch)
- Rarely changes
- Reduces database queries

### 2. Remote Actor Profiles

**Cache Levels**: L1, L2
**TTL**: 24 hours
**Invalidation**: Manual refresh or on error

**Rationale**:
- Expensive to fetch (HTTP request)
- Rarely changes
- Reduces federation overhead

### 3. Activity Feeds

**Cache Levels**: L1, L2
**TTL**: 5 minutes
**Invalidation**: On new activity creation

**Rationale**:
- Expensive to compute (joins, filtering)
- Frequently accessed
- Short TTL for freshness

### 4. User Feeds

**Cache Levels**: L1, L2
**TTL**: 5 minutes
**Invalidation**: On new activity by user

**Rationale**:
- User-specific, frequently accessed
- Short TTL for real-time feel

### 5. Map Images

**Cache Levels**: L2 (metadata), L3 (files)
**TTL**: 24 hours (images don't change)
**Invalidation**: On activity update (if GPX changes)

**Rationale**:
- Expensive to generate
- Large files (better on disk)
- Rarely changes

### 6. OSM Tiles

**Cache Levels**: L3 (disk only)
**TTL**: 30 days (tiles rarely change)
**Invalidation**: Manual cleanup

**Rationale**:
- External resource (rate limits)
- Large volume of files
- Long-term storage

### 7. Shared Inbox URLs

**Cache Levels**: L1, L2
**TTL**: 1 hour
**Invalidation**: On discovery failure

**Rationale**:
- Discovered from actor profiles
- Reduces HTTP requests

## Implementation

### Dependencies

Add to `pom.xml`:

```xml
<!-- Redis Cache (L2) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-redis-cache</artifactId>
</dependency>
```

**Note**: `quarkus-redis-client` is already included for federation delivery queue.

### Configuration

`application.properties`:

```properties
# Redis Cache Configuration (L2)
# Uses default Redis client (same as federation queue)
quarkus.cache.redis.enabled=true

# Actor Profile Cache
quarkus.cache.redis.actorProfile.expire-after-write=1h
quarkus.cache.redis.actorProfile.value-type=org.openpace.core.Actor

# Remote Actor Profile Cache
quarkus.cache.redis.remoteActor.expire-after-write=24h
quarkus.cache.redis.remoteActor.value-type=org.openpace.core.Actor

# Activity Feed Cache
quarkus.cache.redis.activityFeed.expire-after-write=5m
quarkus.cache.redis.activityFeed.value-type=java.util.List

# User Feed Cache
quarkus.cache.redis.userFeed.expire-after-write=5m
quarkus.cache.redis.userFeed.value-type=java.util.List

# Shared Inbox Cache
quarkus.cache.redis.sharedInbox.expire-after-write=1h
quarkus.cache.redis.sharedInbox.value-type=java.lang.String

# Disk Cache Configuration (L3)
cache.disk.base-path=${CACHE_DISK_PATH:/var/cache/open-pace}
cache.disk.map-images.enabled=true
cache.disk.map-images.path=${cache.disk.base-path}/map-images
cache.disk.osm-tiles.enabled=true
cache.disk.osm-tiles.path=${cache.disk.base-path}/osm-tiles
cache.disk.osm-tiles.max-size-gb=10
```

### L1 Cache (In-Memory)

Quarkus automatically uses Caffeine for `@CacheResult` annotations when Redis cache is not configured for that cache name.

```java
@ApplicationScoped
public class ActorService {
    
    @CacheResult(cacheName = "actorProfile")
    public Actor getActor(String username) {
        return Actor.findByUsername(username);
    }
    
    @CacheInvalidate(cacheName = "actorProfile")
    public void invalidateActor(String username) {
        // Cache automatically invalidated
    }
}
```

### L2 Cache (Redis)

Use `@CacheResult` with Redis-backed cache names:

```java
@ApplicationScoped
public class ActorService {
    
    /**
     * Get actor profile (cached in Redis L2).
     */
    @CacheResult(cacheName = "actorProfile")
    public Actor getActor(String username) {
        Log.debugf("Cache miss for actor: %s", username);
        return Actor.findByUsername(username);
    }
    
    /**
     * Get remote actor profile (cached in Redis L2).
     */
    @CacheResult(cacheName = "remoteActor")
    public Actor getRemoteActor(String actorUrl) {
        Log.debugf("Fetching remote actor: %s", actorUrl);
        // Fetch from remote server
        return fetchRemoteActor(actorUrl);
    }
    
    /**
     * Invalidate actor cache on update.
     */
    @CacheInvalidate(cacheName = "actorProfile")
    public void updateActor(String username, Actor actor) {
        actor.persist();
    }
}
```

### L3 Cache (Disk)

Implement custom disk cache service:

```java
@ApplicationScoped
public class DiskCacheService {
    
    @ConfigProperty(name = "cache.disk.base-path")
    String basePath;
    
    @ConfigProperty(name = "cache.disk.map-images.path")
    String mapImagesPath;
    
    @ConfigProperty(name = "cache.disk.osm-tiles.path")
    String osmTilesPath;
    
    private Path mapImagesDir;
    private Path osmTilesDir;
    
    @PostConstruct
    void init() {
        // Create cache directories
        mapImagesDir = Paths.get(mapImagesPath);
        osmTilesDir = Paths.get(osmTilesPath);
        
        try {
            Files.createDirectories(mapImagesDir);
            Files.createDirectories(osmTilesDir);
            Log.infof("Disk cache initialized: %s", basePath);
        } catch (IOException e) {
            throw new RuntimeException("Failed to create cache directories", e);
        }
    }
    
    /**
     * Store map image to disk cache.
     */
    public void storeMapImage(String activityId, byte[] imageData, String format) {
        String filename = String.format("%s.%s", activityId, format);
        Path filePath = mapImagesDir.resolve(filename);
        
        try {
            Files.write(filePath, imageData);
            Log.debugf("Stored map image to cache: %s", filePath);
        } catch (IOException e) {
            Log.warnf(e, "Failed to store map image to cache: %s", filePath);
        }
    }
    
    /**
     * Retrieve map image from disk cache.
     */
    public Optional<byte[]> getMapImage(String activityId, String format) {
        String filename = String.format("%s.%s", activityId, format);
        Path filePath = mapImagesDir.resolve(filename);
        
        if (!Files.exists(filePath)) {
            return Optional.empty();
        }
        
        try {
            byte[] data = Files.readAllBytes(filePath);
            Log.debugf("Retrieved map image from cache: %s", filePath);
            return Optional.of(data);
        } catch (IOException e) {
            Log.warnf(e, "Failed to read map image from cache: %s", filePath);
            return Optional.empty();
        }
    }
    
    /**
     * Store OSM tile to disk cache.
     */
    public void storeOsmTile(int z, int x, int y, byte[] tileData) {
        // OSM tile path structure: z/x/y.png
        Path tilePath = osmTilesDir.resolve(String.format("%d/%d/%d.png", z, x, y));
        
        try {
            Files.createDirectories(tilePath.getParent());
            Files.write(tilePath, tileData);
            Log.debugf("Stored OSM tile to cache: %s", tilePath);
        } catch (IOException e) {
            Log.warnf(e, "Failed to store OSM tile to cache: %s", tilePath);
        }
    }
    
    /**
     * Retrieve OSM tile from disk cache.
     */
    public Optional<byte[]> getOsmTile(int z, int x, int y) {
        Path tilePath = osmTilesDir.resolve(String.format("%d/%d/%d.png", z, x, y));
        
        if (!Files.exists(tilePath)) {
            return Optional.empty();
        }
        
        try {
            byte[] data = Files.readAllBytes(tilePath);
            Log.debugf("Retrieved OSM tile from cache: %s", tilePath);
            return Optional.of(data);
        } catch (IOException e) {
            Log.warnf(e, "Failed to read OSM tile from cache: %s", tilePath);
            return Optional.empty();
        }
    }
    
    /**
     * Clean up old cache files.
     */
    @Scheduled(every = "24h")
    void cleanupOldFiles() {
        cleanupDirectory(mapImagesDir, Duration.ofDays(7));
        cleanupDirectory(osmTilesDir, Duration.ofDays(30));
    }
    
    private void cleanupDirectory(Path dir, Duration maxAge) {
        try {
            long cutoff = Instant.now().minus(maxAge).toEpochMilli();
            Files.walk(dir)
                .filter(Files::isRegularFile)
                .filter(path -> {
                    try {
                        return Files.getLastModifiedTime(path).toMillis() < cutoff;
                    } catch (IOException e) {
                        return false;
                    }
                })
                .forEach(path -> {
                    try {
                        Files.delete(path);
                        Log.debugf("Deleted old cache file: %s", path);
                    } catch (IOException e) {
                        Log.warnf(e, "Failed to delete cache file: %s", path);
                    }
                });
        } catch (IOException e) {
            Log.errorf(e, "Failed to cleanup cache directory: %s", dir);
        }
    }
}
```

### Feed Caching

```java
@ApplicationScoped
public class FeedService {
    
    @Inject
    DiskCacheService diskCache;
    
    /**
     * Get activity feed (cached in Redis L2).
     */
    @CacheResult(cacheName = "activityFeed")
    public List<Activity> getActivityFeed(int page, int limit) {
        Log.debugf("Cache miss for activity feed: page=%d, limit=%d", page, limit);
        return Activity.find("ORDER BY publishedAt DESC")
            .page(page, limit)
            .list();
    }
    
    /**
     * Get user feed (cached in Redis L2).
     */
    @CacheResult(cacheName = "userFeed")
    public List<Activity> getUserFeed(String username, int page, int limit) {
        Log.debugf("Cache miss for user feed: username=%s, page=%d", username, page);
        Actor actor = Actor.findByUsername(username);
        return Activity.find("actor = ?1 ORDER BY publishedAt DESC", actor)
            .page(page, limit)
            .list();
    }
    
    /**
     * Invalidate feeds when new activity is created.
     */
    @CacheInvalidateAll(cacheName = "activityFeed")
    @CacheInvalidate(cacheName = "userFeed")
    public void onActivityCreated(Activity activity) {
        // Caches automatically invalidated
    }
}
```

### Map Image Caching

```java
@ApplicationScoped
public class MapImageService {
    
    @Inject
    DiskCacheService diskCache;
    
    /**
     * Get map image with multi-level cache.
     */
    public byte[] getMapImage(String activityId, String format) {
        // L3: Check disk cache first
        Optional<byte[]> cached = diskCache.getMapImage(activityId, format);
        if (cached.isPresent()) {
            return cached.get();
        }
        
        // Generate map image
        byte[] image = generateMapImage(activityId, format);
        
        // Store in L3 cache
        diskCache.storeMapImage(activityId, image, format);
        
        return image;
    }
}
```

### OSM Tile Caching

```java
@ApplicationScoped
public class OsmTileService {
    
    @Inject
    DiskCacheService diskCache;
    
    /**
     * Get OSM tile with disk cache (L3).
     */
    public byte[] getOsmTile(int z, int x, int y) {
        // Check disk cache
        Optional<byte[]> cached = diskCache.getOsmTile(z, x, y);
        if (cached.isPresent()) {
            return cached.get();
        }
        
        // Fetch from OSM
        byte[] tile = fetchOsmTile(z, x, y);
        
        // Store in cache
        diskCache.storeOsmTile(z, x, y, tile);
        
        return tile;
    }
    
    private byte[] fetchOsmTile(int z, int x, int y) {
        String url = String.format(
            "https://tile.openstreetmap.org/%d/%d/%d.png",
            z, x, y
        );
        
        // Use Vert.x WebClient to fetch
        // ... implementation
    }
}
```

## Cache Invalidation Strategies

### 1. Time-Based (TTL)

**Use**: Most caches
**Configuration**: `expire-after-write` or `expire-after-access`

```properties
quarkus.cache.redis.actorProfile.expire-after-write=1h
```

### 2. Event-Based

**Use**: Feeds, user-specific data
**Implementation**: `@CacheInvalidate` on update methods

```java
@CacheInvalidate(cacheName = "userFeed")
public void createActivity(Activity activity) {
    activity.persist();
}
```

### 3. Manual Invalidation

**Use**: Admin operations, data fixes

```java
@Inject
CacheManager cacheManager;

public void clearAllCaches() {
    cacheManager.getCache("actorProfile").ifPresent(Cache::invalidateAll);
    cacheManager.getCache("activityFeed").ifPresent(Cache::invalidateAll);
}
```

## Cache Key Design

### Pattern: `{cache-name}:{identifier}`

**Examples**:
- Actor: `actorProfile:alice`
- Remote Actor: `remoteActor:https://mastodon.social/users/bob`
- User Feed: `userFeed:alice:0:20` (username:page:limit)
- Activity Feed: `activityFeed:0:20` (page:limit)
- Map Image: `map-image:123:png` (activityId:format)
- OSM Tile: `osm-tile:10:512:256` (z:x:y)

### Redis Key Structure

Redis cache automatically prefixes keys: `cache:{cache-name}:{key}`

So `actorProfile:alice` becomes: `cache:actorProfile:actorProfile:alice`

## Configuration Reference

### Redis Cache (L2)

```properties
# Enable Redis cache
quarkus.cache.redis.enabled=true

# Use specific Redis client (optional)
# quarkus.cache.redis.client-name=default

# Cache-specific configuration
quarkus.cache.redis.actorProfile.expire-after-write=1h
quarkus.cache.redis.actorProfile.value-type=org.openpace.core.Actor

quarkus.cache.redis.remoteActor.expire-after-write=24h
quarkus.cache.redis.remoteActor.value-type=org.openpace.core.Actor

quarkus.cache.redis.activityFeed.expire-after-write=5m
quarkus.cache.redis.activityFeed.value-type=java.util.List

quarkus.cache.redis.userFeed.expire-after-write=5m
quarkus.cache.redis.userFeed.value-type=java.util.List

quarkus.cache.redis.sharedInbox.expire-after-write=1h
quarkus.cache.redis.sharedInbox.value-type=java.lang.String
```

### Disk Cache (L3)

```properties
# Base path for disk cache (externalized, not temp dir)
cache.disk.base-path=${CACHE_DISK_PATH:/var/cache/open-pace}

# Map images
cache.disk.map-images.enabled=true
cache.disk.map-images.path=${cache.disk.base-path}/map-images

# OSM tiles
cache.disk.osm-tiles.enabled=true
cache.disk.osm-tiles.path=${cache.disk.base-path}/osm-tiles
cache.disk.osm-tiles.max-size-gb=10
```

**Environment Variable**: Set `CACHE_DISK_PATH` to override default path.

**Production**: Use persistent volume, not temp directory.

## Cache Warming

### On Application Start

```java
@ApplicationScoped
public class CacheWarmupService {
    
    @Inject
    ActorService actorService;
    
    @Inject
    FeedService feedService;
    
    void onStart(@Observes StartupEvent ev) {
        // Warm up frequently accessed actors
        List<String> popularUsers = getPopularUsers();
        for (String username : popularUsers) {
            actorService.getActor(username); // Triggers cache
        }
        
        // Warm up activity feed
        feedService.getActivityFeed(0, 20);
    }
}
```

## Monitoring

### Metrics to Track

- **Cache Hit Rate**: L1, L2, L3 hit rates
- **Cache Miss Rate**: Per cache name
- **Cache Size**: Redis memory usage, disk usage
- **Eviction Rate**: How often items are evicted
- **Response Time**: Cache access time vs database time

### Logging

```java
// Log cache hits/misses
@CacheResult(cacheName = "actorProfile")
public Actor getActor(String username) {
    Log.debugf("Cache miss for actor: %s", username);
    return Actor.findByUsername(username);
}
```

## Best Practices

### 1. Cache What Makes Sense

✅ **Cache**:
- Frequently accessed data
- Expensive to compute/fetch
- Rarely changes

❌ **Don't Cache**:
- User-specific sensitive data (unless encrypted)
- Frequently changing data
- Very large objects (use disk cache)

### 2. Set Appropriate TTLs

- **Short TTL** (minutes): Feeds, real-time data
- **Medium TTL** (hours): Actor profiles, shared data
- **Long TTL** (days): Static content, OSM tiles

### 3. Invalidate on Updates

Always invalidate cache when data is updated:

```java
@CacheInvalidate(cacheName = "actorProfile")
public void updateActor(Actor actor) {
    actor.persist();
}
```

### 4. Monitor Cache Performance

- Track hit rates
- Monitor memory usage
- Alert on low hit rates

### 5. Use Disk Cache for Large Files

- Map images: Use L3 (disk)
- OSM tiles: Use L3 (disk)
- Small objects: Use L1/L2 (memory)

## Troubleshooting

### Cache Not Working

1. **Check Redis connection**: Verify Redis is running
2. **Check configuration**: Ensure cache names match
3. **Check TTL**: Items may have expired
4. **Check serialization**: Ensure types are serializable

### High Memory Usage

1. **Reduce TTL**: Shorter cache lifetimes
2. **Reduce cache size**: Limit number of cached items
3. **Use disk cache**: Move large items to L3

### Stale Data

1. **Reduce TTL**: Faster expiration
2. **Add invalidation**: Invalidate on updates
3. **Check invalidation logic**: Ensure it's called

## Summary

**Cache Levels**:
- **L1**: In-memory (Caffeine) - Fast, instance-local
- **L2**: Redis - Distributed, shared
- **L3**: Disk - Large files, persistent

**What to Cache**:
- Actor profiles (L1/L2)
- Remote actors (L1/L2)
- Feeds (L1/L2)
- Map images (L2 metadata, L3 files)
- OSM tiles (L3)

**Configuration**:
- Redis cache: `quarkus-redis-cache` extension
- Disk cache: Configurable path via `cache.disk.base-path`
- TTL: Configure per cache name

**See Also**:
- [Quarkus Redis Cache Guide](https://quarkus.io/guides/cache-redis-reference)
- [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md) (uses same Redis)
- [Architectural Gaps](ARCHITECTURAL_GAPS.md)
