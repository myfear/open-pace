# Project Setup Guide

This guide helps you set up the development environment for the Open Pace tutorial series.

## Prerequisites

### Required Software
- **Java 21+** (LTS): [Download](https://adoptium.net/)
- **Maven 3.8+**: [Download](https://maven.apache.org/download.cgi)
- **Docker**: Required for Quarkus Dev Services (automatic database setup)
- **Git**: [Download](https://git-scm.com/downloads)

### Optional but Recommended
- **Postman or Insomnia**: For API testing
- **Mastodon account**: For federation testing (or use a test instance)

## Initial Setup

### 1. Clone the Repository
```bash
git clone <repository-url>
cd open-pace
```

### 2. Verify Java Installation
```bash
java -version
# Should show Java 21 or higher
```

### 3. Verify Maven Installation
```bash
mvn -version
# Should show Maven 3.8 or higher
```

## Database Setup

### Quarkus Dev Services (Automatic)

**No manual setup required!** Quarkus Dev Services automatically:
- Starts a PostgreSQL container when you run `./mvnw quarkus:dev`
- Starts a test database when you run `./mvnw test`
- Configures connection properties automatically
- Tears down containers when you stop the application

Just make sure Docker is running, and Quarkus handles the rest!

### Manual Database (Optional)

If you prefer to use a local PostgreSQL instance instead of Dev Services:

1. Install PostgreSQL locally
2. Create database:
   ```sql
   CREATE DATABASE openpace;
   CREATE USER openpace WITH PASSWORD 'openpace';
   GRANT ALL PRIVILEGES ON DATABASE openpace TO openpace;
   ```
3. Disable Dev Services in `application.properties`:
   ```properties
   quarkus.datasource.devservices.enabled=false
   quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/openpace
   quarkus.datasource.username=openpace
   quarkus.datasource.password=openpace
   ```

## Development Environment

### IDE Setup
Recommended IDEs:
- **IntelliJ IDEA**: Excellent Quarkus support
- **VS Code**: With Java extensions
- **Eclipse**: With Quarkus tools

### Quarkus Dev Mode
Start the application in development mode:
```bash
./mvnw quarkus:dev
```

This provides:
- **Hot reload** on code changes
- **Live coding** support
- **Dev UI** at `http://localhost:8080/q/dev`
- **Automatic database** via Quarkus Dev Services (PostgreSQL container)
- **Automatic configuration** of database connection

## Configuration

### Application Properties
Each tutorial part may have different configuration. Check:
- `src/main/resources/application.properties` (base config)
- `src/main/resources/application-part1.properties` (part-specific)

### Environment Variables
With Quarkus Dev Services, you typically don't need to set these manually. However, if you're using a manual database setup, you can override:

```bash
export QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://localhost:5432/openpace
export QUARKUS_DATASOURCE_USERNAME=openpace
export QUARKUS_DATASOURCE_PASSWORD=openpace
```

## Testing Setup

### Running Tests
```bash
# All tests
./mvnw test

# Specific test class
./mvnw test -Dtest=WebFingerTest

# Integration tests
./mvnw verify
```

### Test Data
Each part includes test data. Check:
- `src/test/resources/` for test fixtures
- `examples/part-X/` for example requests/responses

## Federation Testing

### Local Testing
For local development, you can:
1. Run multiple instances on different ports
2. Use `localhost` with different ports as "domains"
3. Configure hosts file to use different domains

### Hosts File Setup (Optional)
Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
```
127.0.0.1 openpace.local
127.0.0.1 openpace2.local
```

Then access as:
- `http://openpace.local:8080`
- `http://openpace2.local:8081`

### Mastodon Testing
1. Create account on a Mastodon instance (or use test instance)
2. Note your instance URL
3. Follow users from your Open Pace instance
4. Verify activities appear in feeds

## Configuration Management

### Base URL Configuration

**Decision**: Use MicroProfile Config for base URL configuration.

**Rationale**:
- Environment-specific configuration
- Works with Quarkus Dev Services
- Can override via environment variables

**Configuration**:
```properties
# application.properties
openpace.domain=localhost:8080
openpace.scheme=http
```

**Java Code**:
```java
@ConfigProperty(name = "openpace.scheme", defaultValue = "http")
String scheme;

@ConfigProperty(name = "openpace.domain", defaultValue = "localhost:8080")
String domain;

public String getBaseUrl() {
    return scheme + "://" + domain;
}
```

**Production Considerations**:
- Set `openpace.domain` to actual domain
- Set `openpace.scheme` to `https` for SSL
- Can use environment variables: `OPENPACE_DOMAIN`, `OPENPACE_SCHEME`

**Environment Variables**:
```bash
export OPENPACE_DOMAIN=openpace.example.com
export OPENPACE_SCHEME=https
```

### Activity ID Generation

**Pattern**: Generate ActivityPub IDs as URLs using base URL + path pattern.

```
{baseUrl}/users/{username}/activities/{databaseId}
```

**Implementation**:
```java
dbActivity.activityId = getBaseUrl() + "/users/" + actor.username + "/activities/" + dbActivity.id;
```

**Important**: 
- IDs must be stable (don't change after creation)
- Should be globally unique across all instances
- Consider using UUIDs for database IDs to avoid collisions

## Troubleshooting

### Port Already in Use
If port 8080 is in use:
```bash
# Change in application.properties
quarkus.http.port=8081
```

### Database Connection Issues

**If using Quarkus Dev Services:**
1. Verify Docker is running:
   ```bash
   docker ps
   ```
2. Check Dev Services logs in Quarkus Dev UI: `http://localhost:8080/q/dev`
3. Dev Services should automatically start a PostgreSQL container

**If using manual database:**
1. Verify PostgreSQL is running:
   ```bash
   docker ps  # if using Docker
   # or check service status for local PostgreSQL
   ```
2. Check connection string in `application.properties`
3. Verify `quarkus.datasource.devservices.enabled=false` is set

### Maven Wrapper Issues
If `./mvnw` doesn't work:
```bash
# Make executable (Linux/Mac)
chmod +x mvnw

# Or use mvn directly
mvn quarkus:dev
```

## Next Steps

1. ✅ Environment set up
2. 📖 Read [Part 1 Tutorial](part-1-basic-activitypub.md)
3. 🚀 Start building!

## Getting Help

- Check [Troubleshooting](part-1-basic-activitypub.md#common-issues--solutions) in each tutorial part
- Review [ActivityPub Reference](ACTIVITYPUB_REFERENCE.md)
- Consult [Consistency Checklist](CONSISTENCY_CHECKLIST.md) for quality standards
