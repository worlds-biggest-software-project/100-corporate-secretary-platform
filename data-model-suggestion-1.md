# Data Model Suggestion 1: Entity-Centric Normalized Relational Model

> Project: Corporate Secretary Platform · Created: 2026-05-12

## Philosophy

A classically normalized relational model where every concept occupies its own table, relationships are expressed through foreign keys and junction tables, and reference data draws heavily from international standards (ISO 17442 LEI, ISO 3166 jurisdictions, ISO 5009 Official Organizational Roles, GLEIF Entity Legal Forms, BODS Beneficial Ownership). This approach maximises data integrity, supports complex cross-entity queries, and mirrors how government corporate registries (Companies House UK, SEC EDGAR, ASIC Australia) structure their own data.

The core design principle is that every fact about an entity, officer, meeting, or document is stored exactly once, in a typed column with a foreign key to the appropriate reference table. Jurisdiction-specific variations (e.g. UK PSC vs US beneficial ownership, Delaware franchise tax vs UK confirmation statement) are handled through typed subtables rather than flexible columns. This makes the schema self-documenting: a developer can understand the domain by reading the DDL.

Real-world systems that use this pattern include GLEIF's own LEI database (Level 1 LEI-CDF + Level 2 RR-CDF), the UK Companies House internal data model (exposed via their Public Data API), and enterprise entity management platforms like Diligent Entities (which uses a GraphQL layer over a normalized relational core).

**Best for:** Organisations managing large entity portfolios (PE firms, multinational legal departments) where data integrity, multi-jurisdiction compliance, and auditability are paramount. Teams with strong SQL expertise who value referential integrity and well-defined schemas.

**Trade-offs:**
- (+) Maximum data integrity — every relationship enforced at the database level
- (+) Self-documenting schema — the DDL is the documentation
- (+) Standards-aligned reference data enables interoperability with government registries
- (+) Complex cross-entity queries (e.g. "all directors who serve on boards in both the UK and Delaware") are natural JOINs
- (-) Many JOINs required for common queries — can be mitigated with materialised views
- (-) Schema migrations more involved when adding jurisdiction-specific fields
- (-) Higher upfront design effort (40+ tables)
- (-) Adding a new jurisdiction may require new subtables or reference data rows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 17442 (LEI) | `legal_entity.lei` field; GLEIF LEI-CDF 3.1 field structure informs the entity record layout |
| ISO 3166-1 / 3166-2 | `jurisdiction` table with alpha-2/alpha-3 codes and self-referencing hierarchy for subdivisions |
| ISO 5009 (Official Organizational Roles) | `officer_role_type` reference table with OOR codes per jurisdiction; 2000+ roles across 89 jurisdictions |
| ISO 20275 / GLEIF ELF | `entity_legal_form` reference table for legal form codes (LLC, Ltd, GmbH, etc.) |
| GLEIF RR-CDF 2.1 | `entity_relationship` table models IS_DIRECTLY_CONSOLIDATED_BY, IS_ULTIMATELY_CONSOLIDATED_BY, IS_INTERNATIONAL_BRANCH_OF |
| BODS v0.4 | `beneficial_ownership` table structure follows BODS Entity/Person/Ownership statement model |
| Open Cap Format (OCF) | `share_class`, `share_issuance`, `share_transfer` tables align with OCF transaction types |
| OCSF Base Event | `audit_log` table structure follows OCSF category/class/activity pattern |
| DocuSign Envelope Model | `signing_envelope` / `signing_recipient` / `signing_tab` tables mirror DocuSign hierarchy |
| UK Companies House API | Officer and PSC data structures informed by CH JSON schema (officer_role, identification, date_of_birth) |
| SEC EDGAR API | Entity metadata fields align with EDGAR submissions JSON (cik, entityType, sic, exchanges) |
| XBRL 2026 DEI Taxonomy | Regulatory filing fields align with Document and Entity Information taxonomy elements |

---

## Entity Relationship Diagram (Simplified)

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   jurisdiction    │     │ entity_legal_form │     │ officer_role_type│
│ (ISO 3166)        │     │ (GLEIF ELF)       │     │ (ISO 5009 OOR)   │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌──────────────────────────────────────┐         ┌─────────────────────┐
│            legal_entity              │◄────────│  officer_appointment │
│ (corporation, LLC, LP, trust, etc.)  │         │ (person ↔ entity    │
│                                      │────────►│  ↔ role ↔ dates)    │
└──────┬───────┬───────┬───────┬──────┘         └──────────┬──────────┘
       │       │       │       │                           │
       ▼       │       │       ▼                           ▼
┌──────────┐   │  ┌──────────────┐              ┌─────────────────┐
│entity_   │   │  │ compliance_  │              │     person      │
│relation- │   │  │ obligation   │              │ (natural person │
│ship      │   │  │              │              │  or corporate   │
│(GLEIF    │   │  └──────────────┘              │  officer)       │
│ RR-CDF)  │   │                                └─────────────────┘
└──────────┘   │
               ▼
    ┌──────────────────┐     ┌─────────────────┐
    │     meeting      │────►│  agenda_item     │
    │                  │     └────────┬────────┘
    └──────┬───────────┘              │
           │                          ▼
           ▼                ┌─────────────────┐
    ┌──────────────┐        │   resolution     │
    │meeting_      │        │                  │
    │attendance    │        └────────┬────────┘
    └──────────────┘                 │
                                     ▼
                            ┌─────────────────┐
                            │ signing_envelope │
                            │ (DocuSign model) │
                            └─────────────────┘
```

---

## Core Reference Tables

```sql
-- ============================================================
-- REFERENCE DATA
-- ============================================================

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(6) NOT NULL UNIQUE,          -- ISO 3166-1 alpha-2 or 3166-2 subdivision
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES jurisdiction(id),    -- e.g. US-DE → US
    iso_alpha3      VARCHAR(3),                          -- ISO 3166-1 alpha-3
    iso_numeric     VARCHAR(3),                          -- ISO 3166-1 numeric
    region          VARCHAR(100),                        -- e.g. 'North America', 'Europe'
    is_subdivision  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_id);
CREATE INDEX idx_jurisdiction_code ON jurisdiction(code);

-- ISO 20275 / GLEIF Entity Legal Forms
CREATE TABLE entity_legal_form (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    elf_code        VARCHAR(8) NOT NULL UNIQUE,          -- GLEIF ELF code (e.g. '8888' for LLC)
    name_local      VARCHAR(500) NOT NULL,               -- Name in local language
    name_english    VARCHAR(500),                        -- English translation
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    abbreviations   VARCHAR(255),                        -- e.g. 'LLC, L.L.C.'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_elf_jurisdiction ON entity_legal_form(jurisdiction_id);

-- ISO 5009 Official Organizational Roles
CREATE TABLE officer_role_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oor_code        VARCHAR(6) NOT NULL UNIQUE,          -- ISO 5009 OOR code
    name_local      VARCHAR(500) NOT NULL,               -- Role name in local language
    name_english    VARCHAR(500),                        -- English translation
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    elf_code        VARCHAR(8),                          -- Associated entity legal form
    is_signatory    BOOLEAN NOT NULL DEFAULT FALSE,      -- Can this role sign on behalf of entity?
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oor_jurisdiction ON officer_role_type(jurisdiction_id);

-- Document categories
CREATE TABLE document_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,         -- e.g. 'CONSTITUTION', 'RESOLUTION', 'MINUTES'
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    retention_years INTEGER,                             -- ISO 15489 retention guidance
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
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
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',  -- 'starter', 'standard', 'enterprise'
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'US',          -- ISO 3166-1 alpha-2
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Every tenant-scoped table includes tenant_id as the leading column
-- RLS policy enforces tenant isolation at the database level

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'admin', 'secretary', 'director', 'observer', 'member'
    sso_provider_id VARCHAR(255),                           -- External SSO identifier
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- Row-Level Security
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Entity Management Tables

```sql
-- ============================================================
-- ENTITY MANAGEMENT
-- ============================================================

CREATE TABLE legal_entity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- Core identity (GLEIF LEI-CDF 3.1 aligned)
    legal_name          VARCHAR(500) NOT NULL,
    trading_name        VARCHAR(500),
    lei                 VARCHAR(20) UNIQUE,                  -- ISO 17442 Legal Entity Identifier
    entity_legal_form_id UUID REFERENCES entity_legal_form(id),
    
    -- Jurisdiction
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdiction(id),  -- Jurisdiction of incorporation
    
    -- Registration
    registration_number VARCHAR(100),                        -- e.g. Companies House number, Delaware file number
    registration_date   DATE,
    
    -- Status (GLEIF EntityStatus enum)
    entity_status       VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, INACTIVE, DISSOLVED
    
    -- Address (GLEIF LEI-CDF legal address)
    legal_address_line1 VARCHAR(255),
    legal_address_line2 VARCHAR(255),
    legal_address_city  VARCHAR(100),
    legal_address_region VARCHAR(100),
    legal_address_postal_code VARCHAR(20),
    legal_address_country VARCHAR(2),                       -- ISO 3166-1 alpha-2
    
    -- Headquarters address (GLEIF LEI-CDF HQ address)
    hq_address_line1    VARCHAR(255),
    hq_address_line2    VARCHAR(255),
    hq_address_city     VARCHAR(100),
    hq_address_region   VARCHAR(100),
    hq_address_postal_code VARCHAR(20),
    hq_address_country  VARCHAR(2),
    
    -- EDGAR / SEC fields
    cik                 VARCHAR(10),                         -- SEC Central Index Key
    sic_code            VARCHAR(4),                         -- Standard Industrial Classification
    
    -- Financial year
    fiscal_year_end_month INTEGER CHECK (fiscal_year_end_month BETWEEN 1 AND 12),
    fiscal_year_end_day   INTEGER CHECK (fiscal_year_end_day BETWEEN 1 AND 31),
    
    -- Metadata
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_jurisdiction ON legal_entity(jurisdiction_id);
CREATE INDEX idx_entity_lei ON legal_entity(lei) WHERE lei IS NOT NULL;
CREATE INDEX idx_entity_status ON legal_entity(tenant_id, entity_status);

ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Entity relationships (GLEIF RR-CDF 2.1 aligned)
CREATE TABLE entity_relationship (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- GLEIF RR-CDF: StartNode (child) and EndNode (parent)
    child_entity_id     UUID NOT NULL REFERENCES legal_entity(id),
    parent_entity_id    UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Relationship type (GLEIF RelationshipType enum)
    relationship_type   VARCHAR(50) NOT NULL,
    -- 'IS_DIRECTLY_CONSOLIDATED_BY'
    -- 'IS_ULTIMATELY_CONSOLIDATED_BY'
    -- 'IS_INTERNATIONAL_BRANCH_OF'
    -- 'IS_FUND_MANAGED_BY'
    -- 'IS_SUBFUND_OF'
    -- 'IS_FEEDER_TO'
    
    -- GLEIF qualifiers
    accounting_standard VARCHAR(50),                        -- 'IFRS', 'US_GAAP', 'OTHER'
    
    -- Ownership percentage (GLEIF RelationshipQuantifier)
    ownership_percentage DECIMAL(5,2) CHECK (ownership_percentage BETWEEN 0 AND 100),
    voting_percentage    DECIMAL(5,2) CHECK (voting_percentage BETWEEN 0 AND 100),
    
    -- Temporal validity
    valid_from          DATE NOT NULL,
    valid_to            DATE,                               -- NULL = current
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    CONSTRAINT no_self_reference CHECK (child_entity_id != parent_entity_id)
);

CREATE INDEX idx_entrel_child ON entity_relationship(tenant_id, child_entity_id);
CREATE INDEX idx_entrel_parent ON entity_relationship(tenant_id, parent_entity_id);
CREATE INDEX idx_entrel_type ON entity_relationship(relationship_type);

-- Beneficial ownership (BODS v0.4 aligned)
CREATE TABLE beneficial_ownership (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- BODS: subject entity
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- BODS: interested party (person or entity)
    person_id           UUID REFERENCES person(id),
    interested_entity_id UUID REFERENCES legal_entity(id),
    
    -- BODS: interest details
    interest_type       VARCHAR(50) NOT NULL,
    -- 'shareholding', 'votingRights', 'appointmentOfBoard',
    -- 'controlViaCompanyRulesOrArticles', 'controlByLegalFramework',
    -- 'boardMember', 'boardChair', 'otherInfluenceOrControl'
    
    interest_level      VARCHAR(20),                        -- 'direct', 'indirect'
    share_min           DECIMAL(5,2),
    share_max           DECIMAL(5,2),
    share_exact         DECIMAL(5,2),
    
    -- PSC / BOI details
    is_psc              BOOLEAN NOT NULL DEFAULT FALSE,     -- UK PSC register
    boi_filing_status   VARCHAR(20),                        -- FinCEN BOI status
    
    -- Temporal
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    
    -- Source
    source_type         VARCHAR(50),                        -- 'selfDeclaration', 'officialRegister', 'thirdParty'
    source_description  TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    CONSTRAINT has_interested_party CHECK (
        (person_id IS NOT NULL AND interested_entity_id IS NULL) OR
        (person_id IS NULL AND interested_entity_id IS NOT NULL)
    )
);

CREATE INDEX idx_bo_entity ON beneficial_ownership(tenant_id, entity_id);
CREATE INDEX idx_bo_person ON beneficial_ownership(person_id) WHERE person_id IS NOT NULL;
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
    
    -- Identity
    full_name           VARCHAR(500) NOT NULL,
    given_name          VARCHAR(255),
    family_name         VARCHAR(255),
    title               VARCHAR(50),                        -- Mr, Ms, Dr, etc.
    
    -- Date of birth (Companies House model: month/year only for public display)
    date_of_birth       DATE,
    dob_display_month   INTEGER,                            -- For public registry display (CH model)
    dob_display_year    INTEGER,
    
    -- Nationality & residency
    nationality         VARCHAR(2),                         -- ISO 3166-1 alpha-2
    country_of_residence VARCHAR(2),                        -- ISO 3166-1 alpha-2
    
    -- Address (service address — for public registries)
    service_address_line1 VARCHAR(255),
    service_address_line2 VARCHAR(255),
    service_address_city  VARCHAR(100),
    service_address_region VARCHAR(100),
    service_address_postal_code VARCHAR(20),
    service_address_country VARCHAR(2),
    
    -- Residential address (private — GDPR protected)
    residential_address_line1 VARCHAR(255),
    residential_address_line2 VARCHAR(255),
    residential_address_city  VARCHAR(100),
    residential_address_region VARCHAR(100),
    residential_address_postal_code VARCHAR(20),
    residential_address_country VARCHAR(2),
    
    -- Contact
    email               VARCHAR(320),
    phone               VARCHAR(50),
    
    -- Link to app user (if this person is also a platform user)
    app_user_id         UUID REFERENCES app_user(id),
    
    -- Identification (Companies House identification model)
    identification_type VARCHAR(50),                        -- 'passport', 'drivingLicence', 'nationalId'
    
    -- Former names (Companies House model)
    -- Stored in person_former_name table
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_user ON person(app_user_id) WHERE app_user_id IS NOT NULL;
CREATE INDEX idx_person_name ON person(tenant_id, family_name, given_name);

CREATE TABLE person_former_name (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    forenames       VARCHAR(255),
    surname         VARCHAR(255) NOT NULL,
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Officer appointments
CREATE TABLE officer_appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- Who
    person_id           UUID NOT NULL REFERENCES person(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Role (ISO 5009 OOR)
    role_type_id        UUID NOT NULL REFERENCES officer_role_type(id),
    role_description    VARCHAR(255),                       -- Free-text override if needed
    
    -- Dates
    appointed_on        DATE NOT NULL,
    resigned_on         DATE,                               -- NULL = currently appointed
    
    -- Appointing authority
    resolution_id       UUID REFERENCES resolution(id),    -- Resolution that appointed this officer
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, RESIGNED, REMOVED, DISQUALIFIED
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appointment_entity ON officer_appointment(tenant_id, entity_id);
CREATE INDEX idx_appointment_person ON officer_appointment(tenant_id, person_id);
CREATE INDEX idx_appointment_active ON officer_appointment(tenant_id, entity_id, status) 
    WHERE status = 'ACTIVE';

-- Conflict of interest declarations
CREATE TABLE conflict_of_interest (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    nature_of_interest  TEXT NOT NULL,
    declared_on         DATE NOT NULL,
    reviewed_on         DATE,
    reviewed_by         UUID REFERENCES app_user(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'DECLARED',  -- DECLARED, REVIEWED, RESOLVED
    
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
    
    name                VARCHAR(255) NOT NULL,              -- e.g. 'Board of Directors', 'Audit Committee'
    committee_type      VARCHAR(50) NOT NULL,               -- 'BOARD', 'AUDIT', 'REMUNERATION', 'NOMINATION', 'RISK', 'CUSTOM'
    charter             TEXT,                               -- Committee charter / terms of reference
    
    quorum_count        INTEGER,                            -- Minimum members for quorum
    quorum_percentage   DECIMAL(5,2),                       -- Alternative: percentage-based quorum
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_committee_entity ON committee(tenant_id, entity_id);

CREATE TABLE committee_membership (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    committee_id        UUID NOT NULL REFERENCES committee(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    
    role                VARCHAR(50) NOT NULL DEFAULT 'MEMBER',  -- 'CHAIR', 'MEMBER', 'SECRETARY', 'OBSERVER'
    appointed_on        DATE NOT NULL,
    resigned_on         DATE,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(committee_id, person_id, appointed_on)
);

CREATE TABLE meeting (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    committee_id        UUID REFERENCES committee(id),
    
    -- Meeting details
    title               VARCHAR(500) NOT NULL,
    meeting_type        VARCHAR(50) NOT NULL,               -- 'BOARD', 'COMMITTEE', 'AGM', 'EGM', 'SPECIAL'
    meeting_number      INTEGER,                            -- Sequential meeting number
    
    -- Schedule
    scheduled_start     TIMESTAMPTZ NOT NULL,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    timezone            VARCHAR(50) NOT NULL DEFAULT 'UTC',
    
    -- Location
    location_type       VARCHAR(20) NOT NULL DEFAULT 'IN_PERSON',  -- 'IN_PERSON', 'VIRTUAL', 'HYBRID'
    location_address    TEXT,
    virtual_meeting_url VARCHAR(500),
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED',
    -- 'DRAFT', 'SCHEDULED', 'BOARD_PACK_SENT', 'IN_PROGRESS', 'CONCLUDED', 'MINUTES_DRAFT', 'MINUTES_APPROVED', 'CANCELLED'
    
    -- Quorum
    quorum_present      BOOLEAN,
    
    -- Board pack
    board_pack_sent_at  TIMESTAMPTZ,
    
    -- Minutes
    minutes_draft_text  TEXT,                               -- AI-generated draft
    minutes_final_text  TEXT,                               -- Approved minutes
    minutes_approved_at TIMESTAMPTZ,
    minutes_approved_by UUID REFERENCES app_user(id),
    
    -- AI transcript
    transcript_url      VARCHAR(500),
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_meeting_entity ON meeting(tenant_id, entity_id);
CREATE INDEX idx_meeting_date ON meeting(tenant_id, scheduled_start);
CREATE INDEX idx_meeting_status ON meeting(tenant_id, status);

CREATE TABLE agenda_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meeting(id) ON DELETE CASCADE,
    
    item_number         VARCHAR(20) NOT NULL,               -- e.g. '1', '2.1', '2.2'
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    item_type           VARCHAR(50) NOT NULL DEFAULT 'DISCUSSION',
    -- 'PROCEDURAL', 'APPROVAL', 'DISCUSSION', 'INFORMATION', 'RESOLUTION', 'AOB'
    
    presenter_id        UUID REFERENCES person(id),
    duration_minutes    INTEGER,
    sort_order          INTEGER NOT NULL,
    
    -- Outcome (filled after meeting)
    outcome_notes       TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agenda_meeting ON agenda_item(meeting_id);

CREATE TABLE meeting_attendance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL REFERENCES meeting(id) ON DELETE CASCADE,
    person_id           UUID NOT NULL REFERENCES person(id),
    
    attendance_type     VARCHAR(20) NOT NULL DEFAULT 'MEMBER',  -- 'MEMBER', 'SECRETARY', 'OBSERVER', 'INVITEE', 'PROXY'
    rsvp_status         VARCHAR(20),                        -- 'ACCEPTED', 'DECLINED', 'TENTATIVE', 'NO_RESPONSE'
    attended            BOOLEAN,                            -- Actual attendance
    attended_via        VARCHAR(20),                        -- 'IN_PERSON', 'VIDEO', 'PHONE'
    proxy_for_id        UUID REFERENCES person(id),         -- If attending as proxy
    
    joined_at           TIMESTAMPTZ,
    left_at             TIMESTAMPTZ,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(meeting_id, person_id)
);

CREATE INDEX idx_attendance_meeting ON meeting_attendance(meeting_id);
CREATE INDEX idx_attendance_person ON meeting_attendance(person_id);

-- Action items extracted from meetings
CREATE TABLE action_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    meeting_id          UUID REFERENCES meeting(id),
    agenda_item_id      UUID REFERENCES agenda_item(id),
    
    description         TEXT NOT NULL,
    assigned_to         UUID REFERENCES person(id),
    due_date            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'OPEN',  -- 'OPEN', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED'
    completed_at        TIMESTAMPTZ,
    
    -- AI extraction metadata
    ai_extracted        BOOLEAN NOT NULL DEFAULT FALSE,
    ai_confidence       DECIMAL(3,2),                       -- 0.00 to 1.00
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_action_tenant ON action_item(tenant_id);
CREATE INDEX idx_action_meeting ON action_item(meeting_id);
CREATE INDEX idx_action_assignee ON action_item(assigned_to, status);
```

---

## Resolutions & Voting

```sql
-- ============================================================
-- RESOLUTIONS & VOTING
-- ============================================================

CREATE TABLE resolution (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- Resolution identity
    resolution_number   VARCHAR(50),                        -- e.g. 'BR-2026-042'
    title               VARCHAR(500) NOT NULL,
    body_text           TEXT NOT NULL,
    
    -- Type
    resolution_type     VARCHAR(50) NOT NULL,
    -- 'BOARD_RESOLUTION', 'SHAREHOLDER_ORDINARY', 'SHAREHOLDER_SPECIAL',
    -- 'WRITTEN_CONSENT', 'COMMITTEE_RESOLUTION'
    
    -- Source
    meeting_id          UUID REFERENCES meeting(id),        -- NULL for written consents
    agenda_item_id      UUID REFERENCES agenda_item(id),
    
    -- Voting requirements
    approval_threshold  DECIMAL(5,2) NOT NULL DEFAULT 50.01,  -- Percentage required to pass
    
    -- Outcome
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    -- 'DRAFT', 'PROPOSED', 'VOTING_OPEN', 'PASSED', 'FAILED', 'WITHDRAWN'
    
    votes_for           INTEGER,
    votes_against       INTEGER,
    votes_abstained     INTEGER,
    
    passed_at           TIMESTAMPTZ,
    effective_date      DATE,
    
    -- E-signature
    signing_envelope_id UUID REFERENCES signing_envelope(id),
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resolution_entity ON resolution(tenant_id, entity_id);
CREATE INDEX idx_resolution_meeting ON resolution(meeting_id);
CREATE INDEX idx_resolution_status ON resolution(tenant_id, status);

CREATE TABLE resolution_vote (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resolution_id       UUID NOT NULL REFERENCES resolution(id),
    person_id           UUID NOT NULL REFERENCES person(id),
    
    vote                VARCHAR(10) NOT NULL,               -- 'FOR', 'AGAINST', 'ABSTAIN'
    voted_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- For shareholder votes
    shares_voted        BIGINT,
    
    -- Signature
    signature_method    VARCHAR(20),                        -- 'ELECTRONIC', 'WET_INK', 'QUALIFIED'
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(resolution_id, person_id)
);

CREATE INDEX idx_vote_resolution ON resolution_vote(resolution_id);
```

---

## E-Signature Integration (DocuSign Envelope Model)

```sql
-- ============================================================
-- E-SIGNATURE INTEGRATION (DocuSign Envelope Model)
-- ============================================================

CREATE TABLE signing_envelope (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    
    -- External provider reference
    provider            VARCHAR(20) NOT NULL DEFAULT 'DOCUSIGN',  -- 'DOCUSIGN', 'ADOBE_SIGN', 'INTERNAL'
    external_envelope_id VARCHAR(255),                      -- DocuSign envelope ID
    
    -- Subject
    email_subject       VARCHAR(500) NOT NULL,
    message             TEXT,
    
    -- Status (DocuSign EnvelopeStatus)
    status              VARCHAR(20) NOT NULL DEFAULT 'CREATED',
    -- 'CREATED', 'SENT', 'DELIVERED', 'SIGNED', 'COMPLETED', 'DECLINED', 'VOIDED'
    
    -- Compliance
    signature_standard  VARCHAR(20) NOT NULL DEFAULT 'ESIGN',  -- 'ESIGN', 'EIDAS_SES', 'EIDAS_ADES', 'EIDAS_QES'
    
    sent_at             TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    voided_at           TIMESTAMPTZ,
    void_reason         TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_envelope_tenant ON signing_envelope(tenant_id);
CREATE INDEX idx_envelope_status ON signing_envelope(tenant_id, status);

CREATE TABLE signing_recipient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    envelope_id         UUID NOT NULL REFERENCES signing_envelope(id) ON DELETE CASCADE,
    person_id           UUID REFERENCES person(id),
    
    recipient_type      VARCHAR(20) NOT NULL DEFAULT 'SIGNER',  -- 'SIGNER', 'CC', 'IN_PERSON_SIGNER', 'WITNESS'
    routing_order       INTEGER NOT NULL DEFAULT 1,
    
    name                VARCHAR(255) NOT NULL,
    email               VARCHAR(320) NOT NULL,
    
    status              VARCHAR(20) NOT NULL DEFAULT 'CREATED',
    -- 'CREATED', 'SENT', 'DELIVERED', 'SIGNED', 'DECLINED'
    
    signed_at           TIMESTAMPTZ,
    declined_at         TIMESTAMPTZ,
    decline_reason      TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recipient_envelope ON signing_recipient(envelope_id);
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
    
    -- Document identity
    title               VARCHAR(500) NOT NULL,
    category_id         UUID REFERENCES document_category(id),
    document_type       VARCHAR(50) NOT NULL,
    -- 'BOARD_PACK', 'MINUTES', 'RESOLUTION', 'CONSTITUTION',
    -- 'SHARE_CERTIFICATE', 'ANNUAL_RETURN', 'FINANCIAL_STATEMENT', 'OTHER'
    
    -- Version control
    version             INTEGER NOT NULL DEFAULT 1,
    is_current_version  BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID REFERENCES document(id),       -- Previous version
    
    -- Storage
    storage_provider    VARCHAR(20) NOT NULL DEFAULT 'S3',
    storage_key         VARCHAR(500) NOT NULL,
    file_name           VARCHAR(255) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT,
    checksum_sha256     VARCHAR(64),
    
    -- Encryption
    encrypted_at_rest   BOOLEAN NOT NULL DEFAULT TRUE,
    encryption_key_id   VARCHAR(255),
    
    -- Retention (ISO 15489)
    retention_class     VARCHAR(50),                        -- From document_category
    retain_until        DATE,
    disposal_action     VARCHAR(20),                        -- 'DESTROY', 'ARCHIVE', 'REVIEW'
    
    -- Meeting linkage
    meeting_id          UUID REFERENCES meeting(id),
    agenda_item_id      UUID REFERENCES agenda_item(id),
    
    -- Access control
    confidentiality     VARCHAR(20) NOT NULL DEFAULT 'CONFIDENTIAL',
    -- 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED'
    
    uploaded_by         UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_tenant ON document(tenant_id);
CREATE INDEX idx_document_entity ON document(tenant_id, entity_id);
CREATE INDEX idx_document_meeting ON document(meeting_id);
CREATE INDEX idx_document_type ON document(tenant_id, document_type);

-- Granular document permissions
CREATE TABLE document_permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    
    -- Grantee (user or role-based)
    user_id         UUID REFERENCES app_user(id),
    role            VARCHAR(50),                            -- Alternative: grant to all users with this role
    committee_id    UUID REFERENCES committee(id),          -- Alternative: grant to committee members
    
    permission      VARCHAR(20) NOT NULL,                   -- 'VIEW', 'DOWNLOAD', 'ANNOTATE', 'EDIT'
    
    granted_by      UUID REFERENCES app_user(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    
    CONSTRAINT has_grantee CHECK (
        user_id IS NOT NULL OR role IS NOT NULL OR committee_id IS NOT NULL
    )
);

CREATE INDEX idx_docperm_document ON document_permission(document_id);
CREATE INDEX idx_docperm_user ON document_permission(user_id) WHERE user_id IS NOT NULL;

-- Director annotations (private notes on board materials)
CREATE TABLE document_annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    
    page_number     INTEGER,
    position_x      DECIMAL(7,2),
    position_y      DECIMAL(7,2),
    annotation_type VARCHAR(20) NOT NULL DEFAULT 'NOTE',    -- 'NOTE', 'HIGHLIGHT', 'BOOKMARK'
    content         TEXT,
    color           VARCHAR(7),                             -- Hex color
    is_private      BOOLEAN NOT NULL DEFAULT TRUE,          -- Private to the director
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotation_document ON document_annotation(document_id);
CREATE INDEX idx_annotation_user ON document_annotation(user_id);
```

---

## Compliance & Filing Management

```sql
-- ============================================================
-- COMPLIANCE & FILING MANAGEMENT
-- ============================================================

-- Compliance obligation templates (per jurisdiction)
CREATE TABLE compliance_obligation_template (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdiction(id),
    entity_legal_form_id UUID REFERENCES entity_legal_form(id),
    
    name                VARCHAR(500) NOT NULL,
    description         TEXT,
    obligation_type     VARCHAR(50) NOT NULL,
    -- 'ANNUAL_RETURN', 'CONFIRMATION_STATEMENT', 'FRANCHISE_TAX',
    -- 'BENEFICIAL_OWNERSHIP', 'FINANCIAL_FILING', 'REGISTERED_AGENT_RENEWAL'
    
    -- Recurrence
    frequency           VARCHAR(20) NOT NULL,               -- 'ANNUAL', 'QUARTERLY', 'MONTHLY', 'ONE_TIME', 'EVENT_DRIVEN'
    
    -- Deadline calculation
    deadline_months_after_year_end INTEGER,                  -- e.g. 3 = 3 months after fiscal year end
    deadline_fixed_month INTEGER,                           -- Fixed month (e.g. March 1 annually)
    deadline_fixed_day   INTEGER,
    
    -- Penalty information
    penalty_description TEXT,
    penalty_amount      DECIMAL(12,2),
    penalty_currency    VARCHAR(3),                         -- ISO 4217
    
    -- Filing details
    filing_authority    VARCHAR(255),                       -- e.g. 'Delaware Division of Corporations'
    filing_url          VARCHAR(500),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_obligation_template_jurisdiction ON compliance_obligation_template(jurisdiction_id);

-- Instance of a compliance obligation for a specific entity
CREATE TABLE compliance_obligation (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    template_id         UUID REFERENCES compliance_obligation_template(id),
    
    name                VARCHAR(500) NOT NULL,
    description         TEXT,
    
    -- Period
    period_start        DATE,
    period_end          DATE,
    
    -- Deadline
    due_date            DATE NOT NULL,
    
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'UPCOMING',
    -- 'UPCOMING', 'DUE_SOON', 'OVERDUE', 'IN_PROGRESS', 'FILED', 'COMPLETED', 'WAIVED'
    
    filed_on            DATE,
    filed_by            UUID REFERENCES app_user(id),
    confirmation_number VARCHAR(255),
    
    -- Cost
    filing_fee          DECIMAL(12,2),
    filing_fee_currency VARCHAR(3),
    
    -- AI monitoring
    ai_flagged          BOOLEAN NOT NULL DEFAULT FALSE,
    ai_flag_reason      TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_obligation_entity ON compliance_obligation(tenant_id, entity_id);
CREATE INDEX idx_obligation_due ON compliance_obligation(tenant_id, due_date, status);
CREATE INDEX idx_obligation_status ON compliance_obligation(tenant_id, status);

-- Regulatory change tracking
CREATE TABLE regulatory_change (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdiction(id),
    
    title               VARCHAR(500) NOT NULL,
    description         TEXT NOT NULL,
    source_url          VARCHAR(500),
    source_authority    VARCHAR(255),
    
    change_type         VARCHAR(50) NOT NULL,               -- 'NEW_LAW', 'AMENDMENT', 'REPEAL', 'GUIDANCE', 'ENFORCEMENT'
    effective_date      DATE,
    published_date      DATE,
    
    -- Impact assessment
    impact_level        VARCHAR(10),                        -- 'HIGH', 'MEDIUM', 'LOW'
    affected_entity_types TEXT[],                           -- Array of entity legal form codes affected
    
    -- AI detection
    ai_detected         BOOLEAN NOT NULL DEFAULT FALSE,
    ai_summary          TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_regchange_jurisdiction ON regulatory_change(jurisdiction_id);
CREATE INDEX idx_regchange_date ON regulatory_change(effective_date);
```

---

## Share Register (OCF-aligned)

```sql
-- ============================================================
-- SHARE REGISTER (Open Cap Format aligned)
-- ============================================================

CREATE TABLE share_class (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    
    -- OCF StockClass fields
    name                VARCHAR(255) NOT NULL,              -- e.g. 'Class A Common Stock'
    class_type          VARCHAR(20) NOT NULL,               -- 'COMMON', 'PREFERRED'
    
    shares_authorised   BIGINT,
    par_value           DECIMAL(12,6),
    par_value_currency  VARCHAR(3),                         -- ISO 4217
    
    votes_per_share     DECIMAL(10,2) NOT NULL DEFAULT 1.0,
    
    -- Preferences (for preferred stock)
    liquidation_preference_multiple DECIMAL(5,2),
    is_participating    BOOLEAN,
    dividend_rate       DECIMAL(5,4),
    
    seniority           INTEGER,                            -- 1 = most senior
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shareclass_entity ON share_class(tenant_id, entity_id);

-- OCF Transaction model: share issuances, transfers, cancellations
CREATE TABLE share_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    share_class_id      UUID NOT NULL REFERENCES share_class(id),
    
    -- OCF transaction type
    transaction_type    VARCHAR(30) NOT NULL,
    -- 'ISSUANCE', 'TRANSFER', 'CANCELLATION', 'REPURCHASE',
    -- 'SPLIT', 'CONVERSION', 'EXERCISE'
    
    -- Parties
    from_stakeholder_id UUID REFERENCES person(id),         -- NULL for issuance
    to_stakeholder_id   UUID REFERENCES person(id),         -- NULL for cancellation
    
    -- Shares
    quantity            BIGINT NOT NULL,
    price_per_share     DECIMAL(12,6),
    price_currency      VARCHAR(3),
    
    -- Dates
    transaction_date    DATE NOT NULL,
    board_approval_date DATE,
    
    -- Authorising resolution
    resolution_id       UUID REFERENCES resolution(id),
    
    -- Certificate
    certificate_number  VARCHAR(50),
    
    -- Legal basis
    consideration       TEXT,                               -- e.g. 'Cash', 'Services', 'IP Assignment'
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sharetx_entity ON share_transaction(tenant_id, entity_id);
CREATE INDEX idx_sharetx_date ON share_transaction(transaction_date);
CREATE INDEX idx_sharetx_stakeholder ON share_transaction(to_stakeholder_id);
```

---

## Audit Log (OCSF-aligned)

```sql
-- ============================================================
-- AUDIT LOG (OCSF Base Event aligned)
-- ============================================================

CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,                      -- Not FK'd for performance
    
    -- OCSF Base Event fields
    event_time          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- OCSF category (mapped to governance domain)
    category            VARCHAR(50) NOT NULL,
    -- 'ENTITY_MANAGEMENT', 'MEETING_MANAGEMENT', 'DOCUMENT_ACCESS',
    -- 'RESOLUTION_VOTING', 'OFFICER_CHANGE', 'COMPLIANCE', 'AUTHENTICATION'
    
    -- OCSF class
    event_class         VARCHAR(50) NOT NULL,
    -- 'entity.created', 'entity.updated', 'officer.appointed', 'officer.resigned',
    -- 'meeting.created', 'meeting.concluded', 'document.viewed', 'document.downloaded',
    -- 'resolution.passed', 'vote.cast', 'obligation.filed', 'user.login', 'user.logout'
    
    -- OCSF activity
    activity            VARCHAR(20) NOT NULL,               -- 'CREATE', 'READ', 'UPDATE', 'DELETE', 'LOGIN', 'LOGOUT', 'EXPORT'
    
    -- Actor
    actor_user_id       UUID,
    actor_email         VARCHAR(320),
    actor_ip_address    INET,
    actor_user_agent    TEXT,
    
    -- Target
    target_type         VARCHAR(50) NOT NULL,               -- Table name or object type
    target_id           UUID NOT NULL,
    target_name         VARCHAR(500),
    
    -- Change details
    old_values          JSONB,                              -- Previous field values (for updates)
    new_values          JSONB,                              -- New field values (for updates)
    
    -- Outcome
    status              VARCHAR(10) NOT NULL DEFAULT 'SUCCESS',  -- 'SUCCESS', 'FAILURE'
    status_detail       TEXT,
    
    -- Severity (OCSF)
    severity            VARCHAR(10) NOT NULL DEFAULT 'INFO',  -- 'INFO', 'LOW', 'MEDIUM', 'HIGH', 'CRITICAL'
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

-- Partition by month for performance
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional monthly partitions created by cron job

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, event_time DESC);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id, event_time DESC);
CREATE INDEX idx_audit_class ON audit_log(event_class, event_time DESC);
```

---

## Example Queries

```sql
-- All active directors of subsidiaries in Delaware
SELECT p.full_name, le.legal_name, ort.name_english AS role
FROM officer_appointment oa
JOIN person p ON p.id = oa.person_id
JOIN legal_entity le ON le.id = oa.entity_id
JOIN officer_role_type ort ON ort.id = oa.role_type_id
JOIN jurisdiction j ON j.id = le.jurisdiction_id
WHERE oa.tenant_id = :tenant_id
  AND oa.status = 'ACTIVE'
  AND j.code = 'US-DE'
ORDER BY le.legal_name, p.family_name;

-- Corporate structure tree (recursive CTE)
WITH RECURSIVE entity_tree AS (
    SELECT le.id, le.legal_name, er.parent_entity_id, er.ownership_percentage, 1 AS depth
    FROM legal_entity le
    LEFT JOIN entity_relationship er ON er.child_entity_id = le.id
        AND er.relationship_type = 'IS_DIRECTLY_CONSOLIDATED_BY'
        AND er.status = 'ACTIVE'
    WHERE le.id = :ultimate_parent_id
      AND le.tenant_id = :tenant_id
    
    UNION ALL
    
    SELECT le.id, le.legal_name, er.parent_entity_id, er.ownership_percentage, et.depth + 1
    FROM entity_relationship er
    JOIN legal_entity le ON le.id = er.child_entity_id
    JOIN entity_tree et ON et.id = er.parent_entity_id
    WHERE er.relationship_type = 'IS_DIRECTLY_CONSOLIDATED_BY'
      AND er.status = 'ACTIVE'
)
SELECT * FROM entity_tree ORDER BY depth, legal_name;

-- Upcoming compliance deadlines across all entities
SELECT le.legal_name, j.name AS jurisdiction, co.name AS obligation,
       co.due_date, co.status,
       (co.due_date - CURRENT_DATE) AS days_until_due
FROM compliance_obligation co
JOIN legal_entity le ON le.id = co.entity_id
JOIN jurisdiction j ON j.id = le.jurisdiction_id
WHERE co.tenant_id = :tenant_id
  AND co.status IN ('UPCOMING', 'DUE_SOON', 'OVERDUE')
ORDER BY co.due_date;

-- Director overlap analysis (directors who serve on multiple entity boards)
SELECT p.full_name, COUNT(DISTINCT oa.entity_id) AS entities_served,
       ARRAY_AGG(DISTINCT le.legal_name) AS entity_names
FROM officer_appointment oa
JOIN person p ON p.id = oa.person_id
JOIN legal_entity le ON le.id = oa.entity_id
WHERE oa.tenant_id = :tenant_id
  AND oa.status = 'ACTIVE'
GROUP BY p.id, p.full_name
HAVING COUNT(DISTINCT oa.entity_id) > 1
ORDER BY entities_served DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Reference Data | 4 | jurisdiction, entity_legal_form, officer_role_type, document_category |
| Multi-Tenant Foundation | 2 | tenant, app_user |
| Entity Management | 3 | legal_entity, entity_relationship, beneficial_ownership |
| Person & Officers | 4 | person, person_former_name, officer_appointment, conflict_of_interest |
| Meeting Management | 5 | committee, committee_membership, meeting, agenda_item, meeting_attendance |
| Action Items | 1 | action_item |
| Resolutions & Voting | 2 | resolution, resolution_vote |
| E-Signature | 2 | signing_envelope, signing_recipient |
| Document Management | 3 | document, document_permission, document_annotation |
| Compliance & Filing | 3 | compliance_obligation_template, compliance_obligation, regulatory_change |
| Share Register | 2 | share_class, share_transaction |
| Audit Log | 1 | audit_log (partitioned) |
| **Total** | **32** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — standard for modern SaaS; avoids sequential ID enumeration attacks, simplifies multi-region replication.

2. **tenant_id as leading column on all tenant-scoped tables** — enables PostgreSQL Row-Level Security (RLS) policies and ensures composite indexes lead with tenant_id for optimal query performance.

3. **Standards-aligned reference tables** — jurisdiction (ISO 3166), entity_legal_form (GLEIF ELF / ISO 20275), officer_role_type (ISO 5009) are pre-populated from official registries. This enables interoperability with Companies House, EDGAR, and GLEIF APIs.

4. **GLEIF RR-CDF relationship model** — entity_relationship uses the GLEIF relationship type vocabulary (IS_DIRECTLY_CONSOLIDATED_BY, etc.) so ownership structures can be exported in a format compatible with the Global LEI System.

5. **BODS-aligned beneficial ownership** — beneficial_ownership follows the BODS v0.4 three-statement model, supporting both UK PSC and US FinCEN BOI compliance.

6. **Template/instance pattern for compliance obligations** — compliance_obligation_template defines the rules per jurisdiction; compliance_obligation is the instantiated deadline for a specific entity. This separation allows the platform to pre-load compliance rules for 100+ jurisdictions.

7. **DocuSign envelope hierarchy for e-signatures** — signing_envelope / signing_recipient mirrors the DocuSign REST API data model, making integration straightforward while supporting multiple providers.

8. **OCF-aligned share register** — share_class and share_transaction follow the Open Cap Format transaction model, enabling import/export with cap table tools.

9. **OCSF-aligned audit log with time-based partitioning** — audit_log follows the OCSF Base Event pattern and is partitioned monthly for query performance and retention management (ISO 15489).

10. **Separate service and residential addresses for persons** — follows the Companies House UK model where service addresses are public but residential addresses are GDPR-protected.
