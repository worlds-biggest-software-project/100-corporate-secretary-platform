# Corporate Secretary Platform -- Development Plan

> Project: Corporate Secretary Platform (Candidate #100)
> Created: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation and Multi-Tenant Core](#phase-1-foundation-and-multi-tenant-core)
5. [Phase 2: Entity Management](#phase-2-entity-management)
6. [Phase 3: Person and Officer Management](#phase-3-person-and-officer-management)
7. [Phase 4: Meeting Lifecycle](#phase-4-meeting-lifecycle)
8. [Phase 5: Resolutions, Voting, and E-Signatures](#phase-5-resolutions-voting-and-e-signatures)
9. [Phase 6: Document Management](#phase-6-document-management)
10. [Phase 7: Compliance and Filing Management](#phase-7-compliance-and-filing-management)
11. [Phase 8: AI-Powered Governance](#phase-8-ai-powered-governance)
12. [Phase 9: Share Register and Cap Table](#phase-9-share-register-and-cap-table)
13. [Phase 10: External Integrations](#phase-10-external-integrations)
14. [Phase 11: Graph Analytics and Governance Intelligence](#phase-11-graph-analytics-and-governance-intelligence)
15. [Phase 12: Production Hardening and Compliance Certification](#phase-12-production-hardening-and-compliance-certification)
16. [Definition of Done (Global)](#definition-of-done-global)

---

## Technology Decisions

### Database: PostgreSQL 16+ with JSONB and ltree

**Rationale:** The data model analysis produced four candidates: (1) entity-centric normalised relational, (2) event-sourced/CQRS, (3) hybrid relational + JSONB, and (4) graph-relational. The chosen approach is **Model 3 (Hybrid Relational + JSONB) as the primary data layer**, with elements borrowed from Models 1 and 4.

- **Why Hybrid:** Corporate governance spans 100+ jurisdictions, each with unique entity types, filing requirements, and officer roles. A pure normalised model would require hundreds of nullable columns or subtables for jurisdiction-specific fields. JSONB columns on core relational tables provide the flexibility for jurisdiction-specific data (`jurisdiction_data`, `identifiers`) while keeping heavily-queried fields (name, status, jurisdiction code, dates) as typed relational columns with proper indexes.
- **Why not pure Event Sourcing (Model 2):** Event sourcing adds significant complexity (CQRS infrastructure, projection rebuilding, eventual consistency) that is disproportionate for an MVP. However, the **audit log** borrows the OCSF-aligned event pattern from Model 2, and the share register uses an append-only transaction pattern inspired by OCF.
- **Why graph elements from Model 4:** Ownership chain traversal, director network analysis, and conflict-of-interest detection are core differentiators. PostgreSQL's `ltree` extension and recursive CTEs handle these within the relational database without requiring a separate graph store initially. A dedicated graph layer (Neo4j/Neptune) can be added in Phase 11 for advanced analytics at scale.
- **Row-Level Security (RLS):** PostgreSQL RLS enforces tenant isolation at the database level, with `tenant_id` as the leading column on all tenant-scoped tables.

### Backend: Node.js (TypeScript) with NestJS

**Rationale:**
- NestJS provides a modular, testable architecture with built-in support for GraphQL, WebSockets, task scheduling, and guards/interceptors that map well to the platform's permission model.
- TypeScript ensures type safety across the API layer and integrates well with Prisma or Drizzle ORM for database access.
- Strong ecosystem for DocuSign SDK, Azure AD/SAML SSO libraries, and AWS S3 integration.
- The team can share types between the API layer and the frontend.

### API: GraphQL (primary) + REST (webhooks and external integrations)

**Rationale:**
- GraphQL maps naturally to the hybrid relational + JSONB data shape, where core fields are typed schema fields and JSONB extensions are resolved through custom resolvers.
- Board portals have complex data-fetching patterns (meeting with attendees, agenda items, documents, and resolutions in a single view) that benefit from GraphQL's composability over REST.
- REST endpoints are maintained for webhook receivers (DocuSign, SSO callbacks), external API consumers, and SEC EDGAR/Companies House polling.

### Frontend: Next.js 15 (App Router) with React 19

**Rationale:**
- Server Components reduce bundle size for document-heavy board pack views.
- App Router provides file-based routing with layouts suited to the platform's multi-panel governance dashboard.
- React Server Actions simplify form submissions for resolution voting, attendance recording, and compliance filing updates.
- Vercel deployment provides edge functions, ISR for public company data, and built-in analytics.

### UI Components: shadcn/ui + Tailwind CSS

**Rationale:**
- shadcn/ui provides accessible, unstyled primitives that can be themed per tenant (white-labelling requirement for enterprise buyers).
- Tailwind CSS ensures consistent spacing, typography, and responsive behaviour across the meeting builder, entity register, and compliance dashboard views.

### Object Storage: AWS S3 (with S3-compatible alternatives for data residency)

**Rationale:**
- Board packs, constitutional documents, and share certificates require encrypted-at-rest storage with granular access controls.
- AWS KMS integration for per-tenant encryption keys.
- S3-compatible providers (MinIO, Azure Blob, GCS) enable data residency options for EU/APAC deployments.

### AI: Claude API (Anthropic) via the Anthropic SDK

**Rationale:**
- Minutes drafting, document summarisation, resolution generation, and regulatory change analysis are high-ROI AI features that differentiate an AI-native platform.
- Claude's large context window (200K tokens) handles full board packs and meeting transcripts.
- Prompt caching reduces cost for repeated document summarisation tasks.
- No customer data used for model training (contractual commitment required for enterprise buyers).

### Authentication: NextAuth.js v5 with SAML/OAuth adapters

**Rationale:**
- Enterprise buyers require SAML 2.0 SSO (Azure AD, Okta) and MFA.
- NextAuth.js provides a unified authentication layer across email/password (SME tier), OAuth (Google, Microsoft), and SAML (enterprise tier).
- JWT session tokens for API authentication align with the NestJS guard pattern.

### Testing: Vitest (unit) + Playwright (E2E) + Testcontainers (integration)

**Rationale:**
- Vitest for fast unit testing of business logic (quorum calculation, voting threshold evaluation, compliance deadline computation).
- Testcontainers for integration tests against a real PostgreSQL instance with RLS policies.
- Playwright for E2E testing of board book assembly, meeting creation, and resolution voting workflows.

### CI/CD: GitHub Actions

**Rationale:**
- Automated testing, linting, security scanning (Trivy for container images, npm audit for dependencies), and deployment to Vercel (frontend) and AWS ECS/Fargate (backend).

---

## Project Structure

```
corporate-secretary-platform/
├── apps/
│   ├── web/                          # Next.js 15 frontend
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages and layouts
│   │   │   │   ├── (auth)/           # Login, SSO callback, MFA
│   │   │   │   ├── (dashboard)/      # Main authenticated layout
│   │   │   │   │   ├── entities/     # Entity management views
│   │   │   │   │   ├── meetings/     # Meeting lifecycle views
│   │   │   │   │   ├── documents/    # Document library
│   │   │   │   │   ├── resolutions/  # Resolution management
│   │   │   │   │   ├── compliance/   # Compliance dashboard
│   │   │   │   │   ├── people/       # Person and officer directory
│   │   │   │   │   ├── shares/       # Share register and cap table
│   │   │   │   │   └── settings/     # Tenant and user settings
│   │   │   ├── components/           # Shared UI components
│   │   │   │   ├── ui/               # shadcn/ui primitives
│   │   │   │   ├── entity/           # Entity-specific components
│   │   │   │   ├── meeting/          # Meeting-specific components
│   │   │   │   ├── document/         # Document viewer, uploader
│   │   │   │   └── layout/           # Shell, sidebar, navigation
│   │   │   ├── lib/                  # Utilities, API clients, hooks
│   │   │   └── styles/               # Global styles, Tailwind config
│   │   └── public/
│   │
│   └── api/                          # NestJS backend
│       ├── src/
│       │   ├── modules/
│       │   │   ├── auth/             # Authentication, SSO, MFA
│       │   │   ├── tenant/           # Multi-tenant management
│       │   │   ├── entity/           # Legal entity CRUD
│       │   │   ├── person/           # Person and officer management
│       │   │   ├── meeting/          # Meeting lifecycle
│       │   │   ├── resolution/       # Resolutions and voting
│       │   │   ├── document/         # Document storage and permissions
│       │   │   ├── compliance/       # Compliance obligations
│       │   │   ├── share/            # Share register
│       │   │   ├── signing/          # E-signature integration
│       │   │   ├── ai/               # AI services (minutes, summarisation)
│       │   │   ├── audit/            # Audit logging
│       │   │   └── graph/            # Graph analytics (Phase 11)
│       │   ├── common/
│       │   │   ├── guards/           # Auth, tenant, role guards
│       │   │   ├── interceptors/     # Audit logging, RLS context
│       │   │   ├── decorators/       # Custom decorators
│       │   │   └── filters/          # Exception filters
│       │   ├── database/
│       │   │   ├── migrations/       # Database migrations
│       │   │   ├── seeds/            # Reference data seeds
│       │   │   └── rls/              # Row-Level Security policies
│       │   └── graphql/              # GraphQL schema and resolvers
│       └── test/
│           ├── unit/
│           ├── integration/
│           └── e2e/
│
├── packages/
│   ├── shared-types/                 # Shared TypeScript types/enums
│   ├── reference-data/               # ISO 3166, GLEIF ELF, ISO 5009 datasets
│   └── ai-prompts/                   # Versioned prompt templates
│
├── infrastructure/
│   ├── docker/                       # Docker Compose for local dev
│   ├── terraform/                    # AWS infrastructure
│   └── k8s/                          # Kubernetes manifests (optional)
│
├── docs/
│   ├── api/                          # OpenAPI/GraphQL schema docs
│   ├── architecture/                 # ADRs, system diagrams
│   └── compliance/                   # Security and compliance documentation
│
├── turbo.json                        # Turborepo config
├── package.json                      # Root workspace
└── .github/
    └── workflows/                    # CI/CD pipelines
```

---

## Phase Dependency Graph

```
Phase 1: Foundation ─────────────────────────────┐
    │                                              │
    ├──► Phase 2: Entity Management                │
    │        │                                     │
    │        ├──► Phase 3: Person & Officers        │
    │        │        │                             │
    │        │        ├──► Phase 4: Meetings        │
    │        │        │        │                    │
    │        │        │        ├──► Phase 5: Resolutions & Voting
    │        │        │        │        │
    │        │        │        │        └──► Phase 9: Share Register
    │        │        │        │
    │        │        │        └──► Phase 8: AI (Minutes, Summarisation)
    │        │        │
    │        │        └──► Phase 11: Graph Analytics
    │        │
    │        └──► Phase 7: Compliance & Filing
    │
    ├──► Phase 6: Document Management (after Phase 1)
    │        │
    │        └──► (feeds into Phases 4, 5, 7)
    │
    └──► Phase 10: External Integrations
              │     (SSO, DocuSign, EDGAR, Companies House)
              │     Can start after Phase 1; integrates with Phases 3-7
              │
              └──► Phase 12: Production Hardening
                    (after all functional phases)
```

**Critical path:** 1 -> 2 -> 3 -> 4 -> 5 -> 12

**Parallel workstreams:**
- Phase 6 (Documents) can start after Phase 1 and run in parallel with Phases 2-3.
- Phase 7 (Compliance) can start after Phase 2.
- Phase 8 (AI) can start after Phase 4.
- Phase 10 (Integrations) can start after Phase 1 and incrementally connect to later phases.
- Phase 11 (Graph Analytics) can start after Phase 3.

---

## Phase 1: Foundation and Multi-Tenant Core

**Goal:** Establish the monorepo, database, authentication, multi-tenant isolation, audit logging, and a deployable shell application with a working CI/CD pipeline. All subsequent phases build on this foundation.

### Task 1.1: Monorepo and Tooling Setup

**What:** Initialise a Turborepo monorepo with `apps/web` (Next.js 15), `apps/api` (NestJS), and `packages/shared-types`. Configure ESLint, Prettier, Vitest, and Playwright. Set up Docker Compose for local PostgreSQL 16.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "test:integration": { "dependsOn": ["build"] },
    "lint": {},
    "db:migrate": { "cache": false }
  }
}
```

```yaml
# infrastructure/docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: corpsec
      POSTGRES_USER: corpsec
      POSTGRES_PASSWORD: localdev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    command: >
      postgres
        -c shared_preload_libraries=pg_stat_statements
        -c log_statement=all

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| `turbo build` completes without errors | CI | All packages and apps compile successfully |
| `turbo lint` passes on empty project | CI | ESLint reports zero violations |
| Docker Compose services start | Integration | PostgreSQL accepts connections on port 5432 |
| Vitest runs with zero tests | Unit | Vitest exits 0 with "no tests found" warning |
| Playwright config validates | E2E | `npx playwright test --list` exits 0 |

---

### Task 1.2: Database Schema -- Reference Data and Multi-Tenant Tables

**What:** Create the foundational database tables: `reference_data` (unified lookup for jurisdictions, legal forms, officer roles, document categories), `tenant`, and `app_user`. Implement Row-Level Security policies. Seed reference data from ISO 3166, GLEIF ELF, and ISO 5009 datasets.

**Design:**

```sql
-- Migration: 001_foundation.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "ltree";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Unified reference data table
CREATE TABLE reference_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_code       VARCHAR(50) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    name_local      VARCHAR(500),
    description     TEXT,
    parent_code     VARCHAR(20),
    attributes      JSONB NOT NULL DEFAULT '{}',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(list_code, code)
);

CREATE INDEX idx_refdata_list ON reference_data(list_code, is_active);
CREATE INDEX idx_refdata_parent ON reference_data(list_code, parent_code)
    WHERE parent_code IS NOT NULL;
CREATE INDEX idx_refdata_attributes ON reference_data USING GIN (attributes);

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    profile         JSONB NOT NULL DEFAULT '{}',
    password_hash   VARCHAR(255),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);

ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_user ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

```typescript
// packages/reference-data/src/jurisdictions.ts
// Seed data for ISO 3166 jurisdictions
export const jurisdictions = [
  { list_code: 'JURISDICTION', code: 'US', name: 'United States',
    attributes: { iso_alpha3: 'USA', iso_numeric: '840', region: 'North America' } },
  { list_code: 'JURISDICTION', code: 'US-DE', name: 'Delaware',
    parent_code: 'US',
    attributes: { region: 'North America', is_subdivision: true } },
  { list_code: 'JURISDICTION', code: 'GB', name: 'United Kingdom',
    attributes: { iso_alpha3: 'GBR', iso_numeric: '826', region: 'Europe' } },
  // ... 200+ jurisdictions from ISO 3166-1 and key 3166-2 subdivisions
];
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Migration runs without errors | Integration | All tables created; `\dt` shows reference_data, tenant, app_user |
| RLS blocks cross-tenant reads | Integration | Query with tenant A's context returns zero rows from tenant B |
| RLS allows same-tenant reads | Integration | Query with tenant A's context returns tenant A's users |
| Reference data seed populates jurisdictions | Integration | `SELECT COUNT(*) FROM reference_data WHERE list_code = 'JURISDICTION'` >= 200 |
| Unique constraint on (list_code, code) | Integration | Duplicate insert raises unique violation error |
| UUID primary keys are generated | Integration | Inserted rows have valid UUID v4 in `id` column |

---

### Task 1.3: Audit Log Infrastructure

**What:** Create the OCSF-aligned audit log table with time-based partitioning. Build a NestJS interceptor that automatically records all mutating operations. Implement partition management (auto-create monthly partitions).

**Design:**

```sql
-- Migration: 002_audit_log.sql

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    category        VARCHAR(50) NOT NULL,
    event_class     VARCHAR(50) NOT NULL,
    activity        VARCHAR(20) NOT NULL,
    actor_user_id   UUID,
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID NOT NULL,
    target_name     VARCHAR(500),
    status          VARCHAR(10) NOT NULL DEFAULT 'SUCCESS',
    severity        VARCHAR(10) NOT NULL DEFAULT 'INFO',
    event_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, event_time DESC);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id, event_time DESC);
```

```typescript
// apps/api/src/common/interceptors/audit.interceptor.ts
@Injectable()
export class AuditInterceptor implements NestInterceptor {
  constructor(private readonly auditService: AuditService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      tap(async (response) => {
        if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method)) {
          await this.auditService.log({
            tenantId: request.tenantId,
            category: this.resolveCategory(request),
            eventClass: this.resolveEventClass(request),
            activity: this.resolveActivity(request.method),
            actorUserId: request.user?.id,
            targetType: this.resolveTargetType(request),
            targetId: response?.id ?? request.params?.id,
            status: 'SUCCESS',
            eventData: {
              actor: { email: request.user?.email, ip: request.ip },
              duration_ms: Date.now() - startTime,
            },
          });
        }
      }),
    );
  }
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Audit log records created on POST | Integration | After a POST to `/api/tenants`, audit_log contains one row with activity='CREATE' |
| Audit log captures actor user ID | Integration | `actor_user_id` matches the authenticated user's UUID |
| Audit log partitions exist for current month | Integration | `SELECT * FROM pg_tables WHERE tablename LIKE 'audit_log_2026_%'` returns at least one row |
| Partition auto-creation function works | Integration | Calling `create_audit_partitions(3)` creates 3 future monthly partitions |
| Failed operations logged with status='FAILURE' | Integration | A rejected request produces an audit entry with `status='FAILURE'` |
| Audit log query by tenant and time range | Integration | Filtered query returns only matching tenant's events within the specified range |

---

### Task 1.4: Authentication and Authorisation

**What:** Implement NextAuth.js v5 with email/password login (MVP), JWT session management, and role-based access control (admin, secretary, director, observer, member). Build NestJS guards that validate JWT tokens and enforce role-based permissions. Prepare SSO adapter interfaces for SAML (Phase 10).

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    CredentialsProvider({
      name: 'Email',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const user = await verifyCredentials(
          credentials.email,
          credentials.password,
        );
        if (!user) return null;
        return {
          id: user.id,
          email: user.email,
          name: user.full_name,
          role: user.role,
          tenantId: user.tenant_id,
        };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
        token.tenantId = user.tenantId;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.role = token.role;
      session.user.tenantId = token.tenantId;
      return session;
    },
  },
});
```

```typescript
// apps/api/src/common/guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      'roles', [context.getHandler(), context.getClass()],
    );
    if (!requiredRoles) return true;
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.includes(user.role);
  }
}

// Usage:
@Roles('admin', 'secretary')
@UseGuards(AuthGuard, RolesGuard)
@Post('entities')
async createEntity(@Body() dto: CreateEntityDto) { ... }
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Valid credentials return JWT | Integration | POST `/api/auth/signin` returns 200 with session token containing `tenantId` and `role` |
| Invalid credentials return 401 | Integration | POST with wrong password returns 401 Unauthorized |
| JWT contains tenant_id claim | Unit | Decoded JWT token includes `tenantId` field matching user's tenant |
| RolesGuard blocks observer from admin routes | Unit | Request with role='observer' to @Roles('admin') endpoint returns 403 |
| RolesGuard allows admin access | Unit | Request with role='admin' to @Roles('admin') endpoint returns 200 |
| Expired JWT returns 401 | Integration | Token issued with 1-second TTL returns 401 after expiry |
| Tenant context set in database session | Integration | After auth middleware runs, `current_setting('app.current_tenant_id')` returns the correct UUID |

---

### Task 1.5: Application Shell and Dashboard Layout

**What:** Build the Next.js application shell with sidebar navigation, tenant-aware header, role-based menu items, and placeholder pages for each functional area. Implement responsive layout for tablet (director iPad use case).

**Design:**

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await auth();
  if (!session) redirect('/login');

  return (
    <div className="flex h-screen">
      <Sidebar role={session.user.role} />
      <main className="flex-1 overflow-auto">
        <Header user={session.user} />
        <div className="p-6">{children}</div>
      </main>
    </div>
  );
}
```

```typescript
// apps/web/src/components/layout/sidebar.tsx
const navigationItems: NavItem[] = [
  { label: 'Dashboard', href: '/dashboard', icon: HomeIcon, roles: ['*'] },
  { label: 'Entities', href: '/entities', icon: BuildingIcon,
    roles: ['admin', 'secretary'] },
  { label: 'People', href: '/people', icon: UsersIcon,
    roles: ['admin', 'secretary'] },
  { label: 'Meetings', href: '/meetings', icon: CalendarIcon, roles: ['*'] },
  { label: 'Resolutions', href: '/resolutions', icon: VoteIcon, roles: ['*'] },
  { label: 'Documents', href: '/documents', icon: FileIcon, roles: ['*'] },
  { label: 'Compliance', href: '/compliance', icon: ShieldIcon,
    roles: ['admin', 'secretary'] },
  { label: 'Shares', href: '/shares', icon: ChartIcon,
    roles: ['admin', 'secretary'] },
  { label: 'Settings', href: '/settings', icon: SettingsIcon,
    roles: ['admin'] },
];
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Unauthenticated user redirected to /login | E2E | Navigating to /dashboard without session redirects to /login |
| Sidebar renders role-appropriate items | E2E | Director role sees Meetings, Resolutions, Documents but not Settings or Compliance |
| Admin sees all navigation items | E2E | Admin role sees all 9 navigation items |
| Layout is responsive at tablet width (1024px) | E2E | Sidebar collapses to icons-only mode; main content fills remaining width |
| Tenant name appears in header | E2E | Header displays the tenant name from the session |
| Active route is highlighted in sidebar | E2E | When on /meetings, the Meetings nav item has active styling |

---

### Task 1.6: CI/CD Pipeline

**What:** Configure GitHub Actions for build, lint, test, security scan (npm audit, Trivy), and deployment. Set up preview deployments for PRs (Vercel for frontend, staging environment for API).

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: corpsec_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo build
      - run: npx turbo test
      - run: npx turbo test:integration
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/corpsec_test
      - run: npm audit --audit-level=high
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| CI pipeline passes on clean main branch | CI | All jobs complete green |
| Failed lint blocks merge | CI | PR with lint violation shows failing check |
| Integration tests run against real PostgreSQL | CI | Testcontainers (or service container) provides PostgreSQL; tests pass |
| Security scan reports high-severity vulnerabilities | CI | npm audit with `--audit-level=high` fails if high-severity CVE found |

---

### Phase 1 Definition of Done

- [ ] Monorepo builds and passes lint/type-check with zero errors
- [ ] PostgreSQL database running with reference_data, tenant, app_user tables and RLS policies
- [ ] Reference data seeded (200+ jurisdictions, GLEIF ELF codes, ISO 5009 officer roles)
- [ ] Audit log table with monthly partitions and OCSF-aligned schema
- [ ] Email/password authentication working end-to-end
- [ ] Role-based navigation and guards enforced
- [ ] Dashboard shell renders with sidebar, header, and placeholder pages
- [ ] CI/CD pipeline running on every push, blocking merge on failures
- [ ] Minimum 80% code coverage on business logic (guards, interceptors, seed scripts)

---

## Phase 2: Entity Management

**Goal:** Implement the core entity register -- create, read, update, and search legal entities across jurisdictions. Build the entity hierarchy view (corporate structure chart) and entity detail pages.

**Depends on:** Phase 1

### Task 2.1: Legal Entity CRUD

**What:** Create the `legal_entity` table, NestJS module with GraphQL resolvers, and frontend pages for listing, creating, and editing entities. Support jurisdiction-specific data via the `jurisdiction_data` JSONB column.

**Design:**

```sql
-- Migration: 003_legal_entity.sql
CREATE TABLE legal_entity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    legal_name          VARCHAR(500) NOT NULL,
    trading_name        VARCHAR(500),
    lei                 VARCHAR(20) UNIQUE,
    legal_form_code     VARCHAR(20),
    jurisdiction_code   VARCHAR(6) NOT NULL,
    registration_number VARCHAR(100),
    registration_date   DATE,
    entity_status       VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    legal_address       JSONB NOT NULL DEFAULT '{}',
    hq_address          JSONB DEFAULT '{}',
    identifiers         JSONB NOT NULL DEFAULT '{}',
    jurisdiction_data   JSONB NOT NULL DEFAULT '{}',
    fiscal_year_end_month INTEGER CHECK (fiscal_year_end_month BETWEEN 1 AND 12),
    fiscal_year_end_day   INTEGER CHECK (fiscal_year_end_day BETWEEN 1 AND 31),
    ai_metadata         JSONB NOT NULL DEFAULT '{}',
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_jurisdiction ON legal_entity(tenant_id, jurisdiction_code);
CREATE INDEX idx_entity_status ON legal_entity(tenant_id, entity_status);
CREATE INDEX idx_entity_lei ON legal_entity(lei) WHERE lei IS NOT NULL;
CREATE INDEX idx_entity_identifiers ON legal_entity USING GIN (identifiers);
CREATE INDEX idx_entity_jurisdiction_data ON legal_entity USING GIN (jurisdiction_data);

ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_entity ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

```typescript
// apps/api/src/modules/entity/entity.resolver.ts
@Resolver('LegalEntity')
export class EntityResolver {
  constructor(private readonly entityService: EntityService) {}

  @Query(() => [LegalEntity])
  @Roles('admin', 'secretary')
  async entities(
    @Args('filter') filter: EntityFilterInput,
    @Args('pagination') pagination: PaginationInput,
  ): Promise<LegalEntity[]> {
    return this.entityService.findAll(filter, pagination);
  }

  @Mutation(() => LegalEntity)
  @Roles('admin', 'secretary')
  async createEntity(
    @Args('input') input: CreateEntityInput,
    @CurrentUser() user: AuthUser,
  ): Promise<LegalEntity> {
    return this.entityService.create(input, user);
  }
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create entity with required fields | Integration | Entity created with UUID, tenant_id set, entity_status='ACTIVE' |
| Create entity with jurisdiction_data (UK) | Integration | Entity stores `{ "company_number": "12345678" }` in `jurisdiction_data` JSONB |
| Create entity with jurisdiction_data (Delaware) | Integration | Entity stores `{ "delaware_file_number": "4567890" }` |
| List entities filtered by jurisdiction | Integration | `GET /entities?jurisdiction=US-DE` returns only Delaware entities |
| List entities filtered by status | Integration | Filter by `ACTIVE` excludes dissolved entities |
| RLS prevents cross-tenant entity access | Integration | Tenant A cannot see Tenant B's entities |
| LEI uniqueness enforced | Integration | Duplicate LEI insert returns constraint violation |
| GraphQL query returns paginated results | Integration | Query with `first: 10, after: cursor` returns correct page |
| Entity search by registration number | Integration | JSONB query on `identifiers` finds entity by CIK |
| Entity update records audit log entry | Integration | After PUT, audit_log contains event_class='entity.updated' |

---

### Task 2.2: Entity Relationship Hierarchy

**What:** Create the `entity_relationship` table (GLEIF RR-CDF aligned). Build a recursive CTE for corporate structure tree traversal. Implement a visual corporate structure chart using a tree/org-chart component.

**Design:**

```sql
-- Migration: 004_entity_relationship.sql
CREATE TABLE entity_relationship (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    child_entity_id     UUID NOT NULL REFERENCES legal_entity(id),
    parent_entity_id    UUID NOT NULL REFERENCES legal_entity(id),
    relationship_type   VARCHAR(50) NOT NULL,
    ownership_percentage DECIMAL(5,2),
    voting_percentage   DECIMAL(5,2),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    relationship_data   JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_self_ref CHECK (child_entity_id != parent_entity_id)
);

CREATE INDEX idx_rel_child ON entity_relationship(tenant_id, child_entity_id);
CREATE INDEX idx_rel_parent ON entity_relationship(tenant_id, parent_entity_id);
```

```typescript
// apps/api/src/modules/entity/entity.service.ts
async getEntityTree(rootEntityId: string, tenantId: string) {
  return this.db.$queryRaw`
    WITH RECURSIVE entity_tree AS (
      SELECT le.id, le.legal_name, NULL::uuid AS parent_id,
             NULL::decimal AS ownership_pct, 0 AS depth
      FROM legal_entity le
      WHERE le.id = ${rootEntityId}
        AND le.tenant_id = ${tenantId}

      UNION ALL

      SELECT le.id, le.legal_name, er.parent_entity_id,
             er.ownership_percentage, et.depth + 1
      FROM entity_relationship er
      JOIN legal_entity le ON le.id = er.child_entity_id
      JOIN entity_tree et ON et.id = er.parent_entity_id
      WHERE er.status = 'ACTIVE'
        AND er.relationship_type = 'IS_DIRECTLY_CONSOLIDATED_BY'
    )
    SELECT * FROM entity_tree ORDER BY depth, legal_name
  `;
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create parent-child relationship | Integration | Relationship stored with GLEIF type, ownership percentage, valid_from |
| Self-reference prevented | Integration | Attempting child_entity_id = parent_entity_id raises constraint error |
| Entity tree returns 3-level hierarchy | Integration | Root -> 2 children -> 4 grandchildren returned in depth order |
| Circular reference detection | Integration | Recursive CTE with depth limit prevents infinite loops |
| Terminated relationships excluded from active tree | Integration | Relationship with valid_to in the past excluded from tree query |
| Corporate structure chart renders visually | E2E | Org-chart component displays parent at top, children below, with ownership percentages on edges |
| Ownership percentage totals validated | Unit | Service warns if child entity's incoming ownership exceeds 100% |

---

### Task 2.3: Entity Detail Page and Dashboard

**What:** Build the entity detail page showing entity information, jurisdiction-specific fields, active officers, upcoming compliance deadlines (placeholder), and recent audit log entries for the entity.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/entities/[id]/page.tsx
export default async function EntityDetailPage({
  params,
}: {
  params: { id: string };
}) {
  const entity = await getEntity(params.id);
  return (
    <div className="space-y-6">
      <EntityHeader entity={entity} />
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <EntityInfoCard entity={entity} className="lg:col-span-2" />
        <EntityStatusCard entity={entity} />
      </div>
      <Tabs defaultValue="officers">
        <TabsList>
          <TabsTrigger value="officers">Officers</TabsTrigger>
          <TabsTrigger value="subsidiaries">Subsidiaries</TabsTrigger>
          <TabsTrigger value="compliance">Compliance</TabsTrigger>
          <TabsTrigger value="documents">Documents</TabsTrigger>
          <TabsTrigger value="audit">Audit Trail</TabsTrigger>
        </TabsList>
        <TabsContent value="officers">
          <OfficerList entityId={entity.id} />
        </TabsContent>
        {/* ... other tab content */}
      </Tabs>
    </div>
  );
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Entity detail page loads with correct data | E2E | Page displays entity legal name, registration number, jurisdiction |
| Jurisdiction-specific fields displayed for UK entity | E2E | UK entity shows "Company Number" field from jurisdiction_data |
| Jurisdiction-specific fields displayed for Delaware entity | E2E | Delaware entity shows "File Number" and "Franchise Tax Due" |
| Tabs switch content correctly | E2E | Clicking "Subsidiaries" tab shows child entities |
| Entity edit form pre-populates fields | E2E | Edit mode shows current values in all fields |
| Audit trail tab shows recent changes | E2E | Audit entries for this entity displayed with timestamps and actors |

---

### Phase 2 Definition of Done

- [ ] legal_entity and entity_relationship tables created with RLS
- [ ] Full CRUD for entities via GraphQL with role-based access
- [ ] Entity list with filtering by jurisdiction, status, and search by name/registration number
- [ ] Entity detail page with tabs for officers, subsidiaries, compliance, documents, audit trail
- [ ] Corporate structure chart renders multi-level entity hierarchy with ownership percentages
- [ ] Jurisdiction-specific JSONB data stored and displayed correctly for US-DE, GB, SG
- [ ] Audit log entries recorded for all entity create/update/delete operations
- [ ] Minimum 85% test coverage on entity service and resolver

---

## Phase 3: Person and Officer Management

**Goal:** Build the person register, officer appointment and resignation workflows, committee management, and conflict-of-interest declarations.

**Depends on:** Phase 2

### Task 3.1: Person Register

**What:** Create the `person` table with GDPR-aware address handling (service vs residential), the `officer_appointment` table with ISO 5009 role codes, and CRUD operations for both.

**Design:**

```sql
-- Migration: 005_person.sql
CREATE TABLE person (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    full_name           VARCHAR(500) NOT NULL,
    given_name          VARCHAR(255),
    family_name         VARCHAR(255),
    date_of_birth       DATE,
    nationality         VARCHAR(2),
    country_of_residence VARCHAR(2),
    email               VARCHAR(320),
    app_user_id         UUID REFERENCES app_user(id),
    service_address     JSONB NOT NULL DEFAULT '{}',
    residential_address JSONB DEFAULT '{}',
    jurisdiction_data   JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_name ON person(tenant_id, family_name, given_name);
CREATE INDEX idx_person_user ON person(app_user_id) WHERE app_user_id IS NOT NULL;

ALTER TABLE person ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_person ON person
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE TABLE officer_appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    role_code           VARCHAR(20) NOT NULL,
    role_description    VARCHAR(255),
    appointed_on        DATE NOT NULL,
    resigned_on         DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    resolution_id       UUID,  -- FK added after resolution table exists
    appointment_data    JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_entity ON officer_appointment(tenant_id, entity_id);
CREATE INDEX idx_appt_person ON officer_appointment(tenant_id, person_id);
CREATE INDEX idx_appt_active ON officer_appointment(tenant_id, entity_id, status)
    WHERE status = 'ACTIVE';
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create person with service and residential addresses | Integration | Both JSONB address fields stored correctly |
| Residential address not exposed to 'observer' role | Unit | API response for observer role omits `residential_address` field |
| Appoint officer to entity | Integration | officer_appointment created with status='ACTIVE', role_code from ISO 5009 |
| Resign officer | Integration | `resigned_on` set, `status` changed to 'RESIGNED' |
| List active officers for entity | Integration | Only status='ACTIVE' appointments returned |
| Person search by name | Integration | Search for "Smith" returns all persons with matching family_name |
| Link person to app_user | Integration | Setting `app_user_id` enables the person to log in and see their board materials |
| Director overlap query | Integration | Query returns persons serving on multiple entity boards within the tenant |
| GDPR: date_of_birth stored but display restricted | Unit | Public-facing views show only month/year (Companies House model) |

---

### Task 3.2: Committee Management

**What:** Create `committee` and `committee_membership` tables. Build UI for creating committees (Board, Audit, Remuneration, Nomination, Risk, Custom), assigning members with roles (Chair, Member, Secretary, Observer), and viewing committee composition.

**Design:**

```sql
-- Migration: 006_committee.sql
CREATE TABLE committee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    name            VARCHAR(255) NOT NULL,
    committee_type  VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE committee_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    committee_id    UUID NOT NULL REFERENCES committee(id),
    person_id       UUID NOT NULL REFERENCES person(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'MEMBER',
    appointed_on    DATE NOT NULL,
    resigned_on     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(committee_id, person_id, appointed_on)
);
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create Board of Directors committee | Integration | Committee created with type='BOARD' for entity |
| Add 5 members to committee | Integration | 5 committee_membership rows created with correct roles |
| Set quorum via config JSONB | Integration | `config.quorum_count = 3` stored and retrievable |
| List committee members with roles | Integration | API returns members sorted by role (Chair first, then Members) |
| Committee member resignation | Integration | `resigned_on` set; member no longer appears in active member list |
| Prevent duplicate active membership | Integration | Adding same person twice without resignation returns error |

---

### Task 3.3: Conflict of Interest Declarations

**What:** Create the `conflict_of_interest` table and build a workflow for directors to declare, and for the secretary to review and manage conflicts.

**Design:**

```sql
-- Migration: 007_conflict_of_interest.sql
CREATE TABLE conflict_of_interest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    person_id       UUID NOT NULL REFERENCES person(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    nature_of_interest TEXT NOT NULL,
    declared_on     DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'DECLARED',
    coi_data        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Director declares conflict of interest | Integration | COI record created with status='DECLARED' |
| Secretary reviews COI and changes status to 'REVIEWED' | Integration | Status updated, `coi_data.reviewed_by` and `coi_data.reviewed_on` set |
| COI list for entity shows all active declarations | Integration | All non-RESOLVED COIs returned for the entity |
| Director sees only their own COI declarations | Unit | RolesGuard + filter ensures director role only sees own records |

---

### Phase 3 Definition of Done

- [ ] Person register with GDPR-aware address handling
- [ ] Officer appointment and resignation workflows
- [ ] Committee CRUD with membership management
- [ ] Conflict of interest declaration and review workflow
- [ ] Person detail page showing all appointments across entities
- [ ] Director overlap analysis query working
- [ ] All person/officer mutations generate audit log entries
- [ ] Minimum 85% test coverage

---

## Phase 4: Meeting Lifecycle

**Goal:** Build the complete meeting lifecycle: scheduling, agenda building, board pack assembly, attendance recording, digital minutes, and action item tracking.

**Depends on:** Phase 3

### Task 4.1: Meeting CRUD and Scheduling

**What:** Create the `meeting`, `agenda_item`, and `meeting_attendance` tables. Build the meeting creation form with type selection (Board, Committee, AGM, EGM), scheduling, location settings (in-person, virtual, hybrid), and committee association.

**Design:**

```sql
-- Migration: 008_meeting.sql
CREATE TABLE meeting (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    committee_id    UUID REFERENCES committee(id),
    title           VARCHAR(500) NOT NULL,
    meeting_type    VARCHAR(50) NOT NULL,
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    meeting_data    JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_meeting_entity ON meeting(tenant_id, entity_id);
CREATE INDEX idx_meeting_date ON meeting(tenant_id, scheduled_start);
CREATE INDEX idx_meeting_status ON meeting(tenant_id, status);

CREATE TABLE agenda_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meeting(id) ON DELETE CASCADE,
    item_number     VARCHAR(20) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    item_type       VARCHAR(50) NOT NULL DEFAULT 'DISCUSSION',
    presenter_id    UUID REFERENCES person(id),
    duration_minutes INTEGER,
    sort_order      INTEGER NOT NULL,
    outcome_notes   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE meeting_attendance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meeting(id) ON DELETE CASCADE,
    person_id       UUID NOT NULL REFERENCES person(id),
    attendance_type VARCHAR(20) NOT NULL DEFAULT 'MEMBER',
    rsvp_status     VARCHAR(20),
    attended        BOOLEAN,
    attended_via    VARCHAR(20),
    proxy_for_id    UUID REFERENCES person(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(meeting_id, person_id)
);
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create board meeting with agenda | Integration | Meeting and 5 agenda items created; status='DRAFT' |
| Meeting status transitions: DRAFT -> SCHEDULED -> IN_PROGRESS -> CONCLUDED | Integration | Each transition valid; invalid transitions (e.g., DRAFT -> CONCLUDED) rejected |
| Auto-populate attendance from committee members | Integration | When creating a committee meeting, attendance records pre-populated for all active committee members |
| Record RSVP responses | Integration | Attendance rsvp_status updated to 'ACCEPTED' or 'DECLINED' |
| Record actual attendance | Integration | After meeting, `attended` set to true/false with `attended_via` |
| Quorum calculation | Unit | Given 7 members, quorum of 4, with 5 present, `quorum_present = true` |
| Quorum calculation - not met | Unit | Given 7 members, quorum of 4, with 3 present, `quorum_present = false` |
| Meeting calendar view | E2E | Calendar component shows meetings plotted on correct dates |

---

### Task 4.2: Agenda Builder and Board Pack Assembly

**What:** Build a drag-and-drop agenda builder with agenda item types (Procedural, Approval, Discussion, Information, Resolution, AOB). Enable document attachment to agenda items. Generate a compiled board pack (PDF) from the agenda and attached documents.

**Design:**

```typescript
// apps/web/src/components/meeting/agenda-builder.tsx
export function AgendaBuilder({ meetingId }: { meetingId: string }) {
  const { items, reorder, addItem, removeItem } = useAgendaItems(meetingId);

  return (
    <DndContext onDragEnd={handleDragEnd}>
      <SortableContext items={items.map(i => i.id)}>
        {items.map((item) => (
          <SortableAgendaItem
            key={item.id}
            item={item}
            onEdit={updateItem}
            onAttachDocument={attachDocument}
            onRemove={removeItem}
          />
        ))}
      </SortableContext>
      <AddAgendaItemButton onAdd={addItem} />
    </DndContext>
  );
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Drag-and-drop reorders agenda items | E2E | After drag, `sort_order` values updated; items render in new order |
| Add agenda item with type 'RESOLUTION' | E2E | New item appears at end with type badge showing "Resolution" |
| Attach document to agenda item | E2E | Document linked to `agenda_item_id`; appears as attachment on the item |
| Remove agenda item | E2E | Item removed from list; sort_order of remaining items recalculated |
| Board pack includes all agenda items and documents | Integration | PDF generation produces paginated document with table of contents |
| Board pack sent notification | Integration | Setting status to 'BOARD_PACK_SENT' sends notification and records `board_pack_sent_at` |

---

### Task 4.3: Digital Minutes and Action Items

**What:** Build the minutes editor with rich text editing, action item extraction, and linking to agenda items. Create the `action_item` table and action item tracking UI.

**Design:**

```sql
-- Migration: 009_action_item.sql
CREATE TABLE action_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    meeting_id      UUID REFERENCES meeting(id),
    agenda_item_id  UUID REFERENCES agenda_item(id),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES person(id),
    due_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    completed_at    TIMESTAMPTZ,
    ai_data         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// apps/api/src/modules/meeting/minutes.service.ts
async saveMinutes(meetingId: string, minutesText: string): Promise<void> {
  await this.db.meeting.update({
    where: { id: meetingId },
    data: {
      meeting_data: {
        ...existingData,
        minutes: {
          draft_text: minutesText,
          last_edited_at: new Date().toISOString(),
          ai_generated: false,
        },
      },
    },
  });
}

async approveMinutes(meetingId: string, userId: string): Promise<void> {
  await this.db.meeting.update({
    where: { id: meetingId },
    data: {
      status: 'MINUTES_APPROVED',
      meeting_data: {
        ...existingData,
        minutes: {
          ...existingData.minutes,
          final_text: existingData.minutes.draft_text,
          approved_at: new Date().toISOString(),
          approved_by: userId,
        },
      },
    },
  });
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Save draft minutes for meeting | Integration | `meeting_data.minutes.draft_text` updated; status remains 'CONCLUDED' |
| Approve minutes transitions to MINUTES_APPROVED | Integration | Status updated; `approved_at` and `approved_by` recorded |
| Create action item from meeting | Integration | Action item linked to meeting and agenda item |
| Assign action item to person | Integration | `assigned_to` set; person sees action in their dashboard |
| Complete action item | Integration | `status` changed to 'COMPLETED'; `completed_at` set |
| Action item list filterable by status | Integration | Filter by 'OPEN' excludes completed items |
| Action item due date overdue flagging | Unit | Items with `due_date` before today and status='OPEN' flagged as overdue |
| Meeting lifecycle complete flow | E2E | Create meeting -> add agenda -> record attendance -> save minutes -> approve minutes (full workflow) |

---

### Phase 4 Definition of Done

- [ ] Meeting CRUD with status lifecycle (DRAFT through MINUTES_APPROVED)
- [ ] Drag-and-drop agenda builder with document attachments
- [ ] Attendance recording with RSVP, actual attendance, and proxy support
- [ ] Quorum calculation based on committee configuration
- [ ] Digital minutes editor with rich text
- [ ] Action item CRUD with assignment, due dates, and status tracking
- [ ] Meeting calendar view showing upcoming meetings across entities
- [ ] Board pack sent notification flow
- [ ] Full meeting lifecycle E2E test passing

---

## Phase 5: Resolutions, Voting, and E-Signatures

**Goal:** Build resolution creation, voting workflows, approval threshold enforcement, and e-signature integration for signed resolutions and written consents.

**Depends on:** Phase 4

### Task 5.1: Resolution CRUD and Voting

**What:** Create the `resolution` and `resolution_vote` tables. Build resolution drafting, proposal, voting, and outcome recording. Enforce configurable approval thresholds (simple majority, supermajority, unanimous).

**Design:**

```sql
-- Migration: 010_resolution.sql
CREATE TABLE resolution (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    resolution_number VARCHAR(50),
    title           VARCHAR(500) NOT NULL,
    body_text       TEXT NOT NULL,
    resolution_type VARCHAR(50) NOT NULL,
    meeting_id      UUID REFERENCES meeting(id),
    agenda_item_id  UUID REFERENCES agenda_item(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    approval_threshold DECIMAL(5,2) NOT NULL DEFAULT 50.01,
    votes_for       INTEGER,
    votes_against   INTEGER,
    votes_abstained INTEGER,
    passed_at       TIMESTAMPTZ,
    effective_date  DATE,
    resolution_data JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE resolution_vote (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resolution_id   UUID NOT NULL REFERENCES resolution(id),
    person_id       UUID NOT NULL REFERENCES person(id),
    vote            VARCHAR(10) NOT NULL,
    voted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    shares_voted    BIGINT,
    signature_method VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(resolution_id, person_id)
);
```

```typescript
// apps/api/src/modules/resolution/resolution.service.ts
async evaluateVote(resolutionId: string): Promise<ResolutionOutcome> {
  const resolution = await this.db.resolution.findUnique({
    where: { id: resolutionId },
    include: { votes: true },
  });

  const votesFor = resolution.votes.filter(v => v.vote === 'FOR').length;
  const votesAgainst = resolution.votes.filter(v => v.vote === 'AGAINST').length;
  const totalVotes = votesFor + votesAgainst; // Abstentions do not count
  const percentageFor = totalVotes > 0 ? (votesFor / totalVotes) * 100 : 0;

  const passed = percentageFor >= resolution.approval_threshold;

  await this.db.resolution.update({
    where: { id: resolutionId },
    data: {
      votes_for: votesFor,
      votes_against: votesAgainst,
      votes_abstained: resolution.votes.filter(v => v.vote === 'ABSTAIN').length,
      status: passed ? 'PASSED' : 'FAILED',
      passed_at: passed ? new Date() : null,
    },
  });

  return { passed, percentageFor, votesFor, votesAgainst };
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create board resolution linked to meeting agenda item | Integration | Resolution created with `meeting_id` and `agenda_item_id` set |
| Cast vote FOR | Integration | Vote recorded; `vote='FOR'`, `voted_at` set |
| Prevent duplicate vote | Integration | Second vote by same person returns unique constraint violation |
| Simple majority passes (3 FOR, 2 AGAINST) | Unit | `evaluateVote` returns passed=true, percentageFor=60.0 |
| Simple majority fails (2 FOR, 3 AGAINST) | Unit | Returns passed=false, percentageFor=40.0 |
| 75% supermajority passes (6 FOR, 1 AGAINST, 1 ABSTAIN) | Unit | With threshold=75, passed=true (6/7 = 85.7%) |
| 75% supermajority fails (5 FOR, 2 AGAINST, 1 ABSTAIN) | Unit | With threshold=75, passed=false (5/7 = 71.4%) |
| Unanimous resolution (all FOR, 0 AGAINST) | Unit | With threshold=100, passed=true |
| Written consent (no meeting, all directors sign) | Integration | Resolution with type='WRITTEN_CONSENT', no meeting_id, all directors vote FOR |
| Resolution number auto-generated | Integration | Pattern 'BR-{year}-{sequence}' assigned automatically |

---

### Task 5.2: E-Signature Integration (DocuSign)

**What:** Create the `signing_envelope` table. Integrate with the DocuSign eSignature API to send resolutions for signing, track recipient status, and receive webhook callbacks for completion.

**Design:**

```sql
-- Migration: 011_signing.sql
CREATE TABLE signing_envelope (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider        VARCHAR(20) NOT NULL DEFAULT 'DOCUSIGN',
    external_envelope_id VARCHAR(255),
    email_subject   VARCHAR(500) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'CREATED',
    signature_standard VARCHAR(20) NOT NULL DEFAULT 'ESIGN',
    resolution_id   UUID REFERENCES resolution(id),
    provider_data   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// apps/api/src/modules/signing/docusign.service.ts
async sendForSigning(
  resolution: Resolution,
  signers: Person[],
): Promise<SigningEnvelope> {
  const envelopeDefinition = {
    emailSubject: `Resolution ${resolution.resolution_number} for signature`,
    documents: [{
      documentBase64: await this.generateResolutionPdf(resolution),
      name: `${resolution.resolution_number}.pdf`,
      documentId: '1',
    }],
    recipients: {
      signers: signers.map((signer, index) => ({
        email: signer.email,
        name: signer.full_name,
        recipientId: String(index + 1),
        routingOrder: String(index + 1),
        tabs: {
          signHereTabs: [{
            documentId: '1',
            pageNumber: String(this.lastPage),
            xPosition: '100',
            yPosition: String(500 + index * 60),
          }],
          dateSignedTabs: [{
            documentId: '1',
            pageNumber: String(this.lastPage),
            xPosition: '300',
            yPosition: String(500 + index * 60),
          }],
        },
      })),
    },
    status: 'sent',
  };

  const result = await this.docusignClient.createEnvelope(envelopeDefinition);

  return this.db.signingEnvelope.create({
    data: {
      tenantId: resolution.tenantId,
      provider: 'DOCUSIGN',
      externalEnvelopeId: result.envelopeId,
      emailSubject: envelopeDefinition.emailSubject,
      status: 'SENT',
      signatureStandard: 'ESIGN',
      resolutionId: resolution.id,
      providerData: {
        recipients: signers.map(s => ({
          name: s.full_name,
          email: s.email,
          status: 'SENT',
        })),
        sentAt: new Date().toISOString(),
      },
    },
  });
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create signing envelope for resolution | Integration | Envelope created with provider='DOCUSIGN', linked to resolution |
| DocuSign API called with correct envelope definition | Unit (mocked) | Mock verifies envelope has correct document, signers, tabs |
| Webhook callback updates recipient status to SIGNED | Integration | POST to webhook endpoint updates `provider_data.recipients[0].status` to 'SIGNED' |
| All recipients signed -> envelope COMPLETED | Integration | When last signer signs, envelope status changes to 'COMPLETED' |
| Recipient declines -> envelope DECLINED | Integration | Decline webhook updates status; decline reason recorded |
| Resolution status updated when signing completes | Integration | Resolution status transitions to 'PASSED' when envelope 'COMPLETED' |
| Signature standard stored (ESIGN vs eIDAS) | Integration | EU entities use `signature_standard='EIDAS_ADES'` |

---

### Phase 5 Definition of Done

- [ ] Resolution CRUD with type support (board, shareholder ordinary, shareholder special, written consent)
- [ ] Voting workflow with configurable approval thresholds
- [ ] Vote tally and outcome determination
- [ ] Resolution number auto-generation
- [ ] DocuSign integration for sending resolutions for e-signature
- [ ] Webhook receiver for DocuSign status callbacks
- [ ] Resolution detail page showing vote breakdown and signature status
- [ ] Written consent workflow (no meeting, all directors sign)
- [ ] Full resolution lifecycle E2E test

---

## Phase 6: Document Management

**Goal:** Build a secure, version-controlled document library with granular access controls, board pack generation, annotations, and document viewer.

**Depends on:** Phase 1 (can run in parallel with Phases 2-3)

### Task 6.1: Document Storage and Version Control

**What:** Create the `document` and `document_permission` tables. Integrate with AWS S3 for encrypted file storage. Implement version control (new versions linked via `parent_document_id`).

**Design:**

```sql
-- Migration: 012_document.sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID REFERENCES legal_entity(id),
    title           VARCHAR(500) NOT NULL,
    document_type   VARCHAR(50) NOT NULL,
    category_code   VARCHAR(50),
    version         INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id UUID REFERENCES document(id),
    meeting_id      UUID REFERENCES meeting(id),
    confidentiality VARCHAR(20) NOT NULL DEFAULT 'CONFIDENTIAL',
    storage_key     VARCHAR(500) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT,
    checksum_sha256 VARCHAR(64),
    doc_metadata    JSONB NOT NULL DEFAULT '{}',
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE document_permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES app_user(id),
    role            VARCHAR(50),
    committee_id    UUID REFERENCES committee(id),
    permission      VARCHAR(20) NOT NULL,
    granted_by      UUID REFERENCES app_user(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    CONSTRAINT has_grantee CHECK (
        user_id IS NOT NULL OR role IS NOT NULL OR committee_id IS NOT NULL
    )
);
```

```typescript
// apps/api/src/modules/document/document.service.ts
async uploadDocument(
  file: Express.Multer.File,
  metadata: CreateDocumentDto,
  user: AuthUser,
): Promise<Document> {
  const checksum = crypto
    .createHash('sha256')
    .update(file.buffer)
    .digest('hex');

  const storageKey = `${user.tenantId}/${metadata.entityId}/${uuid()}/${file.originalname}`;

  await this.s3Client.send(new PutObjectCommand({
    Bucket: this.configService.get('S3_BUCKET'),
    Key: storageKey,
    Body: file.buffer,
    ContentType: file.mimetype,
    ServerSideEncryption: 'aws:kms',
    SSEKMSKeyId: this.configService.get('KMS_KEY_ID'),
  }));

  return this.db.document.create({
    data: {
      tenantId: user.tenantId,
      entityId: metadata.entityId,
      title: metadata.title,
      documentType: metadata.documentType,
      storageKey,
      fileName: file.originalname,
      mimeType: file.mimetype,
      fileSizeBytes: file.size,
      checksumSha256: checksum,
      uploadedBy: user.id,
      docMetadata: {
        encryption: { encrypted_at_rest: true },
      },
    },
  });
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Upload PDF document | Integration | Document record created; file stored in S3 with KMS encryption |
| Upload new version | Integration | New document created with `parent_document_id` set; previous version's `is_current_version` set to false |
| Download document with valid permission | Integration | Presigned S3 URL returned; file downloadable |
| Download blocked without permission | Integration | User without document_permission receives 403 |
| Grant permission to committee | Integration | All committee members can access the document |
| Permission with expiry | Integration | After `expires_at`, access is denied |
| SHA-256 checksum verified on download | Integration | Checksum of downloaded file matches stored `checksum_sha256` |
| Document version history | Integration | API returns all versions of a document ordered by version number |
| Document search by type and entity | Integration | Filter by `document_type='MINUTES'` and `entity_id` returns correct results |
| Audit log records document views | Integration | Viewing a document creates audit entry with event_class='document.viewed' |

---

### Task 6.2: Document Viewer and Annotations

**What:** Build an in-browser PDF viewer with private annotation support (highlights, notes, bookmarks). Directors can annotate board materials without other directors seeing their notes.

**Design:**

```typescript
// apps/web/src/components/document/document-viewer.tsx
export function DocumentViewer({
  documentId,
  onAnnotation,
}: {
  documentId: string;
  onAnnotation: (annotation: Annotation) => void;
}) {
  const { document, annotations } = useDocument(documentId);
  const [currentPage, setCurrentPage] = useState(1);

  return (
    <div className="flex h-full">
      <div className="flex-1">
        <PdfRenderer
          url={document.presignedUrl}
          currentPage={currentPage}
          annotations={annotations}
          onPageChange={setCurrentPage}
          onAnnotationCreate={onAnnotation}
        />
      </div>
      <AnnotationSidebar
        annotations={annotations}
        currentPage={currentPage}
        onNavigate={setCurrentPage}
      />
    </div>
  );
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| PDF renders in browser viewer | E2E | Document pages visible; navigation between pages works |
| Create highlight annotation | E2E | Highlight appears on page; stored in database |
| Create text note annotation | E2E | Note icon appears at position; clicking shows note content |
| Annotations are private by default | Integration | Director A's annotations not visible to Director B |
| Annotation sidebar lists all notes for current page | E2E | Sidebar shows annotations; clicking one navigates to its position |
| Delete annotation | E2E | Annotation removed from view and database |

---

### Phase 6 Definition of Done

- [ ] Document upload with S3 storage and KMS encryption
- [ ] Version control with version history view
- [ ] Granular permissions (user, role, committee level)
- [ ] In-browser PDF viewer
- [ ] Private annotation support (highlights, notes, bookmarks)
- [ ] Document search and filtering by type, entity, and meeting
- [ ] Audit logging for all document access events
- [ ] Document download with integrity verification (SHA-256)

---

## Phase 7: Compliance and Filing Management

**Goal:** Build jurisdiction-specific compliance calendars, filing deadline tracking, automated alerts, and compliance dashboard.

**Depends on:** Phase 2

### Task 7.1: Compliance Obligation Templates and Instances

**What:** Create the compliance obligation tracking system with jurisdiction-specific templates and entity-level obligation instances. Implement deadline calculation engine.

**Design:**

```sql
-- Migration: 013_compliance.sql
CREATE TABLE compliance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    name            VARCHAR(500) NOT NULL,
    obligation_type VARCHAR(50) NOT NULL,
    jurisdiction_code VARCHAR(6) NOT NULL,
    due_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'UPCOMING',
    obligation_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_entity ON compliance_obligation(tenant_id, entity_id);
CREATE INDEX idx_compliance_due ON compliance_obligation(tenant_id, due_date, status);
CREATE INDEX idx_compliance_status ON compliance_obligation(tenant_id, status);
```

```typescript
// apps/api/src/modules/compliance/deadline.service.ts
async generateObligations(entityId: string): Promise<void> {
  const entity = await this.db.legalEntity.findUnique({
    where: { id: entityId },
  });

  const templates = await this.getTemplatesForJurisdiction(
    entity.jurisdictionCode,
    entity.legalFormCode,
  );

  for (const template of templates) {
    const dueDate = this.calculateDeadline(template, entity);
    await this.db.complianceObligation.create({
      data: {
        tenantId: entity.tenantId,
        entityId: entity.id,
        name: template.name,
        obligationType: template.obligationType,
        jurisdictionCode: entity.jurisdictionCode,
        dueDate,
        obligationData: {
          template_source: 'jurisdiction_rules',
          frequency: template.frequency,
          filing_authority: template.filingAuthority,
          filing_url: template.filingUrl,
          penalty: template.penalty,
        },
      },
    });
  }
}

private calculateDeadline(
  template: ComplianceTemplate,
  entity: LegalEntity,
): Date {
  if (template.deadlineMonthsAfterYearEnd && entity.fiscalYearEndMonth) {
    const yearEndDate = new Date(
      new Date().getFullYear(),
      entity.fiscalYearEndMonth - 1,
      entity.fiscalYearEndDay || 28,
    );
    return addMonths(yearEndDate, template.deadlineMonthsAfterYearEnd);
  }
  if (template.deadlineFixedMonth && template.deadlineFixedDay) {
    return new Date(
      new Date().getFullYear(),
      template.deadlineFixedMonth - 1,
      template.deadlineFixedDay,
    );
  }
  throw new Error(`Cannot calculate deadline for template ${template.id}`);
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Generate obligations for Delaware LLC | Integration | Creates Annual Report and Franchise Tax obligations with correct due dates |
| Generate obligations for UK Ltd | Integration | Creates Confirmation Statement and Annual Accounts obligations |
| Deadline calculation: 3 months after fiscal year end | Unit | FY end Dec 31 + 3 months = March 31 |
| Deadline calculation: fixed date (March 1) | Unit | Delaware franchise tax due March 1 regardless of FY end |
| Compliance dashboard shows upcoming deadlines | Integration | Sorted by due_date, with days-until-due computed |
| Status transition: UPCOMING -> DUE_SOON (14 days) | Integration | Scheduled job moves obligations within 14 days to DUE_SOON |
| Status transition: DUE_SOON -> OVERDUE | Integration | Obligations past due_date moved to OVERDUE |
| Mark obligation as filed | Integration | Status='FILED', `obligation_data.filed_on` and confirmation number recorded |
| Compliance alerts sent for DUE_SOON | Integration | Email notification sent to entity's secretary when status changes to DUE_SOON |

---

### Task 7.2: Compliance Dashboard

**What:** Build a compliance dashboard showing all obligations across entities with traffic-light status indicators, filtering by jurisdiction and entity, and export to CSV/Excel.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Dashboard shows obligations grouped by status | E2E | OVERDUE (red), DUE_SOON (amber), UPCOMING (green) sections visible |
| Filter by jurisdiction | E2E | Selecting 'GB' shows only UK obligations |
| Filter by entity | E2E | Selecting specific entity shows only that entity's obligations |
| Export to CSV | E2E | CSV download contains all visible obligations with correct columns |
| Click obligation navigates to detail | E2E | Clicking an obligation opens detail view with filing information |

---

### Phase 7 Definition of Done

- [ ] Compliance obligation templates for US-DE, GB, SG jurisdictions
- [ ] Automatic obligation generation when entities are created
- [ ] Deadline calculation engine for fiscal-year-relative and fixed-date obligations
- [ ] Status lifecycle (UPCOMING -> DUE_SOON -> OVERDUE, or FILED -> COMPLETED)
- [ ] Compliance dashboard with traffic-light indicators
- [ ] Email alerts for DUE_SOON and OVERDUE status transitions
- [ ] CSV/Excel export of compliance data
- [ ] Filing confirmation recording with reference numbers

---

## Phase 8: AI-Powered Governance

**Goal:** Implement AI-driven minutes drafting from transcripts, document summarisation for pre-meeting briefs, and resolution generation from precedent libraries.

**Depends on:** Phase 4

### Task 8.1: AI Minutes Drafting

**What:** Integrate the Claude API to generate structured meeting minutes from uploaded audio transcripts or typed notes. Minutes are mapped to agenda items with decisions, action items, and attendance automatically extracted.

**Design:**

```typescript
// apps/api/src/modules/ai/minutes-drafting.service.ts
async generateMinutes(meetingId: string): Promise<DraftMinutes> {
  const meeting = await this.db.meeting.findUnique({
    where: { id: meetingId },
    include: {
      agendaItems: { orderBy: { sortOrder: 'asc' } },
      attendance: { include: { person: true } },
      entity: true,
      committee: true,
    },
  });

  const transcript = meeting.meetingData?.transcript?.text
    ?? meeting.meetingData?.notes;

  const response = await this.anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 8192,
    system: MINUTES_SYSTEM_PROMPT,
    messages: [{
      role: 'user',
      content: `Generate formal board meeting minutes from the following transcript.

Meeting: ${meeting.title}
Entity: ${meeting.entity.legalName}
Date: ${meeting.scheduledStart}
Committee: ${meeting.committee?.name ?? 'Board of Directors'}

Agenda Items:
${meeting.agendaItems.map(ai => `${ai.itemNumber}. ${ai.title} (${ai.itemType})`).join('\n')}

Attendees Present:
${meeting.attendance.filter(a => a.attended).map(a => a.person.fullName).join('\n')}

Apologies:
${meeting.attendance.filter(a => !a.attended).map(a => a.person.fullName).join('\n')}

Transcript/Notes:
${transcript}

Output structured minutes mapping each discussion point to its agenda item.
For each item, extract: decisions made, action items (with assignee and due date), and any resolutions proposed or voted on.`,
    }],
  });

  const minutesText = response.content[0].text;
  const actionItems = this.extractActionItems(minutesText);

  // Save draft minutes
  await this.db.meeting.update({
    where: { id: meetingId },
    data: {
      meetingData: {
        ...meeting.meetingData,
        minutes: {
          draft_text: minutesText,
          ai_generated: true,
          ai_confidence: 0.92,
          generated_at: new Date().toISOString(),
        },
      },
    },
  });

  // Create extracted action items
  for (const item of actionItems) {
    await this.db.actionItem.create({
      data: {
        tenantId: meeting.tenantId,
        meetingId: meeting.id,
        agendaItemId: item.agendaItemId,
        description: item.description,
        assignedTo: item.assigneeId,
        dueDate: item.dueDate,
        aiData: { ai_extracted: true, confidence: item.confidence },
      },
    });
  }

  return { minutesText, actionItems };
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Generate minutes from transcript | Integration | Draft minutes text saved to `meeting_data.minutes.draft_text` |
| Minutes mapped to agenda items | Integration | Each agenda item's discussion captured under its heading |
| Action items extracted | Integration | At least one action item created with `ai_data.ai_extracted = true` |
| Action item has assignee and due date | Integration | Extracted action item includes `assigned_to` person_id and `due_date` |
| AI confidence score stored | Integration | `meeting_data.minutes.ai_confidence` between 0 and 1 |
| Minutes editable after generation | E2E | Secretary can edit AI-generated draft in rich text editor |
| No transcript returns helpful error | Unit | Service returns clear error message when no transcript or notes available |
| Rate limiting on AI calls | Unit | More than 5 calls per minute returns 429 |

---

### Task 8.2: Document Summarisation

**What:** Summarise lengthy board materials into pre-meeting briefing documents. Directors receive a 1-2 page summary of each board pack document before the meeting.

**Design:**

```typescript
// apps/api/src/modules/ai/summarisation.service.ts
async summariseDocument(documentId: string): Promise<DocumentSummary> {
  const document = await this.db.document.findUnique({
    where: { id: documentId },
  });

  const fileBuffer = await this.s3Service.getObject(document.storageKey);
  const textContent = await this.extractText(fileBuffer, document.mimeType);

  const response = await this.anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    messages: [{
      role: 'user',
      content: `Summarise the following board document for a director's pre-meeting briefing.

Document Title: ${document.title}
Document Type: ${document.documentType}

Content:
${textContent}

Provide:
1. A 2-3 sentence executive summary
2. Key points (bullet list, max 10 items)
3. Decisions or actions required from the board
4. Risk factors or concerns highlighted`,
    }],
  });

  const summary = response.content[0].text;

  await this.db.document.update({
    where: { id: documentId },
    data: {
      docMetadata: {
        ...document.docMetadata,
        ai: {
          summary,
          summarised_at: new Date().toISOString(),
          confidence: 0.90,
        },
      },
    },
  });

  return { documentId, summary };
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Summarise 50-page PDF board report | Integration | Summary stored in `doc_metadata.ai.summary`; length < 2000 chars |
| Summary includes key points list | Unit | Output contains numbered or bulleted list |
| Summary includes "decisions required" section | Unit | Output contains section on board actions needed |
| Batch summarise all documents in board pack | Integration | All documents for a meeting summarised; aggregate briefing compiled |
| Large document (>200 pages) handled | Integration | Document chunked; summary generated without timeout |
| Non-text document (image PDF) returns error | Unit | OCR fallback attempted; clear error if text extraction fails |

---

### Task 8.3: Resolution Generation from Precedents

**What:** Generate draft resolutions from a library of precedent templates, customised with entity-specific details. AI fills in variable fields (entity name, officer name, dates, financial figures) from the entity register.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Generate "Appointment of Director" resolution | Integration | Draft includes entity name, director name, appointment date, role |
| Generate "Dividend Declaration" resolution | Integration | Draft includes share class, per-share amount, record date, payment date |
| Precedent library contains 20+ templates | Integration | Reference data seeded with resolution precedent types |
| Generated resolution editable before finalisation | E2E | Secretary can modify AI-generated text in editor |
| AI uses correct jurisdiction-specific language | Unit | UK entity gets "Companies Act 2006" references; US entity gets state corporation law references |

---

### Phase 8 Definition of Done

- [ ] AI minutes drafting from transcript/notes with agenda-item mapping
- [ ] Action item auto-extraction from AI-generated minutes
- [ ] Document summarisation for pre-meeting briefs
- [ ] Batch summarisation of all board pack documents
- [ ] Resolution generation from precedent library
- [ ] AI-generated content clearly marked as AI-drafted
- [ ] Secretary review and edit workflow for all AI outputs
- [ ] Rate limiting and cost controls on AI API calls
- [ ] AI prompt templates versioned in `packages/ai-prompts`

---

## Phase 9: Share Register and Cap Table

**Goal:** Build a share register with share class management, transaction history (OCF-aligned), and current holdings computation.

**Depends on:** Phase 5

### Task 9.1: Share Classes and Transactions

**What:** Create `share_class` and `share_transaction` tables aligned with Open Cap Format. Build UI for recording issuances, transfers, cancellations, and computing current holdings.

**Design:**

```sql
-- Migration: 014_shares.sql
CREATE TABLE share_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    name            VARCHAR(255) NOT NULL,
    class_type      VARCHAR(20) NOT NULL,
    shares_authorised BIGINT,
    par_value       DECIMAL(12,6),
    par_value_currency VARCHAR(3),
    votes_per_share DECIMAL(10,2) NOT NULL DEFAULT 1.0,
    class_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE share_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_id       UUID NOT NULL REFERENCES legal_entity(id),
    share_class_id  UUID NOT NULL REFERENCES share_class(id),
    transaction_type VARCHAR(30) NOT NULL,
    from_stakeholder_id UUID REFERENCES person(id),
    to_stakeholder_id UUID REFERENCES person(id),
    quantity        BIGINT NOT NULL,
    transaction_date DATE NOT NULL,
    resolution_id   UUID REFERENCES resolution(id),
    transaction_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// apps/api/src/modules/share/share.service.ts
async getCurrentHoldings(entityId: string): Promise<ShareHolding[]> {
  return this.db.$queryRaw`
    SELECT
      sc.name AS share_class_name,
      sc.class_type,
      p.full_name AS stakeholder_name,
      SUM(CASE
        WHEN st.transaction_type = 'ISSUANCE' AND st.to_stakeholder_id = p.id THEN st.quantity
        WHEN st.transaction_type = 'TRANSFER' AND st.to_stakeholder_id = p.id THEN st.quantity
        WHEN st.transaction_type = 'TRANSFER' AND st.from_stakeholder_id = p.id THEN -st.quantity
        WHEN st.transaction_type = 'CANCELLATION' AND st.from_stakeholder_id = p.id THEN -st.quantity
        WHEN st.transaction_type = 'REPURCHASE' AND st.from_stakeholder_id = p.id THEN -st.quantity
        ELSE 0
      END) AS shares_held
    FROM share_transaction st
    JOIN share_class sc ON sc.id = st.share_class_id
    JOIN person p ON p.id = COALESCE(st.to_stakeholder_id, st.from_stakeholder_id)
    WHERE st.entity_id = ${entityId}
    GROUP BY sc.id, sc.name, sc.class_type, p.id, p.full_name
    HAVING SUM(CASE
      WHEN st.transaction_type = 'ISSUANCE' AND st.to_stakeholder_id = p.id THEN st.quantity
      WHEN st.transaction_type = 'TRANSFER' AND st.to_stakeholder_id = p.id THEN st.quantity
      WHEN st.transaction_type = 'TRANSFER' AND st.from_stakeholder_id = p.id THEN -st.quantity
      WHEN st.transaction_type = 'CANCELLATION' AND st.from_stakeholder_id = p.id THEN -st.quantity
      WHEN st.transaction_type = 'REPURCHASE' AND st.from_stakeholder_id = p.id THEN -st.quantity
      ELSE 0
    END) > 0
    ORDER BY sc.name, shares_held DESC
  `;
}
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Create share class (Common Stock) | Integration | Share class created with type='COMMON', 10M authorised |
| Create preferred share class with liquidation preference | Integration | `class_data` contains `liquidation_preference_multiple: 1.0` |
| Record share issuance | Integration | Transaction created with type='ISSUANCE', quantity, price, to_stakeholder |
| Record share transfer | Integration | Transfer from person A to person B recorded |
| Current holdings computed correctly | Integration | After 1M issuance + 200K transfer, issuer holds 800K, transferee holds 200K |
| Cannot transfer more shares than held | Unit | Service rejects transfer where from_stakeholder holds fewer shares than requested |
| Cap table view shows percentages | E2E | Cap table displays each stakeholder's share count and percentage of total |
| Transaction linked to authorising resolution | Integration | `resolution_id` set on share issuance transaction |
| Share issuance cannot exceed authorised | Unit | Issuance that would exceed `shares_authorised` rejected |

---

### Phase 9 Definition of Done

- [ ] Share class CRUD with common and preferred stock support
- [ ] Transaction recording (issuance, transfer, cancellation, repurchase)
- [ ] Current holdings computed from transaction history
- [ ] Cap table view with ownership percentages
- [ ] Transaction history for each stakeholder
- [ ] Authorisation validation (cannot exceed authorised shares)
- [ ] Transfer validation (cannot transfer more than held)
- [ ] Transactions linked to authorising resolutions

---

## Phase 10: External Integrations

**Goal:** Integrate with external services: SSO providers (SAML/OAuth), SEC EDGAR API, Companies House API, video conferencing (Zoom/Teams), and email notifications.

**Depends on:** Phase 1 (starts early; connects to later phases incrementally)

### Task 10.1: SSO Integration (SAML 2.0 / Azure AD)

**What:** Add SAML 2.0 authentication provider support to NextAuth.js. Support Azure AD, Okta, and generic SAML IdPs. Enable tenant-specific SSO configuration.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| SAML login with Azure AD | E2E | User redirected to Azure AD; after authentication, session created with correct tenant and role |
| SAML IdP metadata stored per tenant | Integration | Tenant config contains IdP metadata URL, entity ID, certificate |
| SAML assertion maps to correct user role | Integration | SAML attribute `role=Secretary` maps to platform role `secretary` |
| SSO-only tenant rejects email/password login | Integration | Email/password auth disabled for SSO-only tenants |
| JIT provisioning creates new user on first login | Integration | First SAML login for unknown email creates app_user record |

---

### Task 10.2: SEC EDGAR Integration

**What:** Build a polling service that retrieves public company filing data from the SEC EDGAR REST API (`data.sec.gov`). Auto-populate entity identifiers (CIK, SIC code) and monitor peer company filings for competitive intelligence.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Look up company by CIK | Integration | EDGAR API returns company name, SIC code, filing history |
| Auto-populate entity identifiers from EDGAR | Integration | Entity's `identifiers.cik` and `identifiers.sic_code` set from EDGAR data |
| Rate limiting respected (10 req/sec) | Unit | Client throttles to stay under EDGAR's 10 requests/second limit |
| Filing history displayed for entity | E2E | Recent 10-K, 10-Q filings shown on entity detail page |

---

### Task 10.3: Companies House API Integration (UK)

**What:** Integrate with the UK Companies House API for officer data, filing history, and PSC (Persons with Significant Control) records. Enable auto-refresh of UK entity data.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Fetch company profile by company number | Integration | Returns company name, status, registered address, date of incorporation |
| Fetch officer list | Integration | Returns current and resigned officers with appointment dates |
| Fetch PSC data | Integration | Returns beneficial owners with nature of control |
| Auto-sync officer register from Companies House | Integration | New officers from CH API added to person and officer_appointment tables |
| Companies House data marked with source='companies_house' | Integration | `jurisdiction_data` includes source attribution |

---

### Task 10.4: Email Notifications

**What:** Build a transactional email service for meeting invitations, board pack distribution, compliance alerts, action item reminders, and resolution voting notifications.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Meeting invitation email sent to all attendees | Integration | Email sent with meeting title, date, location, agenda summary |
| Board pack ready notification | Integration | Email includes link to view board pack |
| Compliance alert for DUE_SOON obligation | Integration | Email includes entity name, obligation, due date, and filing URL |
| Action item reminder for overdue items | Integration | Weekly digest email lists all overdue action items assigned to the person |
| Unsubscribe from non-critical notifications | Integration | User can disable optional notifications in profile settings |

---

### Phase 10 Definition of Done

- [ ] SAML 2.0 SSO working with Azure AD and Okta
- [ ] JIT user provisioning on first SSO login
- [ ] SEC EDGAR API integration for public company data
- [ ] Companies House API integration for UK entity data
- [ ] Transactional email notifications for key workflow events
- [ ] Email template management with tenant branding
- [ ] Integration health monitoring dashboard

---

## Phase 11: Graph Analytics and Governance Intelligence

**Goal:** Build graph-based analytics for ownership chain analysis, director network mapping, conflict-of-interest detection, and AI-powered governance risk scoring.

**Depends on:** Phase 3

### Task 11.1: Graph Layer (PostgreSQL ltree + Recursive CTEs)

**What:** Add `ownership_path` (ltree) column to legal_entity. Build graph query functions for ownership chain traversal, director overlap analysis, and beneficial ownership tracing.

**Design:**

```sql
-- Migration: 015_graph.sql
ALTER TABLE legal_entity ADD COLUMN ownership_path LTREE;
CREATE INDEX idx_entity_path ON legal_entity USING GIST (ownership_path);

-- Function: update ownership path when relationships change
CREATE OR REPLACE FUNCTION update_ownership_paths(root_entity_id UUID)
RETURNS VOID AS $$
DECLARE
  root_path LTREE;
BEGIN
  root_path := replace(root_entity_id::TEXT, '-', '_');

  -- Set root entity's path
  UPDATE legal_entity SET ownership_path = root_path
  WHERE id = root_entity_id;

  -- Recursively set paths for children
  WITH RECURSIVE tree AS (
    SELECT child_entity_id, root_path || replace(child_entity_id::TEXT, '-', '_') AS path
    FROM entity_relationship
    WHERE parent_entity_id = root_entity_id
      AND status = 'ACTIVE'
      AND relationship_type = 'IS_DIRECTLY_CONSOLIDATED_BY'

    UNION ALL

    SELECT er.child_entity_id, t.path || replace(er.child_entity_id::TEXT, '-', '_')
    FROM entity_relationship er
    JOIN tree t ON er.parent_entity_id = (
      SELECT id FROM legal_entity WHERE ownership_path = t.path
    )
    WHERE er.status = 'ACTIVE'
  )
  UPDATE legal_entity le SET ownership_path = tree.path
  FROM tree WHERE le.id = tree.child_entity_id;
END;
$$ LANGUAGE plpgsql;
```

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| ltree path set for 3-level hierarchy | Integration | Root = 'A', child = 'A.B', grandchild = 'A.B.C' |
| Descendant query using ltree | Integration | `WHERE ownership_path <@ 'A'` returns A, B, C |
| Ancestor query using ltree | Integration | `WHERE 'A.B.C' <@ ownership_path` returns A, B, C |
| Director overlap query | Integration | Persons serving on 2+ entity boards returned with entity list |
| Conflict of interest detection | Integration | Persons serving on boards of entities with ownership relationships flagged |
| Ownership effective percentage calculation | Integration | Through 80% -> 60% chain, effective = 48% |

---

### Task 11.2: Governance Risk Scoring

**What:** Build an AI-powered governance risk scoring engine that analyses board composition, meeting attendance, compliance filing history, director tenure, and ownership complexity to produce risk scores per entity.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Risk score computed for entity | Integration | Score between 0.0 (low risk) and 1.0 (high risk) stored in `ai_metadata.last_risk_score` |
| Low independence ratio increases risk | Unit | Entity with <33% independent directors scores higher risk |
| Poor attendance increases risk | Unit | Entity with <75% average board attendance scores higher risk |
| Overdue compliance obligations increase risk | Unit | Entity with 3+ OVERDUE obligations scores higher risk |
| Risk factors listed | Integration | `ai_metadata.risk_factors` array contains specific factor labels |
| Risk dashboard shows all entities ranked by score | E2E | Table sorted by risk score descending; colour-coded rows |

---

### Phase 11 Definition of Done

- [ ] ltree-based ownership hierarchy with efficient ancestor/descendant queries
- [ ] Director overlap analysis
- [ ] Conflict of interest detection via graph relationships
- [ ] Beneficial ownership chain tracing
- [ ] Governance risk scoring engine
- [ ] Risk dashboard with entity rankings
- [ ] Corporate structure visualisation with depth and ownership percentages

---

## Phase 12: Production Hardening and Compliance Certification

**Goal:** Prepare the platform for production deployment with security hardening, performance optimisation, compliance documentation, and operational tooling.

**Depends on:** All functional phases (1-11)

### Task 12.1: Security Hardening

**What:** Conduct security review, implement CSP headers, rate limiting, input sanitisation, SQL injection prevention audit, CSRF protection, and dependency vulnerability scanning. Prepare for SOC 2 Type II audit readiness.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| CSP headers set on all responses | E2E | Response headers include `Content-Security-Policy` |
| Rate limiting enforced on API endpoints | Integration | 101st request within window returns 429 |
| SQL injection attempt rejected | Integration | Parameterised queries prevent injection; malicious input sanitised |
| XSS payload in entity name sanitised | Integration | `<script>alert(1)</script>` stored as escaped text, not executable |
| Dependency vulnerability scan passes | CI | No critical or high CVEs in dependency tree |
| HTTPS enforced (HTTP redirects to HTTPS) | E2E | HTTP request returns 301 redirect to HTTPS |
| Session timeout after inactivity | E2E | After 30 minutes inactive, user redirected to login |

---

### Task 12.2: Performance Optimisation

**What:** Add database query optimisation (materialised views for compliance dashboard, connection pooling), frontend bundle optimisation, CDN configuration, and caching strategy (Redis for session and reference data).

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Entity list page loads in <500ms | E2E | Lighthouse performance score >= 90 |
| Compliance dashboard with 1000 obligations loads in <1s | Integration | Query execution time < 200ms with materialised view |
| Board pack PDF generation completes in <30s for 200 pages | Integration | PDF assembly and S3 upload within timeout |
| Concurrent 50-user load test | Integration | p95 response time < 1s; no errors |
| Reference data cached in Redis | Integration | Second request served from cache; no DB query |

---

### Task 12.3: Data Residency and Backup

**What:** Implement data residency controls (US, EU, APAC region selection per tenant), automated database backups with point-in-time recovery, and disaster recovery documentation.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| EU tenant data stored in EU region | Integration | S3 bucket and database replica in eu-west-1 |
| Daily backup completes successfully | Integration | pg_dump or WAL archiving produces restorable backup |
| Point-in-time recovery tested | Integration | Database restored to 2-hour-old state with data intact |
| Backup encryption at rest | Integration | Backup files encrypted with KMS key |

---

### Task 12.4: Compliance Documentation

**What:** Generate SOC 2 readiness documentation, data processing agreements (DPA) for GDPR, privacy policy, security architecture diagrams, and incident response procedures.

**Testing:**

| Test Case | Type | Expectation |
|-----------|------|-------------|
| Access control matrix documented | Review | All roles and permissions documented and match implementation |
| Data retention policy enforced | Integration | Audit log partitions older than retention period are archived/dropped |
| GDPR data subject access request workflow | E2E | Admin can export all data for a person (DSAR compliance) |
| Data deletion request workflow | E2E | Admin can delete a person's PII with cascade to audit log anonymisation |
| Encryption in transit and at rest documented | Review | TLS 1.3 for transit; AES-256 for S3 and database at rest |

---

### Phase 12 Definition of Done

- [ ] Security review completed with all critical findings resolved
- [ ] CSP, rate limiting, CSRF, and input sanitisation in place
- [ ] Performance benchmarks met (p95 < 1s under 50 concurrent users)
- [ ] Data residency controls operational for US/EU/APAC
- [ ] Automated backups with tested point-in-time recovery
- [ ] SOC 2 readiness documentation complete
- [ ] GDPR DSAR and data deletion workflows operational
- [ ] Incident response procedures documented
- [ ] Monitoring and alerting configured (uptime, error rates, latency)
- [ ] Production deployment checklist validated

---

## Definition of Done (Global)

Every phase must meet these criteria before being considered complete:

1. **Code Quality:** All code passes linting (ESLint), type-checking (TypeScript strict mode), and formatting (Prettier).
2. **Test Coverage:** Minimum 85% line coverage on business logic modules; 100% coverage on security-critical code (auth, permissions, RLS).
3. **Integration Tests:** All database operations tested against real PostgreSQL with RLS policies active.
4. **E2E Tests:** Critical user workflows covered by Playwright tests.
5. **Audit Logging:** All mutating operations produce audit log entries with correct category, class, activity, and actor.
6. **RLS Enforcement:** Every tenant-scoped table has Row-Level Security enabled and verified in tests.
7. **API Documentation:** GraphQL schema documented with descriptions on all types, queries, and mutations.
8. **Accessibility:** WCAG 2.1 AA compliance on all user-facing pages (verified via axe-core).
9. **Security:** No known critical or high-severity vulnerabilities in dependencies.
10. **Review:** Code reviewed by at least one other developer; architecture decisions documented in ADRs.
