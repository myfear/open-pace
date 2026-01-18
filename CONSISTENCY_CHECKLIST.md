# Implementation Consistency Checklist

Use this checklist for each implementation sprint to ensure consistency across the project.

## Pre-Writing Checklist

- [ ] **Clear learning objectives** defined (3-5 specific, measurable goals)
- [ ] **Prerequisites** clearly listed (what from previous sprints is needed)
- [ ] **Deliverable** clearly described (what working system results)
- [ ] **Scope boundaries** defined (what's in vs. out of scope)

## Code Quality Checklist

- [ ] **Code compiles** without errors or warnings
- [ ] **Follows project conventions** (naming, structure, style)
- [ ] **Comments explain ActivityPub concepts** (not just what, but why)
- [ ] **Error handling** for common federation scenarios
- [ ] **Validation** using Hibernate Validator annotations
- [ ] **Error responses** follow consistent format: `{"error": "...", "message": "..."}`
- [ ] **No exceptions exposed** to users (all caught and converted to ErrorResponse)
- [ ] **No hardcoded values** (use configuration)
- [ ] **Vert.x async patterns** used where appropriate
- [ ] **Quarkus best practices** followed

**See**: [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md) for requirements.

## Testing Checklist

- [ ] **Unit tests** for new business logic
- [ ] **Integration tests** for ActivityPub endpoints
- [ ] **Manual test steps** documented and verified
- [ ] **Mastodon compatibility** tested (can follow/post/interact)
- [ ] **Test data** provided (seed scripts or examples)
- [ ] **All tests pass** before marking complete

## Documentation Checklist

- [ ] **Step-by-step instructions** are clear and complete
- [ ] **Code examples** are complete and runnable
- [ ] **Explanations** cover both "what" and "why"
- [ ] **ActivityPub concepts** explained for beginners
- [ ] **Visual examples** included (JSON responses, screenshots, diagrams)
- [ ] **Troubleshooting section** addresses common issues
- [ ] **Links work** (no broken references)
- [ ] **Consistent terminology** (e.g., always "actor" not "user" in ActivityPub context)

## Federation Testing Checklist

- [ ] **WebFinger** works (can discover users)
- [ ] **Actor profile** is valid ActivityPub JSON
- [ ] **Inbox** accepts POST requests (S2S - server-to-server)
- [ ] **Outbox GET** serves activities correctly
- [ ] **Outbox POST** accepts Activity objects (C2S - client-to-server)
- [ ] **Activity type processing** works (server processes based on JSON `type` field)
- [ ] **Follow requests** work end-to-end
- [ ] **Activities appear** in follower feeds
- [ ] **Interoperability** tested with Mastodon
- [ ] **JSON-LD context** is correct

## Reader Experience Checklist

- [ ] **Clear entry point** (where to start)
- [ ] **Progress indicators** (what's been built so far)
- [ ] **Completion criteria** (how to know you're done)
- [ ] **Next steps** clearly linked
- [ ] **Prerequisites** can be verified (checklist format)
- [ ] **Examples work** out-of-the-box
- [ ] **No assumed knowledge** beyond prerequisites

## Technical Accuracy Checklist

- [ ] **ActivityPub spec compliance** (follows W3C spec)
- [ ] **C2S pattern implemented correctly** (POST to outbox, process by activity type)
- [ ] **Activity type determined from JSON** (not from URL path)
- [ ] **Endpoint organization** follows API design (ActivityPub at root, app endpoints under `/api/*`)
- [ ] **JSON-LD** properly formatted
- [ ] **HTTP status codes** correct
- [ ] **Content-Type headers** correct (`application/activity+json` for ActivityPub, `application/json` for app)
- [ ] **Error responses** follow format: `{"error": "...", "message": "..."}`
- [ ] **No exceptions exposed** to users
- [ ] **Validation** using Hibernate Validator
- [ ] **Security considerations** mentioned (authentication, CORS, etc.)
- [ ] **Performance implications** noted where relevant

**See**: 
- [API Design](docs/API_DESIGN.md) for endpoint organization
- [ActivityPub C2S Pattern](docs/ACTIVITYPUB_C2S_PATTERN.md) for correct implementation
- [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md) for error handling requirements

## Cross-Part Consistency

- [ ] **Naming conventions** match previous parts
- [ ] **Code structure** follows established patterns
- [ ] **Documentation style** matches previous parts
- [ ] **Example data** is consistent (same users, activities)
- [ ] **References to previous parts** are accurate
- [ ] **No contradictions** with earlier parts

## Project Structure

- [ ] **Project folder created** (`open-pace-pX/`)
- [ ] **Project is self-contained** (can run independently)
- [ ] **README.md exists** in project folder with part-specific info
- [ ] **Main README updated** with part status
- [ ] **Implementation decisions documented** in relevant strategy documents
- [ ] **Commit messages** are clear

## Final Review

- [ ] **Read through as a beginner** (would you understand?)
- [ ] **Follow instructions step-by-step** (do they work?)
- [ ] **Test all examples** (do they run?)
- [ ] **Check all links** (do they work?)
- [ ] **Verify federation** (does it work with Mastodon?)

## Part-Specific Additions

### Sprint 1: Basic ActivityPub
- [ ] WebFinger endpoint works
- [ ] Actor endpoint returns valid JSON
- [ ] Can follow from Mastodon
- [ ] Simple text posts appear in Mastodon feed

### Sprint 2: Custom Activities
- [ ] Custom object types defined
- [ ] JSON-LD context includes custom types
- [ ] Activities stored with full data
- [ ] FediRun clients can parse custom types

### Sprint 3: Rich Interop
- [ ] Comments work from any server
- [ ] Likes work from any server
- [ ] Attachments supported
- [ ] Activity viewing endpoints work

### Sprint 4: Sports Features
- [ ] Segments tracked
- [ ] Leaderboards work
- [ ] Groups function
- [ ] Cross-instance features work

### Sprint 5: Privacy & Data
- [ ] Private activities work
- [ ] Direct messages work
- [ ] Data export works
- [ ] Federation policies enforced

### Sprint 6: Security Integration
- [ ] User registration works
- [ ] Authentication works (Basic Auth)
- [ ] Password hashing works (BCrypt)
- [ ] Outbox POST requires authentication
- [ ] Actor ownership verified
- [ ] Verified field present

### Sprint 7: Mapping Integration

### Sprint 8: Activity Analytics & Personal Records

### Sprint 9: Social Interactions & Feed

### Sprint 10: Gear & Equipment Tracking
- [ ] GPX parsing works
- [ ] Tracks stored in PostGIS (LineString)
- [ ] Map images generated with OSM tiles
- [ ] OSM attribution present
- [ ] Maps displayed in activity attachments
