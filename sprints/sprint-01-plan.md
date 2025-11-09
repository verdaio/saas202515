# Sprint 1 - Foundation

**Sprint Duration:** Week 1-4 (4 weeks)
**Sprint Goal:** Establish technical foundation and ship mobile crew app core (clock in/out, checklists, photos)
**Status:** Planning
**Trade Name:** Cleaning & Janitorial

---

## Sprint Goal

**Primary Objective:** Build the technical foundation for our multi-tenant SaaS and ship the core mobile crew app functionality (clock in/out with geofencing, checklists, photo capture).

**Success Criteria:**
1. ✅ Azure infrastructure deployed (Container Apps, PostgreSQL, Redis, Blob Storage)
2. ✅ Multi-tenant authentication working (Azure AD B2C + subdomain routing)
3. ✅ Mobile crew app can clock in/out with geofence validation
4. ✅ Mobile crew app can complete checklists and capture photos
5. ✅ CI/CD pipeline automatically deploys backend + mobile app

**Why this matters:** Without a solid foundation, we'll accumulate technical debt and slow down future sprints. The crew app is the most critical UX (adoption depends on it), so we're starting there.

---

## Sprint Capacity

**Available Days:** 20 working days (4 weeks × 5 days)
**Team:** 1-2 developers
**Capacity:** 120-160 hours (assuming 60-80 hours/developer)
**Commitments/Time Off:** None planned

**Velocity Target:** This is Sprint 1, so no historical velocity. Aiming for conservative estimates to avoid over-commitment.

---

## Sprint Backlog

### High Priority (Must Complete) - P0

| Story | Description | Estimate | Assignee | Status | Notes |
|-------|-------------|----------|----------|--------|-------|
| US-001 | Set up Azure infrastructure (IaC with Bicep) | L (16h) | DevOps | 📋 Todo | Container Apps, PostgreSQL, Redis, Key Vault |
| US-002 | Create Prisma database schema (multi-tenant) | M (8h) | Backend | 📋 Todo | Tenants, sites, shifts, checklists, users |
| US-003 | Implement Row-Level Security (RLS) policies | S (4h) | Backend | 📋 Todo | Tenant isolation at database level |
| US-004 | Set up Azure AD B2C multi-tenant auth | M (10h) | Backend | 📋 Todo | Subdomain routing, JWT validation |
| US-005 | Create NestJS backend API (GraphQL setup) | M (8h) | Backend | 📋 Todo | Apollo Server, auth guards, tenant middleware |
| US-006 | Set up CI/CD pipeline (GitHub Actions) | S (6h) | DevOps | 📋 Todo | Build, test, deploy to Azure |
| US-007 | Initialize React Native mobile app (Expo) | S (4h) | Mobile | 📋 Todo | Project setup, navigation, auth flow |
| US-008 | Implement geofencing for clock in/out | L (12h) | Mobile | 📋 Todo | Expo Location, validate 100m radius |
| US-009 | Build clock in/out UI (mobile) | M (8h) | Mobile | 📋 Todo | Button, confirmation, loading states |
| US-010 | Build checklist UI (mobile) | M (10h) | Mobile | 📋 Todo | List view, checkboxes, progress indicator |
| US-011 | Implement photo capture and upload | L (12h) | Mobile + Backend | 📋 Todo | Expo Camera, Blob Storage SAS tokens |
| US-012 | Create backend GraphQL API for shifts | M (8h) | Backend | 📋 Todo | Queries: myShifts, shiftDetails |
| US-013 | Create backend GraphQL API for checklists | S (6h) | Backend | 📋 Todo | Mutations: completeChecklistItem |

**Total Estimated Hours: 112 hours** (within 120-160 hour capacity)

### Medium Priority (Should Complete) - P1

| Story | Description | Estimate | Assignee | Status | Notes |
|-------|-------------|----------|----------|--------|-------|
| US-014 | Add Spanish language toggle (mobile) | S (4h) | Mobile | 📋 Todo | react-i18next, English/Spanish strings |
| US-015 | Implement offline mode (mobile) | M (8h) | Mobile | 📋 Todo | Apollo Cache persistence, queue actions |
| US-016 | Create admin seed script (test data) | XS (2h) | Backend | 📋 Todo | Seed 1 tenant, 3 sites, 10 shifts |
| US-017 | Set up Application Insights monitoring | XS (3h) | DevOps | 📋 Todo | Logs, metrics, custom events |

**Total Estimated Hours (if capacity allows): 17 hours**

### Low Priority (Nice to Have) - P2

| Story | Description | Estimate | Assignee | Status | Notes |
|-------|-------------|----------|----------|--------|-------|
| US-018 | Create basic web admin (tenant dashboard) | M (8h) | Frontend | 📋 Todo | View shifts, sites (read-only) |
| US-019 | Add push notifications (shift reminders) | S (6h) | Mobile | 📋 Todo | Expo Notifications, schedule local |

**Status Legend:**
- 📋 Todo
- 🏗️ In Progress
- 👀 In Review
- ✅ Done
- ❌ Blocked

---

## Technical Debt / Maintenance

Items that need attention but aren't new features:

- [ ] Set up pre-commit hooks (Prettier, ESLint)
- [ ] Configure Dependabot for security updates
- [ ] Create `CONTRIBUTING.md` with coding standards
- [ ] Set up staging environment (separate from production)

---

## User Stories Breakdown

### US-001: Set up Azure infrastructure (IaC with Bicep)

**As a** DevOps engineer
**I want** to deploy Azure infrastructure via Bicep templates
**So that** we can provision environments consistently and quickly

**Acceptance Criteria:**
- [ ] Bicep templates exist for all Azure resources
- [ ] Deploying `main.bicep` creates: Container Apps (2), PostgreSQL, Redis, Blob Storage, Key Vault
- [ ] GitHub Actions workflow can deploy infrastructure
- [ ] Staging and production environments separable via parameters
- [ ] Outputs include connection strings (stored in Key Vault)

**Technical Notes:**
- Use existing templates in `infrastructure/bicep/`
- Follow Azure naming convention: `app-vrd-202515-dev-eus2-01`
- Estimated cost: $200-300/month for dev environment

---

### US-002: Create Prisma database schema (multi-tenant)

**As a** backend developer
**I want** a Prisma schema with multi-tenant support
**So that** we can store tenant data with proper isolation

**Acceptance Criteria:**
- [ ] Prisma schema includes: Tenant, User, Site, Shift, Checklist, ChecklistItem, Photo, Timesheet, Exception
- [ ] All tenant-scoped tables have `tenant_id` column with foreign key to `tenants`
- [ ] Indexes on `(tenant_id, created_at)` for time-series queries
- [ ] Migration scripts work (can seed database)
- [ ] TypeScript types auto-generated from schema

**Schema Preview:**
```prisma
model Tenant {
  id        String   @id @default(uuid())
  slug      String   @unique  // for subdomain
  name      String
  status    String   @default("active")
  createdAt DateTime @default(now())

  sites     Site[]
  users     User[]
  shifts    Shift[]
}

model Site {
  id         String   @id @default(uuid())
  tenantId   String
  name       String
  address    String?
  latitude   Float?
  longitude  Float?

  tenant     Tenant   @relation(fields: [tenantId], references: [id])
  shifts     Shift[]

  @@index([tenantId, createdAt])
}

model Shift {
  id             String   @id @default(uuid())
  tenantId       String
  siteId         String
  userId         String?  // assigned crew member
  scheduledStart DateTime
  scheduledEnd   DateTime
  status         String   @default("scheduled") // scheduled, in_progress, completed, missed

  tenant         Tenant   @relation(fields: [tenantId], references: [id])
  site           Site     @relation(fields: [siteId], references: [id])
  user           User?    @relation(fields: [userId], references: [id])
  checklist      Checklist?
  timesheets     Timesheet[]
  photos         Photo[]

  @@index([tenantId, scheduledStart])
}

// ... additional models
```

---

### US-008: Implement geofencing for clock in/out

**As a** crew member
**I want** the app to automatically detect when I'm at the site
**So that** I can clock in quickly without entering location manually

**Acceptance Criteria:**
- [ ] App requests location permission on first launch
- [ ] Clock in button enabled only when within 100m of site GPS coordinates
- [ ] Display distance from site (e.g., "52m away")
- [ ] Error message if outside geofence: "You must be at the site to clock in"
- [ ] Fallback: "Override" button for dispatcher (requires reason)
- [ ] Geofence accuracy > 95% (tested on 3 devices: iOS, Android flagship, Android mid-tier)

**Technical Notes:**
- Use Expo Location API: `Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.High })`
- Calculate distance with Haversine formula
- Cache site coordinates locally (offline mode)
- Log geofence distance in Application Insights for accuracy tracking

**Test Cases:**
- ✅ Within 50m of site → Clock in succeeds
- ✅ 150m from site → Clock in blocked
- ✅ GPS unavailable → Show "Enable location services" message
- ✅ Site has no GPS coordinates → Allow clock in (log warning)

---

### US-010: Build checklist UI (mobile)

**As a** crew member
**I want** to see a checklist of tasks for this shift
**So that** I know exactly what to clean and can mark items complete

**Acceptance Criteria:**
- [ ] Checklist loads when shift starts (after clock in)
- [ ] Tasks grouped by area (e.g., "Restrooms", "Lobby", "Break Room")
- [ ] Checkbox for each task, tap to toggle complete
- [ ] Photo button next to each task (optional photo proof)
- [ ] Progress indicator: "4 of 12 tasks complete"
- [ ] "Submit Checklist" button enabled when all tasks complete
- [ ] Works offline (sync when connectivity restored)

**UI Mockup:**
```
┌─────────────────────────────┐
│ Shift: Office Building A    │
│ Started: 6:05 AM             │
│ Progress: ████░░░░░░  40%   │
├─────────────────────────────┤
│ Restrooms                    │
│ ☑️ Clean toilets       📷    │
│ ☑️ Restock supplies    📷    │
│ ☐ Mop floors           📷    │
│                              │
│ Lobby                        │
│ ☑️ Vacuum carpets      📷    │
│ ☐ Dust furniture       📷    │
│ ☐ Empty trash bins     📷    │
│                              │
│        [Submit Checklist]    │
└─────────────────────────────┘
```

**Technical Notes:**
- Apollo Client local state for checkbox toggles (offline)
- GraphQL mutation when checklist submitted: `completeChecklistItem(itemId, photoUrl)`
- Use React Native FlatList for performance (100+ tasks per shift)

---

### US-011: Implement photo capture and upload

**As a** crew member
**I want** to take photos of completed work
**So that** I have proof for my manager and the client

**Acceptance Criteria:**
- [ ] Tap camera icon next to checklist item → Opens camera
- [ ] Capture photo → Shows preview with "Retake" and "Use Photo" buttons
- [ ] Photo compresses to < 500KB (optimize for upload speed)
- [ ] Photo uploads to Azure Blob Storage via SAS token
- [ ] Photo URL saved to database (associated with shift + checklist item)
- [ ] Upload works offline (queued, retry when connectivity restored)
- [ ] Shows upload progress (spinner or progress bar)

**Technical Notes:**
- Expo Camera API: `Camera.takePictureAsync({ quality: 0.7 })`
- Backend generates SAS token (15 min expiry): `/api/photos/upload` → `{ uploadUrl, photoId }`
- Mobile uploads directly to Blob Storage (no backend bottleneck)
- Photo metadata: `{ tenant_id, shift_id, checklist_item_id, uploaded_at, uploaded_by }`

**Test Cases:**
- ✅ Capture photo → Upload succeeds → Photo appears in checklist
- ✅ Capture photo offline → Queued → Upload when online
- ✅ Upload fails (network error) → Retry 3 times → Show error message
- ✅ Photo > 500KB → Compressed before upload

---

## Daily Progress

*(To be filled out during sprint execution)*

### Week 1 - Day 1 (Monday)
**What I worked on:**
- Sprint kickoff meeting
- Azure infrastructure setup (US-001 started)
- Prisma schema design (US-002 started)

**Blockers:**
- None

**Plan for tomorrow:**
- Continue Azure infrastructure deployment
- Complete Prisma schema and migrations

---

### Week 1 - Day 2 (Tuesday)
**What I worked on:**
-

**Blockers:**
-

**Plan for tomorrow:**
-

---

*(Continue for all 20 days)*

---

## Scope Changes

Document any stories added or removed during the sprint:

| Date | Change | Reason |
|------|--------|--------|
| - | - | - |

---

## Sprint Metrics

### Planned vs Actual
- **Planned:** 112 hours (P0) + 17 hours (P1 if capacity)
- **Completed:** [TBD at sprint end]
- **Completion Rate:** [TBD]%

### Velocity
- **Previous Sprint:** N/A (first sprint)
- **This Sprint:** [TBD]
- **Trend:** Baseline

---

## Wins & Learnings

*(To be filled out during sprint retrospective)*

### What Went Well
-

### What Could Be Improved
-

### Action Items for Sprint 2
- [ ]

---

## Sprint Review Notes

*(To be completed at sprint end)*

**What We Shipped:**
- Feature 1: Mobile crew app with clock in/out
- Feature 2: Checklist and photo capture
- Feature 3: Azure infrastructure deployed

**Demo Notes:**
- [Demo to stakeholders/team]

**Feedback Received:**
-

---

## Links & References

- Product Roadmap: `product/roadmap.md`
- Technical Architecture: `technical/ARCHITECTURE-OVERVIEW.md`
- ADR-001: Tech Stack Selection
- ADR-002: Multi-Tenancy Strategy
- Q1 OKRs: `business/okr-2025-q1.md`

---

## Sprint 2 Preview

**Next Sprint Goal:** Build dispatcher board and client QA dashboard (the money features)

**Key Stories (preview):**
- Drag-drop shift assignment board
- No-show escalation workflow
- Client QA dashboard (who cleaned, when, photos)
- SMS notifications (Twilio integration)

**Dependencies from Sprint 1:**
- ✅ Backend API working (shifts, users)
- ✅ Authentication and multi-tenancy solid
- ✅ Mobile app foundation (for testing dispatcher flows)

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Geofencing accuracy issues | High | Medium | Test on multiple devices, allow manual override |
| Azure AD B2C setup complexity | High | Low | Follow Microsoft docs closely, allocate buffer time |
| Photo upload performance | Medium | Medium | Compress images, use SAS tokens for direct upload |
| Team capacity (1-2 devs) | High | Medium | Focus on P0 only, defer P1/P2 to Sprint 2 |
| Prisma migration issues | Medium | Low | Test migrations locally, backup database |

---

**Sprint Status:** 📋 Planning → 🏗️ In Progress (Week 1) → ✅ Complete (Week 4)
