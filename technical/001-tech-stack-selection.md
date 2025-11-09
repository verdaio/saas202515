# ADR-001: Tech Stack Selection for Cleaning & Janitorial SaaS

**Date:** 2025-11-09
**Status:** Proposed
**Deciders:** Product Team, Engineering Lead
**Technical Story:** MVP Phase (Sprints 1-3)

---

## Context

We need to select a technology stack for our commercial janitorial management SaaS platform that will support:

1. **Mobile crew app** - Geofencing, photo capture, offline support, language toggle
2. **Web dispatcher board** - Drag-drop UI, real-time updates
3. **Client QA dashboard** - Photo galleries, real-time data
4. **Multi-tenant architecture** - Subdomain-based tenant isolation
5. **Integrations** - Stripe, QuickBooks, Twilio (SMS)
6. **Azure deployment** - Azure-native services preferred

**Key Constraints:**
- 8-12 week MVP timeline (aggressive)
- Small team (1-2 developers)
- Budget-conscious (startup)
- Must scale to 50+ accounts in Year 1
- Mobile UX critical for adoption (crews are ESL, low tech-savviness)

**Forces at Play:**
- Speed to market vs. technical debt
- Native mobile performance vs. development velocity
- Azure-native vs. best-of-breed tools
- Cost optimization vs. feature richness

---

## Decision

We will use the following technology stack:

### Backend
- **Runtime:** Node.js 20 LTS (TypeScript)
- **Framework:** NestJS
- **API:** GraphQL with Apollo Server
- **Authentication:** Azure AD B2C (multi-tenant support)

### Frontend (Web Apps)
- **Framework:** React 18 with TypeScript
- **State Management:** Apollo Client (GraphQL cache) + React Context
- **UI Library:** Tailwind CSS + shadcn/ui components
- **Build Tool:** Vite

### Mobile App
- **Framework:** React Native with Expo
- **Navigation:** React Navigation
- **Offline:** Redux Persist + Apollo Cache
- **Maps/Geofencing:** Expo Location + react-native-maps

### Database
- **Primary:** PostgreSQL 16 (Azure Database for PostgreSQL)
- **Schema:** Multi-tenant (tenant_id on all tables)
- **Migrations:** Prisma ORM
- **Cache:** Azure Cache for Redis

### Storage
- **Photos:** Azure Blob Storage (Hot tier)
- **Documents:** Azure Blob Storage (Cool tier)

### Infrastructure
- **Hosting:** Azure Container Apps
- **CI/CD:** GitHub Actions
- **IaC:** Bicep templates
- **Monitoring:** Azure Application Insights
- **Secrets:** Azure Key Vault

### Integrations
- **Payments:** Stripe API (subscriptions)
- **SMS:** Twilio API
- **Accounting:** QuickBooks Online API
- **Analytics:** Custom (PostgreSQL aggregations)

---

## Consequences

### Positive Consequences

**Development Velocity:**
- React Native + Expo allows shared code with web apps (60-70% code reuse)
- TypeScript across full stack reduces bugs and improves maintainability
- Prisma ORM speeds up database development with type-safe queries
- NestJS provides structure and convention (reduces decision fatigue)
- GraphQL allows frontend teams to iterate without backend changes

**Mobile UX:**
- React Native provides near-native performance (critical for crew adoption)
- Expo simplifies geofencing, camera, and offline support (no native code needed initially)
- Fast refresh speeds up mobile development
- Over-the-air updates (Expo) allow rapid bug fixes without app store approval

**Azure Integration:**
- Azure Container Apps provides managed Kubernetes-like experience (low ops burden)
- Azure AD B2C solves multi-tenant auth out-of-box (subdomain routing)
- Blob Storage is cost-effective for photos ($0.018/GB/month)
- Application Insights provides excellent observability

**Scalability:**
- PostgreSQL scales to 1000s of tenants with proper indexing
- Redis cache handles real-time dispatcher board updates
- Container Apps auto-scales based on load
- GraphQL reduces over-fetching (lower bandwidth costs for mobile)

**Budget:**
- Open-source stack (no licensing costs)
- Azure for Startups credits available ($1,000/month free)
- Expo is free for < 1GB OTA updates/month
- Estimated cost: $200-500/month for MVP phase

### Negative Consequences

**Technical Debt:**
- React Native may require native modules later (e.g., advanced geofencing, background tasks)
- GraphQL complexity for simple CRUD operations (learning curve)
- Multi-tenant schema requires careful migration planning (can't easily separate tenants)

**Vendor Lock-in:**
- Azure-specific services (AD B2C, Container Apps) make migration harder
- Stripe is hard to replace once integrated (though industry standard)

**Performance Trade-offs:**
- React Native slower than native Swift/Kotlin (acceptable trade-off for velocity)
- GraphQL adds overhead vs. REST for simple queries

### Neutral Consequences

- Team must learn NestJS and GraphQL (1-2 week ramp-up)
- Prisma migrations require discipline (can't manually edit database)
- TypeScript compilation adds build time (negligible with Vite)

---

## Alternatives Considered

### Alternative 1: Ruby on Rails + Native Mobile

**Description:** Ruby on Rails backend with separate native iOS (Swift) and Android (Kotlin) apps

**Pros:**
- Rails is extremely productive for CRUD operations
- Native mobile apps have best performance and UX
- Rails conventions reduce decision fatigue

**Cons:**
- Need 3 developers (Rails, iOS, Android) vs. 1-2 with React Native
- Slower mobile iteration (need to update 2 codebases)
- App store review delays (vs. Expo OTA updates)
- Not Azure-native (would deploy to Heroku or generic VMs)

**Why rejected:** Too slow for 8-12 week MVP with small team; Ruby not as strong in Azure ecosystem

---

### Alternative 2: .NET Core + Blazor + MAUI

**Description:** Microsoft stack - .NET 8 backend, Blazor for web, .NET MAUI for mobile

**Pros:**
- Fully Azure-native (excellent integration)
- C# across full stack (strong typing)
- MAUI is true native compilation
- Enterprise-grade tooling (Visual Studio)

**Cons:**
- MAUI is less mature than React Native (more bugs, smaller community)
- Blazor WebAssembly has slow initial load time (critical for crew app UX)
- Smaller talent pool (harder to hire)
- Team has stronger JavaScript/TypeScript experience

**Why rejected:** MAUI immaturity is risky for mobile-first product; team expertise in TypeScript/React

---

### Alternative 3: Python (Django/FastAPI) + Flutter

**Description:** Python backend (Django or FastAPI) with Flutter for cross-platform mobile

**Pros:**
- Python excellent for data processing / analytics (exception trends, benchmarking)
- Django admin provides instant admin UI
- Flutter has excellent performance and UI flexibility

**Cons:**
- Flutter uses Dart (less transferable skill than JavaScript)
- Smaller web component ecosystem vs. React
- Python async less mature than Node.js for real-time features
- Team has stronger Node.js experience

**Why rejected:** Dart learning curve + less web ecosystem; Python async complexity

---

### Alternative 4: Serverless (AWS Lambda + DynamoDB + React Native)

**Description:** Serverless Functions (Lambda/Azure Functions) + NoSQL database + React Native

**Pros:**
- Ultimate scalability (infinite scale, pay-per-use)
- Very low cost at low volume
- Forces stateless architecture

**Cons:**
- Cold starts hurt UX (500-1000ms latency for first request)
- DynamoDB/CosmosDB requires denormalization (complex queries harder)
- Multi-tenant isolation more complex
- Debugging/local development harder
- Not cost-effective until 100k+ requests/month

**Why rejected:** Cold starts unacceptable for real-time dispatcher board; SQL better fit for relational data (shifts, checklists, sites)

---

## References

- [React Native vs Native Performance](https://medium.com/@the.nickweb/react-native-vs-native-performance-comparison-2024-8f0d4e5c1e7a)
- [Azure Container Apps Pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/)
- [Multi-tenancy with Prisma](https://www.prisma.io/docs/guides/deployment/multi-tenancy)
- [Expo Geofencing Guide](https://docs.expo.dev/versions/latest/sdk/location/)
- [GraphQL vs REST for Mobile](https://www.apollographql.com/blog/graphql/basics/why-use-graphql/)

---

## Notes

### Implementation Notes

**Phase 1 (Sprint 1):**
- Set up monorepo (Nx or Turborepo) with backend, web, mobile apps
- Configure Prisma with multi-tenant schema
- Set up Azure Container Apps deployment
- Implement Azure AD B2C authentication

**Phase 2 (Sprint 2-3):**
- Build React Native crew app with Expo
- Implement geofencing with background location tracking
- Set up Apollo GraphQL subscriptions for real-time updates
- Configure Blob Storage for photo uploads

**Phase 3 (Post-MVP):**
- Evaluate if native modules needed (advanced geofencing, background tasks)
- Consider ejecting from Expo if needed
- Optimize database queries with proper indexes
- Implement caching strategy with Redis

### Security Considerations

- Row-level security (RLS) in PostgreSQL for tenant isolation
- JWT tokens with tenant_id claim for API authorization
- Blob Storage SAS tokens for photo uploads (no public access)
- Rate limiting on API (prevent abuse)

### Cost Estimates (MVP Phase, 10-20 beta accounts)

| Service | Cost/Month |
|---------|------------|
| Azure Container Apps (2 apps) | $100 |
| PostgreSQL (Basic tier) | $25 |
| Redis (Basic tier) | $15 |
| Blob Storage (1TB) | $20 |
| Azure AD B2C (10k MAU free) | $0 |
| Application Insights | $10 |
| Twilio SMS (1000 SMS) | $7.50 |
| Domain + SSL | $5 |
| **Total** | **$182.50/month** |

**Note:** Azure for Startups provides $1,000/month credits for first year

---

## Follow-Up Items

- [ ] Create Prisma schema with multi-tenant support
- [ ] Set up GitHub Actions CI/CD pipeline
- [ ] Configure Azure Container Apps with Bicep
- [ ] Evaluate GraphQL schema design (schema-first vs. code-first)
- [ ] Research background task handling for geofencing (iOS/Android limitations)
- [ ] Create ADR-002 for database schema multi-tenancy approach

---

## Superseded By

[Not superseded]

---

## Template Information

This ADR follows the format described in [Michael Nygard's article](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).
