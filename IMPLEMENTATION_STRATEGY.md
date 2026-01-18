# Implementation Strategy: Open Pace

**Purpose**: This document defines the **overall strategy, principles, structure, and workflow** for the implementation sprints. It explains the "what", "why", and "how" of the implementation approach.

## Overview
This document outlines the strategy for building a consistent, high-quality federated ActivityPub application through 10 implementation sprints. Each sprint delivers working functionality that builds incrementally toward a complete "Open Pace" application - a federated Strava alternative using Quarkus + Vert.x.

## Core Principles

### 1. **Incremental Implementation**
Each sprint builds on the previous, adding one major feature at a time:
- Sprint 1: Basic federation (foundation)
- Sprint 2: Custom types (extension)
- Sprint 3: Rich features (enhancement - comments, likes, attachments)
- Sprint 4: Sports features (specialization)
- Sprint 5: Privacy & polish (production-ready)
- Sprint 6: Security integration (authentication & authorization)
- Sprint 7: Mapping integration (geospatial data, OSM tiles)
- Sprint 8: Activity analytics & personal records (data analysis)
- Sprint 9: Social interactions & feed (social features)
- Sprint 10: Gear & equipment tracking (equipment management)

### 2. **Working Deliverables**
Each sprint should result in a **working, testable feature** that can be demonstrated independently.

### 3. **Clear Scope**
Each sprint has:
- **Goal**: What feature will be implemented
- **Prerequisites**: What functionality from previous sprints is needed
- **Deliverables**: What working feature you'll have at the end
- **Testing**: How to verify it works

## Project Structure

Since this is part of a mono-repo, each implementation sprint is a **standalone, runnable project** in its own folder:

```
mono-repo-root/
├── open-pace/                    # This folder (strategy & shared docs)
│   ├── README.md                 # Main project overview
│   ├── IMPLEMENTATION_STRATEGY.md      # This file
│   └── docs/                     # Shared documentation
│       ├── part-1-basic-activitypub.md
│       ├── part-2-custom-activities.md
│       ├── part-3-rich-interop.md
│       ├── part-4-sports-features.md
│       ├── part-5-privacy-data.md
│       ├── ACTIVITYPUB_REFERENCE.md
│       └── PROJECT_SETUP.md
│
├── open-pace-p1/                 # Sprint 1: Standalone project
│   ├── pom.xml                   # Quarkus project
│   ├── README.md                 # Sprint 1 specific README
│   ├── src/
│   │   └── main/
│   │       └── java/org/openpace/
│   │           └── core/         # Core ActivityPub logic
│   ├── src/test/                 # Sprint 1 tests
│   └── (no docker-compose needed - Quarkus Dev Services handles it)
│
├── open-pace-p2/                 # Sprint 2: Standalone project
│   ├── pom.xml                   # Quarkus project (builds on Sprint 1)
│   ├── README.md                 # Sprint 2 specific README
│   ├── src/
│   │   └── main/
│   │       └── java/org/openpace/
│   │           ├── core/         # Core ActivityPub (refined)
│   │           └── activities/   # Custom activity types
│   └── ...
│
├── open-pace-p3/                 # Sprint 3: Standalone project
├── open-pace-p4/                 # Sprint 4: Standalone project
├── open-pace-p5/                 # Sprint 5: Standalone project
├── open-pace-p6/                 # Sprint 6: Standalone project
├── open-pace-p7/                 # Sprint 7: Standalone project
├── open-pace-p8/                 # Sprint 8: Standalone project
├── open-pace-p9/                 # Sprint 9: Standalone project
└── open-pace-p10/                # Sprint 10: Standalone project
```

**Key Benefits of This Structure:**
- ✅ Each sprint is **completely runnable** independently
- ✅ Developers can start at any sprint (with prerequisites)
- ✅ No need to manage branches or tags for checkpoints
- ✅ Clear separation of concerns
- ✅ Easy to test each sprint in isolation
- ✅ Works well in mono-repo structure

## Git Workflow Strategy

### Mono-Repo Approach: Separate Project Folders

Since each sprint is a **standalone project folder**, the git workflow is simplified:

#### Structure
- Each sprint lives in its own folder: `open-pace-p1/`, `open-pace-p2/`, etc.
- All sprints are in the same mono-repo
- Shared documentation stays in `open-pace/docs/`

#### Development Workflow
1. **Create new sprint folder**: `open-pace-pX/`
2. **Initialize as Quarkus project**: Set up `pom.xml`, structure
3. **Implement sprint features**: Each sprint is self-contained
4. **Reference previous sprints**: Copy/adapt code from `open-pace-p(X-1)/` as needed
5. **Commit to mono-repo**: All sprints committed together

#### Advantages
- ✅ **No branch management**: Each sprint is a folder, not a branch
- ✅ **Easy navigation**: Clear folder structure
- ✅ **Independent testing**: Each sprint can be tested separately
- ✅ **Clear progression**: Readers can see evolution across folders
- ✅ **Mono-repo friendly**: Fits naturally into existing structure

#### Versioning (Optional)
If you want to tag releases:
- Tag the mono-repo: `open-pace-v1.0-sprint1`, `open-pace-v1.0-sprint2`
- Each tag represents a complete state of all sprints up to that point

#### Migration from Previous Sprints
When building Sprint 2+:
- **Option A**: Copy relevant code from `open-pace-p1/` to `open-pace-p2/`
- **Option B**: Reference Sprint 1 in documentation, build Sprint 2 from scratch
- **Recommended**: Copy and refine (shows progression, but keeps sprints independent)

## Documentation Template

Each implementation sprint should follow this structure:

```markdown
# Part X: [Title]

## Learning Objectives
- [ ] Objective 1
- [ ] Objective 2

## Prerequisites
- Completed Sprint X-1
- Understanding of [concepts]

## What You'll Implement
[Clear description of the working feature]

## Implementation Steps

### Step 1: [Title]
[Detailed instructions]

### Step 2: [Title]
[Detailed instructions]

## Testing Your Implementation

### Manual Testing
1. [Test step 1]
2. [Test step 2]

### Automated Tests
[How to run tests]

## Key Concepts Explained
[Deep dive into important concepts]

## Common Issues & Solutions
[FAQ for this part]

## Next Steps
[Link to Sprint X+1]
```

## Code Organization Strategy

### 1. **Package Structure**
Each sprint is in its own project folder, but packages can evolve:

```
Sprint 1 (open-pace-p1/): org.openpace.core.*
Sprint 2 (open-pace-p2/): org.openpace.core.* + org.openpace.activities.*
Sprint 3 (open-pace-p3/): + org.openpace.rendering.*
Sprint 4 (open-pace-p4/): + org.openpace.sports.*
Sprint 5 (open-pace-p5/): + org.openpace.privacy.*
Sprint 6 (open-pace-p6/): + org.openpace.security.*
Sprint 7 (open-pace-p7/): + org.openpace.mapping.*
Sprint 8 (open-pace-p8/): + org.openpace.analytics.*
Sprint 9 (open-pace-p9/): + org.openpace.social.*
Sprint 10 (open-pace-p10/): + org.openpace.gear.*
```

**Note**: Each sprint is independent. You can copy/adapt code from previous sprints, but each sprint should be runnable on its own.

### 1a. **API Endpoint Organization**

**ActivityPub Endpoints** (Root level - required by spec):
- `/.well-known/webfinger`
- `/users/{username}`
- `/users/{username}/inbox`
- `/users/{username}/outbox`
- `/users/{username}/followers`
- `/users/{username}/following`
- `/activities/{activityId}`

**Application Endpoints** (`/api/*` prefix - for UI/internal):
- `/api/auth/*` - Authentication
- `/api/activities/*` - Activities (UI format)
- `/api/users/*` - User management
- `/api/feed/*` - Feeds
- `/api/settings/*` - Settings

**See**: [API Design](docs/API_DESIGN.md) for detailed endpoint organization strategy.

### 2. **Configuration Management**
- Each sprint has its own `application.properties`
- **Quarkus Dev Services** automatically configures database connections in dev/test mode
- For production, use environment variables or explicit configuration
- Keep configuration simple and documented

### 3. **Database Migrations**
- Use Flyway or Liquibase
- Each sprint has its own migration set
- Sprint 1: `V1__initial.sql`, `V2__actors.sql`, etc.
- Sprint 2: Can start fresh or build on Sprint 1's schema (document the choice)
- **Quarkus Dev Services** automatically runs migrations when starting the dev database

## Testing Strategy

### Per-Part Testing
Each sprint should have:
1. **Unit tests** for new logic
2. **Integration tests** for federation (using test ActivityPub servers)
3. **Manual test scripts** (curl commands, Postman collections)

### Test Data
- Provide seed data for each part
- Include example activities, users, followers

## Consistency Checklist

For each implementation sprint, ensure:

- [ ] **Clear entry point**: Where does the code start?
- [ ] **Working example**: Can it be run and tested immediately?
- [ ] **Federation test**: Can it interact with Mastodon/test server?
- [ ] **Code comments**: Explain ActivityPub-specific concepts
- [ ] **Error handling**: Consistent error responses, no exceptions exposed
- [ ] **Validation**: Input validation using Hibernate Validator
- [ ] **Performance notes**: Where Vert.x async helps
- [ ] **Visual examples**: Screenshots or example JSON responses

**See**: [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md) for error handling requirements.

## Part-Specific Strategies

### Part 1: Basic ActivityPub Server
**Focus**: Get federation working end-to-end
- Minimal code, maximum clarity
- Test with Mastodon following
- Show WebFinger, Actor, Inbox/Outbox
- **Important**: Implement C2S pattern (POST to outbox, process by activity type)

**Deliverable**: Can post text, Mastodon users can follow and see posts

**Note**: See [ActivityPub C2S Pattern](docs/ACTIVITYPUB_C2S_PATTERN.md) for correct implementation approach.

### Part 2: Custom Activity Types
**Focus**: Extend ActivityPub with sports data
- Schema definition
- JSON-LD context
- Object storage

**Deliverable**: Can post runs with metrics, FediRun clients can parse them

### Part 3: Rich Data & Interop
**Focus**: Make it beautiful and interoperable
- **Comment/Like Handling**: Accept from any ActivityPub server
- **Activity Rendering**: Full activity viewer for FediRun clients
- **Attachments**: Support for images and previews
- **Activity Viewing**: Endpoints to view activities with comments and likes

**Deliverable**: Mastodon users can comment and like, FediRun users see full activity details

### Part 4: Sports-Specific Features
**Focus**: Strava-like features
- Leaderboards
- Segments
- Groups

**Deliverable**: Competitive features work across instances

### Part 5: Privacy & Data
**Focus**: Production-ready privacy
- Access control
- Data export
- Federation policies

**Deliverable**: Users control their data, can export, can set privacy

### Part 6: Security Integration
**Focus**: Authentication and authorization
- User registration and authentication
- Password hashing (BCrypt)
- Secure C2S endpoints (actor ownership)
- Verified field for email verification
- Actor-User relationship

**Deliverable**: Secure application with user accounts and proper authorization

**Note**: See [Security Integration](docs/SECURITY_INTEGRATION.md) for detailed strategy.

### Part 7: Mapping Integration
**Focus**: Geospatial data and map visualization
- **GPX Processing**: Parse GPX files, extract tracks, simplify paths
- **PostGIS Storage**: Store tracks as LineString, start/end points
- **Map Image Generation**: Generate static maps with tracks overlaid on OSM tiles
- **OSM Integration**: Use OpenStreetMap tiles with proper attribution
- **ActivityPub Integration**: Display maps in activity attachments

**Deliverable**: Activities with GPX data display beautiful map images with tracks

**Note**: See [Mapping Integration](docs/MAPPING_INTEGRATION.md) for detailed strategy.

## Tools & Dependencies

For detailed versions and technology choices, see **[Quarkus Tech Stack](docs/QUARKUS_TECH_STACK.md)**.

### Core Stack
- **Quarkus**: Java framework
- **Vert.x**: Async processing
- **Jackson**: JSON/ActivityPub serialization
- **PostgreSQL**: Data storage
- **Flyway**: Migrations
- **Hibernate Validator**: Input validation and error handling

### Testing
- **REST Assured**: API testing
- **Quarkus Dev Services**: Automatic test database (uses Testcontainers under the hood)
- **Mock ActivityPub server**: For federation testing

### Development
- **Quarkus Dev Mode**: Hot reload
- **Quarkus Dev Services**: Automatic PostgreSQL container in dev mode
- **OpenAPI/Swagger**: API documentation for application endpoints (see [API Design](docs/API_DESIGN.md#openapi-documentation))

### Error Handling
- **Hibernate Validator**: Input validation
- **Exception Mappers**: Consistent error responses
- **Error Response Format**: `{"error": "...", "message": "..."}`

**See**: [Error Handling Strategy](docs/ERROR_HANDLING_STRATEGY.md) for detailed approach.

## Content Creation Workflow

### Phase 1: Planning (Before Coding)
1. **Read the strategy** (this document)
2. **Create detailed outline** using `docs/SPRINT_OUTLINE_TEMPLATE.md`
3. **Review ActivityPub reference** (`docs/ACTIVITYPUB_REFERENCE.md`) to ensure correct terminology
4. **Plan code structure** (packages, classes, endpoints)

### Phase 2: Implementation
1. **Create project folder**: `open-pace-pX/` (e.g., `open-pace-p1/`)
2. **Set up Quarkus project** in that folder
3. **Implement the part** following your outline
4. **Write tests** as you go
5. **Test with Mastodon** to verify federation works

**Starting Part 1 - Quick Setup**:
```bash
# Create the project folder
mkdir open-pace-p1
cd open-pace-p1

# Create Quarkus project (or initialize manually)
# Ensure it has:
# - Quarkus REST
# - Quarkus Vert.x
# - PostgreSQL driver
# - Flyway (for migrations)
```

### Phase 3: Documentation
1. **Write implementation guide** using `docs/SPRINT_TEMPLATE.md`
2. **Add code examples** (complete, runnable)
3. **Create test scenarios** (manual and automated)
4. **Add troubleshooting** section

### Phase 4: Review
1. **Run consistency checklist** (`CONSISTENCY_CHECKLIST.md`)
2. **Test as a beginner** (follow your own instructions)
3. **Verify federation** (test with Mastodon)
4. **Get feedback** (if possible)

### Phase 5: Document Decisions
1. **Document patterns** established in this part
2. **Note lessons learned** and trade-offs
3. **Update relevant strategy documents** with new patterns:
   - Database patterns → `docs/DATABASE_DESIGN.md`
   - API patterns → `docs/API_DESIGN.md`
   - Error handling → `docs/ERROR_HANDLING_STRATEGY.md`
   - Federation patterns → `docs/FEDERATION_DELIVERY_STRATEGY.md`
   - ActivityPub patterns → `docs/ACTIVITYPUB_REFERENCE.md`

**Important**: Document key implementation decisions and lessons learned in the relevant strategy documents. This helps maintain consistency and guides future parts.

### Phase 6: Finalize
1. **Ensure project is complete** and runnable in its folder
2. **Update main README** with status
3. **Create README.md** in the part's folder with part-specific info
4. **Prepare for next part** (create `open-pace-p(X+1)/` folder)

### Key Success Factors

1. **Test Early, Test Often**
   - Test with Mastodon after each major feature
   - Don't assume federation works - verify it
   - Keep a test Mastodon account handy

2. **Keep It Simple**
   - Each part should be minimal but working
   - Don't add features from later parts
   - Focus on getting federation working

3. **Explain the "Why"**
   - Not just "what" the code does
   - Explain ActivityPub concepts
   - Show how it fits into federation

4. **Provide Working Examples**
   - Every code example should run
   - Include complete, not partial, examples
   - Test all examples before publishing

5. **Be Consistent**
   - Use terminology from ActivityPub reference
   - Follow the template structure
   - Maintain code style throughout

### Common Pitfalls to Avoid

❌ **Don't: Jump Ahead**
- Don't add Part 2 features in Part 1
- Each part should be complete but focused
- Readers need working checkpoints

❌ **Don't: Assume Knowledge**
- Explain ActivityPub concepts
- Don't assume readers know federation
- Provide context for each step

❌ **Don't: Skip Testing**
- Federation is complex - test everything
- Manual testing is as important as automated
- Test with real Mastodon instances

❌ **Don't: Ignore Errors**
- Document common issues
- Provide solutions, not just problems
- Include troubleshooting in each part

❌ **Don't: Forget the Reader**
- Write for someone new to ActivityPub
- Use clear, simple language
- Provide visual aids (diagrams, screenshots)

## Quality Gates

Before marking a part as complete:

- [ ] Code compiles and runs in its project folder (`open-pace-pX/`)
- [ ] All tests pass
- [ ] Can follow from Mastodon
- [ ] Tutorial is clear to a beginner
- [ ] Examples work out-of-the-box
- [ ] No broken links or references
- [ ] Project folder is complete and self-contained
- [ ] README.md exists in project folder
- [ ] **Implementation decisions documented** in relevant strategy documents

## Reader Experience

### Onboarding
- Quick start guide in main README
- Prerequisites clearly listed
- Development environment setup

### Progress Tracking
- Each part has a completion checklist
- Visual indicators of progress
- "What you've built so far" summary

### Troubleshooting
- Common issues section in each part
- Debug tips
- Community support links

## Questions to Ask Yourself

Before marking a part complete:
- ✅ Can a beginner follow this?
- ✅ Does it work with Mastodon?
- ✅ Are all examples complete and runnable?
- ✅ Is the code structure clear?
- ✅ Are ActivityPub concepts explained?
- ✅ Would I understand this if I were new to federation?

## Next Steps

1. **Create `open-pace-p1/` folder** and set up Quarkus project
2. **Create Part 1** (establish pattern)
3. **Document decisions** in relevant strategy documents
4. **Review Part 1** (refine approach)
5. **Create `open-pace-p2/` folder** and apply pattern
6. **Continue for Parts 3-7**
7. **Cross-part review** (consistency check)
8. **Final polish** (examples, screenshots, etc.)

## Migration Between Parts

When building Part 2+:

### Option A: Copy and Refine (Recommended)
- Copy relevant code from `open-pace-p1/` to `open-pace-p2/`
- Refine and extend it
- **Pros**: Shows progression, readers can see evolution
- **Cons**: More code to maintain

### Option B: Reference Only
- Build Part 2 from scratch
- Reference Part 1 concepts in documentation
- **Pros**: Cleaner, focused code
- **Cons**: Less continuity, more duplication

**Recommendation**: Use Option A for Parts 1-3 (shows clear progression), Option B for Parts 4-7 (more specialized features).

## Implementation Decisions Documentation

**Critical**: Each tutorial part must document its key implementation decisions in the relevant strategy documents (Database Design, API Design, Error Handling, etc.).

### What to Document

For each part, document:
- **Design decisions**: Why certain approaches were chosen
- **Patterns established**: Reusable patterns for future parts
- **Lessons learned**: What worked, what didn't, what to avoid
- **Code patterns**: Example code patterns that should be reused
- **Trade-offs**: What was considered and why decisions were made
- **Future considerations**: What might change in later parts

### Format

```markdown
## Part X: [Title]

### [Decision Name]

**Decision**: [What was decided]

**Rationale**: [Why this decision was made]

**Implementation**: [How it's implemented]

**Code Pattern**: [Example code if applicable]

**Future Considerations**: [What might change]
```

### Benefits

- ✅ **Consistency**: Future parts follow established patterns
- ✅ **Learning**: Readers understand why, not just what
- ✅ **Reference**: Quick lookup for implementation details
- ✅ **Evolution**: Shows how decisions evolved across parts

**See**: Relevant strategy documents (Database Design, API Design, Error Handling, etc.) for patterns and examples from completed parts.
