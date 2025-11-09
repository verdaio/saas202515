# Architecture Overview - Cleaning & Janitorial SaaS

**Project:** saas202515 (Cleaning & Janitorial)
**Last Updated:** 2025-11-09
**Status:** Planning Phase
**Related ADRs:** ADR-001 (Tech Stack), ADR-002 (Multi-Tenancy)

---

## System Overview

Commercial janitorial management platform focused on QA verification and staffing control. Multi-tenant SaaS serving 10-100 person cleaning companies with mobile crew app, web dispatcher board, and client QA dashboard.

**Core User Flows:**
1. **Crew clocks in** → Geofence verifies location → Checklist loads → Photos captured → Clock out → Data synced
2. **Dispatcher assigns shifts** → Conflict detection → SMS alerts → No-show escalation → Real-time status
3. **Client views QA dashboard** → See who cleaned, when → Photo proof → Rate service → Report issues

---

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Azure Cloud                              │
│                                                                   │
│  ┌────────────────┐        ┌──────────────────────────────┐    │
│  │  Azure AD B2C  │        │   Azure Container Apps       │    │
│  │  (Auth/Tenant) │───────▶│  ┌────────────────────────┐  │    │
│  └────────────────┘        │  │  NestJS Backend API    │  │    │
│                            │  │  (GraphQL + REST)      │  │    │
│                            │  │  - Multi-tenant        │  │    │
│                            │  │  - Row-level security  │  │    │
│                            │  └────────────────────────┘  │    │
│                            └──────────────────────────────┘    │
│                                      │                          │
│                 ┌────────────────────┼────────────────────┐    │
│                 │                    │                     │    │
│      ┌──────────▼─────────┐  ┌──────▼──────────┐  ┌──────▼────┐
│      │  Azure Database    │  │  Azure Cache    │  │   Azure   │
│      │  for PostgreSQL    │  │  for Redis      │  │   Blob    │
│      │  (Multi-tenant)    │  │  (Sessions)     │  │  Storage  │
│      └────────────────────┘  └─────────────────┘  │  (Photos) │
│                                                    └───────────┘
│                                                                   │
│  ┌────────────────┐        ┌──────────────────────────────┐    │
│  │  Key Vault     │        │   Application Insights        │    │
│  │  (Secrets)     │        │   (Monitoring/Logs)           │    │
│  └────────────────┘        └──────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                │
                │ HTTPS (subdomain routing)
                │
    ┌───────────┴────────────────────────────────┐
    │                                             │
┌───▼────────────┐   ┌────────────────┐   ┌─────▼──────────┐
│  Mobile App    │   │  Web Dispatcher│   │  Client Portal │
│  (React Native)│   │  (React + Vite)│   │  (React + Vite)│
│                │   │                │   │                │
│  - Geofence    │   │  - Drag-drop   │   │  - QA Dashboard│
│  - Camera      │   │  - Real-time   │   │  - Photos      │
│  - Offline     │   │  - Scheduling  │   │  - Reports     │
│  - i18n (ES)   │   │                │   │                │
└────────────────┘   └────────────────┘   └────────────────┘
```

---

## Technology Stack Summary

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Backend** | NestJS (Node.js + TypeScript) | API server, business logic |
| **API** | GraphQL (Apollo) + REST | Flexible queries, real-time subscriptions |
| **Database** | PostgreSQL 16 | Relational data, multi-tenant |
| **Cache** | Redis | Sessions, real-time updates |
| **Auth** | Azure AD B2C | Multi-tenant authentication |
| **Storage** | Azure Blob Storage | Photo/video storage |
| **Hosting** | Azure Container Apps | Managed containers, auto-scale |
| **Frontend** | React 18 + TypeScript + Vite | Web applications |
| **Mobile** | React Native + Expo | Cross-platform mobile app |
| **ORM** | Prisma | Type-safe database access |
| **CI/CD** | GitHub Actions | Automated deployment |
| **IaC** | Bicep | Infrastructure as Code |
| **Monitoring** | Application Insights | Logs, metrics, alerts |

---

## Data Model (Core Entities)

```
┌──────────┐
│ Tenants  │  (Cleaning companies)
└────┬─────┘
     │
     ├─── Sites  (Cleaning locations)
     │     └─── SiteNotes  (Keys, codes, instructions)
     │
     ├─── Users  (Crews, managers)
     │     └─── Roles  (crew_lead, dispatcher, manager)
     │
     ├─── ChecklistTemplates  (Reusable checklists)
     │     └─── ChecklistItems
     │
     ├─── Shifts  (Scheduled work)
     │     ├─── Checklists  (Shift-specific checklist)
     │     │     └─── ChecklistItemCompletions
     │     ├─── Photos  (Proof-of-work)
     │     ├─── Timesheets  (Clock in/out records)
     │     └─── Exceptions  (No-shows, issues)
     │
     ├─── Invoices  (Billing)
     │     └─── InvoiceItems
     │
     └─── Notifications  (SMS/email log)
```

---

## Multi-Tenancy Architecture

**Strategy:** Shared database, shared schema with Row-Level Security (RLS)

### Tenant Isolation

```sql
-- Every tenant-scoped table has tenant_id
CREATE TABLE shifts (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  site_id UUID NOT NULL,
  scheduled_start TIMESTAMP NOT NULL,
  ...
);

-- RLS policy enforces isolation
ALTER TABLE shifts ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON shifts
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

### Tenant Resolution Flow

1. User visits `acme-cleaning.cleaningapp.com`
2. Backend extracts slug: `acme-cleaning`
3. Database lookup: `SELECT id FROM tenants WHERE slug = 'acme-cleaning'`
4. Set session: `SET app.current_tenant_id = <tenant_id>`
5. All queries automatically scoped to tenant via RLS

---

## Authentication & Authorization

### Azure AD B2C Setup

- **Separate B2C tenant per cleaning company** (e.g., acme-cleaning.b2clogin.com)
- **Subdomain mapped to B2C tenant** in configuration
- **JWT token includes claims:**
  - `tenant_id` - UUID of cleaning company
  - `user_id` - UUID of user
  - `role` - crew_lead, dispatcher, manager, client
  - `sub` - Azure AD B2C user identifier

### Authorization Model

| Role | Permissions |
|------|-------------|
| **crew_lead** | Clock in/out, view assigned shifts, complete checklists, upload photos |
| **dispatcher** | Assign shifts, view all crews, manage schedules, escalate exceptions |
| **manager** | Full access to tenant data, billing, analytics, site management |
| **client** | View QA dashboard for assigned sites, report issues, view invoices |

**Implementation:** Role-based access control (RBAC) in NestJS guards, using JWT role claim.

---

## API Design

### GraphQL Schema (Core Queries/Mutations)

```graphql
# Queries
type Query {
  # Crew app
  myShifts(date: Date!): [Shift!]!
  shiftDetails(id: UUID!): Shift!
  siteNotes(siteId: UUID!): [SiteNote!]!

  # Dispatcher
  shiftsForDate(date: Date!): [Shift!]!
  availableCrews(date: Date!): [User!]!
  exceptions(date: Date!): [Exception!]!

  # Client portal
  siteQADashboard(siteId: UUID!): QADashboard!
  recentShifts(siteId: UUID!, limit: Int): [Shift!]!

  # Analytics
  attendanceReport(startDate: Date!, endDate: Date!): AttendanceReport!
  exceptionTrends(days: Int!): ExceptionTrends!
}

# Mutations
type Mutation {
  # Crew app
  clockIn(shiftId: UUID!, location: LocationInput!): ClockInResult!
  clockOut(shiftId: UUID!, location: LocationInput!): ClockOutResult!
  completeChecklistItem(itemId: UUID!, photoUrl: String): ChecklistItemCompletion!

  # Dispatcher
  assignShift(shiftId: UUID!, userId: UUID!): Shift!
  createShift(input: CreateShiftInput!): Shift!
  escalateException(exceptionId: UUID!): Exception!

  # Admin
  createSite(input: CreateSiteInput!): Site!
  createChecklistTemplate(input: CreateChecklistTemplateInput!): ChecklistTemplate!
}

# Subscriptions (Real-time)
type Subscription {
  shiftUpdated(date: Date!): Shift!
  exceptionCreated: Exception!
  crewClockedIn(date: Date!): ClockInEvent!
}
```

### REST Endpoints (Integrations)

```
POST   /api/webhooks/stripe           - Stripe webhook for billing events
POST   /api/webhooks/twilio           - Twilio webhook for SMS status
GET    /api/export/quickbooks         - Export timesheets for QuickBooks
POST   /api/photos/upload              - Upload photo to Blob Storage (returns URL)
GET    /api/photos/:id/signed-url     - Get signed URL for photo download
```

---

## Mobile App Architecture

### React Native + Expo

**Key Features:**
- **Geofencing:** Expo Location API for clock in/out verification
- **Camera:** Expo Camera for photo capture
- **Offline:** Apollo Cache + Redux Persist for offline checklist completion
- **Push Notifications:** Expo Notifications for shift reminders
- **i18n:** react-i18next for English/Spanish toggle

### Offline Strategy

1. **Sync on app open:** Download today's shifts, checklists, site notes
2. **Queue actions offline:** Clock in, checklist completions, photos stored locally
3. **Background sync:** When connectivity restored, upload queued actions
4. **Conflict resolution:** Server timestamp wins (optimistic UI updates)

### Geofencing Implementation

```typescript
// Geofence radius: 100 meters
const GEOFENCE_RADIUS = 100; // meters

async function clockIn(shiftId: string) {
  const location = await Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.High
  });

  const distance = calculateDistance(
    location.coords,
    shift.site.coordinates
  );

  if (distance > GEOFENCE_RADIUS) {
    throw new Error('You are not at the site location');
  }

  // Proceed with clock in
}
```

---

## Real-Time Updates (Dispatcher Board)

### GraphQL Subscriptions

**Use case:** Dispatcher board shows live crew clock-ins, shift status changes

**Implementation:**
- WebSocket connection to backend (Apollo Server subscriptions)
- Redis Pub/Sub for message broadcasting across Container App instances
- Client subscribes to `shiftUpdated(date: today)`
- Server publishes event when crew clocks in/out

```typescript
// Client subscription
const { data } = useSubscription(SHIFT_UPDATED_SUBSCRIPTION, {
  variables: { date: today }
});

// Server resolver
const pubsub = new PubSub({ connection: redis });

pubsub.publish('SHIFT_UPDATED', {
  shiftUpdated: updatedShift
});
```

---

## Photo Storage & CDN

### Azure Blob Storage

**Strategy:**
- **Hot tier** for recent photos (< 30 days): Fast access, $0.018/GB/month
- **Cool tier** for older photos (30-90 days): Slower access, $0.01/GB/month
- **Archive tier** for ancient photos (> 90 days): Rare access, $0.002/GB/month

### Upload Flow

1. Mobile app requests signed upload URL from backend
2. Backend generates Blob Storage SAS token (15 min expiry)
3. Mobile app uploads directly to Blob Storage (no backend bottleneck)
4. Mobile app calls GraphQL mutation with photo URL
5. Backend saves photo URL to database

**Cost estimate:** 10 photos/shift × 100 shifts/day × 365 days = 365k photos/year
365k photos × 2MB/photo = 730 GB/year = $13.14/month (Hot tier) + $7.30/month (Cool tier)

---

## Integrations

### Stripe (Billing)

**Subscription Model:**
- **Base subscription:** $149/month (commercial) or $99/month (residential)
- **Per-user metering:** $12/active cleaner/month
- **Add-on:** $49/site/month for client QA dashboard access

**Implementation:**
- Stripe Customer created per tenant
- Subscription with metered billing (active cleaners counted daily)
- Webhook handles `invoice.paid`, `customer.subscription.deleted` events

### Twilio (SMS)

**Use cases:**
- No-show alerts to dispatchers
- Shift reminders to crews
- Client notifications (QA reports ready)

**Cost:** $0.0075/SMS, estimated 1000 SMS/month = $7.50/month for 20 tenants

### QuickBooks Online (Accounting)

**Export functionality:**
- Timesheets (shift start/end, crew member, hours)
- Invoices (monthly billing per site)
- API uses OAuth 2.0 for tenant-specific authorization

---

## Deployment Architecture

### Azure Container Apps

**Configuration:**
- **2 Container Apps:** Backend API, Web frontend
- **Auto-scaling:** 1-10 instances based on HTTP requests
- **Custom domain:** `*.cleaningapp.com` with wildcard SSL
- **Environment variables:** Stored in Key Vault, injected at runtime

### CI/CD Pipeline (GitHub Actions)

```yaml
name: Deploy Backend

on:
  push:
    branches: [main]
    paths: [backend/**]

jobs:
  deploy:
    - Build Docker image
    - Push to Azure Container Registry
    - Deploy to Azure Container Apps (staging)
    - Run smoke tests
    - Deploy to production (manual approval)
```

---

## Monitoring & Observability

### Application Insights

**Metrics tracked:**
- Request latency (P50, P95, P99)
- Error rate by endpoint
- Database query performance
- Photo upload success rate
- Geofencing accuracy (distance from site)

**Custom Events:**
- `crew_clocked_in` - Track geofence distance
- `shift_completed` - Track completion time vs. scheduled
- `exception_created` - Track exception types
- `client_viewed_qa_dashboard` - Track client engagement

### Alerts

- Error rate > 5% for 5 minutes
- Database connection pool exhausted
- Photo upload failure rate > 10%
- Container App unhealthy (restart)

---

## Security Measures

### Application Security

- **JWT validation:** Every request validates token signature and expiry
- **Row-level security:** PostgreSQL RLS prevents cross-tenant data access
- **Rate limiting:** 100 requests/minute per tenant
- **Input validation:** All GraphQL inputs validated with Joi schemas
- **SQL injection:** Prisma ORM prevents SQL injection
- **XSS protection:** React escapes output by default

### Data Security

- **Encryption at rest:** Azure Blob Storage, PostgreSQL encrypted
- **Encryption in transit:** TLS 1.3 for all connections
- **Secrets management:** Azure Key Vault (never in code/env files)
- **Photo access:** SAS tokens with 15-minute expiry (no public URLs)

### Compliance Prep

- **GDPR:** Tenant data deletion API, data export API
- **SOC 2:** Audit logs for data access, change tracking
- **HIPAA (future):** PHI data flagging, additional encryption

---

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Mobile app startup | < 2 seconds | Time to first render |
| Clock in/out | < 3 seconds | Button press to confirmation |
| Photo upload | < 5 seconds | Capture to uploaded |
| Dispatcher board load | < 1 second | Page load with 100 shifts |
| GraphQL query P95 | < 200ms | Application Insights |
| Database query P95 | < 50ms | PostgreSQL logs |

---

## Scalability Plan

### Current Capacity (MVP)

- **Database:** PostgreSQL Basic tier (2 vCores, 50 GB) = 100 TPS
- **Backend:** Container Apps (1-5 instances, 0.5 vCPU each) = 500 req/s
- **Tenants:** 50-100 tenants comfortably

### Scale Triggers

| Metric | Action |
|--------|--------|
| Database CPU > 80% | Upgrade to General Purpose tier (4 vCores) |
| Container App instances maxed | Increase max replicas to 20 |
| Tenant count > 500 | Consider sharding by region |
| Large tenant (100+ cleaners) | Move to dedicated database (ADR-002) |

---

## Cost Estimates

### MVP Phase (10-20 beta accounts)

| Service | Tier | Cost/Month |
|---------|------|------------|
| Container Apps (backend) | 1-3 instances | $60 |
| Container Apps (frontend) | 1-2 instances | $40 |
| PostgreSQL | Basic (2 vCores, 50 GB) | $25 |
| Redis | Basic (250 MB) | $15 |
| Blob Storage | Hot tier (100 GB) | $2 |
| Application Insights | Basic (5 GB/month) | $10 |
| Key Vault | Standard | $3 |
| Azure AD B2C | 10k MAU (free tier) | $0 |
| Twilio SMS | 1000 SMS | $7.50 |
| Domain + SSL | *.cleaningapp.com | $5 |
| **Total** | | **$167.50/month** |

**Note:** Azure for Startups provides $1,000/month credits for Year 1

### Year 1 (50 accounts)

- Estimated cost: $400-600/month
- Revenue: $250 ARPU × 50 = $12,500/month
- Gross margin: ~95%

---

## Disaster Recovery

### Backup Strategy

- **Database:** Automated daily backups (7-day retention)
- **Blob Storage:** Geo-redundant storage (GRS) - automatic replication to secondary region
- **Configuration:** Infrastructure as Code (Bicep) in Git

### RTO/RPO Targets

- **RPO (Recovery Point Objective):** 1 hour (last database backup)
- **RTO (Recovery Time Objective):** 4 hours (restore from backup + redeploy)

### Failure Scenarios

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Container App down | API unavailable | Auto-restart + health checks (5 min RTO) |
| Database unavailable | Read-only mode | Failover to read replica (15 min RTO) |
| Blob Storage outage | Photos unavailable | Geo-replication (automatic failover) |
| Region outage | Full outage | Manual failover to secondary region (4 hr RTO) |

---

## Future Architecture Considerations

### Year 2 Enhancements

- **Multi-region deployment:** Secondary region for HA
- **CDN for photos:** Azure Front Door for faster photo loads
- **Data warehouse:** Azure Synapse for advanced analytics
- **API gateway:** Azure API Management for rate limiting, caching

### Year 3 Scalability

- **Database sharding:** Shard by tenant geography
- **Microservices:** Split monolith into services (billing, notifications, analytics)
- **Event-driven:** Azure Event Hubs for async processing
- **ML/AI:** Azure Machine Learning for predictive staffing

---

## Related Documents

- ADR-001: Tech Stack Selection
- ADR-002: Multi-Tenancy Strategy
- Product Roadmap (product/roadmap.md)
- Database Schema (technical/database-schema.md) - TBD
- API Reference (technical/api-reference.md) - TBD
- Deployment Runbook (workflows/deployment-runbook.md) - TBD

---

**Last Updated:** 2025-11-09
**Next Review:** After Sprint 1 (Week 4)
