# Product Backlog & Ideation

This document captures potential features, enhancements, and future sprints for Open Pace, organized by priority and phase.

## Current Status

**Completed Sprints**: 1-7 (MVP Foundation)  
**Planned Sprints**: 8-10 (Analytics, Social, Gear)  
**Total Planned**: 10 sprints

## Feature Comparison: Open Pace vs Strava

### ✅ Covered in Current Sprints (1-7)

| Feature | Sprint | Status |
|---------|--------|--------|
| Basic activity tracking (Run, Ride, Swim) | Sprint 2 | ✅ Planned |
| GPX file handling | Sprint 3 | ✅ Planned |
| Activity visualization | Sprint 3 | ✅ Planned |
| Segments | Sprint 4 | ✅ Planned |
| Leaderboards | Sprint 4 | ✅ Planned |
| Privacy controls | Sprint 5 | ✅ Planned |
| Data export | Sprint 5 | ✅ Planned |
| Authentication/Authorization | Sprint 6 | ✅ Planned |
| Maps and routes | Sprint 7 | ✅ Planned |
| PostGIS geospatial storage | Sprint 7 | ✅ Planned |

### ❌ Missing Features (Potential Future Sprints)

| Feature | Priority | Suggested Sprint |
|---------|----------|------------------|
| Personal records & analytics | HIGH | Sprint 8 ✅ |
| Activity feed & timeline | HIGH | Sprint 9 ✅ |
| Comments & kudos | HIGH | Sprint 9 ✅ |
| Advanced metrics (HR, power, cadence) | HIGH | Sprint 11 |
| Device integration (Garmin, etc.) | HIGH | Sprint 11 |
| Gear tracking | MEDIUM | Sprint 10 ✅ |
| Clubs & groups | MEDIUM | Sprint 10 (alternative) |
| Goals & training plans | MEDIUM | Sprint 13 |
| Route planning & builder | MEDIUM | Sprint 14 |
| Achievements & badges | MEDIUM | Sprint 15 |
| Photos & media | MEDIUM | Sprint 17 |
| Advanced analytics (premium) | LOW | Sprint 16 |
| Mobile app | LOW | Sprint 18 |
| Notifications & digests | LOW | Sprint 19 |
| Federation enhancements | LOW | Sprint 20 |

## Planned Sprints (8-10)

### Sprint 8: Activity Analytics & Personal Records ✅
**Status**: Planned  
**Priority**: HIGH  
**Phase**: Core Sports Platform

**Features**:
- Automatic personal record detection (fastest 5K, longest ride, etc.)
- Split analysis (per km/mile)
- Pace zones and heart rate zones
- Elevation profile visualization
- Comparative analysis (this run vs. average)
- Activity statistics dashboard

**Technical**:
- Time-series data analysis
- Efficient queries for PRs across all activities
- Chart generation (Recharts in React)
- Database indexes for performance queries

### Sprint 9: Social Interactions & Feed ✅
**Status**: Planned  
**Priority**: HIGH  
**Phase**: Core Sports Platform

**Features**:
- Activity feed (timeline of followed users)
- Comments on activities (ActivityPub replies)
- Kudos/Likes (ActivityPub Like activities)
- Activity sharing to feed
- @mentions in comments
- Notifications for interactions

**Technical**:
- ActivityPub Create/Like/Announce activities
- Feed aggregation and ranking
- Real-time notifications (WebSocket or SSE)
- Efficient timeline queries

### Sprint 10: Gear & Equipment Tracking ✅
**Status**: Planned  
**Priority**: MEDIUM  
**Phase**: Core Sports Platform

**Features**:
- Add bikes, shoes, other gear
- Track mileage/usage per gear
- Assign gear to activities (retroactively)
- Maintenance reminders
- Gear retirement tracking
- Default gear per activity type

**Technical**:
- Gear database schema
- Activity-gear relationship
- Aggregation queries for mileage
- Scheduled notifications for maintenance

## Future Sprint Ideas (11-20)

### Sprint 11: Advanced Metrics & Device Integration
**Priority**: HIGH  
**Phase**: Core Sports Platform

**Features**:
- Heart rate tracking and zones
- Power meter data (cycling)
- Cadence tracking
- Temperature data
- Multiple file format support (FIT, TCX, GPX)
- Automatic activity sync from devices

**Technical**:
- FIT file parsing (Garmin format)
- TCX file parsing (Training Center XML)
- Webhook integrations for device manufacturers
- OAuth with Garmin Connect, etc.
- Data normalization across formats

**Dependencies**: Sprint 2 (Custom Activity Types), Sprint 8 (Analytics)

### Sprint 12: Clubs & Groups
**Priority**: MEDIUM  
**Phase**: Engagement

**Features**:
- Create/join clubs
- Club activity feed
- Club leaderboards
- Club challenges
- Club announcements
- Member management

**Technical**:
- ActivityPub Group actors
- Group inbox/outbox handling
- Membership management
- Aggregate statistics for groups

**Dependencies**: Sprint 9 (Social Feed), ActivityPub Groups spec

### Sprint 13: Goals & Training Plans
**Priority**: MEDIUM  
**Phase**: Training & Planning

**Features**:
- Set distance/time goals (weekly, monthly, yearly)
- Goal progress tracking
- Training calendar
- Workout builder (intervals, repeats)
- Training plan templates
- Goal achievement notifications

**Technical**:
- Recurring goal calculations
- Calendar view with activity slots
- Workout definition schema
- Progress calculation algorithms

**Dependencies**: Sprint 8 (Analytics)

### Sprint 14: Route Planning & Discovery
**Priority**: MEDIUM  
**Phase**: Training & Planning

**Features**:
- Interactive route builder
- Save/share routes
- Browse popular routes in area
- Route export (GPX)
- Route difficulty estimation
- Route search and filtering

**Technical**:
- Map drawing tools (Leaflet.draw or similar)
- Elevation API integration
- Route storage and retrieval
- Spatial queries for route discovery
- Route similarity algorithms

**Dependencies**: Sprint 7 (Mapping)

### Sprint 15: Achievements & Gamification
**Priority**: MEDIUM  
**Phase**: Engagement

**Features**:
- Automatic achievement detection
- Badge system
- Monthly challenges
- Streaks (consecutive days)
- Milestone celebrations
- Achievement sharing

**Technical**:
- Achievement rule engine
- Background jobs for achievement checking
- Badge asset management
- Challenge participation tracking

**Dependencies**: Sprint 8 (Analytics), Sprint 9 (Social)

### Sprint 16: Advanced Analytics (Premium Features)
**Priority**: LOW  
**Phase**: Premium & Advanced

**Features**:
- Fitness & Freshness (CTL/ATL/TSB)
- Power curves and analysis
- Pace analysis across time
- Comparative heat maps
- Suffer score / relative effort
- Year in review statistics

**Technical**:
- Complex time-series calculations
- Power duration curve algorithms
- Statistical aggregations
- Data visualization (charts, graphs)
- Cached calculation results

**Dependencies**: Sprint 8 (Analytics), Sprint 11 (Advanced Metrics)

### Sprint 17: Photos & Media
**Priority**: MEDIUM  
**Phase**: Engagement

**Features**:
- Upload multiple photos per activity
- Photo metadata (location, timestamp)
- Photo ordering/gallery
- Photo comments
- Activity highlights with photos
- Media storage management

**Technical**:
- File upload (multipart)
- Image resizing/optimization
- CDN integration or object storage (S3)
- Photo ActivityPub attachments
- Storage quota management

**Dependencies**: Sprint 3 (Attachments), Sprint 9 (Social)

### Sprint 18: Mobile App Support
**Priority**: LOW  
**Phase**: Mobile & Scale

**Features**:
- Mobile-optimized API
- Activity recording (GPS tracking)
- Live stats during activity
- Auto-pause detection
- Background location tracking
- Offline support

**Technical**:
- Mobile app (React Native / Flutter)
- GPS data collection
- Battery optimization
- Local data storage
- Sync protocols

**Dependencies**: All core sprints (1-12)

### Sprint 19: Notifications & Communication
**Priority**: LOW  
**Phase**: Premium & Advanced

**Features**:
- Activity notifications (kudos, comments)
- Challenge reminders
- Goal progress updates
- Weekly/monthly summaries
- Email digests
- In-app notifications

**Technical**:
- Notification service
- Email templates
- Push notification support
- Notification preferences
- Digest generation jobs

**Dependencies**: Sprint 9 (Social), Sprint 13 (Goals)

### Sprint 20: Federation Features
**Priority**: LOW  
**Phase**: Mobile & Scale

**Features**:
- Cross-instance clubs
- Federated challenges
- Segment synchronization across instances
- Activity discovery across fediverse
- Relay support for better discoverability

**Technical**:
- Enhanced ActivityPub Group support
- Relay protocol implementation
- Cross-instance data synchronization
- Federated search

**Dependencies**: Sprint 12 (Clubs), ActivityPub Relay spec

## Prioritized Roadmap

### Phase 1: MVP Foundation ✅
**Sprints 1-7** (Current)
- Foundation + basic features
- Basic federation working
- Core ActivityPub functionality

### Phase 2: Core Sports Platform
**Sprints 8-12** (High Priority)
- Sprint 8: Analytics & PRs ✅
- Sprint 9: Social Feed & Interactions ✅
- Sprint 10: Gear Tracking ✅
- Sprint 11: Advanced Metrics
- Sprint 12: Clubs & Groups

**Why**: Essential for Strava-like experience

### Phase 3: Training & Planning
**Sprints 13-14** (Medium Priority)
- Sprint 13: Goals & Training Plans
- Sprint 14: Route Planning

**Why**: Differentiate from simple tracking apps

### Phase 4: Engagement
**Sprints 10, 15, 17** (Medium Priority)
- Sprint 10: Clubs & Groups (if not in Phase 2)
- Sprint 15: Achievements
- Sprint 17: Photos & Media

**Why**: Keep users engaged long-term

### Phase 5: Premium & Advanced
**Sprints 16, 19** (Low Priority)
- Sprint 16: Advanced Analytics
- Sprint 19: Notifications

**Why**: Potential monetization, power users

### Phase 6: Mobile & Scale
**Sprints 18, 20** (Low Priority)
- Sprint 18: Mobile App
- Sprint 20: Federation Features

**Why**: Growth and platform maturity

## Detailed Feature Gaps

### Core Activity Features

#### Missing Detailed Metrics
- ❌ Heart rate zones and analysis
- ❌ Power meter data (cycling)
- ❌ Cadence tracking (running/cycling)
- ❌ Elevation gain/loss detailed analysis
- ❌ Temperature data
- ❌ Weather conditions

**Suggested Sprint**: Sprint 11 (Advanced Metrics)

#### Missing Activity Types
- ❌ 20+ activity types (hike, yoga, ski, row, etc.)
- ❌ Custom activity type creation
- ❌ Activity type-specific metrics

**Suggested Sprint**: Extend Sprint 2 or new sprint

#### Missing Device Integration
- ❌ Garmin Connect sync
- ❌ Wahoo sync
- ❌ Apple Watch sync
- ❌ Polar sync
- ❌ Generic FIT/TCX import

**Suggested Sprint**: Sprint 11 (Advanced Metrics)

#### Missing Activity Features
- ❌ Multiple photos per activity
- ❌ Activity tagging (workout, race, commute)
- ❌ Activity notes/description
- ❌ Activity visibility controls (public/followers/private)

**Suggested Sprint**: Sprint 17 (Photos), Sprint 5 extension

### Social Features

#### Missing Social Interactions
- ❌ Activity feed/timeline
- ❌ Comments on activities
- ❌ Kudos/likes
- ❌ Activity sharing
- ❌ @mentions

**Suggested Sprint**: Sprint 9 ✅

#### Missing Social Groups
- ❌ Clubs/groups
- ❌ Group activities
- ❌ Group leaderboards
- ❌ Group challenges

**Suggested Sprint**: Sprint 12 (Clubs)

#### Missing Gamification
- ❌ Achievements/badges
- ❌ Challenges (monthly, brand)
- ❌ Streaks
- ❌ Milestones

**Suggested Sprint**: Sprint 15 (Achievements)

### Training & Analysis

#### Missing Analytics
- ❌ Personal records
- ❌ Split analysis
- ❌ Pace zones
- ❌ Comparative analysis

**Suggested Sprint**: Sprint 8 ✅

#### Missing Training Features
- ❌ Goals (distance/time)
- ❌ Training calendar
- ❌ Training plans
- ❌ Workout builder

**Suggested Sprint**: Sprint 13 (Goals & Training)

#### Missing Advanced Analytics
- ❌ Fitness & Freshness (CTL/ATL/TSB)
- ❌ Power curves
- ❌ Suffer score
- ❌ Year in review

**Suggested Sprint**: Sprint 16 (Advanced Analytics)

### Routes & Planning

#### Missing Route Features
- ❌ Route builder
- ❌ Route library
- ❌ Route sharing
- ❌ Route discovery
- ❌ Heatmaps (popular routes)

**Suggested Sprint**: Sprint 14 (Route Planning)

#### Missing Navigation
- ❌ Turn-by-turn navigation
- ❌ Live route following
- ❌ Offline route support

**Suggested Sprint**: Sprint 18 (Mobile App) or separate

### Equipment

#### Missing Gear Features
- ❌ Gear tracking
- ❌ Maintenance reminders
- ❌ Multiple gear assignment

**Suggested Sprint**: Sprint 10 ✅

## Implementation Notes

### ActivityPub Advantages
- ✅ **Federation**: Built-in (Strava doesn't have this)
- ✅ **Interoperability**: Works with Mastodon, other ActivityPub servers
- ✅ **Decentralization**: Users can run their own instances
- ✅ **Open Standards**: No vendor lock-in

### Technical Considerations

#### Performance
- Efficient queries for PRs, analytics
- Caching strategies for feeds
- Background job processing for calculations
- Database indexing for time-series data

#### Scalability
- Federation delivery optimization
- Feed aggregation strategies
- Media storage (CDN/S3)
- Real-time notification systems

#### Data Formats
- Support multiple file formats (GPX, FIT, TCX)
- Normalize data across formats
- Handle missing data gracefully
- Version data schemas

## Decision Log

### Why These Priorities?

**Phase 2 (Sprints 8-12) is critical** because:
1. Analytics (Sprint 8) provides immediate value to users
2. Social features (Sprint 9) leverage ActivityPub's strengths
3. Advanced metrics (Sprint 11) differentiate from basic trackers
4. Gear tracking (Sprint 10) is unique and useful
5. Clubs (Sprint 12) showcase ActivityPub Groups

**Phase 3-6 can be adjusted** based on:
- User feedback
- Community needs
- Resource availability
- Competitive landscape

## Future Considerations

### Potential New Sprints
- **Sprint 21**: Multi-sport support (triathlon, duathlon)
- **Sprint 22**: Race/event management
- **Sprint 23**: Coach-athlete relationships
- **Sprint 24**: API for third-party integrations
- **Sprint 25**: Premium subscription features

### Research Areas
- ActivityPub Relay protocol for better discovery
- WebSub for real-time updates
- Linked Data Fragments for efficient queries
- Microformats for better SEO

## References

- [Strava Features](https://www.strava.com/features)
- [ActivityPub Specification](https://www.w3.org/TR/activitypub/)
- [ActivityPub Groups](https://www.w3.org/TR/activitypub/#groups)
- [FIT File Format](https://developer.garmin.com/fit/overview/)
