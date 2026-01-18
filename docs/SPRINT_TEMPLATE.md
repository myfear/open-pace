# Sprint X: [Title]

## Implementation Goals
By the end of this sprint, you will have implemented:
- [ ] Feature 1
- [ ] Feature 2
- [ ] Feature 3

## Prerequisites
- ✅ Completed [Sprint X-1: Previous Sprint Name]
- ✅ Understanding of [key concepts from previous sprints]
- ✅ [Any new concepts needed]

## What You'll Implement
[2-3 sentence description of what working feature will be delivered at the end of this sprint]

**Example**: "A basic ActivityPub server that can receive follows from Mastodon and display your activities in their feed."

## Architecture Overview
[Brief diagram or explanation of what we're adding in this sprint]

**Note**: Endpoints are organized as:
- **ActivityPub endpoints** (root level): For federation compatibility
- **Application endpoints** (`/api/*`): For UI/internal use

See [API Design](API_DESIGN.md) for endpoint organization strategy.

## Implementation Steps

### Step 1: [Clear Action Title]

**Goal**: [What this step accomplishes]

**Implementation**:

```java
// Code example with comments
```

**Explanation**: [Why we're doing this, how it fits into ActivityPub]

**Test**: [How to verify this step works]

---

### Step 2: [Next Step]

[Repeat structure]

---

## Testing Your Implementation

### Manual Testing Steps

1. **Start the server**:
   ```bash
   ./mvnw quarkus:dev
   ```

2. **Test [Feature 1]**:
   ```bash
   curl -X GET http://localhost:8080/.well-known/webfinger?resource=acct:alice@localhost:8080
   ```

3. **Test with Mastodon**:
   - [Specific steps to test federation]

### Automated Tests

Run the test suite:
```bash
./mvnw test
```

**Expected Results**: [What should pass]

### Integration Testing

[How to test with real ActivityPub servers]

## Key Concepts Explained

### [Concept Name]
[Deep dive into important ActivityPub/federation concepts introduced in this sprint]

### [Another Concept]
[More explanation]

## Error Handling

All endpoints use consistent error handling:
- **Validation**: Hibernate Validator annotations on request DTOs
- **Error Format**: `{"error": "<ERROR_CODE>", "message": "<MESSAGE>"}`
- **No Exceptions**: All exceptions are caught and converted to ErrorResponse
- **Status Codes**: Appropriate HTTP status codes per error type

**See**: [Error Handling Strategy](ERROR_HANDLING_STRATEGY.md) for details.

## Common Issues & Solutions

### Issue: [Problem Description]
**Symptoms**: [How you know you have this issue]
**Solution**: [How to fix it]
**Prevention**: [How to avoid it]

## Code Review: What We Built

[Summary of the code structure, key files, and how they work together]

## Implementation Decisions

**Important**: Document key design decisions and lessons learned in the relevant strategy documents (Database Design, API Design, Error Handling, etc.).

For this sprint, document:
- Key architectural decisions and rationale
- Patterns established for future sprints
- Lessons learned (what worked, what didn't)
- Code patterns that should be reused
- Trade-offs considered

**See**: Relevant strategy documents (Database Design, API Design, Error Handling, etc.) for patterns from previous sprints.

## Performance Notes

[Where Vert.x async processing helps, performance considerations]

## Next Steps

- ✅ You've completed Sprint X!
- 🎯 **Next**: [Sprint X+1: Next Title]
- 📚 **Related**: [Links to ActivityPub spec, relevant docs]

## Additional Resources

- [ActivityPub Specification](https://www.w3.org/TR/activitypub/)
- [JSON-LD Context](https://www.w3.org/ns/activitystreams)
- [Relevant documentation]
