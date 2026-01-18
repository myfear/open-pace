# HTTP Signatures Strategy

This document defines the strategy for implementing HTTP Signatures (RFC 9421) for ActivityPub Server-to-Server (S2S) authentication, ensuring secure federation between ActivityPub servers.

## Overview

**Goal**: Implement HTTP Signatures for secure S2S authentication in ActivityPub federation.

**Standard**: [HTTP Signatures (RFC 9421)](https://www.rfc-editor.org/rfc/rfc9421.html)

**Note**: ActivityPub historically used the older "draft-cavage" HTTP Signatures specification. Many existing Fediverse servers still use draft-cavage. RFC 9421 is the newer standard (published February 2024). For maximum compatibility, this strategy includes support for both formats, with RFC 9421 as the primary implementation and draft-cavage as a fallback.

**Purpose**: 
- **Outgoing**: Sign requests when delivering activities to remote servers' inboxes
- **Incoming**: Verify signatures on requests received in our inbox

**Status**: Future work (Part 8+ or later). Part 6 focuses on C2S authentication (Basic Auth). S2S authentication can be added after core security is in place.

**See**: [Security Integration](SECURITY_INTEGRATION.md) for C2S authentication strategy.

## Core Principles

### 1. S2S vs C2S Authentication

**C2S (Client-to-Server)**:
- **Method**: HTTP Basic Authentication
- **Use**: Users posting to their own outbox
- **Implementation**: `quarkus-security-jpa` with Basic Auth
- **Status**: Part 6

**S2S (Server-to-Server)**:
- **Method**: HTTP Signatures (RFC 9421)
- **Use**: Servers delivering activities to other servers' inboxes
- **Implementation**: Custom HTTP Signature signing/verification
- **Status**: Part 8+ (future)

### 2. Signature Algorithm

**Recommended**: RSA-SHA256 (widely supported)

**Alternative**: Ed25519 (more modern, smaller keys)

**Decision**: Start with RSA-SHA256 for maximum compatibility.

### 3. Key Management

**Pattern**: Each Actor has a public/private key pair

**Storage**:
- **Private Key**: Stored securely (encrypted at rest)
- **Public Key**: Exposed in Actor profile (`publicKey` field)

**Key Generation**: Generate on Actor creation

## HTTP Signature Format

### Signature Header

```
Signature: keyId="https://openpace.example/users/alice#main-key",
  algorithm="rsa-sha256",
  headers="(request-target) host date digest",
  signature="base64(signature)"
```

### Components

1. **keyId**: URL pointing to the public key (Actor's publicKey.id)
2. **algorithm**: Signature algorithm (`rsa-sha256` or `ed25519`)
3. **headers**: List of headers included in signature
4. **signature**: Base64-encoded signature

### Required Headers

**Minimum** (ActivityPub standard):
- `(request-target)`: Method and path
- `host`: Target host
- `date`: Request date

**Recommended** (for security):
- `digest`: SHA-256 hash of request body

## Implementation

### Dependencies

Add to `pom.xml`:

```xml
<!-- HTTP Message Signatures (RFC 9421) -->
<dependency>
    <groupId>com.authlete</groupId>
    <artifactId>http-message-signatures</artifactId>
    <version>1.0.0</version>
</dependency>

<!-- Quarkus JWT Build API for key generation and handling -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-jwt-build</artifactId>
</dependency>

<!-- Quarkus Vault for secure key storage -->
<dependency>
    <groupId>io.quarkiverse.vault</groupId>
    <artifactId>quarkus-vault</artifactId>
</dependency>
```

**Note**: 
- Authlete's `http-message-signatures` library provides full RFC 9421 support
- `quarkus-smallrye-jwt-build` provides `io.smallrye.jwt.util.KeyUtils` for key generation (RSA, EC) and key handling (PEM, JWK)
- `quarkus-vault` provides secure storage for encrypted private keys using HashiCorp Vault
- No official Quarkus extension exists yet for HTTP Signatures
- For draft-cavage compatibility, may need additional library or manual implementation

**Reference**: 
- [Quarkus JWT Build Guide](https://quarkus.io/guides/security-jwt-build)
- [Quarkus Vault Guide](https://docs.quarkiverse.io/quarkus-vault/dev/index.html)

### Database Schema

#### Actor Keys Table

```sql
CREATE TABLE actor_keys (
    id BIGSERIAL PRIMARY KEY,
    actor_id BIGINT REFERENCES actors(id) NOT NULL,
    key_id VARCHAR(500) UNIQUE NOT NULL,  -- Full URL: https://domain/users/username#main-key
    private_key_pem VARCHAR(500) NOT NULL,  -- Vault path reference (e.g., "actor-keys/{actorId}/private-key")
    public_key_pem TEXT NOT NULL,   -- Public key (safe to store in database)
    algorithm VARCHAR(50) NOT NULL DEFAULT 'rsa-sha256',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_actor_keys_actor_id ON actor_keys(actor_id);
CREATE INDEX idx_actor_keys_key_id ON actor_keys(key_id);
```

**Security**: 
- **Private keys**: Stored in HashiCorp Vault (encrypted at rest automatically)
- **Database column**: `private_key_pem` stores a Vault path reference, NOT the actual key
- **Public keys**: Safe to store in database (public by design)
- See [Key Encryption Service](#key-encryption-service) section below for implementation details
- See [Database Design](DATABASE_DESIGN.md) for schema details

### Actor Entity Update

```java
@Entity
@Table(name = "actors")
public class Actor extends PanacheEntityBase {
    // ... existing fields ...
    
    @OneToOne(mappedBy = "actor", cascade = CascadeType.ALL)
    public ActorKey key;  // Public/private key pair
}
```

### ActorKey Entity

```java
@Entity
@Table(name = "actor_keys")
public class ActorKey extends PanacheEntityBase {
    @Id
    @GeneratedValue
    public Long id;
    
    @OneToOne
    @JoinColumn(name = "actor_id")
    public Actor actor;
    
    @Column(name = "key_id", unique = true, nullable = false)
    public String keyId;  // Full URL
    
    @Column(name = "private_key_pem", nullable = false)
    public String privateKeyPem;  // Vault path reference (e.g., "actor-keys/{actorId}/private-key")
    
    @Column(name = "public_key_pem", nullable = false)
    public String publicKeyPem;
    
    @Column(nullable = false)
    public String algorithm = "rsa-sha256";
    
    public Instant createdAt;
}
```

### Key Generation Service

```java
import io.smallrye.jwt.util.KeyUtils;
import java.security.KeyPair;
import java.security.PrivateKey;
import java.security.PublicKey;

@ApplicationScoped
public class KeyGenerationService {
    
    @Inject
    KeyEncryptionService encryptionService;
    
    /**
     * Generate RSA key pair for an actor using Quarkus KeyUtils.
     */
    public ActorKey generateKeyPair(Actor actor) throws Exception {
        // Generate RSA key pair (2048 bits) using Quarkus KeyUtils
        KeyPair keyPair = KeyUtils.generateKeyPair(2048);
        
        // Convert to PEM format using KeyUtils
        String privateKeyPem = KeyUtils.convertPrivateKeyToPem(keyPair.getPrivate());
        String publicKeyPem = KeyUtils.convertPublicKeyToPem(keyPair.getPublic());
        
        // Create key ID (ActivityPub format)
        String keyId = actor.getActivityPubId(getBaseUrl()) + "#main-key";
        
        // Store private key in Vault (encrypted at rest)
        String vaultPath = "actor-keys/" + actor.id + "/private-key";
        encryptionService.storePrivateKey(vaultPath, "private-key-pem", privateKeyPem);
        
        // Create ActorKey entity
        ActorKey actorKey = new ActorKey();
        actorKey.actor = actor;
        actorKey.keyId = keyId;
        actorKey.privateKeyPem = vaultPath;  // Store Vault path, not the key itself
        actorKey.publicKeyPem = publicKeyPem;
        actorKey.algorithm = "rsa-sha256";
        actorKey.createdAt = Instant.now();
        
        return actorKey;
    }
    
    /**
     * Retrieve private key from Vault.
     */
    public String retrievePrivateKeyFromVault(String vaultPath) {
        return encryptionService.retrievePrivateKey(vaultPath, "private-key-pem");
    }
    
    /**
     * Parse private key from PEM format (for use in signing).
     */
    public PrivateKey parsePrivateKey(String privateKeyPem) throws Exception {
        return KeyUtils.decodePrivateKey(privateKeyPem);
    }
    
    /**
     * Parse public key from PEM format (for use in verification).
     */
    public PublicKey parsePublicKey(String publicKeyPem) throws Exception {
        return KeyUtils.decodePublicKey(publicKeyPem);
    }
}
```

**Note**: `KeyUtils` from `quarkus-smallrye-jwt-build` provides:
- Key generation: `generateKeyPair(int keySize)` for RSA or EC key pairs
- PEM conversion: Methods to convert keys to/from PEM format
- Key parsing: Methods to decode keys from PEM or JWK format
- Support for JWK and JWK Set formats

**Reference**: See [Quarkus JWT Build Guide - Dealing with the keys](https://quarkus.io/guides/security-jwt-build#dealing-with-the-keys) for exact API methods and usage patterns.

**Note**: The exact method names may vary. Consult the `io.smallrye.jwt.util.KeyUtils` API documentation for the current implementation. The pattern shown here demonstrates using Quarkus-native utilities instead of third-party libraries.

### Actor Profile with Public Key

**ActivityPub Format**:
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Person",
  "id": "https://openpace.example/users/alice",
  "preferredUsername": "alice",
  "publicKey": {
    "id": "https://openpace.example/users/alice#main-key",
    "owner": "https://openpace.example/users/alice",
    "publicKeyPem": "-----BEGIN PUBLIC KEY-----\n..."
  }
}
```

**Implementation**:
```java
public class Actor {
    // ... existing fields ...
    
    public PublicKeyObject getPublicKey() {
        if (key == null) {
            return null;
        }
        
        PublicKeyObject publicKey = new PublicKeyObject();
        publicKey.id = key.keyId;
        publicKey.owner = getActivityPubId(getBaseUrl());
        publicKey.publicKeyPem = key.publicKeyPem;
        return publicKey;
    }
}
```

### HTTP Signature Signing Service

```java
@ApplicationScoped
public class HttpSignatureService {
    
    @Inject
    KeyGenerationService keyService;
    
    @Inject
    KeyEncryptionService encryptionService;
    
    /**
     * Sign an HTTP request for federation delivery (RFC 9421).
     */
    public void signRequest(HttpRequest request, Actor actor) throws Exception {
        ActorKey actorKey = actor.key;
        if (actorKey == null) {
            throw new IllegalStateException("Actor has no key pair");
        }
        
        // Retrieve private key from Vault
        String privateKeyPem = keyService.retrievePrivateKeyFromVault(actorKey.privateKeyPem);
        PrivateKey privateKey = keyService.parsePrivateKey(privateKeyPem);
        
        // Using Authlete library (RFC 9421)
        SignatureBaseBuilder builder = new SignatureBaseBuilder();
        builder.setMethod(request.method());
        builder.setPath(request.uri().getPath());
        builder.setAuthority(request.uri().getHost());
        builder.setDate(request.getHeader("Date"));
        builder.setContentDigest(request.getHeader("Digest")); // If present
        
        // Build signature base
        SignatureBase signatureBase = builder.build();
        
        // Sign with RSA-SHA256
        HttpSigner signer = new HttpSigner();
        signer.setKeyId(actorKey.keyId);
        signer.setAlgorithm("rsa-sha256");
        signer.setPrivateKey(privateKey);
        
        // Generate signature
        HttpSignature signature = signer.sign(signatureBase);
        
        // Add Signature-Input and Signature headers (RFC 9421)
        request.putHeader("Signature-Input", signature.getSignatureInput());
        request.putHeader("Signature", signature.getSignature());
    }
    
    /**
     * Sign request using draft-cavage format (for compatibility).
     */
    public void signRequestLegacy(HttpRequest request, Actor actor) throws Exception {
        // Legacy draft-cavage implementation
        // Used as fallback for servers that don't support RFC 9421
        // Similar to RFC 9421 but uses single "Signature" header
    }
}
```

### Integration with Federation Delivery

Update `FederationDeliveryService` to sign outgoing requests:

```java
@ApplicationScoped
public class FederationDeliveryService {
    
    @Inject
    HttpSignatureService signatureService;
    
    /**
     * Deliver activity to recipient inbox (with HTTP signature).
     */
    private void deliver(DeliveryJob job) {
        // ... existing server reputation check ...
        
        try {
            // Get actor (sender)
            Actor actor = Actor.findById(job.actorId);
            
            // Create HTTP request
            HttpRequest request = webClient.postAbs(job.recipientInbox)
                .putHeader("Content-Type", "application/activity+json")
                .putHeader("Accept", "application/activity+json")
                .putHeader("Date", Instant.now().toString())
                .putHeader("Digest", calculateDigest(job.activityJson));
            
            // Sign request
            signatureService.signRequest(request, actor);
            
            // Send request
            request.sendJson(new JsonObject(job.activityJson))
                .await()
                .atMost(Duration.ofSeconds(30));
            
            // ... handle success ...
        } catch (Exception e) {
            // ... handle failure ...
        }
    }
    
    private String calculateDigest(String jsonBody) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(jsonBody.getBytes(StandardCharsets.UTF_8));
            String hashBase64 = Base64.getEncoder().encodeToString(hash);
            return "SHA-256=" + hashBase64;
        } catch (Exception e) {
            throw new RuntimeException("Failed to calculate digest", e);
        }
    }
}
```

### HTTP Signature Verification Service

```java
@ApplicationScoped
public class HttpSignatureVerificationService {
    
    @Inject
    RemoteActorService remoteActorService;
    
    @Inject
    KeyGenerationService keyService;
    
    /**
     * Verify HTTP signature on incoming inbox request.
     * Supports both RFC 9421 and draft-cavage formats.
     */
    public boolean verifySignature(HttpRequest request) throws Exception {
        String signatureInput = request.getHeader("Signature-Input");
        String signature = request.getHeader("Signature");
        
        if (signatureInput != null && signature != null) {
            // RFC 9421 format
            return verifyRfc9421Signature(request, signatureInput, signature);
        } else if (signature != null) {
            // Draft-cavage format (legacy)
            return verifyDraftCavageSignature(request, signature);
        }
        
        // No signature present
        return false;
    }
    
    /**
     * Verify RFC 9421 signature.
     */
    private boolean verifyRfc9421Signature(
            HttpRequest request, 
            String signatureInput, 
            String signatureValue) throws Exception {
        
        // Parse Signature-Input header
        SignatureInputParser parser = new SignatureInputParser();
        SignatureInput signatureInputObj = parser.parse(signatureInput);
        
        // Extract keyId
        String keyId = signatureInputObj.getKeyId();
        
        // Fetch public key
        PublicKey publicKey = fetchPublicKey(keyId);
        if (publicKey == null) {
            Log.warnf("Public key not found for keyId: %s", keyId);
            return false;
        }
        
        // Build signature base
        SignatureBaseBuilder builder = new SignatureBaseBuilder();
        builder.setMethod(request.method());
        builder.setPath(request.uri().getPath());
        builder.setAuthority(request.uri().getHost());
        builder.setDate(request.getHeader("Date"));
        builder.setContentDigest(request.getHeader("Digest"));
        
        SignatureBase signatureBase = builder.build();
        
        // Verify signature
        HttpVerifier verifier = new HttpVerifier();
        verifier.setPublicKey(publicKey);
        verifier.setAlgorithm(signatureInputObj.getAlgorithm());
        
        return verifier.verify(signatureBase, signatureValue);
    }
    
    /**
     * Verify draft-cavage signature (legacy format).
     */
    private boolean verifyDraftCavageSignature(
            HttpRequest request, 
            String signatureHeader) throws Exception {
        
        // Parse draft-cavage Signature header
        SignatureComponents components = parseDraftCavageSignature(signatureHeader);
        
        // Fetch public key
        PublicKey publicKey = fetchPublicKey(components.keyId);
        if (publicKey == null) {
            return false;
        }
        
        // Build signature string (draft-cavage format)
        String signatureString = buildDraftCavageSignatureString(request, components.headers);
        
        // Verify signature
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initVerify(publicKey);
        signature.update(signatureString.getBytes(StandardCharsets.UTF_8));
        
        byte[] signatureBytes = Base64.getDecoder().decode(components.signature);
        return signature.verify(signatureBytes);
    }
    
    @Inject
    KeyGenerationService keyService;
    
    /**
     * Fetch public key from remote actor profile.
     */
    private PublicKey fetchPublicKey(String keyId) throws Exception {
        // Extract actor URL from keyId
        // keyId format: https://domain/users/username#main-key
        String actorUrl = keyId.split("#")[0];
        
        // Fetch and cache actor profile
        Actor actor = remoteActorService.fetchRemoteActor(actorUrl);
        if (actor == null || actor.publicKey == null) {
            return null;
        }
        
        // Parse public key PEM using KeyGenerationService
        return keyService.parsePublicKey(actor.publicKey.publicKeyPem);
    }
    
    private static class SignatureComponents {
        String keyId;
        String algorithm;
        List<String> headers;
        String signature;
    }
}
```

### Inbox Verification

Update `InboxResource` to verify signatures:

```java
@Path("/users/{username}/inbox")
public class InboxResource {
    
    @Inject
    HttpSignatureVerificationService signatureVerification;
    
    @POST
    @Consumes("application/activity+json")
    public Response receiveActivity(
            @PathParam("username") String username,
            @HeaderParam("Signature") String signatureHeader,
            JsonNode activityJson) {
        
        // Find target actor
        Actor targetActor = Actor.findByUsername(username);
        if (targetActor == null) {
            return Response.status(404).build();
        }
        
        // Verify signature (if enabled)
        if (signatureVerificationEnabled) {
            try {
                HttpRequest request = getCurrentRequest(); // Get request context
                boolean valid = signatureVerification.verifySignature(request, signatureHeader);
                
                if (!valid) {
                    Log.warnf("Invalid HTTP signature for inbox POST to %s", username);
                    return Response.status(401)
                        .entity(new ErrorResponse(
                            "INVALID_SIGNATURE",
                            "HTTP signature verification failed"
                        ))
                        .build();
                }
            } catch (Exception e) {
                Log.errorf(e, "Error verifying HTTP signature");
                return Response.status(401)
                    .entity(new ErrorResponse(
                        "SIGNATURE_ERROR",
                        "Error verifying HTTP signature"
                    ))
                    .build();
            }
        }
        
        // Process activity (existing logic)
        // ...
        
        // Always return 202 Accepted (ActivityPub spec)
        return Response.status(Response.Status.ACCEPTED).build();
    }
}
```

## Key Management

### Key Generation on Actor Creation

```java
@ApplicationScoped
public class ActorService {
    
    @Inject
    KeyGenerationService keyService;
    
    @Transactional
    public Actor createActor(User user, String username) {
        // Create actor
        Actor actor = new Actor();
        actor.username = username;
        actor.user = user;
        actor.persist();
        
        // Generate key pair
        ActorKey key = keyService.generateKeyPair(actor);
        key.persist();
        actor.key = key;
        
        return actor;
    }
}
```

### Key Rotation (Future)

**Pattern**: Support multiple keys per actor

```sql
-- Allow multiple keys (for rotation)
ALTER TABLE actor_keys DROP CONSTRAINT IF EXISTS unique_key_per_actor;
-- Add key_type: 'active', 'rotating', 'deprecated'
ALTER TABLE actor_keys ADD COLUMN key_type VARCHAR(50) DEFAULT 'active';
```

## Configuration

`application.properties`:

```properties
# HTTP Signatures
http.signatures.enabled=true
http.signatures.algorithm=rsa-sha256
http.signatures.key-size=2048
http.signatures.require-signature=true  # Reject unsigned inbox POSTs

# Vault Configuration (for private key storage)
quarkus.vault.url=http://localhost:8200
quarkus.vault.authentication.userpass.username=vault-user
quarkus.vault.authentication.userpass.password=vault-password
quarkus.vault.kv-secret-engine-mount-path=secret
quarkus.vault.kv-secret-engine-version=2

# Key Management
http.signatures.vault-path-prefix=actor-keys  # Vault path prefix for actor keys
http.signatures.key-rotation-enabled=false
```

## Testing

### Unit Tests

```java
@Test
public void testSignatureGeneration() {
    Actor actor = createTestActor();
    ActorKey key = keyService.generateKeyPair(actor);
    
    assertNotNull(key.privateKeyPem);
    assertNotNull(key.publicKeyPem);
    assertTrue(key.keyId.endsWith("#main-key"));
}

@Test
public void testSignatureVerification() {
    // Create request
    // Sign request
    // Verify signature
    assertTrue(verificationService.verifySignature(request, signatureHeader));
}
```

### Integration Tests

```java
@Test
public void testFederationDeliveryWithSignature() {
    // Create activity
    // Queue for delivery
    // Verify signature is included in request
    // Verify remote server accepts signature
}
```

### Manual Testing

```bash
# Test signature generation
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secure123"}'

# Check actor profile includes publicKey
curl http://localhost:8080/users/alice

# Test inbox with signature (from another server)
# Use ActivityPub client that signs requests
```

## Security Considerations

### Key Encryption Service

**Implementation**: Use HashiCorp Vault via Quarkus Vault extension to store private keys securely.

```java
import io.quarkus.vault.VaultKVSecretEngine;
import java.util.Map;

@ApplicationScoped
public class KeyEncryptionService {
    
    @Inject
    VaultKVSecretEngine vaultKVSecretEngine;
    
    /**
     * Store private key in Vault.
     * @param vaultPath Path in Vault (e.g., "actor-keys/{actorId}/private-key")
     * @param keyName Key name in the secret (e.g., "private-key-pem")
     * @param privateKeyPem The private key in PEM format
     */
    public void storePrivateKey(String vaultPath, String keyName, String privateKeyPem) {
        Map<String, String> secret = Map.of(keyName, privateKeyPem);
        vaultKVSecretEngine.writeSecret(vaultPath, secret);
    }
    
    /**
     * Retrieve private key from Vault.
     * @param vaultPath Path in Vault
     * @param keyName Key name in the secret
     * @return Private key in PEM format
     */
    public String retrievePrivateKey(String vaultPath, String keyName) {
        Map<String, String> secret = vaultKVSecretEngine.readSecret(vaultPath);
        return secret.get(keyName);
    }
    
    /**
     * Delete private key from Vault (for key rotation or actor deletion).
     */
    public void deletePrivateKey(String vaultPath) {
        vaultKVSecretEngine.deleteSecret(vaultPath);
    }
}
```

**Configuration** (`application.properties`):
```properties
# Vault connection
quarkus.vault.url=http://localhost:8200

# Vault authentication (use appropriate method for your environment)
# For development: userpass
quarkus.vault.authentication.userpass.username=vault-user
quarkus.vault.authentication.userpass.password=vault-password

# For production: Kubernetes authentication (recommended)
# quarkus.vault.authentication.kubernetes.role=open-pace-role
# quarkus.vault.authentication.kubernetes.jwt-token-path=/var/run/secrets/kubernetes.io/serviceaccount/token

# KV secret engine mount path (default: secret)
quarkus.vault.kv-secret-engine-mount-path=secret
quarkus.vault.kv-secret-engine-version=2
```

**Vault Path Structure**:
- `secret/actor-keys/{actorId}/private-key` - Stores private key for actor
- Example: `secret/actor-keys/123/private-key` contains `{"private-key-pem": "-----BEGIN PRIVATE KEY-----\n..."}`

**Benefits**:
- ✅ Private keys never stored in database
- ✅ Vault handles encryption at rest automatically
- ✅ Access control via Vault policies
- ✅ Audit logging via Vault
- ✅ Key rotation without database changes
- ✅ Integration with Quarkus Dev Services for development

**Reference**: [Quarkus Vault Guide](https://docs.quarkiverse.io/quarkus-vault/dev/index.html)

### Private Key Protection

**Critical**: Private keys must be encrypted at rest and never stored in the database.

**Strategy**: Use HashiCorp Vault via Quarkus Vault extension to store private keys securely. See [Key Encryption Service](#key-encryption-service) section above for implementation details.

### Key Exposure

**Never**:
- Log private keys
- Include private keys in responses
- Store private keys in version control
- Transmit private keys over unencrypted connections

**Always**:
- Store private keys in Vault (encrypted at rest automatically)
- Use HTTPS for all federation communication
- Rotate keys periodically (delete old key from Vault, generate new)
- Revoke compromised keys immediately (delete from Vault)
- Use Vault policies to restrict access to actor keys

### Signature Replay Attacks

**Protection**: Include `date` header and verify it's recent (within 5 minutes).

```java
private boolean verifyDateHeader(String dateHeader) {
    try {
        Instant requestDate = Instant.parse(dateHeader);
        Instant now = Instant.now();
        Duration difference = Duration.between(requestDate, now);
        
        // Reject if older than 5 minutes
        return difference.abs().toMinutes() <= 5;
    } catch (Exception e) {
        return false;
    }
}
```

## Migration Strategy

### Step 1: Add Key Generation

1. Create `ActorKey` entity
2. Add migration for `actor_keys` table
3. Generate keys for existing actors
4. Update Actor profile to include `publicKey`

### Step 2: Implement Signing

1. Create `HttpSignatureService`
2. Update `FederationDeliveryService` to sign requests
3. Test outgoing signatures

### Step 3: Implement Verification

1. Create `HttpSignatureVerificationService`
2. Update `InboxResource` to verify signatures
3. Test incoming signature verification

### Step 4: Enable Verification

1. Set `http.signatures.require-signature=true`
2. Reject unsigned requests
3. Monitor for issues

## Compatibility

### RFC 9421 vs Draft-Cavage

**Challenge**: Many existing ActivityPub servers use the older draft-cavage HTTP Signatures specification, not RFC 9421.

**Strategy**: 
- **Primary**: Implement RFC 9421 (modern standard)
- **Fallback**: Support draft-cavage for compatibility with older servers
- **Detection**: Check for `Signature-Input` header (RFC 9421) vs `Signature` header (draft-cavage)

**Implementation**:
```java
public boolean verifySignature(HttpRequest request) {
    String signatureInput = request.getHeader("Signature-Input");
    
    if (signatureInput != null) {
        // RFC 9421 format
        return verifyRfc9421Signature(request, signatureInput);
    } else {
        // Draft-cavage format (legacy)
        String signature = request.getHeader("Signature");
        if (signature != null) {
            return verifyDraftCavageSignature(request, signature);
        }
    }
    
    return false;
}
```

### Servers Without Signatures

**Current**: Accept unsigned requests (for tutorial simplicity)

**Future**: Optionally require signatures:
- `http.signatures.require-signature=false`: Accept unsigned (current)
- `http.signatures.require-signature=true`: Reject unsigned (production)

### Algorithm Support

**Priority**:
1. RSA-SHA256 (widely supported, works with draft-cavage)
2. Ed25519 (modern, efficient, RFC 9421)

**Fallback**: If server doesn't support algorithm, log warning and continue (for compatibility).

**Note**: Draft-cavage typically only supports RSA-PKCS#1-v1.5 or RSA-PSS. RFC 9421 supports Ed25519, ECDSA, and others.

## Integration with Security Integration

### Relationship to Part 6

**Part 6 (Security Integration)**:
- ✅ C2S authentication (HTTP Basic Auth)
- ✅ User registration and login
- ✅ Password hashing (BCrypt)
- ✅ Actor-User relationship

**Part 8+ (HTTP Signatures)**:
- ✅ S2S authentication (HTTP Signatures)
- ✅ Key generation and management
- ✅ Signature signing for outgoing requests
- ✅ Signature verification for incoming requests

### Actor Profile Structure

**With Public Key**:
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Person",
  "id": "https://openpace.example/users/alice",
  "preferredUsername": "alice",
  "publicKey": {
    "id": "https://openpace.example/users/alice#main-key",
    "owner": "https://openpace.example/users/alice",
    "publicKeyPem": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A...\n-----END PUBLIC KEY-----"
  }
}
```

## Summary

**Technology**: HTTP Signatures (RFC 9421)

**Purpose**:
- **Outgoing**: Sign federation delivery requests
- **Incoming**: Verify inbox POST requests

**Algorithm**: RSA-SHA256 (recommended), Ed25519 (alternative)

**Key Management**:
- Generate key pair per Actor
- Store public key in Actor profile
- Encrypt private key at rest
- Expose public key via ActivityPub

**Integration**:
- Federation Delivery: Sign outgoing requests
- Inbox: Verify incoming requests
- Actor Profile: Include publicKey field

**Status**: Future work (Part 8+), after Part 6 security integration

**See Also**:
- [Security Integration](SECURITY_INTEGRATION.md) - C2S authentication (Part 6)
- [Federation Delivery Strategy](FEDERATION_DELIVERY_STRATEGY.md) - Outgoing delivery
- [HTTP Signatures RFC 9421](https://www.rfc-editor.org/rfc/rfc9421.html)
- [ActivityPub Security](https://www.w3.org/TR/activitypub/#security-considerations)
- [Quarkus JWT Build Guide](https://quarkus.io/guides/security-jwt-build) - Key generation and handling
- [Authlete HTTP Message Signatures Library](https://github.com/authlete/http-message-signatures)

## References

- [RFC 9421: HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html)
- [ActivityPub Security Considerations](https://www.w3.org/TR/activitypub/#security-considerations)
- [Authlete HTTP Message Signatures (Java)](https://github.com/authlete/http-message-signatures)
