# Security Integration Strategy

This document outlines the security integration approach for the Open Pace tutorial series, covering authentication, authorization, and ActivityPub-specific security requirements.

## Should Security Be Part 6?

**Recommendation: Yes, make it Part 6**, but with an important consideration:

### Option A: Part 6 (Recommended)
**Pros:**
- ✅ Keeps security concerns separate from core functionality
- ✅ Allows Parts 1-5 to focus on ActivityPub features
- ✅ Security can be added as a complete, cohesive unit
- ✅ Easier to test federation without auth complexity initially

**Cons:**
- ❌ Parts 1-5 won't have realistic authentication
- ❌ C2S pattern requires auth (only actor can post to their outbox)
- ❌ May need to retrofit security into earlier parts

### Option B: Integrate Earlier (Alternative)
**Pros:**
- ✅ More realistic from the start
- ✅ C2S endpoints properly secured from Part 1
- ✅ Better security posture throughout

**Cons:**
- ❌ Adds complexity to early tutorial parts
- ❌ May distract from learning ActivityPub concepts
- ❌ Harder to test federation without auth setup

**Decision**: **Part 6** - Add security as a complete integration after core functionality is built.

## Security Requirements

### 1. User Registration & Authentication

#### User Signup
- **Endpoint**: `POST /api/auth/register`
- **Fields**: username, password, email (optional for now)
- **Validation**: Username uniqueness, password strength
- **Response**: User created, can log in

#### User Login
- **Method**: HTTP Basic Authentication
- **Configuration**: `quarkus.http.auth.basic=true`
- **Provider**: Quarkus Security JPA
- **Storage**: Jakarta Persistence entity

### 2. Password Security

#### Password Hashing
- **Algorithm**: BCrypt (default for Quarkus Security JPA)
- **Implementation**: Built-in BCrypt support via `BcryptUtil`
- **Storage**: Modular Crypt Format (MCF)
- **Salt**: Random, per-password (handled automatically)
- **Iterations**: 10 rounds (default, configurable)

**Why BCrypt?**
- Industry standard, well-tested
- Built-in Quarkus support (no custom provider needed)
- Adequate security for most use cases
- Simple implementation

**Reference**: [Quarkus Security JPA Password Storage](https://quarkus.io/guides/security-jpa#password-storage-and-hashing)

### 3. User Entity Structure

```java
@Entity
@Table(name = "users")  // Note: not "user" (reserved keyword)
@UserDefinition
public class User extends PanacheEntity {
    @Username
    @Column(unique = true, nullable = false)
    public String username;
    
    @Password  // Default: BCrypt
    @Column(nullable = false)
    public String password;
    
    @Column
    public String email;  // Optional for now
    
    @Column(nullable = false)
    public Boolean verified = true;  // Defaults to true, for future email verification
    
    @Roles
    @Column(nullable = false)
    public String role = "user";  // Default role
    
    @OneToOne(mappedBy = "user")
    public Actor actor;  // Link to ActivityPub Actor
    
    @Column(name = "created_at")
    public Instant createdAt;
    
    @Column(name = "updated_at")
    public Instant updatedAt;
}
```

### 4. Actor-User Relationship

**Important**: Link `User` (authentication) with `Actor` (ActivityPub identity)

```java
// In Actor entity
@OneToOne
@JoinColumn(name = "user_id")
public User user;
```

**Why?**
- `User` handles authentication (username/password)
- `Actor` handles ActivityPub identity (ActivityPub ID, inbox, outbox)
- One-to-one relationship ensures each user has one actor

### 5. Endpoint Security

#### Public Endpoints (No Auth Required)
- `GET /.well-known/webfinger` - User discovery
- `GET /users/{username}` - Actor profile (public)
- `GET /users/{username}/outbox` - Public activity stream
- `GET /users/{username}/followers` - Public followers list
- `GET /users/{username}/following` - Public following list
- `POST /users/{username}/inbox` - Server-to-server (uses HTTP signatures, not Basic auth)

#### Protected Endpoints (Require Authentication)
- `POST /users/{username}/outbox` - **C2S: Only the actor can post to their own outbox**
- `POST /api/auth/register` - Registration (public, but rate-limited in production)
- `GET /api/auth/me` - Current user info
- `PUT /users/{username}` - Update actor profile (only own profile)
- `DELETE /users/{username}/activities/{id}` - Delete activity (only own activities)

#### Authorization Rules
```java
// Only actor can post to their own outbox
@POST
@Path("/users/{username}/outbox")
@RolesAllowed("user")
public Response postToOutbox(
        @PathParam("username") String username,
        @Context SecurityIdentity securityIdentity,
        JsonNode activityJson) {
    
    // Verify the authenticated user owns this outbox
    if (!securityIdentity.getPrincipal().getName().equals(username)) {
        return Response.status(403).entity("Cannot post to another user's outbox").build();
    }
    
    // Process activity...
}
```

### 6. ActivityPub-Specific Security

#### Client-to-Server (C2S) Authentication
- **Requirement**: Only the actor can POST to their own outbox
- **Method**: HTTP Basic Authentication
- **Verification**: Check authenticated username matches outbox owner
- **Implementation**: Use `@RolesAllowed` + manual username check

#### Server-to-Server (S2S) Authentication
- **Requirement**: Verify HTTP signatures on inbox POSTs
- **Method**: HTTP Signatures (RFC 9421)
- **Status**: **Future work** (Part 6 focuses on C2S, S2S can be Part 8 or later)
- **Note**: Inbox currently accepts all POSTs (for tutorial simplicity)

### 7. Configuration

#### application.properties
```properties
# Enable Basic Authentication
quarkus.http.auth.basic=true

# Security JPA
quarkus.security.users.embedded.enabled=false
quarkus.security.jpa.enabled=true

# Password Provider (BCrypt is default, no configuration needed)
# quarkus.security.jpa uses BCrypt by default

# CORS (for web clients)
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:3000,http://localhost:8080
quarkus.http.cors.headers=accept,authorization,content-type
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS
```

## Implementation Plan

### Step 1: Add Dependencies

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-security-jpa</artifactId>
</dependency>
```

**Note**: BCrypt support is included in `quarkus-security-jpa`, no additional dependencies needed.

### Step 2: Create User Entity

- Create `User` entity with `@UserDefinition`
- Add `verified` field (defaults to `true`)
- Link to `Actor` entity
- Add password field with custom provider

### Step 3: Use BCrypt for Password Hashing

```java
import io.quarkus.elytron.security.common.BcryptUtil;

public class User extends PanacheEntity {
    // ... fields ...
    
    /**
     * Hash password using BCrypt.
     * Use this when creating/updating user passwords.
     */
    public static String hashPassword(String plainPassword) {
        return BcryptUtil.bcryptHash(plainPassword);
    }
    
    /**
     * Create user with hashed password.
     */
    public static void add(String username, String password, String role) {
        User user = new User();
        user.username = username;
        user.password = hashPassword(password);  // Hash with BCrypt
        user.role = role;
        user.verified = true;
        user.persist();
    }
}
```

**Note**: BCrypt is the default for Quarkus Security JPA, so no custom provider is needed. The `@Password` annotation without parameters uses BCrypt automatically.

### Step 4: Create Registration Endpoint

```java
@Path("/api/auth")
public class AuthResource {
    
    @POST
    @Path("/register")
    @PermitAll  // Public endpoint
    public Response register(RegistrationRequest request) {
        // Validate username uniqueness
        // Hash password
        // Create User
        // Create Actor (linked to User)
        // Return success
    }
}
```

### Step 5: Secure Outbox POST

```java
@POST
@Path("/users/{username}/outbox")
@RolesAllowed("user")  // Require authentication
@Consumes("application/activity+json")
public Response postToOutbox(
        @PathParam("username") String username,
        @Context SecurityIdentity securityIdentity,
        JsonNode activityJson) {
    
    // Verify ownership
    if (!securityIdentity.getPrincipal().getName().equals(username)) {
        return Response.status(403).build();
    }
    
    // Process activity...
}
```

### Step 6: Update Actor Entity

- Add `user` relationship
- Ensure Actor is created when User registers
- Update Actor creation logic

## Database Migration

```sql
-- Create users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,  -- MCF format hash
    email VARCHAR(255),
    verified BOOLEAN NOT NULL DEFAULT true,
    role VARCHAR(50) NOT NULL DEFAULT 'user',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Add user_id to actors table
ALTER TABLE actors ADD COLUMN user_id BIGINT;
ALTER TABLE actors ADD CONSTRAINT fk_actor_user 
    FOREIGN KEY (user_id) REFERENCES users(id);

-- Create index
CREATE INDEX idx_actors_user_id ON actors(user_id);
```

## What Else Might Be Missing?

### ✅ Covered
- [x] User registration
- [x] User authentication (Basic Auth)
- [x] Password hashing (BCrypt)
- [x] Verified field for future email verification
- [x] Endpoint security (public vs protected)
- [x] C2S authorization (actor ownership)

### ⚠️ Future Considerations (Not in Part 6)

#### Short Term (Part 6+)
- [ ] **CORS Configuration** - For web client access
- [x] **Rate Limiting** - Prevent abuse (registration, login attempts) - See [Rate Limiting Strategy](RATE_LIMITING_STRATEGY.md)
- [ ] **Session Management** - If moving away from Basic Auth
- [ ] **Password Reset** - Forgot password flow
- [ ] **Email Verification** - Use the `verified` field
- [ ] **Account Deletion** - User can delete their account

#### Medium Term (Part 8+)
- [x] **HTTP Signatures** - For S2S (server-to-server) authentication - See [HTTP Signatures Strategy](HTTP_SIGNATURES_STRATEGY.md)
- [ ] **OAuth2/OIDC** - Alternative to Basic Auth for web clients
- [ ] **Two-Factor Authentication (2FA)** - Enhanced security
- [ ] **API Keys** - For programmatic access
- [ ] **Role-Based Access Control (RBAC)** - Admin, moderator roles
- [ ] **Audit Logging** - Track security events

#### Long Term (Production)
- [ ] **Account Lockout** - After failed login attempts
- [ ] **IP Whitelisting** - For admin endpoints
- [ ] **Security Headers** - HSTS, CSP, etc.
- [ ] **Input Validation** - Prevent injection attacks
- [ ] **Content Security Policy** - XSS protection

## Testing Strategy

### Unit Tests
- Password hashing/verification
- User registration validation
- Authorization checks

### Integration Tests
- Registration flow
- Login flow
- Outbox POST with authentication
- Outbox POST with wrong user (should fail)

### Manual Tests
```bash
# Register user
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secure123"}'

# Login (Basic Auth)
curl -X GET http://localhost:8080/api/auth/me \
  -u alice:secure123

# Post to outbox (authenticated)
curl -X POST http://localhost:8080/users/alice/outbox \
  -u alice:secure123 \
  -H "Content-Type: application/activity+json" \
  -d '{"type":"Create","object":{"type":"Note","content":"Hello"}}'

# Try to post to another user's outbox (should fail)
curl -X POST http://localhost:8080/users/bob/outbox \
  -u alice:secure123 \
  -H "Content-Type: application/activity+json" \
  -d '{"type":"Create","object":{"type":"Note","content":"Hello"}}'
```

## Security Best Practices

### Password Requirements
- Minimum length: 12 characters (configurable)
- Require mix of character types (optional, but recommended)
- No common passwords (check against common password list)

### Authentication
- Use HTTPS in production (Basic Auth sends credentials in header)
- Consider OAuth2/OIDC for web clients (future)
- Implement rate limiting on login attempts

### Authorization
- **Principle of Least Privilege**: Users can only access their own resources
- **Verify ownership**: Always check username matches path parameter
- **Fail securely**: Return 403, not 404 (don't reveal existence)

### Data Protection
- Hash passwords (never store plaintext)
- Use parameterized queries (prevent SQL injection)
- Validate all input
- Sanitize output (prevent XSS)

## Migration from Parts 1-5

When adding security to existing parts:

1. **Add User entity** and migration
2. **Link Actor to User** (one-to-one)
3. **Create registration endpoint**
4. **Secure outbox POST** (add auth check)
5. **Update tests** to include authentication
6. **Update documentation** with auth requirements

## References

- [Quarkus Security JPA Guide](https://quarkus.io/guides/security-jpa)
- [Quarkus Security Overview](https://quarkus.io/guides/security-overview)
- [BCrypt Algorithm](https://en.wikipedia.org/wiki/Bcrypt)
- [ActivityPub C2S Specification](https://www.w3.org/TR/activitypub/#client-to-server-interactions)
- [HTTP Signatures (RFC 9421)](https://www.rfc-editor.org/rfc/rfc9421.html)

## Summary

**Part 6: Security Integration** will add:
- ✅ User registration and authentication
- ✅ Password hashing (BCrypt)
- ✅ Verified field for future email verification
- ✅ Secure C2S endpoints (outbox POST)
- ✅ Actor-User relationship
- ✅ Basic authorization (actor ownership)

**Future work** (beyond Part 6):
- HTTP Signatures for S2S - See [HTTP Signatures Strategy](HTTP_SIGNATURES_STRATEGY.md)
- OAuth2/OIDC for web clients
- Email verification
- Rate limiting - See [Rate Limiting Strategy](RATE_LIMITING_STRATEGY.md)
- Enhanced security features
