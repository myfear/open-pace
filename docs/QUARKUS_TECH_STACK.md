# Quarkus Technology Stack & Versions

This document details the Quarkus technologies and versions used in the Open Pace tutorial series, based on `open-pace-p1/pom.xml`.

## Platform Version

**Quarkus Platform**: `3.30.6`

All Quarkus extensions are managed through the Quarkus BOM (Bill of Materials), ensuring version compatibility across all dependencies.

## Java Version

**Java**: `21` (LTS)
- Source: `maven.compiler.source=21`
- Target: `maven.compiler.target=21`
- Preview features enabled (for future Java features)

**Note**: All parts require Java 21+. This is consistent across the entire tutorial series.

## Core Quarkus Extensions

### 1. **quarkus-arc**
**Purpose**: Dependency Injection and Contexts
- Provides CDI (Contexts and Dependency Injection) support
- Enables `@ApplicationScoped`, `@RequestScoped`, `@Singleton` annotations
- Used for: Service classes, repositories, REST resources

### 2. **quarkus-rest-jackson**
**Purpose**: REST API with Jackson JSON serialization
- RESTEasy Reactive for REST endpoints
- Jackson for JSON serialization/deserialization
- Used for: ActivityPub endpoints (WebFinger, Actor, Inbox, Outbox)
- **Why**: Perfect for ActivityPub JSON-LD serialization

### 3. **quarkus-hibernate-validator**
**Purpose**: Input validation and error handling
- Bean Validation (JSR 303) support
- Automatic validation of request DTOs
- Consistent error responses
- Used for: All REST endpoints requiring input validation
- **Why**: Ensures consistent validation and error handling across all parts

### 4. **quarkus-jdbc-postgresql**
**Purpose**: PostgreSQL JDBC Driver
- Database connectivity for PostgreSQL
- Used with: Hibernate ORM, Flyway
- **Why**: Standard database for ActivityPub data storage

### 5. **quarkus-hibernate-orm-panache**
**Purpose**: Hibernate ORM with Panache (Active Record pattern)
- Object-relational mapping
- Panache provides simplified data access
- Used for: Entity models (Actor, Activity, etc.)
- **Why**: Simplifies database operations, fits well with ActivityPub entities

### 6. **quarkus-flyway**
**Purpose**: Database Migration Tool
- Version-controlled database schema changes
- Runs migrations automatically on startup
- Used for: Creating tables, indexes, initial data
- **Why**: Essential for tutorial progression (each part adds migrations)

### 7. **quarkus-vertx**
**Purpose**: Vert.x Integration
- Async, non-blocking processing
- Event-driven architecture
- Used for: GPX processing, map generation, async federation delivery
- **Why**: Critical for performance (as outlined in tutorial concept)

### 8. **quarkus-smallrye-openapi**
**Purpose**: OpenAPI (Swagger) Documentation
- Automatic OpenAPI spec generation
- Swagger UI for interactive API documentation
- Used for: Documenting application endpoints (`/api/*`)
- **Why**: Essential for UI/mobile app developers to understand the API
- **Note**: Only documents application endpoints, not ActivityPub endpoints (see [API Design](API_DESIGN.md))

## Testing Extensions

### 9. **quarkus-junit5**
**Purpose**: JUnit 5 Testing Support
- Quarkus-specific test annotations (`@QuarkusTest`)
- Automatic test resource management
- Used for: Unit tests, integration tests
- **Scope**: `test`

### 10. **rest-assured**
**Purpose**: REST API Testing
- Fluent API for HTTP testing
- JSON validation
- Used for: Testing ActivityPub endpoints
- **Scope**: `test`
- **Version**: Managed by Quarkus BOM

## Maven Plugins

### Quarkus Maven Plugin
- **Version**: `3.30.6` (matches platform version)
- **Goals**:
  - `build`: Build the application
  - `generate-code`: Generate code at build time
  - `generate-code-tests`: Generate test code
  - `native-image-agent`: Native image support

### Maven Compiler Plugin
- **Version**: `3.11.0`
- **Configuration**: Java 17, preview features enabled

### Maven Surefire Plugin
- **Version**: `3.0.0`
- **Purpose**: Unit test execution

### Maven Failsafe Plugin
- **Version**: `3.0.0`
- **Purpose**: Integration test execution

## Build Profiles

### Native Profile
- **Activation**: `-Pnative` or `-Dnative`
- **Purpose**: Build native executable
- **Properties**:
  - `quarkus.native.enabled=true`
  - `quarkus.package.jar.enabled=false`

## Quarkus Dev Services

**Automatic Services** (no explicit dependency needed):
- **PostgreSQL**: Automatically starts PostgreSQL container in dev/test mode
- **Configuration**: Automatically configures database connection
- **Migrations**: Runs Flyway migrations automatically

## Technology Choices Rationale

### Why Quarkus 3.30.6?
- Latest stable version at project start
- Excellent ActivityPub support (REST, JSON, async)
- Strong Vert.x integration
- Great developer experience

### Why REST with Jackson?
- ActivityPub requires JSON-LD serialization
- Jackson handles complex JSON structures
- RESTEasy Reactive provides async REST

### Why Hibernate Panache?
- Simplifies entity management
- Active Record pattern is intuitive
- Good for tutorial progression

### Why Vert.x?
- **Core requirement**: Async GPX processing, map generation
- Non-blocking federation delivery
- WebSocket support for live tracking
- Perfect fit for ActivityPub webhook-style inbox POSTs

### Why Flyway?
- Version-controlled migrations
- Each tutorial part can add migrations
- Clear database evolution

## Version Management

All Quarkus extensions are managed through the **Quarkus BOM**:
```xml
<quarkus.platform.version>3.30.6</quarkus.platform.version>
```

This ensures:
- ✅ All extensions are compatible
- ✅ No version conflicts
- ✅ Easy upgrades (change one version number)

## Upgrading Versions

To upgrade Quarkus:

1. Update `quarkus.platform.version` in `pom.xml`
2. Test all functionality
3. Check for breaking changes in [Quarkus Migration Guide](https://quarkus.io/guides/quarkus-upgrade-guide)

## Dependencies Not Included (Yet)

These may be added in later parts:
- **hibernate-spatial**: For PostGIS/geospatial support (Part 7)
- **jts-core**: JTS geometry library for LineString, Point (Part 7)
- **jpx**: GPX file parsing library (Part 7)
- **quarkus-security-jpa**: For user authentication (Part 6) - includes BCrypt support
- **quarkus-redis-client**: For federation delivery queue (Part 5+) - see [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md)
- **quarkus-redis-cache**: For distributed caching (Part 3+) - see [Caching Strategy](CACHING_STRATEGY.md)
- **quarkus-smallrye-fault-tolerance**: For rate limiting and fault tolerance (Part 6+) - see [Rate Limiting Strategy](RATE_LIMITING_STRATEGY.md)
- **quarkus-scheduler**: For background job processing (Part 5+) - federation delivery workers
- **quarkus-websockets**: For live activity tracking (Part 3+)
- **quarkus-smallrye-jwt**: For OAuth2/JWT (Part 6+)
- **quarkus-mailer**: For email notifications (Part 6+)
- **quarkus-cache**: For performance optimization (Part 4)

**Note**: 
- `quarkus-hibernate-validator` is included from Part 1 for consistent error handling across all parts.
- `quarkus-smallrye-openapi` should be added when implementing application endpoints (`/api/*`) for API documentation.
- `quarkus-redis-client` and `quarkus-scheduler` are needed for reliable federation delivery (Part 5+).
- `quarkus-redis-cache` is needed for distributed caching (Part 3+).
- `quarkus-smallrye-fault-tolerance` is needed for rate limiting (Part 6+).

## Testing Stack

- **JUnit 5**: Test framework
- **REST Assured**: API testing
- **Quarkus Dev Services**: Automatic test database
- **Quarkus Test Resources**: For integration testing

## Development Tools

- **Quarkus Dev Mode**: `./mvnw quarkus:dev`
  - Hot reload
  - Dev Services (automatic database)
  - Dev UI at `/q/dev`

## Summary Table

| Technology | Version | Purpose | Part |
|------------|---------|---------|------|
| Quarkus Platform | 3.30.6 | Core framework | All |
| Java | 21 | Runtime | All |
| quarkus-arc | 3.30.6 | Dependency injection | All |
| quarkus-rest-jackson | 3.30.6 | REST API | All |
| quarkus-hibernate-validator | 3.30.6 | Input validation | All |
| quarkus-jdbc-postgresql | 3.30.6 | Database driver | All |
| quarkus-hibernate-orm-panache | 3.30.6 | ORM | All |
| quarkus-flyway | 3.30.6 | Migrations | All |
| quarkus-vertx | 3.30.6 | Async processing | All |
| quarkus-smallrye-openapi | 3.30.6 | OpenAPI documentation | Part 3+ (when /api/* endpoints added) |
| quarkus-redis-client | 3.30.6 | Redis client (federation queue) | Part 5+ |
| quarkus-redis-cache | 3.30.6 | Redis cache (L2 distributed cache) | Part 3+ |
| quarkus-smallrye-fault-tolerance | 3.30.6 | Rate limiting and fault tolerance | Part 6+ |
| quarkus-scheduler | 3.30.6 | Scheduled tasks (delivery workers) | Part 5+ |
| quarkus-junit5 | 3.30.6 | Testing | All |
| rest-assured | (BOM) | API testing | All |
| hibernate-spatial | (Hibernate) | PostGIS support | Part 7+ |
| jts-core | 1.18.2 | Geometry types (LineString, Point) | Part 7+ |
| jpx | 3.0.1 | GPX file parsing | Part 7+ |
| quarkus-security-jpa | 3.30.6 | User authentication (BCrypt) | Part 6+ |

## References

- [Quarkus Documentation](https://quarkus.io/)
- [Quarkus Extensions](https://quarkus.io/extensions/)
- [Quarkus Upgrade Guide](https://quarkus.io/guides/quarkus-upgrade-guide)
- [Vert.x Documentation](https://vertx.io/docs/)
- [Hibernate Panache](https://quarkus.io/guides/hibernate-orm-panache)
