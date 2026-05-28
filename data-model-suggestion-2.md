# Data Model Suggestion 2: Event-Sourced / Audit-First Model

> Project: Corporate Secretary Platform · Created: 2026-05-12

## Philosophy

Every change to every entity is modelled as an immutable **event** in an append-only log. The "current state" of any object is derived by replaying its events. This aligns naturally with corporate governance, where auditability is non-negotiable: every officer appointment, share transfer, resolution vote, and document upload must be traceable to who authorised it, when, and under what authority.

This pattern is already proven in the domain:
- **Open Cap Format (OCF)** models cap tables as a sequence of typed transactions (`TX_STOCK_ISSUANCE`, `TX_STOCK_TRANSFER`, etc.) — current holdings are derived from the transaction history.
- **Government corporate registries** (Companies House UK, SEC EDGAR, ASIC) are fundamentally event-sourced: a filing history is the source of truth, and the company profile is a projection from that history.
- **GLEIF Level 2 data** carries explicit period start/end dates and registration status transitions — each relationship record is effectively an event with temporal bounds.
- **OCSF** (Open Cybersecurity Schema Framework) defines all events as typed, immutable records with a shared base schema — a pattern we extend to governance events.
- **BODS v0.4** structures beneficial ownership as timestamped statements, each representing a claim at a point in time — a natural fit for event sourcing.

The architecture follows **CQRS** (Command Query Responsibility Segregation): writes go to the event store as immutable events, reads come from materialised projections optimised for specific query patterns. This separation enables time-travel queries ("what was the board composition on 15 March 2024?"), full auditability without a separate audit log, and AI-powered compliance monitoring that analyses change patterns over time.

**Best for:** Regulated environments where complete auditability is required; platforms that need to reconstruct historical state at any point in time; AI-powered compliance monitoring that analyses change patterns; organisations subject to multiple jurisdictional regulators who may request different historical snapshots.

**Trade-offs:**
- (+) Complete audit trail is inherent — no separate audit log needed
- (+) Time-travel queries are trivial ("show me the state on any past date")
- (+) Natural fit for AI analytics — the event stream is a structured input for anomaly detection
- (+) Domain events map directly to government registry filing events
- (+) Event replay enables migration, disaster recovery, and schema evolution
- (-) Querying current state requires materialised views — more infrastructure
- (-) More storage (events are never deleted) — mitigated by compression and cheap storage
- (-) Developers need to understand event sourcing and CQRS patterns
- (-) Eventual consistency between event store and read models adds complexity
- (-) Event schema evolution requires versioning discipline

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCSF Base Event | Event envelope structure (category, class, activity, severity, timestamp) follows OCSF patterns |
| Open Cap Format (OCF) | Share/equity transaction events directly mirror OCF transaction types |
| GLEIF LEI-CDF 3.1 | LegalEntityEventType enum informs the entity event taxonomy |
| GLEIF RR-CDF 2.1 | Relationship period dates map to event temporal bounds |
| BODS v0.4 | Beneficial ownership as timestamped statements = events |
| Companies House Filing History | UK CH filing types map directly to governance event types |
| SEC EDGAR Submissions | EDGAR filing history = event stream per entity |
| ISO 15489 | Record retention applied per event type/category |
| DocuSign Envelope Status | Envelope lifecycle transitions map to signing events |
| ISO 5009 / GLEIF ELF | Officer role codes and legal form codes embedded in event payloads |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     WRITE SIDE (Commands)                     │
│                                                               │
│   Command → Validate → Produce Event(s) → Event Store         │
│                                                               │
│   "Appoint John as Director of Entity X"                      │
│       → Validate: entity exists, person exists, no conflict   │
│       → OfficerAppointedEvent stored immutably                │
│                                                               │
└─────────────────────┬───────────────────────────────────────┘
                      │ Event published (via LISTEN/NOTIFY or message bus)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                   EVENT STORE (Append-Only)                    │
│                                                               │
│   Single source of truth for all domain state                 │
│   Partitioned by: tenant → aggregate_type → aggregate_id      │
│   Supports replay from any point in time                      │
│   Never updated, never deleted (soft-archive after retention) │
│                                                               │
└─────────────────────┬───────────────────────────────────────┘
                      │ Projected by event processors
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                   READ SIDE (Projections)                      │
│                                                               │
│   ┌─────────────────┐  ┌──────────────────┐                  │
│   │ Entity Current   │  │ Meeting Current   │                  │
│   │ State View       │  │ State View        │                  │
│   └─────────────────┘  └──────────────────┘                  │
│   ┌─────────────────┐  ┌──────────────────┐                  │
│   │ Officer Register │  │ Compliance        │                  │
│   │ View             │  │ Dashboard View    │                  │
│   └─────────────────┘  └──────────────────┘                  │
│   ┌─────────────────┐  ┌──────────────────┐                  │
│   │ Share Register   │  │ AI Analytics      │                  │
│   │ View             │  │ View              │                  │
│   └─────────────────┘  └──────────────────┘                  │
│                                                               │
│   Each projection is rebuilt by replaying events              │
│   Projections are disposable — delete and rebuild any time    │
└─────────────────────────────────────────────────────────────┘
```

---

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (Source of Truth)
-- ============================================================

-- Core event store — append-only, immutable
CREATE TABLE domain_event (
    -- Event identity
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number     BIGINT GENERATED ALWAYS AS IDENTITY, -- Global ordering
    
    -- Tenant isolation
    tenant_id           UUID NOT NULL,
    
    -- Aggregate identification (DDD pattern)
    aggregate_type      VARCHAR(50) NOT NULL,
    -- 'LegalEntity', 'Person', 'Meeting', 'Resolution', 'Document',
    -- 'Committee', 'ShareClass', 'SigningEnvelope', 'ComplianceObligation'
    
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,                   -- Optimistic concurrency control
    
    -- Event type (hierarchical: category.class)
    event_category      VARCHAR(50) NOT NULL,
    -- 'ENTITY_MANAGEMENT', 'OFFICER_MANAGEMENT', 'MEETING_LIFECYCLE',
    -- 'RESOLUTION_WORKFLOW', 'DOCUMENT_MANAGEMENT', 'SHARE_REGISTER',
    -- 'COMPLIANCE', 'SIGNING', 'ACCESS_CONTROL'
    
    event_type          VARCHAR(100) NOT NULL,
    -- Entity events:
    --   'entity.incorporated', 'entity.name_changed', 'entity.address_changed',
    --   'entity.status_changed', 'entity.dissolved', 'entity.merged',
    --   'entity.lei_assigned', 'entity.relationship_established',
    --   'entity.relationship_terminated'
    -- Officer events:
    --   'officer.appointed', 'officer.resigned', 'officer.removed',
    --   'officer.role_changed', 'officer.address_changed'
    -- Meeting events:
    --   'meeting.scheduled', 'meeting.rescheduled', 'meeting.cancelled',
    --   'meeting.board_pack_sent', 'meeting.started', 'meeting.quorum_confirmed',
    --   'meeting.concluded', 'meeting.attendance_recorded',
    --   'meeting.minutes_drafted', 'meeting.minutes_approved'
    -- Resolution events:
    --   'resolution.proposed', 'resolution.vote_cast', 'resolution.passed',
    --   'resolution.failed', 'resolution.withdrawn'
    -- Document events:
    --   'document.uploaded', 'document.versioned', 'document.viewed',
    --   'document.downloaded', 'document.annotated', 'document.permission_granted',
    --   'document.permission_revoked', 'document.archived', 'document.destroyed'
    -- Share events (OCF-aligned):
    --   'share.class_created', 'share.issued', 'share.transferred',
    --   'share.cancelled', 'share.repurchased', 'share.split', 'share.converted'
    -- Compliance events:
    --   'compliance.obligation_created', 'compliance.deadline_approaching',
    --   'compliance.filing_submitted', 'compliance.filing_confirmed',
    --   'compliance.overdue', 'compliance.regulatory_change_detected'
    -- Signing events (DocuSign lifecycle):
    --   'signing.envelope_created', 'signing.envelope_sent',
    --   'signing.recipient_signed', 'signing.recipient_declined',
    --   'signing.envelope_completed', 'signing.envelope_voided'
    
    -- Event payload (the actual data)
    payload             JSONB NOT NULL,
    -- Example for 'officer.appointed':
    -- {
    --   "person_id": "uuid",
    --   "person_name": "John Smith",
    --   "entity_id": "uuid",
    --   "entity_name": "Acme Corp",
    --   "role_oor_code": "ABCDEF",
    --   "role_name": "Director",
    --   "appointed_on": "2026-03-15",
    --   "resolution_id": "uuid",
    --   "jurisdiction_code": "GB"
    -- }
    
    -- Metadata payload (non-domain data)
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",       -- The event or command that caused this
    --   "source": "web_ui",           -- 'web_ui', 'api', 'ai_agent', 'system', 'import'
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "...",
    --   "ai_generated": false,
    --   "ai_confidence": null
    -- }
    
    -- Actor (who caused this event)
    actor_user_id       UUID,
    actor_type          VARCHAR(20) NOT NULL DEFAULT 'USER',  -- 'USER', 'SYSTEM', 'AI_AGENT', 'API_KEY'
    
    -- Timestamp (immutable — when the event was recorded)
    event_time          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Schema version for payload evolution
    schema_version      INTEGER NOT NULL DEFAULT 1,
    
    -- Retention (ISO 15489)
    retention_class     VARCHAR(50),
    retain_until        DATE
    
) PARTITION BY RANGE (event_time);

-- Partitioned monthly
CREATE TABLE domain_event_2026_01 PARTITION OF domain_event
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE domain_event_2026_02 PARTITION OF domain_event
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created automatically

-- Primary access patterns
CREATE INDEX idx_event_aggregate ON domain_event(tenant_id, aggregate_type, aggregate_id, aggregate_version);
CREATE INDEX idx_event_type ON domain_event(tenant_id, event_type, event_time DESC);
CREATE INDEX idx_event_category ON domain_event(tenant_id, event_category, event_time DESC);
CREATE INDEX idx_event_sequence ON domain_event(sequence_number);
CREATE INDEX idx_event_actor ON domain_event(actor_user_id, event_time DESC) WHERE actor_user_id IS NOT NULL;
CREATE INDEX idx_event_correlation ON domain_event((metadata->>'correlation_id')) WHERE metadata->>'correlation_id' IS NOT NULL;

-- Optimistic concurrency: prevent duplicate aggregate versions
CREATE UNIQUE INDEX idx_event_aggregate_version 
    ON domain_event(tenant_id, aggregate_type, aggregate_id, aggregate_version);

-- Row-Level Security
ALTER TABLE domain_event ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON domain_event
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- ============================================================
-- EVENT SNAPSHOTS (Performance optimisation)
-- ============================================================

-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE aggregate_snapshot (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    aggregate_type      VARCHAR(50) NOT NULL,
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,
    
    -- Snapshot payload (current state as JSON)
    state               JSONB NOT NULL,
    
    snapshot_time       TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Unique: one snapshot per version
    UNIQUE(tenant_id, aggregate_type, aggregate_id, aggregate_version)
);

CREATE INDEX idx_snapshot_aggregate ON aggregate_snapshot(tenant_id, aggregate_type, aggregate_id, aggregate_version DESC);
```

---

## Multi-Tenant & User Tables (Operational — Not Event-Sourced)

```sql
-- ============================================================
-- OPERATIONAL TABLES (not event-sourced — pure infrastructure)
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'US',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    sso_provider_id VARCHAR(255),
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- Reference data (same as Model 1 — shared across tenants)
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(6) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES jurisdiction(id),
    iso_alpha3      VARCHAR(3),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE entity_legal_form (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    elf_code        VARCHAR(8) NOT NULL UNIQUE,
    name_local      VARCHAR(500) NOT NULL,
    name_english    VARCHAR(500),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE officer_role_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oor_code        VARCHAR(6) NOT NULL UNIQUE,
    name_local      VARCHAR(500) NOT NULL,
    name_english    VARCHAR(500),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model Projections

```sql
-- ============================================================
-- READ MODEL PROJECTIONS (Derived from events — disposable)
-- ============================================================

-- These tables are populated by event processors.
-- They can be dropped and rebuilt by replaying the event store.

-- Projection: Current state of all legal entities
CREATE TABLE v_entity_current (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    legal_name          VARCHAR(500) NOT NULL,
    trading_name        VARCHAR(500),
    lei                 VARCHAR(20),
    entity_legal_form_code VARCHAR(8),
    jurisdiction_code   VARCHAR(6) NOT NULL,
    registration_number VARCHAR(100),
    registration_date   DATE,
    entity_status       VARCHAR(20) NOT NULL,
    legal_address       JSONB,
    hq_address          JSONB,
    cik                 VARCHAR(10),
    fiscal_year_end     JSONB,                              -- { "month": 12, "day": 31 }
    last_event_version  INTEGER NOT NULL,
    last_event_time     TIMESTAMPTZ NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_entity_tenant ON v_entity_current(tenant_id);
CREATE INDEX idx_v_entity_status ON v_entity_current(tenant_id, entity_status);

-- Projection: Current officer register
CREATE TABLE v_officer_register (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    entity_id           UUID NOT NULL,
    entity_name         VARCHAR(500),
    person_id           UUID NOT NULL,
    person_name         VARCHAR(500),
    role_oor_code       VARCHAR(6),
    role_name           VARCHAR(255),
    appointed_on        DATE NOT NULL,
    resigned_on         DATE,
    status              VARCHAR(20) NOT NULL,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_officer_entity ON v_officer_register(tenant_id, entity_id);
CREATE INDEX idx_v_officer_person ON v_officer_register(tenant_id, person_id);
CREATE INDEX idx_v_officer_active ON v_officer_register(tenant_id, entity_id) WHERE status = 'ACTIVE';

-- Projection: Current entity relationships (ownership tree)
CREATE TABLE v_entity_relationship (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    child_entity_id     UUID NOT NULL,
    child_entity_name   VARCHAR(500),
    parent_entity_id    UUID NOT NULL,
    parent_entity_name  VARCHAR(500),
    relationship_type   VARCHAR(50) NOT NULL,
    ownership_percentage DECIMAL(5,2),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    status              VARCHAR(20) NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_rel_child ON v_entity_relationship(tenant_id, child_entity_id);
CREATE INDEX idx_v_rel_parent ON v_entity_relationship(tenant_id, parent_entity_id);

-- Projection: Current meeting state
CREATE TABLE v_meeting_current (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    entity_id           UUID NOT NULL,
    entity_name         VARCHAR(500),
    committee_id        UUID,
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL,
    quorum_present      BOOLEAN,
    agenda_items        JSONB,                              -- Denormalised for read performance
    attendance          JSONB,                              -- Denormalised for read performance
    action_items        JSONB,                              -- Denormalised for read performance
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_meeting_entity ON v_meeting_current(tenant_id, entity_id);
CREATE INDEX idx_v_meeting_date ON v_meeting_current(tenant_id, scheduled_start);

-- Projection: Share register (current holdings derived from transaction events)
CREATE TABLE v_share_holding (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    entity_id           UUID NOT NULL,
    share_class_id      UUID NOT NULL,
    share_class_name    VARCHAR(255),
    stakeholder_id      UUID NOT NULL,
    stakeholder_name    VARCHAR(500),
    shares_held         BIGINT NOT NULL,
    percentage_of_class DECIMAL(7,4),
    percentage_of_total DECIMAL(7,4),
    last_transaction_date DATE,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_holding_entity ON v_share_holding(tenant_id, entity_id);
CREATE INDEX idx_v_holding_stakeholder ON v_share_holding(tenant_id, stakeholder_id);

-- Projection: Compliance dashboard
CREATE TABLE v_compliance_dashboard (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    entity_id           UUID NOT NULL,
    entity_name         VARCHAR(500),
    jurisdiction_code   VARCHAR(6),
    obligation_name     VARCHAR(500),
    obligation_type     VARCHAR(50),
    due_date            DATE NOT NULL,
    status              VARCHAR(20) NOT NULL,
    days_until_due      INTEGER,
    filed_on            DATE,
    confirmation_number VARCHAR(255),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_compliance_due ON v_compliance_dashboard(tenant_id, due_date);
CREATE INDEX idx_v_compliance_status ON v_compliance_dashboard(tenant_id, status);

-- Projection: Document index
CREATE TABLE v_document_index (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    entity_id           UUID,
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50),
    category_code       VARCHAR(50),
    version             INTEGER NOT NULL,
    is_current_version  BOOLEAN NOT NULL,
    file_name           VARCHAR(255),
    mime_type           VARCHAR(100),
    file_size_bytes     BIGINT,
    meeting_id          UUID,
    confidentiality     VARCHAR(20),
    uploaded_by_name    VARCHAR(255),
    uploaded_at         TIMESTAMPTZ,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_doc_entity ON v_document_index(tenant_id, entity_id);
CREATE INDEX idx_v_doc_type ON v_document_index(tenant_id, document_type);
CREATE INDEX idx_v_doc_meeting ON v_document_index(meeting_id) WHERE meeting_id IS NOT NULL;

-- ============================================================
-- PROJECTION METADATA (Tracks rebuild state)
-- ============================================================

CREATE TABLE projection_metadata (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_event_sequence BIGINT NOT NULL DEFAULT 0,          -- Last processed event
    last_rebuilt_at     TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- 'ACTIVE', 'REBUILDING', 'ERROR'
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Processing / Projection Rebuild

```sql
-- ============================================================
-- EVENT PROCESSING QUERIES
-- ============================================================

-- Replay all events for a specific aggregate (e.g. rebuild entity state)
SELECT event_type, payload, metadata, event_time, aggregate_version
FROM domain_event
WHERE tenant_id = :tenant_id
  AND aggregate_type = 'LegalEntity'
  AND aggregate_id = :entity_id
ORDER BY aggregate_version ASC;

-- Replay from snapshot (performance optimisation)
-- 1. Load latest snapshot
SELECT state, aggregate_version
FROM aggregate_snapshot
WHERE tenant_id = :tenant_id
  AND aggregate_type = 'LegalEntity'
  AND aggregate_id = :entity_id
ORDER BY aggregate_version DESC
LIMIT 1;

-- 2. Replay events since snapshot
SELECT event_type, payload, metadata, event_time, aggregate_version
FROM domain_event
WHERE tenant_id = :tenant_id
  AND aggregate_type = 'LegalEntity'
  AND aggregate_id = :entity_id
  AND aggregate_version > :snapshot_version
ORDER BY aggregate_version ASC;

-- TIME TRAVEL: What was the board composition on a specific date?
SELECT de.payload->>'person_name' AS director,
       de.payload->>'role_name' AS role,
       de.payload->>'appointed_on' AS appointed_on,
       de.event_time AS recorded_at
FROM domain_event de
WHERE de.tenant_id = :tenant_id
  AND de.aggregate_type = 'LegalEntity'
  AND de.aggregate_id = :entity_id
  AND de.event_type = 'officer.appointed'
  AND de.event_time <= :as_of_date
  AND de.payload->>'person_id' NOT IN (
      -- Exclude officers who resigned before the as_of_date
      SELECT payload->>'person_id'
      FROM domain_event
      WHERE tenant_id = :tenant_id
        AND aggregate_type = 'LegalEntity'
        AND aggregate_id = :entity_id
        AND event_type = 'officer.resigned'
        AND event_time <= :as_of_date
  );

-- AUDIT: Full history of changes to a specific entity
SELECT event_type, payload, actor_user_id, event_time,
       metadata->>'source' AS source
FROM domain_event
WHERE tenant_id = :tenant_id
  AND aggregate_type = 'LegalEntity'
  AND aggregate_id = :entity_id
ORDER BY event_time ASC;

-- AI ANALYTICS: Detect unusual patterns (e.g. rapid officer turnover)
SELECT aggregate_id,
       COUNT(*) FILTER (WHERE event_type = 'officer.resigned') AS resignations,
       COUNT(*) FILTER (WHERE event_type = 'officer.appointed') AS appointments,
       MIN(event_time) AS period_start,
       MAX(event_time) AS period_end
FROM domain_event
WHERE tenant_id = :tenant_id
  AND aggregate_type = 'LegalEntity'
  AND event_type IN ('officer.appointed', 'officer.resigned')
  AND event_time >= now() - INTERVAL '90 days'
GROUP BY aggregate_id
HAVING COUNT(*) FILTER (WHERE event_type = 'officer.resigned') >= 3;

-- COMPLIANCE: Track all filing events for regulatory audit
SELECT de.payload->>'entity_name' AS entity,
       de.payload->>'obligation_name' AS obligation,
       de.event_type,
       de.payload->>'due_date' AS due_date,
       de.payload->>'filed_on' AS filed_on,
       de.event_time AS event_recorded_at,
       u.full_name AS filed_by
FROM domain_event de
LEFT JOIN app_user u ON u.id = de.actor_user_id
WHERE de.tenant_id = :tenant_id
  AND de.event_category = 'COMPLIANCE'
  AND de.event_time BETWEEN :audit_start AND :audit_end
ORDER BY de.event_time;

-- CATCH-UP: Process events since last projection update
SELECT event_id, event_type, payload, metadata, tenant_id, 
       aggregate_type, aggregate_id, event_time, sequence_number
FROM domain_event
WHERE sequence_number > :last_processed_sequence
ORDER BY sequence_number ASC
LIMIT 1000;
```

---

## Event Payload Examples

```jsonc
// officer.appointed event payload
{
    "person_id": "550e8400-e29b-41d4-a716-446655440000",
    "person_name": "Jane Smith",
    "entity_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "entity_name": "Acme Holdings Ltd",
    "role_oor_code": "GBLTDI",
    "role_name": "Director",
    "appointed_on": "2026-03-15",
    "resolution_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "jurisdiction_code": "GB",
    "service_address": {
        "line1": "10 Downing Street",
        "city": "London",
        "postal_code": "SW1A 2AA",
        "country": "GB"
    }
}

// share.issued event payload (OCF-aligned)
{
    "entity_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "share_class_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
    "share_class_name": "Series A Preferred",
    "stakeholder_id": "550e8400-e29b-41d4-a716-446655440000",
    "stakeholder_name": "Venture Capital Fund LP",
    "transaction_type": "ISSUANCE",
    "quantity": 1000000,
    "price_per_share": 1.50,
    "price_currency": "USD",
    "transaction_date": "2026-01-15",
    "board_approval_date": "2026-01-10",
    "resolution_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "certificate_number": "PA-001",
    "consideration": "Cash"
}

// meeting.concluded event payload
{
    "meeting_id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
    "entity_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "entity_name": "Acme Holdings Ltd",
    "meeting_type": "BOARD",
    "actual_start": "2026-03-20T14:00:00Z",
    "actual_end": "2026-03-20T16:30:00Z",
    "quorum_present": true,
    "attendees_present": 7,
    "attendees_absent": 1,
    "resolutions_passed": 3,
    "resolutions_failed": 0,
    "action_items_created": 5
}

// compliance.filing_submitted event payload
{
    "entity_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "entity_name": "Acme Holdings Ltd",
    "obligation_type": "CONFIRMATION_STATEMENT",
    "obligation_name": "Annual Confirmation Statement (Companies House)",
    "jurisdiction_code": "GB",
    "due_date": "2026-04-15",
    "filed_on": "2026-04-10",
    "filing_authority": "Companies House",
    "confirmation_number": "CS-2026-0042",
    "filing_fee": 13.00,
    "filing_fee_currency": "GBP"
}

// signing.recipient_signed event payload (DocuSign-aligned)
{
    "envelope_id": "d9428888-122b-11e1-b85c-61cd3cbb3210",
    "envelope_subject": "Board Resolution BR-2026-015",
    "recipient_name": "Jane Smith",
    "recipient_email": "jane.smith@acme.com",
    "recipient_type": "SIGNER",
    "signed_at": "2026-03-21T09:15:00Z",
    "signature_standard": "EIDAS_ADES",
    "provider": "DOCUSIGN",
    "external_envelope_id": "abc123-def456"
}
```

---

## File/Blob Storage (Not Event-Sourced)

```sql
-- ============================================================
-- FILE STORAGE METADATA (Operational — referenced by events)
-- ============================================================

-- Document files are stored in object storage (S3/GCS/Azure Blob).
-- This table tracks the storage metadata. The document lifecycle
-- (upload, version, view, archive, destroy) is tracked via events.

CREATE TABLE document_blob (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    
    storage_provider    VARCHAR(20) NOT NULL DEFAULT 'S3',
    storage_key         VARCHAR(500) NOT NULL,
    file_name           VARCHAR(255) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT,
    checksum_sha256     VARCHAR(64),
    
    encrypted_at_rest   BOOLEAN NOT NULL DEFAULT TRUE,
    encryption_key_id   VARCHAR(255),
    
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_blob_tenant ON document_blob(tenant_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | domain_event (partitioned), aggregate_snapshot |
| Operational / Infrastructure | 2 | tenant, app_user |
| Reference Data | 3 | jurisdiction, entity_legal_form, officer_role_type |
| File Storage | 1 | document_blob |
| Projection Metadata | 1 | projection_metadata |
| Read Model Projections | 7 | v_entity_current, v_officer_register, v_entity_relationship, v_meeting_current, v_share_holding, v_compliance_dashboard, v_document_index |
| **Total** | **16** | (+ 7 disposable projections = 23 effective) |

---

## Key Design Decisions

1. **Single event table for all domain events** — rather than separate tables per aggregate type, all events share the same schema. This simplifies cross-aggregate queries (e.g. "all events for this entity including officer changes, meetings, and filings") and makes event replay straightforward.

2. **JSONB payloads with schema versioning** — event payloads are stored as JSONB with a `schema_version` field. When the payload structure changes, the version is bumped and event processors handle both old and new versions. This avoids ALTER TABLE migrations on the immutable event store.

3. **Aggregate version for optimistic concurrency** — `aggregate_version` with a UNIQUE constraint prevents conflicting concurrent writes to the same aggregate. Two users editing the same entity simultaneously will have one succeed and one fail with a version conflict, which the application can retry.

4. **Snapshots for performance** — rather than replaying potentially thousands of events for long-lived aggregates, periodic snapshots store the current state. Event replay resumes from the latest snapshot.

5. **Disposable projections** — all `v_*` projection tables can be dropped and rebuilt by replaying the event store. This means schema changes to read models are trivial: create the new projection, replay events, switch traffic.

6. **Monthly time-based partitioning** — the domain_event table is partitioned by event_time, enabling efficient time-range queries and clean retention management (drop old partitions when retention period expires per ISO 15489).

7. **Correlation and causation tracking** — `metadata.correlation_id` links related events across aggregates (e.g. a resolution that triggers an officer appointment), enabling causal chain analysis for audit and AI analytics.

8. **No separate audit log** — the event store IS the audit log. Every state change is recorded immutably with actor, timestamp, and full payload. This eliminates the audit log maintenance burden and ensures the audit trail is always complete.

9. **OCF-aligned share events** — share transaction events mirror the Open Cap Format transaction types, enabling import/export compatibility with cap table tools.

10. **Event-driven AI analytics** — the event stream is a natural input for AI/ML models. Change patterns (rapid officer turnover, missed filing deadlines, unusual voting patterns) can be detected by streaming event processors without querying the read models.
