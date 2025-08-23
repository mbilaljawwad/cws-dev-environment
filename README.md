# Note: This architecture blueprint is used as reference for starting the product.
It may change from the actual scope. 

# Coworking Admin Portal – Architecture Blueprint (Full Document)

## 1) Scope & Key Principles
**In-scope (MVP → Scale)**
- Multi-tenant orgs → locations → floors → spaces (desks, rooms, private offices)  
- Users (staff + members), teams, roles/permissions (RBAC + policy checks)  
- Bookings & calendars (rooms/desks), check-ins, access rules  
- Memberships, plans, pricing, coupons, day-passes  
- Billing: invoices, payments, refunds, credit notes, taxes  
- Assets & inventory, maintenance work orders  
- Visitor management, support tickets, notifications  
- Audit logs & activity streams, reports/analytics  
- Integrations: payments, accounting, identity/SSO, calendar, door locks/IoT  

**Non-goals**  
- Full-featured CRM  
- Complex procurement/warehouse ops  

**Principles**  
- Multi-tenancy by default  
- Modular boundaries with async events  
- API-first with OpenAPI  
- Security-first, performance-conscious, operational excellence  

---

## 2) High-Level Architecture
```
[ Admin Web (Next.js) ] <---> [ API Gateway / BFF ] ---> [ Core Services (Go) ]
                                     |                        |   \
                                     |                        |    +-- Notifications
                                     |                        |    +-- Search
                                     v                        v
                               [Auth Service]           [Postgres, Redis]
                                     |                        |
                                  [OIDC]                [Object Storage]
                                     |                        |
                               [SSO/IdP]               [Message Bus]
                                     |                        |
                               [RBAC/ABAC]           [Analytics/BigQuery]
```
**Deployment Phases**:  
1. MVP: modular monolith (Go) + Postgres + Redis  
2. Scale: extract services, add event bus, analytics  

---

## 3) Tenancy, Identity & Security
- Tenant = Organization; users may belong to multiple orgs.  
- OIDC + SSO, JWT tokens.  
- RBAC roles (SuperAdmin, OrgOwner, Admin, Manager, Finance, Support, Member, Visitor).  
- ABAC for fine-grained rules.  
- Security: encrypt PII, audit trail, TLS everywhere, rate limits.  

---

## 4) Core Domain Modules
1. **Organizations & People** – orgs, users, org-users, teams  
2. **Locations & Spaces** – floors, spaces, bookings, calendars  
3. **Memberships & Pricing** – plans, credits, pricing rules  
4. **Billing & Payments** – invoices, payments, refunds, taxes  
5. **Assets & Maintenance** – assets, consumables, work orders  
6. **Visitors & Access Control** – guest mgmt, access rules/events  
7. **Support & Ops** – tickets, notifications, audit logs  
8. **Search & Reporting** – search index, reports  

---

## 5) Data Model (PostgreSQL – Indicative)
- UUIDv7 IDs, `created_at/updated_at` timestamptz, soft deletes.  
- Tables: `organizations`, `users`, `org_users`, `locations`, `spaces`, `bookings`, `invoices`, `invoice_items`, `audit_logs`.  
- Indexes: org/time-based, exclusion constraints for bookings.  

---

## 6) Services & Boundaries (Go)
- Packages: `auth`, `org`, `spaces`, `booking`, `billing`, `assets`, `support`, `notify`, `search`, `analytics`.  
- External: REST (OpenAPI), internal: gRPC.  
- Events on NATS/Kafka with outbox pattern.  
- Common packages: `pkg/db`, `pkg/cache`, `pkg/log`, `pkg/authz`, `pkg/telemetry`.  

---

## 7) APIs
- **Auth**: `/login`, `/refresh`, `/me`  
- **Orgs**: CRUD orgs & users  
- **Spaces**: list locations/spaces, create bookings, check availability  
- **Billing**: invoices, payments  
- **Support**: tickets  
- **Search**: `/search?q=`  
- **Webhooks**: payments, door-locks  
- JSON:API style errors & pagination.  

---

## 8) Frontend (Next.js Admin)
- Stack: Next.js App Router + TypeScript, TanStack Query, RHF + Zod, Tailwind + shadcn/ui, Zustand, MSW.  
- Guards & role checks, schema-first forms.  
- Screens: Orgs, Users, Spaces, Bookings, Memberships, Billing, Assets, Visitors, Support, Reports, Audit/Settings.  
- Optimizations: virtualization, code-splitting, infinite tables.  

---

## 9) Availability & Booking Logic
- Conflict detection: exclusion constraints on `(space_id, tsrange)`.  
- Recurrence with RRULE.  
- Check-in grace period, auto-cancel, refund rules.  
- Enforce entitlements (credits, quotas).  
- Calendar sync with Google/Outlook.  

---

## 10) Billing Design
- Pricing engine with modifiers (time, tier).  
- Invoicing: monthly subscriptions & ad-hoc.  
- Payments via gateways + webhooks.  
- Dunning workflows.  
- Tax per jurisdiction.  

---

## 11) Access & IoT Integration
- Vendor abstraction: `OpenDoor(spaceId, userId)`.  
- Local edge agent in Go for offline handling.  
- Signed commands, validated events.  

---

## 12) Observability & Ops
- Logging: structured with correlation IDs.  
- Metrics: Prometheus/OTel → Grafana.  
- Tracing: OTel spans across stack.  
- Health checks: `/healthz`, `/readyz`.  
- Backups with PITR.  

---

## 13) Security & Compliance
- Secrets via vault/KMS.  
- Field-level encryption for sensitive PII.  
- Immutable audit logs.  
- Role review + diffs.  
- GDPR-style export/delete.  

---

## 14) Tech Choices
- **Backend**: Go (Gin/Echo/Fiber), Postgres + sqlc, Redis, NATS, Argon2id, OTel.  
- **Frontend**: Next.js, Tailwind, shadcn/ui, TanStack Query.  
- **Search**: Meilisearch (→ OpenSearch later).  
- **CI/CD**: GitHub Actions, OpenAPI SDK generation.  

---

## 15) Testing Strategy
- Backend: unit, repo tests (testcontainers), contract tests, policy tests.  
- Frontend: unit, integration, e2e (Playwright/Cypress).  
- Load testing with k6.  

---

## 16) Incremental Delivery Plan
**MVP (6–8 weeks):** Auth, Orgs, Spaces, basic bookings, memberships, invoicing, payments, audit logs.  
**MVP+ (4–6 weeks):** Recurring bookings, credits, coupons, assets, support, notifications, reports.  
**Scale:** Search, advanced billing, IoT access, BI dashboards.  

---

## 17) RBAC Matrix (Excerpt)
| Resource      | SuperAdmin | OrgOwner | Manager | Member | Support | Finance |
|---------------|------------|----------|---------|--------|---------|---------|
| Orgs          | CRUD       | R/U      | R       | —      | R       | R       |
| Users/Teams   | R          | CRUD     | R       | R(self)| R       | R       |
| Spaces        | R          | CRUD     | CRUD    | R      | R       | R       |
| Bookings      | R          | CRUD     | CRUD    | CRUD(self)| R     | R       |
| Memberships   | R          | CRUD     | R       | R(self)| R       | R       |
| Invoices      | R          | R        | —       | R(own) | R       | CRUD    |
| Assets        | R          | R        | CRUD    | —      | R       | R       |
| Tickets       | R          | R        | R       | R(own) | CRUD    | R       |
| Audit Logs    | R          | R        | —       | —      | —       | —       |  

---

## 18) Example Go Snippets
**Router & Middleware**
```go
r := gin.New()
r.Use(telemetryMiddleware(), requestID(), rateLimit())

api := r.Group("/v1")
{
  api.POST("/auth/login", auth.Login)
  api.Use(auth.Require())

  orgs := api.Group("/orgs/:orgId", auth.OrgScope())
  {
    orgs.GET("/users", users.List)
    orgs.POST("/users", users.Invite)
    orgs.GET("/spaces", spaces.List)
    orgs.POST("/bookings", bookings.Create)
    orgs.GET("/availability", bookings.Availability)
  }
}
```

**Time Overlap Query**
```sql
SELECT * FROM bookings
WHERE org_id = $1 AND space_id = $2
  AND tstzrange(starts_at, ends_at, '[)') && tstzrange($3, $4, '[)');
```

---

## 19) Risks & Mitigations
- Double-bookings → DB exclusion + idempotent API  
- Payment webhook drift → retries + signatures  
- Multi-tenant bleed → strict `org_id` filters  
- Calendar sync → versioning, prompts  
- IoT vendor lock-in → abstraction layer  

---

## 20) Next Steps
1. Confirm MVP scope  
2. Decide on framework choices (Gin/Echo, Meilisearch/OpenSearch)  
3. Generate OpenAPI + sqlc queries  
4. Provision infra (Postgres, Redis, S3/MinIO)  
5. Implement core flow: Org → Booking → Invoice → Payment  

---

### Appendices
**A: Data Warehouse Events** – user events, booking events, invoices, payments, tickets, access events.  
**B: Config Flags** – enable/disable modules, plan-tier limits.  
**C: Importers** – CSV import (users, spaces), ICS mapping.  
