# Documentation Overview

This document explains the purpose and relationship between the main documentation files in the Open Pace tutorial series.

## Core Strategy Documents

### IMPLEMENTATION_STRATEGY.md
**Purpose**: Overall strategy, principles, and structure  
**Audience**: Tutorial authors, project maintainers  
**Content**: 
- Core principles (incremental complexity, working checkpoints)
- Project structure (mono-repo, separate folders)
- Part-specific strategies (what each part focuses on)
- Tools and dependencies
- Code organization strategy
- Quality gates

**Think of it as**: The "what" and "why" - the strategic vision

### Relationship
- **IMPLEMENTATION_STRATEGY.md** contains strategy, principles, structure, and workflow
- It covers the "what", "why", and "how" in one comprehensive document
- Read IMPLEMENTATION_STRATEGY.md to understand the approach and workflow

## Quality & Consistency

### CONSISTENCY_CHECKLIST.md
**Purpose**: Quality standards checklist for each part  
**When to use**: Before marking a part complete  
**Content**: Comprehensive checklist covering code, testing, documentation, federation, etc.

## Reference Documents

### docs/ACTIVITYPUB_REFERENCE.md
**Purpose**: Quick reference for ActivityPub concepts  
**When to use**: When implementing ActivityPub features, need terminology  
**Content**: Actor, Activity, Inbox, Outbox, common patterns, examples

### docs/DATABASE_DESIGN.md
**Purpose**: Database schema design, indexing, and query patterns  
**When to use**: Understanding database structure, optimizing queries, designing new tables  
**Content**: Entity relationships, indexing strategy, JSONB storage, query patterns, migration strategy

### docs/QUARKUS_TECH_STACK.md
**Purpose**: Technologies and versions used  
**When to use**: Need to know versions, understand tech choices  
**Content**: Quarkus version, Java version, all extensions, dependencies

## Feature-Specific Guides

### docs/API_DESIGN.md
**Purpose**: Endpoint organization strategy  
**When to use**: Creating new endpoints, deciding where they should live  
**Content**: ActivityPub vs Application endpoints, `/api/*` prefix strategy

### docs/ERROR_HANDLING_STRATEGY.md
**Purpose**: Validation and error handling approach  
**When to use**: Implementing endpoints, validation, error handling  
**Content**: Hibernate Validator, error response format, exception mapping

### docs/FEDERATION_DELIVERY_STRATEGY.md
**Purpose**: Reliable activity delivery to remote servers  
**When to use**: Implementing federation delivery, retry mechanisms, queue management  
**Content**: Redis queue architecture, exponential backoff, server reputation, shared inbox optimization

### docs/CACHING_STRATEGY.md
**Purpose**: Multi-level caching strategy for performance optimization  
**When to use**: Implementing caching, optimizing performance, reducing database load  
**Content**: L1 (memory), L2 (Redis), L3 (disk) cache architecture, cache invalidation, TTL configuration

### docs/RATE_LIMITING_STRATEGY.md
**Purpose**: Rate limiting strategy for abuse prevention and resource protection  
**When to use**: Implementing rate limits, preventing abuse, protecting endpoints  
**Content**: Per-IP and per-user rate limiting, @RateLimit annotations, custom rate limiters, configuration

### docs/BACKGROUND_JOBS_STRATEGY.md
**Purpose**: Background job processing strategy for asynchronous task execution  
**When to use**: Implementing background jobs, async processing, scheduled tasks  
**Content**: Redis queues, Quarkus Scheduler, job types (map generation, aggregation, cleanup), retry strategy

### docs/HTTP_SIGNATURES_STRATEGY.md
**Purpose**: HTTP Signatures strategy for S2S ActivityPub authentication  
**When to use**: Implementing S2S authentication, signing outgoing requests, verifying incoming requests  
**Content**: RFC 9421 implementation, key management, signature signing/verification, Actor publicKey integration

### docs/ACTIVITYPUB_C2S_PATTERN.md
**Purpose**: Correct C2S implementation pattern  
**When to use**: Implementing client-to-server endpoints  
**Content**: POST to outbox, process by activity type, examples

### docs/MAPPING_INTEGRATION.md
**Purpose**: Mapping and geospatial data strategy  
**When to use**: Implementing Part 7 (mapping features)  
**Content**: GPX processing, PostGIS, OSM tiles, map generation

### docs/SECURITY_INTEGRATION.md
**Purpose**: Authentication and authorization strategy  
**When to use**: Implementing Part 6 (security)  
**Content**: User registration, BCrypt, secure endpoints, Actor-User relationship

## Templates

### docs/SPRINT_TEMPLATE.md
**Purpose**: Template for writing implementation guides for each sprint  
**When to use**: When creating documentation for a new sprint  
**Content**: Structure for sprint documentation including goals, prerequisites, implementation steps, testing

### docs/BACKLOG.md
**Purpose**: Product backlog and ideation document  
**When to use**: Planning future sprints, prioritizing features, comparing with competitors  
**Content**: Feature comparison with Strava, future sprint ideas (11-20), prioritized roadmap, implementation notes
**Purpose**: Structure template for tutorial documentation  
**When to use**: Writing tutorial docs for each part  
**Content**: Template with sections for objectives, steps, testing, etc.

### docs/SPRINT_OUTLINE_TEMPLATE.md
**Purpose**: Planning template for each part  
**When to use**: Planning a part before implementation  
**Content**: Detailed outline structure with time estimates, concepts, etc.

## Setup & Reference

### docs/PROJECT_SETUP.md
**Purpose**: Development environment setup  
**When to use**: Setting up to work on the project  
**Content**: Prerequisites, database setup, Quarkus Dev Services

### docs/MONOREPO_STRUCTURE.md
**Purpose**: Mono-repo structure explanation  
**When to use**: Understanding folder structure, creating new parts  
**Content**: Folder organization, how parts relate, examples

### docs/HOW_TO_REQUEST_IMPLEMENTATION.md
**Purpose**: How to reference docs when asking for implementations  
**When to use**: When requesting implementations from AI/team  
**Content**: How to reference documents effectively, examples

## Quick Reference

### For Tutorial Authors

**Starting a new part**:
1. Read `IMPLEMENTATION_STRATEGY.md` (understand strategy and workflow)
2. Use `docs/SPRINT_OUTLINE_TEMPLATE.md` (plan the sprint)
3. Use `docs/SPRINT_TEMPLATE.md` (write the implementation guide)
4. Check `CONSISTENCY_CHECKLIST.md` (before completing)

**During implementation**:
- `docs/ACTIVITYPUB_REFERENCE.md` - Terminology, JSON-LD context, activity types
- `docs/API_DESIGN.md` - Endpoint organization
- `docs/ERROR_HANDLING_STRATEGY.md` - Error handling
- `docs/DATABASE_DESIGN.md` - Database schema, indexing, query patterns
- `docs/CACHING_STRATEGY.md` - Caching (Part 3+)
- `docs/RATE_LIMITING_STRATEGY.md` - Rate limiting (Part 6+)
- `docs/BACKGROUND_JOBS_STRATEGY.md` - Background jobs (Part 3+)
- `docs/HTTP_SIGNATURES_STRATEGY.md` - HTTP Signatures (Part 8+)
- `docs/FEDERATION_DELIVERY_STRATEGY.md` - Federation delivery (Part 5+)

**Feature-specific**:
- `docs/ACTIVITYPUB_C2S_PATTERN.md` - C2S implementation
- `docs/MAPPING_INTEGRATION.md` - Part 7 mapping
- `docs/SECURITY_INTEGRATION.md` - Part 6 security

### For Tutorial Readers

**Getting started**:
1. `README.md` - Project overview
2. `docs/PROJECT_SETUP.md` - Environment setup
3. `docs/part-1-basic-activitypub.md` - Start tutorial

**Understanding concepts**:
- `docs/ACTIVITYPUB_REFERENCE.md` - ActivityPub concepts
- `docs/QUARKUS_TECH_STACK.md` - Technologies used

## Document Relationships

```
IMPLEMENTATION_STRATEGY.md (Strategy & Workflow)
    ↓
SPRINT_OUTLINE_TEMPLATE.md (Planning)
    ↓
Implementation
    ↓
SPRINT_TEMPLATE.md (Documentation)
    ↓
CONSISTENCY_CHECKLIST.md (Review)
```

## Summary

**Strategy Documents**:
- `IMPLEMENTATION_STRATEGY.md` - Strategy, principles, structure, and workflow (what, why, and how)

**Reference Documents**:
- Feature guides (C2S, Mapping, Security, API Design, Error Handling)
- Quick references (ActivityPub, Tech Stack)
- Templates (Tutorial, Outline)

**Quality Documents**:
- `CONSISTENCY_CHECKLIST.md` - Quality standards
