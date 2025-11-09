# Product Roadmap - Cleaning & Janitorial SaaS

**Period:** 2025 (Year 1)
**Trade Name:** Cleaning & Janitorial
**Owner:** Product Team
**Last Updated:** 2025-11-09
**Status:** Draft

---

## Vision & Strategy

### Product Vision

Build the **industry-standard QA platform** for commercial janitorial services by making quality assurance visible, verifiable, and automatic. We protect cleaning companies from chargebacks and contract loss by providing proof-of-work through photos, geofenced time tracking, and real-time client dashboards.

**Year 1 Goal:** 50 paying accounts averaging $250/month ARPU, proving 30% fewer exceptions and +10% crew utilization.

### Strategic Themes for Year 1

1. **Depth in QA/Inspections** - Best-in-class proof-of-work and client visibility
2. **Mobile-first UX** - Dead-simple crew app (< 5 taps per checklist)
3. **Fast ROI** - "Save two hours a day" promise with < 4 month payback
4. **Stickiness** - Become part of client contracts via QA dashboard

---

## Roadmap Overview

### Now (Weeks 1-4 | Sprint 1) - Foundation

**Focus:** Core infrastructure + Mobile crew app foundation

| Feature/Initiative | Status | Owner | Target Date | Priority |
|--------------------|--------|-------|-------------|----------|
| Project setup (Azure, repos, CI/CD) | Not Started | Eng | Week 1 | P0 |
| Database schema (sites, shifts, checklists) | Not Started | Eng | Week 2 | P0 |
| Authentication & multi-tenant setup | Not Started | Eng | Week 2 | P0 |
| Mobile crew app - Clock in/out (geofence) | Not Started | Eng | Week 3-4 | P0 |
| Mobile crew app - Checklist UI | Not Started | Eng | Week 3-4 | P0 |
| Photo capture & upload | Not Started | Eng | Week 4 | P0 |

### Next (Weeks 5-8 | Sprint 2) - Dispatcher & Client Portals

**Focus:** Dispatcher board + Client QA dashboard (the money features)

| Feature/Initiative | Description | Strategic Theme | Target Date | Priority |
|--------------------|-------------|-----------------|-------------|----------|
| Dispatcher board | Drag-drop shifts, conflict detection | Mobile-first UX | Week 5-6 | P0 |
| No-show escalation workflow | Automated alerts to supervisors | Fast ROI | Week 6 | P0 |
| Client QA dashboard | Who cleaned, when, photos | Depth in QA | Week 7-8 | P0 |
| SMS notifications | Twilio integration for alerts | Fast ROI | Week 7 | P0 |
| Site notes management | Keys/codes, special instructions | Mobile-first UX | Week 8 | P1 |

### Later (Weeks 9-12 | Sprint 3) - Billing & Analytics

**Focus:** Complete MVP with billing, integrations, analytics

| Feature/Initiative | Description | Strategic Theme | Target Date | Priority |
|--------------------|-------------|-----------------|-------------|----------|
| Stripe billing | Recurring + tips/surcharges | Fast ROI | Week 9 | P0 |
| QuickBooks Online sync | Export timesheets, invoices | Fast ROI | Week 10 | P0 |
| Analytics dashboard | Attendance, exceptions, revenue/hour | Depth in QA | Week 11 | P0 |
| QR station scanning | Proof-of-presence via QR codes | Depth in QA | Week 11 | P1 |
| Language toggle (Spanish) | Crew app i18n | Mobile-first UX | Week 12 | P0 |
| Beta onboarding flow | Migration kit, on-site setup | Stickiness | Week 12 | P0 |

### Future/Backlog (Months 4-12+)

**Q2 2025 (Months 4-6):**
- Exception trend analytics (data exhaust moat)
- WhatsApp Business integration
- Advanced route optimization
- Vertical template: Medical facilities

**Q3-Q4 2025 (Months 7-12):**
- Supply tracking and re-order flow
- Residential rescheduling module (secondary wedge)
- Client rating system
- API for property management integrations
- Predictive staffing recommendations

**Year 2+ Ideas:**
- Industry benchmarking dashboard
- Training/certification program
- Marketplace for cleaning supplies
- AI-powered QA anomaly detection

---

## Detailed Feature Breakdown

### Mobile Crew App - Clock In/Out (Sprint 1)

**Problem:** Time disputes between crews and managers; no proof of when work started/ended
**Solution:** Geofenced clock in/out with automatic location detection (< 30 seconds to clock in)
**Impact:** Eliminates time disputes, proves crews were on-site
**Effort:** Medium (geofencing library, permissions handling)
**Dependencies:** Database schema, authentication
**Status:** Not Started
**PRD:** TBD

---

### Mobile Crew App - Checklists (Sprint 1)

**Problem:** Crews don't know exactly what to clean; no standard process
**Solution:** Task-by-task checklists with photo capture for each area (< 5 taps to complete)
**Impact:** Clear expectations, proof of work completion
**Effort:** Medium (mobile UI, photo handling, offline support)
**Dependencies:** Photo upload infrastructure
**Status:** Not Started
**PRD:** TBD

---

### Dispatcher Board (Sprint 2)

**Problem:** Manual scheduling in Excel/paper; 2+ hours/day wasted on logistics
**Solution:** Drag-drop shift assignment with conflict detection and auto-alerts
**Impact:** Saves 2+ hours/day (core ROI promise)
**Effort:** Large (drag-drop UI, real-time updates, conflict logic)
**Dependencies:** Shift schema, crew availability data
**Status:** Not Started
**PRD:** TBD

---

### Client QA Dashboard (Sprint 2) - MONEY FEATURE

**Problem:** Clients can't verify work was done; chargebacks and contract loss
**Solution:** Real-time dashboard showing who cleaned, when, pass/fail, photo proof
**Impact:** Reduces chargebacks 30%+, becomes contract requirement (sticky!)
**Effort:** Medium (dashboard UI, photo gallery, filtering)
**Dependencies:** Completed shifts data, photo storage
**Status:** Not Started
**PRD:** TBD
**Pricing:** +$49/site/month per client

---

### No-Show Escalation (Sprint 2)

**Problem:** Crew call-outs discovered too late; sites go uncleaned
**Solution:** Automated SMS alerts when crew doesn't clock in within 15 min of shift start
**Impact:** Faster response to exceptions, reduces missed services
**Effort:** Small (Twilio integration, cron job)
**Dependencies:** SMS notification system
**Status:** Not Started
**PRD:** TBD

---

### Stripe Billing (Sprint 3)

**Problem:** Manual invoicing is time-consuming and error-prone
**Solution:** Automated recurring billing based on active cleaners + client QA seats + tips
**Impact:** Reduces admin overhead, enables self-service pricing
**Effort:** Medium (Stripe API, subscription management, proration)
**Dependencies:** User account setup, pricing tiers
**Status:** Not Started
**PRD:** TBD

---

### QuickBooks Online Sync (Sprint 3)

**Problem:** Manual export of timesheets and invoices for accounting
**Solution:** One-click export of shift data, timesheets, invoices to QuickBooks
**Impact:** Saves 1-2 hours/week on accounting, increases adoption
**Effort:** Medium (QuickBooks API, OAuth, field mapping)
**Dependencies:** Completed shifts data, timesheet calculation
**Status:** Not Started
**PRD:** TBD

---

### Analytics Dashboard (Sprint 3)

**Problem:** No visibility into crew performance, exception trends, revenue metrics
**Solution:** Dashboard showing attendance rate, completion rate, exceptions, revenue/hour
**Impact:** Data-driven operations decisions, proves ROI
**Effort:** Medium (data aggregation, charting library, filters)
**Dependencies:** Historical shift data (3+ weeks)
**Status:** Not Started
**PRD:** TBD

---

### QR Station Scanning (Sprint 3)

**Problem:** Need extra proof crews visited specific areas within sites
**Solution:** Print QR codes per station/area; crews scan to prove presence
**Impact:** Additional verification for high-value/compliance sites
**Effort:** Small (QR generation, camera scanning)
**Dependencies:** Mobile camera, checklist association
**Status:** Not Started
**PRD:** TBD

---

### Language Toggle - Spanish (Sprint 3)

**Problem:** Many crews are ESL (Spanish primary); English-only reduces adoption
**Solution:** Full Spanish translation toggle in mobile app
**Impact:** Eliminates language barrier, critical for adoption
**Effort:** Medium (i18n setup, translation, testing)
**Dependencies:** Mobile app UI complete
**Status:** Not Started
**PRD:** TBD

---

## Success Metrics

### Key Results for MVP (Weeks 1-12)

| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Beta accounts signed | 0 | 10 | 🟡 Not Started |
| MVP delivered | 0% | 100% | 🟡 Sprint 1 |
| Exception rate reduction | - | 30% | 🟡 To measure |
| Dispatcher time savings | - | 2+ hrs/day | 🟡 To measure |
| Payback period | - | < 4 months | 🟡 To prove |

### Key Results for Post-MVP (Months 4-12)

| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Paying accounts | 0 | 50 | 🟡 On Track |
| ARPU | $0 | $250 | 🟡 TBD |
| Churn rate | - | < 5%/mo | 🟡 To track |
| NPS score | - | 60+ | 🟡 To survey |
| CAC payback | - | < 4 months | 🟡 To optimize |

---

## Resource Allocation

### Team Capacity
- **Engineering:** 1-2 developers
- **Design:** 0.5 designer (templates/Figma)
- **Product:** Solo founder/PM

### Effort Distribution (MVP Phase)
- 70% - MVP features (P0)
- 15% - Infrastructure (Azure, CI/CD, security)
- 10% - Bug fixes / Testing
- 5% - Beta onboarding prep

### Effort Distribution (Post-MVP)
- 50% - New features (roadmap)
- 20% - Beta feedback / iterations
- 15% - Bug fixes / Performance
- 10% - Sales/onboarding support
- 5% - Technical debt

---

## Risks and Dependencies

| Risk/Dependency | Impact | Mitigation | Owner |
|-----------------|--------|------------|-------|
| Geofencing accuracy issues | High | Test multiple libraries; allow manual override | Eng |
| Crew adoption (won't use app) | High | Simple UX, Spanish support, on-site training | Product |
| Photo storage costs scale fast | Medium | Compress images, expire old photos (90 days) | Eng |
| QuickBooks API complexity | Medium | Start with CSV export fallback | Eng |
| Beta account recruitment | High | Partner with suppliers/franchises early | Business |
| SMS costs exceed budget | Medium | SMS-only for critical alerts, email optional | Product |
| Azure costs exceed estimate | Medium | Set up cost alerts, optimize resources | Eng |

---

## What We're NOT Doing

It's important to be explicit about what we're deprioritizing:

### NOT in MVP (Weeks 1-12):
- ❌ **Residential rescheduling** - Focus on commercial first (lower churn)
- ❌ **Supply tracking** - Not core to QA wedge, add in Q2
- ❌ **Advanced routing/optimization** - Manual dispatch OK for beta
- ❌ **Mobile app for clients** - Web dashboard sufficient initially
- ❌ **Custom reports/exports** - Standard analytics only for MVP
- ❌ **Multiple roles/permissions** - Single owner/manager role OK
- ❌ **Crew performance scores** - Focus on QA, not crew ratings (v1)

### NOT in Year 1:
- ❌ **Marketplace/supply ordering** - Too complex, focus on SaaS
- ❌ **Training/certification** - Nice-to-have, not monetizable yet
- ❌ **Payroll integration** - QuickBooks export sufficient
- ❌ **Native iOS app** - React Native/PWA OK for MVP
- ❌ **Video proof-of-work** - Photos sufficient, video = storage costs

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-11-09 | Prioritize commercial over residential | Lower churn, clearer ROI, better wedge per brief |
| 2025-11-09 | Target 8-12 weeks for MVP | Per brief recommendation (3 sprints) |
| 2025-11-09 | Client QA dashboard as paid add-on | $49/site creates high-margin upsell, proves value |
| 2025-11-09 | Spanish language support in MVP | Critical for crew adoption (ESL users) |
| 2025-11-09 | SMS-first notifications | Crews more responsive to SMS than email |
| 2025-11-09 | QuickBooks over other accounting | Most common in SMB janitorial space |

---

## Phased Beta Rollout Plan

### Phase 1: Friends & Family (Accounts 1-5)
- **Timeline:** Weeks 9-10
- **Goal:** Test core workflows, find critical bugs
- **Profile:** Friendly accounts, willing to give feedback
- **Features:** Core MVP only (crew app, dispatcher, basic QA)

### Phase 2: Pilot Beta (Accounts 6-20)
- **Timeline:** Weeks 11-16
- **Goal:** Prove ROI metrics (30% exceptions, 2hr savings)
- **Profile:** Target size (10-50 cleaners), multi-site
- **Features:** Full MVP including billing, analytics, QR codes

### Phase 3: Expansion (Accounts 21-50)
- **Timeline:** Months 4-6
- **Goal:** Refine product, establish unit economics
- **Profile:** Referred accounts, 1-2 additional cities
- **Features:** MVP + exception trends + vertical templates

---

## Feedback and Questions

**For stakeholders/team:**
1. Is 8-12 weeks realistic for MVP given team size?
2. Should we delay QR codes to post-MVP?
3. Are we prioritizing the right P0 features?
4. Should client QA dashboard be included in base pricing vs. add-on?

---

## Revision History

| Date | Changes | Updated By |
|------|---------|------------|
| 2025-11-09 | Initial roadmap based on project brief | Product Team |

---

## Next Steps

1. **Validate roadmap with team** - Review effort estimates
2. **Create Sprint 1 plan** - Break down Weeks 1-4 into user stories
3. **Recruit beta accounts** - Start outreach to 20 target accounts
4. **Set up project infrastructure** - Azure resources, GitHub repo
5. **Create ADR for tech stack** - Document key architectural decisions
