# Mono-Repo Structure Guide

This document explains how the Open Pace tutorial series is organized in a mono-repo structure.

## Overview

Each tutorial part is a **standalone, runnable project** in its own folder. This approach works well for mono-repos and provides clear separation between parts.

## Folder Structure

```
mono-repo-root/
├── open-pace/                    # Strategy & shared documentation
│   ├── README.md                 # Main overview
│   ├── IMPLEMENTATION_STRATEGY.md      # Overall strategy
│   ├── IMPLEMENTATION_STRATEGY.md      # Strategy, principles, and workflow
│   ├── CONSISTENCY_CHECKLIST.md  # Quality standards
│   └── docs/                     # Shared documentation
│       ├── part-1-basic-activitypub.md
│       ├── part-2-custom-activities.md
│       ├── part-3-rich-interop.md
│       ├── part-4-sports-features.md
│       ├── part-5-privacy-data.md
│       ├── ACTIVITYPUB_REFERENCE.md
│       ├── PROJECT_SETUP.md
│       └── SPRINT_TEMPLATE.md
│
├── open-pace-p1/                 # Part 1: Basic ActivityPub Server
│   ├── pom.xml                   # Quarkus Maven project
│   ├── README.md                 # Part 1 specific guide
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/org/openpace/
│   │   │   │   └── core/         # Core ActivityPub implementation
│   │   │   └── resources/
│   │   │       └── application.properties
│   │   └── test/                 # Part 1 tests
│   └── (Quarkus Dev Services handles database automatically)
│
├── open-pace-p2/                 # Part 2: Custom Activity Types
│   ├── pom.xml
│   ├── README.md
│   ├── src/
│   │   └── main/java/org/openpace/
│   │       ├── core/             # Core (refined from P1)
│   │       └── activities/       # Custom activity types
│   └── ...
│
├── open-pace-p3/                 # Part 3: Rich Data & Interop
├── open-pace-p4/                 # Part 4: Sports Features
├── open-pace-p5/                 # Part 5: Privacy & Data
├── open-pace-p6/                 # Part 6: Security Integration
└── open-pace-p7/                 # Part 7: Mapping Integration
```

## Key Principles

### 1. **Independence**
Each part (`open-pace-pX/`) is a complete, runnable project:
- Has its own `pom.xml`
- Has its own `application.properties`
- Has its own database migrations
- **Quarkus Dev Services** automatically handles database setup (no docker-compose needed)
- Can be run independently

### 2. **Progression**
While independent, parts build on concepts:
- Part 1: Basic federation
- Part 2: Adds custom types (can copy/adapt Part 1 code)
- Part 3: Adds comments/likes/attachments (builds on Part 2)
- Part 4: Adds sports features (builds on Part 2)
- Part 5: Adds privacy features (builds on Parts 1-4)
- Part 6: Adds security (builds on Parts 1-5)
- Part 7: Adds mapping (builds on Parts 1-6)

### 3. **Shared Documentation**
Common docs live in `open-pace/docs/`:
- Tutorial guides reference these
- ActivityPub reference for consistency
- Setup guides apply to all parts

## Working with Parts

### Starting a New Part

1. **Create the folder**:
   ```bash
   mkdir open-pace-p1
   cd open-pace-p1
   ```

2. **Initialize Quarkus project**:
   ```bash
   # Use Quarkus CLI or Maven archetype
   mvn io.quarkus.platform:quarkus-maven-plugin:3.0.0.Final:create
   ```

3. **Add dependencies** (REST, Vert.x, PostgreSQL, Flyway, etc.)
   - Quarkus Dev Services will automatically handle PostgreSQL setup

4. **Create README.md** in the part's folder

5. **Start implementing** following the tutorial guide
   - No need to set up database manually - Dev Services handles it!

### Referencing Previous Parts

When building Part 2+:

**Option A: Copy and Refine** (Recommended for P1-P3)
- Copy relevant code from `open-pace-p1/` to `open-pace-p2/`
- Refine and extend it
- Shows clear progression

**Option B: Reference Only** (Good for P4-P5)
- Build from scratch
- Reference concepts in documentation
- More focused, less duplication

### Running a Part

```bash
# Navigate to the part's folder
cd open-pace-p1

# Start the application
./mvnw quarkus:dev

# Run tests
./mvnw test
```

## Benefits of This Structure

### ✅ For Tutorial Authors
- **Clear boundaries**: Each part is self-contained
- **Easy testing**: Test each part independently
- **Simple git workflow**: No branch management needed
- **Flexible**: Can reorganize without breaking things

### ✅ For Tutorial Readers
- **Clear progression**: See evolution across folders
- **Easy navigation**: Know exactly where each part lives
- **Independent learning**: Can focus on one part at a time
- **Easy to reset**: Delete folder and start fresh

### ✅ For Mono-Repo
- **Fits naturally**: Standard mono-repo pattern
- **No conflicts**: Parts don't interfere with each other
- **Easy to find**: Clear naming convention
- **Scalable**: Easy to add more parts

## Documentation Structure

### Main Documentation (`open-pace/docs/`)
- **Tutorial guides**: `part-X-*.md` - Step-by-step instructions
- **Reference**: `ACTIVITYPUB_REFERENCE.md` - Consistent terminology
- **Setup**: `PROJECT_SETUP.md` - Development environment

### Part-Specific Documentation (`open-pace-pX/README.md`)
Each part folder should have a README with:
- Quick start for that part
- What you'll build
- Prerequisites
- Link to full tutorial in `open-pace/docs/`

## Git Workflow

Since each part is a folder (not a branch):

1. **Develop in the folder**: Work directly in `open-pace-pX/`
2. **Commit to mono-repo**: All parts committed together
3. **No branch switching**: Just navigate to different folders
4. **Optional tagging**: Tag the mono-repo if you want versioned releases

## Example: Creating Part 1

```bash
# 1. Create folder
mkdir open-pace-p1
cd open-pace-p1

# 2. Initialize Quarkus
mvn io.quarkus.platform:quarkus-maven-plugin:3.0.0.Final:create \
    -DprojectGroupId=org.openpace \
    -DprojectArtifactId=open-pace-p1 \
    -DclassName="org.openpace.core.WebFingerResource" \
    -Dextensions="resteasy-reactive,vertx,postgresql,flyway"

# 3. Create README
cat > README.md << 'EOF'
# Part 1: Basic ActivityPub Server

Quick start for Part 1. See full tutorial in `../open-pace/docs/part-1-basic-activitypub.md`.

## Run
./mvnw quarkus:dev

## Test
./mvnw test
EOF

# 4. Start implementing following the tutorial guide
```

## Migration Strategy

When moving from Part 1 to Part 2:

1. **Review Part 1**: Understand what was built
2. **Create Part 2 folder**: `mkdir open-pace-p2`
3. **Copy relevant code**: From `open-pace-p1/` to `open-pace-p2/`
4. **Refine and extend**: Add Part 2 features
5. **Update documentation**: Reference Part 1 concepts

## Best Practices

1. **Keep parts independent**: Each should run on its own
2. **Document dependencies**: If Part 2 needs Part 1 concepts, document it
3. **Consistent naming**: Use same package structure across parts
4. **Clear READMEs**: Each part folder needs a helpful README
5. **Test thoroughly**: Each part should be fully testable

## Troubleshooting

### "Where do I start?"
→ Go to `open-pace-p1/` and follow `../open-pace/docs/part-1-basic-activitypub.md`

### "How do I see what changed from Part 1 to Part 2?"
→ Compare `open-pace-p1/` and `open-pace-p2/` folders

### "Can I skip Part 1?"
→ Technically yes (each part is independent), but not recommended (concepts build on each other)

### "Where are the shared docs?"
→ In `open-pace/docs/` folder

## Summary

This mono-repo structure provides:
- ✅ Clear organization
- ✅ Independent, runnable projects
- ✅ Easy navigation
- ✅ Natural progression
- ✅ Mono-repo friendly

Each part is a complete project, but they work together to teach a complete federated ActivityPub application.
