# ADR-002: Multi-Tenancy Strategy

**Date:** 2025-11-09
**Status:** Proposed
**Deciders:** Product Team, Engineering Lead
**Technical Story:** Foundation Sprint (Sprint 1)
**Related:** ADR-001 (Tech Stack Selection)

---

## Context

Our Cleaning & Janitorial SaaS needs to support multiple cleaning companies (tenants) on a shared infrastructure while ensuring complete data isolation and supporting subdomain-based routing (e.g., `acme-cleaning.cleaningapp.com`).

**Requirements:**
1. **Complete data isolation** - Tenant A cannot access Tenant B's data (critical for security/compliance)
2. **Subdomain routing** - Each tenant has `{company-slug}.cleaningapp.com` URL
3. **Cost-effective scaling** - Shared infrastructure for small tenants, ability to scale large tenants separately
4. **Simple tenant provisioning** - Create new tenant in < 5 minutes (automated)
5. **Query performance** - No N+1 queries, efficient filtering

**Tenant Characteristics:**
- 10-100 cleaners per tenant
- 5-50 sites per tenant
- 100-500 shifts/month per tenant
- Year 1: 50 tenants
- Year 3: 500 tenants
- Year 5: 2,000 tenants

**Forces at Play:**
- Strong isolation vs. cost efficiency
- Query simplicity vs. performance
- Shared schema vs. database-per-tenant
- Developer experience vs. operational complexity

---

## Decision

We will implement a **shared database, shared schema** multi-tenancy model with the following characteristics:

### Database Architecture
- **Single PostgreSQL database** with `tenant_id` column on all tenant-scoped tables
- **Row-level security (RLS)** policies to enforce tenant isolation at database level
- **Tenant resolution** via subdomain in middleware (extracted from Host header)
- **Connection pooling** with Prisma to share database connections across tenants

### Schema Design
```sql
-- Core tenant table
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug VARCHAR(50) UNIQUE NOT NULL,  -- for subdomain
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  status VARCHAR(20) NOT NULL DEFAULT 'active'  -- active, suspended, churned
);

-- Tenant-scoped table example
CREATE TABLE sites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  address TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  CONSTRAINT fk_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- Index for fast tenant-scoped queries
CREATE INDEX idx_sites_tenant_id ON sites(tenant_id);

-- Row-level security policy
ALTER TABLE sites ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON sites
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

### Application-Level Enforcement
1. **Middleware extracts tenant** from subdomain: `acme-cleaning.cleaningapp.com` → tenant_id lookup
2. **Set session variable** in PostgreSQL: `SET app.current_tenant_id = <tenant_id>`
3. **RLS automatically filters** all queries to that tenant
4. **Prisma middleware** adds `tenant_id` to all queries as additional safety layer

### Authentication Flow
1. User visits `acme-cleaning.cleaningapp.com`
2. Azure AD B2C authenticates user (separate B2C tenant per customer)
3. JWT token includes `tenant_id` claim
4. Backend validates JWT and sets `app.current_tenant_id` for database session
5. All queries automatically scoped to tenant via RLS

---

## Consequences

### Positive Consequences

**Cost Efficiency:**
- Shared database infrastructure (single PostgreSQL instance serves all tenants)
- Shared connection pool (fewer connections = lower resource usage)
- Estimated cost: $25/month for 50 tenants vs. $25/tenant with separate databases

**Simple Operations:**
- Single database to backup/restore
- Single schema migration (apply to all tenants at once)
- Easy to aggregate cross-tenant metrics (admin dashboard)

**Developer Experience:**
- Simple Prisma queries (RLS handles filtering automatically)
- No need to manually add `WHERE tenant_id = ?` to every query
- Easier testing (seed single database)

**Security:**
- Database-level isolation via RLS (application bugs can't leak data across tenants)
- Explicit tenant_id in JWT and session (multiple layers of defense)
- Foreign key constraints ensure data integrity

**Performance:**
- Fast queries with proper indexes on (tenant_id, ...)
- Shared cache in Redis (tenant-scoped cache keys)
- No cross-database joins needed

### Negative Consequences

**Noisy Neighbor Risk:**
- Large tenant's queries can slow down small tenants (mitigation: query timeouts, monitoring)
- Need to monitor per-tenant query patterns and optimize

**Limited Isolation:**
- All tenants share same database instance (security concern if database compromised)
- Can't easily move single tenant to separate instance (requires data migration)

**Schema Changes:**
- Single schema change affects all tenants (must be backward-compatible)
- Can't customize schema per tenant (e.g., custom fields)

**Scale Ceiling:**
- Single database has upper limit (~500-1000 tenants before performance degrades)
- Will need to shard or move large tenants eventually

### Neutral Consequences

- RLS adds small query overhead (~1-2ms per query)
- Requires PostgreSQL (can't use simpler databases like SQLite in dev)
- Must be disciplined about tenant_id in all queries (Prisma helps)

---

## Alternatives Considered

### Alternative 1: Database-per-Tenant

**Description:** Each tenant gets their own PostgreSQL database

**Pros:**
- Strongest isolation (complete database separation)
- Can customize schema per tenant
- Easy to move tenant to different server
- Better for compliance (HIPAA, SOC2)
- Noisy neighbor eliminated

**Cons:**
- High cost (50 tenants = 50 databases = $1,250/month vs. $25)
- Complex operations (50 backups, 50 schema migrations)
- Connection pool exhaustion (each tenant needs connections)
- Can't easily aggregate cross-tenant metrics
- Overhead for small tenants (10-user tenant gets full database)

**Why rejected:** Too expensive for early-stage SaaS with small tenants; operational complexity not justified

---

### Alternative 2: Schema-per-Tenant (PostgreSQL Schemas)

**Description:** Single database, but each tenant gets a separate PostgreSQL schema

**Pros:**
- Good isolation (better than shared schema)
- Shared database instance (lower cost than database-per-tenant)
- Can customize schema per tenant
- Easier than database-per-tenant

**Cons:**
- Still need to create/migrate N schemas (50 tenants = 50 schema migrations)
- Can't use connection pooling effectively (need to SET search_path per query)
- PostgreSQL schemas not designed for high cardinality (500+ schemas)
- Complex query routing

**Why rejected:** Operational complexity still high; PostgreSQL schemas not optimized for this use case

---

### Alternative 3: Table-per-Tenant (Table Prefixes)

**Description:** Shared database, but tables have tenant prefix (e.g., `tenant_acme_sites`)

**Pros:**
- Strong isolation at table level
- Can customize columns per tenant

**Cons:**
- Insane complexity (500 tenants × 20 tables = 10,000 tables)
- Schema migrations become nightmare
- No connection pooling
- Dynamic SQL (security risk, hard to optimize)
- Prisma doesn't support this pattern

**Why rejected:** Completely impractical; anti-pattern in modern SaaS

---

### Alternative 4: Hybrid (Shared Database + Large Tenants Get Dedicated DB)

**Description:** Start with shared database, move large tenants (100+ cleaners) to dedicated databases

**Pros:**
- Best of both worlds (cost-effective for small, performant for large)
- Gradual migration (no need to architect for scale on Day 1)
- Can offer "dedicated instance" as premium feature

**Cons:**
- Application must support both models (complexity)
- Need to identify when to migrate tenant (monitoring, thresholds)
- Migration process is complex (zero-downtime?)

**Why considered but deferred:** Good future option, but adds complexity for MVP; revisit when we have 100+ cleaners tenant

---

## References

- [Prisma Multi-Tenancy Guide](https://www.prisma.io/docs/guides/deployment/multi-tenancy)
- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Azure AD B2C Multi-Tenancy](https://learn.microsoft.com/en-us/azure/active-directory-b2c/multi-tenant)
- [Multi-Tenant Data Architecture (Microsoft)](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/data)

---

## Notes

### Implementation Checklist

**Sprint 1:**
- [ ] Create `tenants` table with slug, name, status
- [ ] Add `tenant_id` to all tenant-scoped tables
- [ ] Create indexes on `(tenant_id, created_at)` for time-series queries
- [ ] Implement RLS policies on all tenant-scoped tables
- [ ] Create Prisma middleware to set `app.current_tenant_id`
- [ ] Test tenant isolation (verify Tenant A can't access Tenant B data)

**Sprint 2:**
- [ ] Implement subdomain routing middleware
- [ ] Configure Azure AD B2C for multi-tenant auth
- [ ] Add `tenant_id` claim to JWT
- [ ] Create tenant provisioning API (automated onboarding)

**Sprint 3:**
- [ ] Add monitoring for per-tenant query performance
- [ ] Implement query timeouts (prevent runaway queries)
- [ ] Create admin dashboard for cross-tenant metrics

### Security Checklist

- [x] RLS policies on all tenant-scoped tables
- [ ] Prisma middleware validates tenant_id in JWT matches session
- [ ] API endpoints reject requests without valid tenant_id
- [ ] Foreign keys prevent cross-tenant data references
- [ ] Audit logging for tenant creation/deletion
- [ ] Penetration testing for tenant isolation

### Tables Requiring tenant_id

**Tenant-scoped (require tenant_id):**
- `sites` - Cleaning sites
- `shifts` - Scheduled cleaning shifts
- `checklists` - QA checklists
- `checklist_templates` - Checklist templates
- `photos` - Proof-of-work photos
- `users` - Crew members, managers (belongs to tenant)
- `notifications` - SMS/email notifications
- `invoices` - Billing records

**Global (no tenant_id):**
- `tenants` - Tenant metadata
- `system_logs` - Application logs
- `migrations` - Database migrations
- `audit_log` - Security audit trail

### Testing Strategy

**Unit Tests:**
- Verify Prisma middleware adds `tenant_id` to queries
- Test RLS policies with different `current_setting` values

**Integration Tests:**
- Create 2 test tenants, verify data isolation
- Test subdomain routing (mock Host header)
- Verify JWT validation and tenant_id extraction

**Security Tests:**
- Try to access other tenant's data (should fail)
- Try to SET `app.current_tenant_id` to different tenant (should fail)
- SQL injection attempts with tenant_id

---

## Migration Path to Alternative 4 (Hybrid Model)

**When to consider migrating large tenant to dedicated database:**
1. Tenant has 100+ active users
2. Tenant's queries consume > 20% of database resources
3. Tenant requests dedicated instance for compliance

**Migration Process:**
1. Create new PostgreSQL instance for tenant
2. Dump tenant's data (filtered by tenant_id)
3. Restore to new database
4. Update application config (tenant_id → database connection string)
5. Monitor for 24 hours
6. Delete tenant data from shared database

**Estimated effort:** 2-3 days for first migration, then automated

---

## Follow-Up Items

- [ ] Create ADR-003 for authentication strategy (Azure AD B2C details)
- [ ] Document tenant provisioning API
- [ ] Create runbook for tenant data migration (if needed)
- [ ] Set up monitoring alerts for per-tenant query performance

---

## Superseded By

[Not superseded]
