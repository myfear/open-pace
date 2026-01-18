# Mapping Integration with OpenStreetMap

This document outlines the mapping integration strategy for Open Pace, covering GPX processing, geospatial storage, map image generation, and OSM tile integration.

## Overview

This integration adds comprehensive mapping capabilities:
- **GPX Processing**: Parse GPX files to extract track data
- **Geospatial Storage**: Store simplified tracks as PostGIS LineString
- **Map Image Generation**: Generate static map images with tracks overlaid on OSM tiles
- **OSM Integration**: Use public OpenStreetMap tiles with proper attribution
- **ActivityPub Integration**: Display maps in ActivityPub posts

## Part Placement

**Decision**: This is **Part 7: Mapping Integration**.

**Rationale**:
- Mapping requires PostGIS, geospatial processing, and OSM integration
- Complex enough to warrant its own focused part
- Builds on Parts 1-6 (needs activities, custom types, and security)
- Natural progression after security (Part 6)
- Allows Part 3 to focus on comments/likes/attachments without mapping complexity

## Requirements

### 1. GPX Processing

#### Parse GPX Files
- **Input**: GPX file (XML format) or base64-encoded GPX data
- **Extract**: Track points (lat/lon, elevation, timestamps)
- **Simplify**: Reduce point count using Douglas-Peucker algorithm
- **Output**: Simplified coordinate array

#### GPX Structure
```xml
<gpx>
  <trk>
    <trkseg>
      <trkpt lat="40.7128" lon="-74.0060">
        <ele>10.5</ele>
        <time>2026-01-17T07:00:00Z</time>
      </trkpt>
      <!-- more points -->
    </trkseg>
  </trk>
</gpx>
```

#### Implementation
```java
@Service
public class GPXService {
    
    /**
     * Parse GPX file and extract track points.
     */
    public List<Coordinate> parseGPX(String gpxData) {
        // Parse XML
        // Extract trkpt elements
        // Return List<Coordinate>
    }
    
    /**
     * Simplify track using Douglas-Peucker algorithm.
     * Reduces point count while preserving track shape.
     */
    public List<Coordinate> simplifyTrack(
            List<Coordinate> points, 
            double tolerance) {
        // Apply Douglas-Peucker algorithm
        // Return simplified points
    }
    
    /**
     * Calculate bounding box for track.
     */
    public BoundingBox calculateBoundingBox(List<Coordinate> points) {
        // Find min/max lat/lon
        // Return bounding box
    }
}
```

### 2. Geospatial Storage with PostGIS

#### Dependencies
```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-spatial</artifactId>
</dependency>
<dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.18.2</version>
</dependency>
```

#### Configuration
```properties
# Enable PostGIS dialect
quarkus.hibernate-orm.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect

# PostGIS extension (in migration)
# CREATE EXTENSION IF NOT EXISTS postgis;
```

#### Entity with LineString
```java
@Entity
@Table(name = "activities")
public class Activity extends PanacheEntity {
    
    // ... other fields ...
    
    /**
     * Simplified track stored as PostGIS LineString.
     * SRID 4326 = WGS84 (lat/lon)
     */
    @Column(columnDefinition = "geometry(LineString, 4326)")
    public LineString track;
    
    /**
     * Start point (for quick access).
     */
    @Column(columnDefinition = "geometry(Point, 4326)")
    public Point startPoint;
    
    /**
     * End point (for quick access).
     */
    @Column(columnDefinition = "geometry(Point, 4326)")
    public Point endPoint;
}
```

#### Database Migration
```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Add track column to activities table
ALTER TABLE activities 
ADD COLUMN track geometry(LineString, 4326);

-- Add start/end points
ALTER TABLE activities 
ADD COLUMN start_point geometry(Point, 4326),
ADD COLUMN end_point geometry(Point, 4326);

-- Create spatial index for performance
CREATE INDEX idx_activities_track ON activities USING GIST (track);
CREATE INDEX idx_activities_start_point ON activities USING GIST (start_point);
```

### 3. Map Image Generation

#### Static Map Image Creation
- **Input**: Track coordinates, bounding box
- **Process**: 
  1. Fetch OSM tiles for bounding box
  2. Composite tiles into base map
  3. Draw simplified track on map
  4. Draw start/end markers
  5. Add attribution text
- **Output**: PNG image (or SVG for scalability)

#### Implementation
```java
@Service
public class MapImageService {
    
    /**
     * Generate static map image with track overlay.
     */
    public byte[] generateMapImage(
            LineString track,
            Point startPoint,
            Point endPoint) {
        
        // 1. Calculate bounding box
        BoundingBox bbox = calculateBoundingBox(track);
        
        // 2. Determine zoom level (based on bbox size)
        int zoom = calculateZoomLevel(bbox);
        
        // 3. Fetch OSM tiles
        List<Tile> tiles = fetchOSMTiles(bbox, zoom);
        
        // 4. Composite tiles into base map image
        BufferedImage baseMap = compositeTiles(tiles);
        
        // 5. Draw track on map
        Graphics2D g = baseMap.createGraphics();
        drawTrack(g, track, bbox, zoom);
        
        // 6. Draw start/end markers
        drawMarker(g, startPoint, Color.GREEN, "Start");
        drawMarker(g, endPoint, Color.RED, "End");
        
        // 7. Add attribution
        drawAttribution(g, "© OpenStreetMap contributors");
        
        // 8. Convert to PNG
        return imageToBytes(baseMap);
    }
    
    /**
     * Fetch OSM tiles for bounding box.
     * Uses public OSM tile server: https://tile.openstreetmap.org/{z}/{x}/{y}.png
     */
    private List<Tile> fetchOSMTiles(BoundingBox bbox, int zoom) {
        // Calculate tile coordinates
        // Fetch tiles (with caching)
        // Return list of tiles
    }
    
    /**
     * Draw track line on map.
     * Convert lat/lon to pixel coordinates.
     */
    private void drawTrack(
            Graphics2D g, 
            LineString track, 
            BoundingBox bbox, 
            int zoom) {
        
        // Convert coordinates to pixels
        // Draw polyline
        g.setStroke(new BasicStroke(3));
        g.setColor(Color.BLUE);
        // ... draw path
    }
}
```

### 4. Coordinate Transformation

#### Lat/Lon to Pixel Conversion
```java
/**
 * Convert lat/lon to pixel coordinates on map.
 * Uses Web Mercator projection (EPSG:3857) for tile coordinates.
 */
public class CoordinateConverter {
    
    /**
     * Convert lat/lon to tile coordinates.
     */
    public static Point2D latLonToTile(double lat, double lon, int zoom) {
        double n = Math.pow(2, zoom);
        double x = (lon + 180.0) / 360.0 * n;
        double latRad = Math.toRadians(lat);
        double y = (1.0 - Math.log(Math.tan(latRad) + 
            (1.0 / Math.cos(latRad))) / Math.PI) / 2.0 * n;
        return new Point2D.Double(x, y);
    }
    
    /**
     * Convert tile coordinates to pixel coordinates.
     */
    public static Point2D tileToPixel(
            double tileX, 
            double tileY, 
            int tileSize) {
        int pixelX = (int) ((tileX % 1) * tileSize);
        int pixelY = (int) ((tileY % 1) * tileSize);
        return new Point2D.Double(pixelX, pixelY);
    }
}
```

### 5. OSM Tile Integration

#### Tile Server
- **URL Pattern**: `https://tile.openstreetmap.org/{z}/{x}/{y}.png`
- **Tile Size**: 256x256 pixels
- **Max Zoom**: 19 (typically)
- **Usage Policy**: [OSM Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)

#### Tile Caching
- **Local Cache**: Cache tiles to disk/Redis
- **Respect Rate Limits**: OSM has usage policies
- **User-Agent**: Set proper user-agent header

#### Implementation
```java
@Service
public class OSMTileService {
    
    private static final String OSM_TILE_URL = 
        "https://tile.openstreetmap.org/{z}/{x}/{y}.png";
    
    /**
     * Fetch OSM tile with caching.
     */
    public BufferedImage fetchTile(int z, int x, int y) {
        // Check cache first
        // If not cached, fetch from OSM
        // Cache tile
        // Return image
    }
    
    /**
     * Calculate tile coordinates for bounding box.
     */
    public List<TileCoordinate> getTilesForBoundingBox(
            BoundingBox bbox, 
            int zoom) {
        // Calculate min/max tile coordinates
        // Return list of tile coordinates
    }
}
```

### 6. Attribution

#### OSM Attribution Requirements
- **Required**: "© OpenStreetMap contributors"
- **Placement**: Bottom-right corner of map
- **Style**: Small text, readable
- **Link**: Optional link to OSM (if clickable)

#### Implementation
```java
private void drawAttribution(Graphics2D g, int width, int height) {
    String attribution = "© OpenStreetMap contributors";
    
    Font font = new Font("Arial", Font.PLAIN, 10);
    g.setFont(font);
    g.setColor(Color.BLACK);
    
    FontMetrics fm = g.getFontMetrics();
    int textWidth = fm.stringWidth(attribution);
    int x = width - textWidth - 10;
    int y = height - 10;
    
    // Draw with background for readability
    g.setColor(new Color(255, 255, 255, 200));
    g.fillRect(x - 5, y - fm.getAscent() - 2, 
               textWidth + 10, fm.getHeight() + 4);
    
    g.setColor(Color.BLACK);
    g.drawString(attribution, x, y);
}
```

### 7. ActivityPub Integration

#### Store Map Image
```java
// When creating activity
Activity activity = new Activity();
// ... set other fields ...

// Generate map image
byte[] mapImage = mapImageService.generateMapImage(
    activity.track, 
    activity.startPoint, 
    activity.endPoint);

// Store image (save to disk/S3, get URL)
String mapImageUrl = imageStorageService.storeImage(mapImage, "map-" + activity.id + ".png");

// Add to activity attachment
activity.attachment = List.of(
    new Attachment("Image", mapImageUrl, "image/png")
);
```

#### ActivityPub Object with Map
```json
{
  "type": "Create",
  "object": {
    "type": "https://fedisports.example/ns#Run",
    "name": "Morning 5K",
    "distance": 5000,
    "attachment": [
      {
        "type": "Image",
        "url": "https://openpace.example/activities/123/map.png",
        "mediaType": "image/png"
      }
    ]
  }
}
```

## Implementation Steps

### Step 1: Add Dependencies
```xml
<!-- Hibernate Spatial -->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-spatial</artifactId>
</dependency>

<!-- JTS Geometry -->
<dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.18.2</version>
</dependency>

<!-- GPX Parsing -->
<dependency>
    <groupId>io.jenetics</groupId>
    <artifactId>jpx</artifactId>
    <version>3.0.1</version>
</dependency>
```

### Step 2: Enable PostGIS
```sql
-- Migration: V3__enable_postgis.sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

### Step 3: Update Activity Entity
- Add `track` (LineString) field
- Add `startPoint` and `endPoint` (Point) fields
- Add spatial indexes

### Step 4: Create GPX Service
- Parse GPX files
- Extract track points
- Simplify track (Douglas-Peucker)

### Step 5: Create Map Image Service
- Fetch OSM tiles
- Composite tiles
- Draw track
- Add attribution

### Step 6: Integrate with Activity Creation
- Parse GPX when activity created
- Store track in PostGIS
- Generate map image
- Add to activity attachment

## Performance Considerations

### Track Simplification
- **Why**: Reduce storage and rendering complexity
- **Algorithm**: Douglas-Peucker
- **Tolerance**: Configurable (e.g., 10 meters)
- **Result**: 90%+ point reduction while preserving shape

### Tile Caching
- **Strategy**: Cache tiles locally
- **Storage**: Disk cache or Redis
- **TTL**: Tiles don't change often, long cache
- **Rate Limiting**: Respect OSM usage policies

### Image Generation
- **Async**: Use Vert.x for non-blocking image generation
- **Caching**: Cache generated map images
- **Size**: Optimize image size (e.g., 800x600px)

### Database Indexing
- **Spatial Index**: GIST index on track column
- **Queries**: Fast spatial queries (distance, intersection)

## Testing

### Unit Tests
- GPX parsing
- Track simplification
- Coordinate transformation
- Map image generation

### Integration Tests
- PostGIS storage/retrieval
- OSM tile fetching
- End-to-end: GPX → Track → Map Image

### Manual Tests
```bash
# Create activity with GPX
curl -X POST http://localhost:8080/users/alice/outbox \
  -u alice:password \
  -H "Content-Type: application/activity+json" \
  -d '{
    "type": "Create",
    "object": {
      "type": "Run",
      "gpxData": "<base64-encoded-gpx>"
    }
  }'

# Check map image URL in response
# Verify map image is accessible
curl http://localhost:8080/activities/123/map.png
```

## OSM Usage Policy

**Important**: Follow [OSM Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)

- ✅ **Allowed**: Moderate use for displaying maps
- ⚠️ **Required**: Proper attribution
- ⚠️ **Required**: Respectful user-agent
- ❌ **Not Allowed**: Heavy tile scraping
- 💡 **Alternative**: Consider self-hosting tiles or using commercial providers for high volume

## Future Enhancements

### Short Term
- [ ] Elevation profile generation
- [ ] Multiple track colors (by speed/elevation)
- [ ] Custom map styles
- [ ] Interactive maps (web UI)

### Medium Term
- [ ] Offline tile caching
- [ ] Alternative tile providers (Mapbox, etc.)
- [ ] Vector tiles
- [ ] 3D visualization

### Long Term
- [ ] Real-time tracking
- [ ] Heat maps
- [ ] Route planning
- [ ] Segment matching

## References

- [Hibernate Spatial Documentation](https://docs.jboss.org/hibernate/orm/5.0/userguide/html_single/chapters/query/spatial/Spatial.html)
- [PostGIS Documentation](https://postgis.net/documentation/)
- [OpenStreetMap Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)
- [OSM Tile Server](https://wiki.openstreetmap.org/wiki/Tile_servers)
- [JTS Geometry Library](https://github.com/locationtech/jts)
- [Douglas-Peucker Algorithm](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)

## Summary

**Part 7: Mapping Integration** adds:
- ✅ GPX parsing and track extraction
- ✅ PostGIS storage (LineString)
- ✅ Static map image generation
- ✅ OSM tile integration
- ✅ Proper attribution
- ✅ ActivityPub integration (map in attachments)

This makes activities visually rich and interoperable with Mastodon and other ActivityPub clients.
