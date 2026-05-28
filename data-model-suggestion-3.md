# Data Model Suggestion 3: Hybrid Relational + JSONB Flexible Model

> Project: Corporate Secretary Platform · Created: 2026-05-12

## Philosophy

A pragmatic middle ground: core entities and relationships are relational (typed, indexed, foreign-keyed), but jurisdiction-specific fields, event-type-specific metadata, and extensible attributes live in JSONB columns. This is how modern multi-jurisdiction SaaS platforms actually work — including Diligent Entities (GraphQL with flexible entity attributes), Legalinc (API-first with extensible entity payloads), and the Companies House API (which returns structured JSON with jurisdiction-specific fields nested in objects).

The key insight: corporate governance spans 100+ jurisdictions, each with unique entity types, officer roles, filing requirements, and legal forms. A fully normalised model either has hundreds of nullable columns or hundreds of junction tables. JSONB lets you store `{"registered_charity_number": "1234567", "charity_commission_class": "C2"}` for a UK charity and `{"franchise_tax_id": "98765", "delaware_file_number": "4567890"}` for a Delaware LLC without either polluting the other's schema.

PostgreSQL's JSONB capabilities are the enabler: GIN indexes support efficient containment queries (`@>`), `jsonb_path_query` enables SQL/JSON path expressions, and CHECK constraints with `jsonb_typeof` can enforce structure at the database level. The schema is self-documenting for core fields (relational columns) while remaining extensible for jurisdiction-specific and version-specific attributes (JSONB).

**Best for:** Rapid development, multi-jurisdiction flexibility, API-first design (GraphQL maps naturally to this shape), teams comfortable with PostgreSQL's JSONB capabilities. Ideal for startups that need to ship an MVP quickly while preserving the ability to refine the schema as jurisdiction coverage expands. Also suits platforms that need to integrate with many external APIs that each return slightly different data shapes.

**Trade-offs:**
- (+) Fewer tables (~25 vs. ~32 in normalized) — simpler to understand and migrate
- (+) Jurisdiction-specific fields don't require schema changes — add them via JSONB
- (+) API responses naturally map to the hybrid structure (core fields + flexible attributes)
- (+) GIN indexes on JSONB provide good query performance for containment queries
- (+) GraphQL schema maps directly to relational core + JSONB resolver
- (-) JSONB fields are harder to validate at the database level (need application-layer validation or JSON Schema)
- (-) Cross-JSONB queries are possible but slower than indexed relational columns
- (-) Schema documentation requires discipline — JSONB contents must be documented in code or schema registry
- (-) Harder to enforce referential integrity within JSONB (e.g. a jurisdiction code inside JSONB can't be FK'd)
- (-) Data quality depends more on application-layer validation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/3166-2 | `jurisdiction` table (relational — heavily queried, stable schema) |
| ISO 17442 (LEI) | `legal_entity.lei` column (relational) + additional identifiers in `legal_entity.identifiers` JSONB |
| ISO 5009 / ISO 20275 | Lookup values stored in `reference_data` table; referenced by code in JSONB attributes |
| GLEIF LEI-CDF 3.1 | Entity record layout for core relational fields; GLEIF-specific extended fields in JSONB |
| GLEIF RR-CDF 2.1 | Relationship type vocabulary used in `entity_relationship.relationship_type` |
| BODS v0.4 | Beneficial ownership interests stored with core relational fields + JSONB for jurisdiction-specific PSC/BOI data |
| Diligent Entities GraphQL | Entity model inspired by Diligent's Company type with typed core fields and flexible attributes |
| DocuSign Envelope | E-signature integration data stored in `signing_envelope.provider_data` JSONB |
| Companies House API | UK-specific officer fields (identification, former_names) stored in `person.jurisdiction_data` JSONB |
| OCSF | Audit log with structured envelope + JSONB payload |
| OCF | Share transaction details in `share_transaction.transaction_data` JSONB |

---

## Schema Overview

```
┌─────────────────────┐
│   reference_data     │  ← Unified lookup table (jurisdictions, legal forms, roles, doc categories)
└──────────┬──────────┘
           │
     ┌─────┴──────────────────────────────────────┐
     ▼                                             ▼
┌──────────────┐                          ┌──────────────────┐
│ legal_entity  │◄─────────────────────── │ entity_relationship│
│ (core + JSONB)│                          │ (core + JSONB)    │
└──┬──┬──┬──┬──┘                          └──────────────────┘
   │  │  │  │
   │  │  │  └─► person (core + JSONB)
   │  │  │       └─► officer_appointment (relational)
   │  │  │
   │  │  └─► meeting (core + JSONB)
   │  │       ├─► agenda_item (relational)
   │  │       ├─► attendance (relational)
   │  │       └─► resolution (core + JSONB)
   │  │             └─► vote (relational)
   │  │
   │  └─► document (core + JSONB)
   │       └─► document_permission (relational)
   │
   └─► compliance_obligation (core + JSONB)
        └─► share_transaction (core + JSONB)
```

---

## Unified Reference Data

```sql
-- ============================================================
-- UNIFIED REFERENCE DATA (Replaces multiple lookup tables)
-- ============================================================

-- A single reference data table for all code lists.
-- This simplifies administration and supports dynamic code lists
-- that can be extended without schema changes.

CREATE TABLE reference_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Code list identity
    list_code       VARCHAR(50) NOT NULL,
    -- 'JURISDICTION'         (ISO 3166)
    -- 'ENTITY_LEGAL_FORM'    (GLEIF ELF / ISO 20275)
    -- 'OFFICER_ROLE_TYPE'    (ISO 5009 OOR)
    -- 'DOCUMENT_CATEGORY'    (internal)
    -- 'MEETING_TYPE'         (internal)
    -- 'RESOLUTION_TYPE'      (internal)
    -- 'COMPLIANCE_TYPE'      (internal)
    
    -- Item identity
    code            VARCHAR(20) NOT NULL,                   -- The code value (e.g. 'US-DE', 'LLC', 'GBLTDI')
    
    -- Display
    name            VARCHAR(500) NOT NULL,                  -- Display name (English)
    name_local      VARCHAR(500),                           -- Display name (local language)
    description     TEXT,
    
    -- Hierarchy (for jurisdictions: US-DE → US)
    parent_code     VARCHAR(20),
    
    -- Additional attributes (varies by list type)
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- For JURISDICTION: { "iso_alpha3": "USA", "iso_numeric": "840", "region": "North America" }
    -- For ENTITY_LEGAL_FORM: { "jurisdiction_code": "US-DE", "abbreviations": "LLC, L.L.C." }
    -- For OFFICER_ROLE_TYPE: { "jurisdiction_code": "GB", "elf_code": "8888", "is_signatory": true }
    -- For DOCUMENT_CATEGORY: { "retention_years": 7, "disposal_action": "ARCHIVE" }
    
    -- Ordering
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(list_code, code)
);

CREATE INDEX idx_refdata_list ON reference_data(list_code, is_active);
CREATE INDEX idx_refdata_parent ON reference_data(list_code, parent_code) WHERE parent_code IS NOT NULL;
CREATE INDEX idx_refdata_attributes ON reference_data USING GIN (attributes);
```

---

## Multi-Tenant Foundation

```sql
-- ============================================================
-- MULTI-TENANT FOUNDATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    
    -- Configuration stored as JSONB for flexibility
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "subscription_tier": "standard",
    --   "data_residency": "US",
    --   "features": {
    --     "ai_minutes": true,
    --     "entity_management": true,
    --     "share_register": false,
    --     "xbrl_export": false
    --   },
    --   "branding": {
    --     "logo_url": "...",
    --     "primary_color": "#1a73e8"
    --   },
    --   "compliance": {
    --     "default_retention_years": 7,
    --     "require_mfa": true
    --   }
    -- }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    
    -- User preferences and SSO data in JSONB
    profile         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "sso_provider_id": "...",
    --   "mfa_enabled": true,
    --   "timezone": "America/New_York",
    --   "notification_preferences": {
    --     "email_board_pack": true,
    --     "email_compliance_alert": true,
    --     "push_meeting_reminder": true
    --   }
    -- }
    
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- RLS for all tenant-scoped tables
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Entity Management (Hybrid Core)

```sql
-- ============================================================
-- ENTITY MANAGEMENT (Hybrid: relational core + JSONB extensions)
-- ============================================================

CREATE TABLE legal_entity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- Core identity (relational — indexed, queryable, FK-able)
    legal_name          VARCHAR(500) NOT NULL,
    trading_name        VARCHAR(500),
    lei                 VARCHAR(20) UNIQUE,                  -- ISO 17442
    legal_form_code     VARCHAR(20),                         -- References reference_data('ENTITY_LEGAL_FORM', code)
    jurisdiction_code   VARCHAR(6) NOT NULL,                 -- References reference_data('JURISDICTION', code)
    registration_number VARCHAR(100),
    registration_date   DATE,
    entity_status       VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- Addresses (structured JSONB — flexible for international formats)
    legal_address       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Main Street",
    --   "line2": "Suite 400",
    --   "city": "Wilmington",
    --   "region": "Delaware",
    --   "postal_code": "19801",
    --   "country": "US"
    -- }
    
    hq_address          JSONB DEFAULT '{}',
    
    -- Additional identifiers (varies by jurisdiction)
    identifiers         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "cik": "0001234567",              -- SEC Central Index Key (US)
    --   "company_number": "12345678",     -- Companies House number (UK)
    --   "abn": "12 345 678 901",          -- Australian Business Number
    --   "siren": "123456789",             -- French SIREN
    --   "kvk_number": "12345678",         -- Dutch KvK number
    --   "sic_code": "6199",               -- Standard Industrial Classification
    --   "nace_code": "64.19"              -- EU NACE code
    -- }
    
    -- Jurisdiction-specific data (the key JSONB flexibility point)
    jurisdiction_data   JSONB NOT NULL DEFAULT '{}',
    -- UK example:
    -- {
    --   "confirmation_statement_due": "2026-09-15",
    --   "last_accounts_made_up_to": "2025-12-31",
    --   "registered_office_type": "SAIL",
    --   "has_been_liquidated": false,
    --   "has_charges": true,
    --   "jurisdiction_specific_type": "private-limited-shares-section-30-exemption"
    -- }
    -- Delaware example:
    -- {
    --   "franchise_tax_due": "2026-03-01",
    --   "franchise_tax_method": "authorized_shares",
    --   "delaware_file_number": "4567890",
    --   "registered_agent": "Corporation Service Company",
    --   "registered_agent_address": "251 Little Falls Drive, Wilmington, DE 19808"
    -- }
    -- Singapore example:
    -- {
    --   "uen": "202012345A",
    --   "acra_status": "live",
    --   "agm_due_date": "2026-06-30",
    --   "constitution_type": "model"
    -- }
    
    -- Financial year
    fiscal_year_end_month INTEGER CHECK (fiscal_year_end_month BETWEEN 1 AND 12),
    fiscal_year_end_day   INTEGER CHECK (fiscal_year_end_day BETWEEN 1 AND 31),
    
    -- AI-managed metadata
    ai_metadata         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "last_risk_score": 0.23,
    --   "last_risk_assessment": "2026-04-01",
    --   "risk_factors": ["late_filing_history", "high_director_turnover"],
    --   "extracted_from_documents": true,
    --   "extraction_confidence": 0.95
    -- }
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_jurisdiction ON legal_entity(tenant_id, jurisdiction_code);
CREATE INDEX idx_entity_status ON legal_entity(tenant_id, entity_status);
CREATE INDEX idx_entity_lei ON legal_entity(lei) WHERE lei IS NOT NULL;
CREATE INDEX idx_entity_reg ON legal_entity(tenant_id, jurisdiction_code, registration_number);

-- GIN indexes for JSONB querying
CREATE INDEX idx_entity_identifiers ON legal_entity USING GIN (identifiers);
CREATE INDEX idx_entity_jurisdiction_data ON legal_entity USING GIN (jurisdiction_data);

ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Entity relationships
CREATE TABLE entity_relationship (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    child_entity_id     UUID NOT NULL REFERENCES legal_entity(id),
    parent_entity_id    UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Core fields (relational)
    relationship_type   VARCHAR(50) NOT NULL,                -- GLEIF: IS_DIRECTLY_CONSOLIDATED_BY, etc.
    ownership_percentage DECIMAL(5,2),
    voting_percentage   DECIMAL(5,2),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- Extended relationship data
    relationship_data   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "accounting_standard": "IFRS",
    --   "consolidation_method": "full",
    --   "control_mechanism": "voting_rights",
    --   "relationship_source": "annual_return",
    --   "last_verified": "2026-01-15"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_self_ref CHECK (child_entity_id != parent_entity_id)
);

CREATE INDEX idx_rel_child ON entity_relationship(tenant_id, child_entity_id);
CREATE INDEX idx_rel_parent ON entity_relationship(tenant_id, parent_entity_id);
```

---

## Person & Officer Management

```sql
-- ============================================================
-- PERSON & OFFICER MANAGEMENT
-- ============================================================

CREATE TABLE person (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- Core identity (relational)
    full_name           VARCHAR(500) NOT NULL,
    given_name          VARCHAR(255),
    family_name         VARCHAR(255),
    date_of_birth       DATE,
    nationality         VARCHAR(2),                         -- ISO 3166-1
    country_of_residence VARCHAR(2),
    email               VARCHAR(320),
    
    -- Link to platform user
    app_user_id         UUID REFERENCES app_user(id),
    
    -- Addresses (JSONB — flexible international formats)
    service_address     JSONB NOT NULL DEFAULT '{}',
    residential_address JSONB DEFAULT '{}',                 -- GDPR protected
    
    -- Jurisdiction-specific person data (Companies House model)
    jurisdiction_data   JSONB NOT NULL DEFAULT '{}',
    -- UK example:
    -- {
    --   "former_names": [
    --     { "forenames": "Jane Elizabeth", "surname": "Doe", "effective_to": "2020-06-15" }
    --   ],
    --   "identification": {
    --     "identification_type": "passport",
    --     "legal_authority": "United Kingdom",
    --     "registration_number": "123456789"
    --   },
    --   "dob_display": { "month": 3, "year": 1985 },
    --   "nationality_description": "British"
    -- }
    -- US example:
    -- {
    --   "ssn_last_four": "1234",
    --   "fincen_id": "BOI-2026-0042",
    --   "identity_verification": {
    --     "document_type": "drivers_license",
    --     "issuing_jurisdiction": "US-CA",
    --     "document_number": "D1234567",
    --     "expiry_date": "2028-03-15"
    --   }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_name ON person(tenant_id, family_name, given_name);
CREATE INDEX idx_person_user ON person(app_user_id) WHERE app_user_id IS NOT NULL;

-- Officer appointments (relational — heavily queried)
CREATE TABLE officer_appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Core fields (relational)
    role_code           VARCHAR(20) NOT NULL,                -- ISO 5009 OOR code → reference_data
    role_description    VARCHAR(255),
    appointed_on        DATE NOT NULL,
    resigned_on         DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    resolution_id       UUID REFERENCES resolution(id),
    
    -- Extended appointment data
    appointment_data    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "is_executive": true,
    --   "committee_memberships": ["AUDIT", "REMUNERATION"],
    --   "compensation": {
    --     "annual_fee": 50000,
    --     "currency": "GBP",
    --     "equity_component": true
    --   },
    --   "independence_declaration": {
    --     "is_independent": true,
    --     "declared_on": "2026-01-15",
    --     "factors_considered": ["no_material_relationship", "no_family_ties"]
    --   }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_entity ON officer_appointment(tenant_id, entity_id);
CREATE INDEX idx_appt_person ON officer_appointment(tenant_id, person_id);
CREATE INDEX idx_appt_active ON officer_appointment(tenant_id, entity_id, status) WHERE status = 'ACTIVE';

-- Beneficial ownership
CREATE TABLE beneficial_ownership (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Interested party (person or entity)
    person_id           UUID REFERENCES person(id),
    interested_entity_id UUID REFERENCES legal_entity(id),
    
    -- Core BODS fields (relational)
    interest_type       VARCHAR(50) NOT NULL,
    interest_level      VARCHAR(20),
    share_percentage    DECIMAL(5,2),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    
    -- Jurisdiction-specific beneficial ownership data
    bo_data             JSONB NOT NULL DEFAULT '{}',
    -- UK PSC example:
    -- {
    --   "psc_nature_of_control": [
    --     "ownership-of-shares-75-to-100-percent",
    --     "right-to-appoint-and-remove-directors"
    --   ],
    --   "psc_kind": "individual-person-with-significant-control",
    --   "notified_on": "2026-01-15",
    --   "ceased_on": null
    -- }
    -- US FinCEN BOI example:
    -- {
    --   "boi_report_type": "initial",
    --   "boi_filed_on": "2026-02-01",
    --   "boi_confirmation_id": "BOI-2026-0042",
    --   "boi_beneficial_owner_type": "direct_ownership",
    --   "ownership_percentage_range": "75-100"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    CONSTRAINT has_interested_party CHECK (
        (person_id IS NOT NULL AND interested_entity_id IS NULL) OR
        (person_id IS NULL AND interested_entity_id IS NOT NULL)
    )
);

CREATE INDEX idx_bo_entity ON beneficial_ownership(tenant_id, entity_id);
CREATE INDEX idx_bo_person ON beneficial_ownership(person_id) WHERE person_id IS NOT NULL;

-- Conflict of interest declarations
CREATE TABLE conflict_of_interest (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    nature_of_interest  TEXT NOT NULL,
    declared_on         DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'DECLARED',
    
    -- Extended data
    coi_data            JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "related_entities": ["uuid1", "uuid2"],
    --   "financial_interest": true,
    --   "estimated_value": 50000,
    --   "currency": "USD",
    --   "reviewed_by": "uuid",
    --   "reviewed_on": "2026-02-01",
    --   "board_aware": true,
    --   "recusal_required": true,
    --   "recusal_topics": ["merger_discussion", "vendor_selection"]
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coi_person ON conflict_of_interest(tenant_id, person_id);
```

---

## Meeting Management

```sql
-- ============================================================
-- MEETING MANAGEMENT
-- ============================================================

CREATE TABLE committee (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    name                VARCHAR(255) NOT NULL,
    committee_type      VARCHAR(50) NOT NULL,
    
    -- Committee configuration in JSONB
    config              JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "charter": "...",
    --   "quorum_count": 3,
    --   "quorum_percentage": null,
    --   "meeting_frequency": "quarterly",
    --   "default_agenda_template": [
    --     { "title": "Apologies", "type": "PROCEDURAL" },
    --     { "title": "Minutes of Previous Meeting", "type": "APPROVAL" },
    --     { "title": "Matters Arising", "type": "DISCUSSION" }
    --   ]
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_committee_entity ON committee(tenant_id, entity_id);

CREATE TABLE committee_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    committee_id    UUID NOT NULL REFERENCES committee(id),
    person_id       UUID NOT NULL REFERENCES person(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'MEMBER',
    appointed_on    DATE NOT NULL,
    resigned_on     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(committee_id, person_id, appointed_on)
);

CREATE TABLE meeting (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    committee_id        UUID REFERENCES committee(id),
    
    -- Core fields (relational — heavily queried)
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    
    -- Extended meeting data
    meeting_data        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "meeting_number": 42,
    --   "timezone": "Europe/London",
    --   "location": {
    --     "type": "HYBRID",
    --     "physical_address": "1 Board Room Lane, London EC2V 7QN",
    --     "virtual_url": "https://zoom.us/j/123456789",
    --     "dial_in": "+44 20 1234 5678"
    --   },
    --   "quorum": {
    --     "required": 4,
    --     "present": 5,
    --     "confirmed": true
    --   },
    --   "timing": {
    --     "actual_start": "2026-03-20T14:05:00Z",
    --     "actual_end": "2026-03-20T16:25:00Z"
    --   },
    --   "board_pack": {
    --     "sent_at": "2026-03-15T09:00:00Z",
    --     "page_count": 142
    --   },
    --   "minutes": {
    --     "draft_text": "...",
    --     "final_text": "...",
    --     "approved_at": "2026-04-17T14:30:00Z",
    --     "approved_by": "uuid",
    --     "ai_generated": true,
    --     "ai_confidence": 0.94
    --   },
    --   "transcript": {
    --     "url": "s3://...",
    --     "duration_seconds": 8400,
    --     "language": "en"
    --   }
    -- }
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_meeting_entity ON meeting(tenant_id, entity_id);
CREATE INDEX idx_meeting_date ON meeting(tenant_id, scheduled_start);
CREATE INDEX idx_meeting_status ON meeting(tenant_id, status);

-- Agenda items (relational — ordered, referenced by resolutions)
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

CREATE INDEX idx_agenda_meeting ON agenda_item(meeting_id);

-- Meeting attendance (relational — queried for quorum, reports)
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

-- Action items
CREATE TABLE action_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    meeting_id      UUID REFERENCES meeting(id),
    agenda_item_id  UUID REFERENCES agenda_item(id),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES person(id),
    due_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    
    -- AI metadata
    ai_data         JSONB NOT NULL DEFAULT '{}',
    -- { "ai_extracted": true, "confidence": 0.92, "source_text": "..." }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_action_tenant ON action_item(tenant_id);
CREATE INDEX idx_action_assignee ON action_item(assigned_to, status);
```

---

## Resolutions, Voting & E-Signatures

```sql
-- ============================================================
-- RESOLUTIONS, VOTING & E-SIGNATURES
-- ============================================================

CREATE TABLE resolution (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Core fields (relational)
    resolution_number   VARCHAR(50),
    title               VARCHAR(500) NOT NULL,
    body_text           TEXT NOT NULL,
    resolution_type     VARCHAR(50) NOT NULL,
    meeting_id          UUID REFERENCES meeting(id),
    agenda_item_id      UUID REFERENCES agenda_item(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    
    -- Voting results (relational)
    approval_threshold  DECIMAL(5,2) NOT NULL DEFAULT 50.01,
    votes_for           INTEGER,
    votes_against       INTEGER,
    votes_abstained     INTEGER,
    passed_at           TIMESTAMPTZ,
    effective_date      DATE,
    
    -- Extended resolution data
    resolution_data     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "precedent_source": "template_library",
    --   "precedent_id": "uuid",
    --   "ai_generated": true,
    --   "ai_confidence": 0.91,
    --   "legal_review": {
    --     "reviewed_by": "uuid",
    --     "reviewed_at": "2026-03-19T10:00:00Z",
    --     "approved": true
    --   },
    --   "related_resolutions": ["uuid1", "uuid2"],
    --   "regulatory_filing_required": true,
    --   "filing_jurisdictions": ["GB", "US-DE"]
    -- }
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resolution_entity ON resolution(tenant_id, entity_id);
CREATE INDEX idx_resolution_meeting ON resolution(meeting_id);
CREATE INDEX idx_resolution_status ON resolution(tenant_id, status);

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

-- E-signature envelopes
CREATE TABLE signing_envelope (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- Core fields (relational)
    provider            VARCHAR(20) NOT NULL DEFAULT 'DOCUSIGN',
    external_envelope_id VARCHAR(255),
    email_subject       VARCHAR(500) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'CREATED',
    signature_standard  VARCHAR(20) NOT NULL DEFAULT 'ESIGN',
    
    -- Resolution linkage
    resolution_id       UUID REFERENCES resolution(id),
    
    -- Provider-specific data (varies by DocuSign, Adobe Sign, etc.)
    provider_data       JSONB NOT NULL DEFAULT '{}',
    -- DocuSign example:
    -- {
    --   "account_id": "...",
    --   "brand_id": "...",
    --   "recipients": [
    --     {
    --       "recipient_id": "1",
    --       "person_id": "uuid",
    --       "name": "Jane Smith",
    --       "email": "jane@acme.com",
    --       "recipient_type": "SIGNER",
    --       "routing_order": 1,
    --       "status": "SIGNED",
    --       "signed_at": "2026-03-21T09:15:00Z",
    --       "tabs": [
    --         { "type": "signHere", "page": 3, "x": 100, "y": 500 },
    --         { "type": "dateSigned", "page": 3, "x": 100, "y": 550 }
    --       ]
    --     }
    --   ],
    --   "documents": [
    --     { "document_id": "1", "name": "Resolution BR-2026-015.pdf", "pages": 4 }
    --   ],
    --   "sent_at": "2026-03-20T17:00:00Z",
    --   "completed_at": "2026-03-21T14:30:00Z"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_envelope_tenant ON signing_envelope(tenant_id);
CREATE INDEX idx_envelope_status ON signing_envelope(tenant_id, status);
CREATE INDEX idx_envelope_resolution ON signing_envelope(resolution_id) WHERE resolution_id IS NOT NULL;
```

---

## Document Management

```sql
-- ============================================================
-- DOCUMENT MANAGEMENT
-- ============================================================

CREATE TABLE document (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID REFERENCES legal_entity(id),
    
    -- Core fields (relational)
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL,
    category_code       VARCHAR(50),                        -- → reference_data('DOCUMENT_CATEGORY')
    version             INTEGER NOT NULL DEFAULT 1,
    is_current_version  BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID REFERENCES document(id),
    meeting_id          UUID REFERENCES meeting(id),
    confidentiality     VARCHAR(20) NOT NULL DEFAULT 'CONFIDENTIAL',
    
    -- Storage (relational — indexed)
    storage_key         VARCHAR(500) NOT NULL,
    file_name           VARCHAR(255) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT,
    checksum_sha256     VARCHAR(64),
    
    -- Extended document metadata
    doc_metadata        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "encryption": {
    --     "encrypted_at_rest": true,
    --     "encryption_key_id": "arn:aws:kms:..."
    --   },
    --   "retention": {
    --     "class": "CORPORATE_RECORD",
    --     "retain_until": "2033-12-31",
    --     "disposal_action": "ARCHIVE"
    --   },
    --   "ai": {
    --     "summary": "Board resolution approving the acquisition of...",
    --     "key_topics": ["acquisition", "due_diligence", "shareholder_approval"],
    --     "extracted_entities": ["Acme Corp", "Target Inc"],
    --     "language": "en",
    --     "confidence": 0.93
    --   },
    --   "annotations_count": 5,
    --   "download_count": 12,
    --   "last_viewed_at": "2026-04-10T09:00:00Z"
    -- }
    
    uploaded_by         UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_tenant ON document(tenant_id);
CREATE INDEX idx_doc_entity ON document(tenant_id, entity_id);
CREATE INDEX idx_doc_type ON document(tenant_id, document_type);
CREATE INDEX idx_doc_meeting ON document(meeting_id) WHERE meeting_id IS NOT NULL;
CREATE INDEX idx_doc_metadata ON document USING GIN (doc_metadata);

-- Document permissions (relational — enforced at query time)
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

CREATE INDEX idx_docperm_doc ON document_permission(document_id);
CREATE INDEX idx_docperm_user ON document_permission(user_id) WHERE user_id IS NOT NULL;
```

---

## Compliance & Share Register

```sql
-- ============================================================
-- COMPLIANCE OBLIGATIONS
-- ============================================================

CREATE TABLE compliance_obligation (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Core fields (relational)
    name                VARCHAR(500) NOT NULL,
    obligation_type     VARCHAR(50) NOT NULL,
    jurisdiction_code   VARCHAR(6) NOT NULL,
    due_date            DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'UPCOMING',
    
    -- Extended obligation data
    obligation_data     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "template_source": "jurisdiction_rules",
    --   "frequency": "ANNUAL",
    --   "filing_authority": "Companies House",
    --   "filing_url": "https://...",
    --   "filing_fee": { "amount": 13.00, "currency": "GBP" },
    --   "filed_on": "2026-04-10",
    --   "confirmation_number": "CS-2026-0042",
    --   "filed_by": "uuid",
    --   "penalty": {
    --     "description": "Late filing penalty",
    --     "amount": 150.00,
    --     "currency": "GBP"
    --   },
    --   "period": {
    --     "start": "2025-04-16",
    --     "end": "2026-04-15"
    --   },
    --   "ai": {
    --     "flagged": true,
    --     "flag_reason": "Due in 14 days, no preparation started",
    --     "priority_score": 0.85
    --   }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_entity ON compliance_obligation(tenant_id, entity_id);
CREATE INDEX idx_compliance_due ON compliance_obligation(tenant_id, due_date, status);
CREATE INDEX idx_compliance_status ON compliance_obligation(tenant_id, status);

-- ============================================================
-- SHARE REGISTER
-- ============================================================

CREATE TABLE share_class (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    name                VARCHAR(255) NOT NULL,
    class_type          VARCHAR(20) NOT NULL,
    shares_authorised   BIGINT,
    par_value           DECIMAL(12,6),
    par_value_currency  VARCHAR(3),
    votes_per_share     DECIMAL(10,2) NOT NULL DEFAULT 1.0,
    
    -- Class-specific details (varies by class type)
    class_data          JSONB NOT NULL DEFAULT '{}',
    -- Preferred stock example:
    -- {
    --   "liquidation_preference_multiple": 1.0,
    --   "is_participating": false,
    --   "dividend_rate": 0.08,
    --   "seniority": 1,
    --   "anti_dilution": "broad_based_weighted_average",
    --   "conversion_ratio": 1.0,
    --   "protective_provisions": ["change_of_control", "new_debt", "new_equity"]
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shareclass_entity ON share_class(tenant_id, entity_id);

CREATE TABLE share_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    share_class_id      UUID NOT NULL REFERENCES share_class(id),
    
    -- Core fields (relational)
    transaction_type    VARCHAR(30) NOT NULL,
    from_stakeholder_id UUID REFERENCES person(id),
    to_stakeholder_id   UUID REFERENCES person(id),
    quantity            BIGINT NOT NULL,
    transaction_date    DATE NOT NULL,
    resolution_id       UUID REFERENCES resolution(id),
    
    -- Extended transaction data (OCF-aligned)
    transaction_data    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "price_per_share": 1.50,
    --   "price_currency": "USD",
    --   "total_consideration": 1500000.00,
    --   "consideration_type": "cash",
    --   "board_approval_date": "2026-01-10",
    --   "certificate_number": "PA-001",
    --   "vesting_schedule": {
    --     "cliff_months": 12,
    --     "total_months": 48,
    --     "start_date": "2026-01-15"
    --   },
    --   "tax_implications": {
    --     "section_83b_filed": true,
    --     "filing_date": "2026-02-10"
    --   }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sharetx_entity ON share_transaction(tenant_id, entity_id);
CREATE INDEX idx_sharetx_date ON share_transaction(transaction_date);
```

---

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG (OCSF-aligned, JSONB payload)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    
    -- Core envelope (relational — for indexing and filtering)
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    category        VARCHAR(50) NOT NULL,
    event_class     VARCHAR(50) NOT NULL,
    activity        VARCHAR(20) NOT NULL,
    actor_user_id   UUID,
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID NOT NULL,
    status          VARCHAR(10) NOT NULL DEFAULT 'SUCCESS',
    severity        VARCHAR(10) NOT NULL DEFAULT 'INFO',
    
    -- Full event data in JSONB (flexible, queryable)
    event_data      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "actor": {
    --     "email": "jane@acme.com",
    --     "ip_address": "192.168.1.1",
    --     "user_agent": "Mozilla/5.0..."
    --   },
    --   "target": {
    --     "name": "Acme Holdings Ltd",
    --     "entity_id": "uuid"
    --   },
    --   "changes": {
    --     "legal_name": { "old": "Acme Corp", "new": "Acme Holdings Ltd" },
    --     "entity_status": { "old": "ACTIVE", "new": "ACTIVE" }
    --   },
    --   "context": {
    --     "session_id": "uuid",
    --     "request_id": "uuid"
    --   }
    -- }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, event_time DESC);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id, event_time DESC);
CREATE INDEX idx_audit_data ON audit_log USING GIN (event_data);
```

---

## Example Queries (JSONB-Specific)

```sql
-- Find all entities in Delaware with franchise tax due
SELECT le.legal_name, le.registration_number,
       le.jurisdiction_data->>'franchise_tax_due' AS franchise_tax_due,
       le.jurisdiction_data->>'delaware_file_number' AS file_number
FROM legal_entity le
WHERE le.tenant_id = :tenant_id
  AND le.jurisdiction_code = 'US-DE'
  AND le.jurisdiction_data ? 'franchise_tax_due';

-- Find entities with a specific identifier (cross-jurisdiction search)
SELECT le.legal_name, le.jurisdiction_code, le.identifiers
FROM legal_entity le
WHERE le.tenant_id = :tenant_id
  AND le.identifiers @> '{"cik": "0001234567"}';

-- UK PSC register query (JSONB containment)
SELECT p.full_name, bo.interest_type, bo.share_percentage,
       bo.bo_data->'psc_nature_of_control' AS nature_of_control
FROM beneficial_ownership bo
JOIN person p ON p.id = bo.person_id
WHERE bo.tenant_id = :tenant_id
  AND bo.entity_id = :entity_id
  AND bo.valid_to IS NULL
  AND bo.bo_data @> '{"psc_kind": "individual-person-with-significant-control"}';

-- Director independence analysis
SELECT p.full_name, oa.role_code,
       oa.appointment_data->'independence_declaration'->>'is_independent' AS independent,
       oa.appointment_data->'compensation'->>'annual_fee' AS fee
FROM officer_appointment oa
JOIN person p ON p.id = oa.person_id
WHERE oa.tenant_id = :tenant_id
  AND oa.entity_id = :entity_id
  AND oa.status = 'ACTIVE';

-- Full-text search across document AI summaries
SELECT d.title, d.document_type,
       d.doc_metadata->'ai'->>'summary' AS ai_summary
FROM document d
WHERE d.tenant_id = :tenant_id
  AND d.doc_metadata->'ai'->>'summary' ILIKE '%acquisition%';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Reference Data | 1 | reference_data (unified lookup) |
| Multi-Tenant Foundation | 2 | tenant, app_user |
| Entity Management | 3 | legal_entity, entity_relationship, beneficial_ownership |
| Person & Officers | 3 | person, officer_appointment, conflict_of_interest |
| Meeting Management | 5 | committee, committee_membership, meeting, agenda_item, meeting_attendance |
| Action Items | 1 | action_item |
| Resolutions & Voting | 2 | resolution, resolution_vote |
| E-Signature | 1 | signing_envelope (recipients in JSONB) |
| Document Management | 2 | document, document_permission |
| Compliance | 1 | compliance_obligation |
| Share Register | 2 | share_class, share_transaction |
| Audit Log | 1 | audit_log (partitioned) |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Unified reference_data table** — replaces multiple lookup tables (jurisdiction, entity_legal_form, officer_role_type, document_category) with a single table differentiated by `list_code`. Reduces table count and simplifies administration while using JSONB `attributes` for list-specific fields.

2. **Relational core + JSONB extensions pattern** — every table has strongly-typed relational columns for fields that are queried, indexed, or referenced by foreign keys, plus JSONB columns for jurisdiction-specific, provider-specific, or AI-generated data. This gives the best of both worlds.

3. **jurisdiction_data JSONB on legal_entity** — the primary flexibility point. UK entities carry Companies House-specific fields, Delaware entities carry franchise tax fields, Singapore entities carry ACRA fields — all without schema changes. New jurisdictions are added by documenting the expected JSONB structure.

4. **provider_data JSONB on signing_envelope** — the entire DocuSign recipient/tab hierarchy is stored in a single JSONB column rather than separate tables. This dramatically simplifies the e-signature integration layer and supports multiple providers without schema changes.

5. **Addresses as JSONB** — international address formats vary too much for relational columns (Japan has no "street", UK has "county", Singapore has "block" and "unit"). JSONB accommodates all formats while the application layer validates per-jurisdiction rules.

6. **GIN indexes on JSONB columns** — all major JSONB columns have GIN indexes to support containment queries (`@>`) and existence queries (`?`). This ensures JSONB queries perform well for common access patterns.

7. **AI metadata as JSONB** — AI-generated data (summaries, risk scores, extraction confidence) is stored in JSONB because the structure evolves rapidly as AI capabilities improve. No schema migration needed when adding a new AI feature.

8. **24 tables vs. 32** — the hybrid approach reduces total table count by ~25% compared to the normalized model, primarily by collapsing reference tables, removing junction tables for e-signature recipients, and storing variable-structure data in JSONB rather than subtables.

9. **GraphQL-natural shape** — the hybrid model maps directly to a GraphQL schema where core fields are typed and JSONB fields are resolved as JSON scalar types or through custom resolvers. This suits the API-first architecture that Diligent Entities and similar platforms use.

10. **Progressive normalisation** — as the platform matures and jurisdiction-specific patterns stabilise, frequently-queried JSONB fields can be promoted to relational columns (e.g. `jurisdiction_data->>'franchise_tax_due'` could become a `franchise_tax_due DATE` column) without breaking existing queries, since JSONB queries continue to work alongside relational columns.
