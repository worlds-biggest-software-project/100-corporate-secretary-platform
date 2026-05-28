# Data Model Suggestion 4: Graph-Relational Model for Ownership & Director Networks

> Project: Corporate Secretary Platform · Created: 2026-05-12

## Philosophy

Corporate governance is fundamentally a domain of **relationships**: entities own other entities, people serve on multiple boards, committees report to boards, resolutions authorise appointments, and documents evidence transactions. The most valuable analytical questions — "who controls this entity chain?", "which directors overlap across our portfolio?", "trace the ownership path from ultimate parent to this subsidiary", "does this proposed board appointment create a conflict of interest?" — are **graph traversal problems** that relational databases handle awkwardly with recursive CTEs and self-joins.

This model uses a **property graph** approach implemented on top of PostgreSQL, combining relational tables for operational CRUD (meetings, documents, compliance) with graph-optimised structures for relationship queries (ownership chains, director networks, control paths). It draws from:

- **GLEIF Level 2 Data ("Who Owns Whom")** — relationship records with typed edges, period dates, and accounting standard qualifiers. The GLEIF data model is inherently a directed graph where entities are nodes and ownership/consolidation relationships are edges.
- **BODS v0.4 (Beneficial Ownership Data Standard)** — three node types (Entity Statement, Person Statement, Ownership-or-Control Statement) forming a directed graph. The `interestType` codelist includes edge types like `shareholding`, `votingRights`, `appointmentOfBoard`, `boardMember`, `boardChair`.
- **OpenCorporates** — models corporate control as a directed graph with typed control mechanisms.
- **Neo4j/property graph patterns** — nodes with labels and properties, edges with types, directions, and properties.
- **PostgreSQL ltree** — materialised path extension that stores hierarchical labels as dot-separated strings, enabling efficient ancestor/descendant queries without recursive CTEs.

**Best for:** Platforms where complex corporate structures are the primary challenge — PE portfolio management, multinational subsidiary governance, beneficial ownership compliance (FinCEN BOI, UK PSC, EU AMLD6), director network analysis, and conflict-of-interest detection. Also excellent for AI-powered features like governance risk scoring, director overlap analysis, and anomaly detection across corporate structures.

**Trade-offs:**
- (+) Ownership chain traversal is O(depth) not O(n) — dramatically faster for deep hierarchies
- (+) Director network analysis (overlaps, conflicts, independence) is a natural graph query
- (+) Beneficial ownership tracing through intermediate entities is trivial
- (+) ltree enables ancestor/descendant queries without recursive CTEs
- (+) AI analytics on graph structure (centrality, clustering, anomaly detection) are first-class
- (-) Dual-store architecture (relational for CRUD + graph for analytics) adds operational complexity
- (-) Graph queries require different developer skills than SQL
- (-) In PostgreSQL-only mode, deep traversals with recursive CTEs are still expensive
- (-) Less differentiated for meetings/documents/compliance (those remain relational)
- (-) Change Data Capture needed to keep graph store in sync with relational tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GLEIF RR-CDF 2.1 | Edge types: IS_DIRECTLY_CONSOLIDATED_BY, IS_ULTIMATELY_CONSOLIDATED_BY, IS_INTERNATIONAL_BRANCH_OF map directly to graph edge types |
| GLEIF LEI-CDF 3.1 | Entity node properties align with LEI record fields |
| BODS v0.4 | Three-node model (Entity Statement, Person Statement, Ownership Statement) maps to graph nodes and edges. Interest types become edge properties |
| ISO 5009 (OOR) | Officer role codes become edge labels/properties on SERVES_AS edges |
| ISO 3166 | Jurisdiction codes as node properties for geographic filtering |
| ISO 20275 / GLEIF ELF | Legal form codes as node properties |
| Companies House API | UK officer and PSC data structures map to Person nodes + SERVES_AS / CONTROLS edges |
| OpenCorporates | Control statement model informs edge typing for beneficial ownership |
| PostgreSQL ltree | Materialised paths for efficient hierarchy queries without recursive CTEs |

---

## Architecture: Dual-Store Approach

```
┌────────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                               │
│                                                                     │
│   CRUD Operations          │    Graph Queries / Analytics          │
│   (meetings, documents,    │    (ownership traversal, director    │
│    resolutions, compliance)│     networks, conflict detection,    │
│           │                │     AI governance risk scoring)      │
│           │                │              │                        │
└───────────┼────────────────┼──────────────┼────────────────────────┘
            ▼                               ▼
┌───────────────────────┐       ┌──────────────────────────┐
│    PostgreSQL          │       │    Graph Layer             │
│                        │  CDC  │                            │
│  Operational tables    │──────►│  Option A: PostgreSQL      │
│  (meetings, docs,      │       │    with ltree + recursive  │
│   resolutions, etc.)   │       │    CTEs + graph tables     │
│                        │       │                            │
│  Graph source tables:  │       │  Option B: Neo4j / Neptune │
│  • graph_node          │       │    (dedicated graph DB)    │
│  • graph_edge          │       │                            │
│                        │       │  Nodes: Entity, Person     │
│  Hierarchy support:    │       │  Edges: OWNS, CONTROLS,    │
│  • entity.ownership_   │       │    SERVES_AS, REPORTS_TO,  │
│    path (ltree)        │       │    BENEFITS_FROM           │
│                        │       │                            │
└───────────────────────┘       └──────────────────────────┘
```

---

## Graph Node & Edge Tables (PostgreSQL Implementation)

```sql
-- ============================================================
-- GRAPH LAYER (PostgreSQL-native implementation)
-- ============================================================

-- Enable ltree extension for materialised path hierarchies
CREATE EXTENSION IF NOT EXISTS ltree;

-- ============================================================
-- GRAPH NODES
-- ============================================================

-- Graph nodes represent entities and persons in the corporate structure.
-- Each node has a type, a reference to the operational table, and properties.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    
    -- Node type (label in graph terminology)
    node_type       VARCHAR(20) NOT NULL,
    -- 'ENTITY'   → references legal_entity
    -- 'PERSON'   → references person
    -- 'TRUST'    → references legal_entity (BODS trust modeling)
    -- 'STATE'    → references reference_data (government/regulator node)
    
    -- Reference to operational table
    ref_id          UUID NOT NULL,                          -- FK to legal_entity.id or person.id
    
    -- Display properties (denormalised for graph query performance)
    display_name    VARCHAR(500) NOT NULL,
    
    -- Node properties (JSONB for flexibility)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- For ENTITY nodes:
    -- {
    --   "legal_name": "Acme Holdings Ltd",
    --   "lei": "549300ABCDEF12345678",
    --   "jurisdiction_code": "GB",
    --   "legal_form_code": "8888",
    --   "entity_status": "ACTIVE",
    --   "registration_number": "12345678",
    --   "incorporation_date": "2015-03-20",
    --   "sic_code": "6199"
    -- }
    -- For PERSON nodes:
    -- {
    --   "full_name": "Jane Smith",
    --   "nationality": "GB",
    --   "country_of_residence": "GB",
    --   "date_of_birth_year": 1985,
    --   "is_pep": false
    -- }
    
    -- Hierarchy path (ltree — for ownership tree)
    -- Format: "tenant_slug.ultimate_parent_id.intermediate_id.this_id"
    ownership_path  LTREE,
    
    -- Computed metrics (updated by graph processors)
    centrality_score DECIMAL(5,4),                         -- Betweenness centrality
    cluster_id       INTEGER,                              -- Community detection result
    
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, node_type, ref_id)
);

CREATE INDEX idx_gnode_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gnode_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gnode_ref ON graph_node(ref_id);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties);
CREATE INDEX idx_gnode_path ON graph_node USING GIST (ownership_path);
CREATE INDEX idx_gnode_cluster ON graph_node(tenant_id, cluster_id) WHERE cluster_id IS NOT NULL;

-- ============================================================
-- GRAPH EDGES
-- ============================================================

-- Graph edges represent relationships between nodes.
-- Each edge has a type, source/target, direction, temporal bounds, and properties.

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    
    -- Edge type (relationship label in graph terminology)
    edge_type       VARCHAR(30) NOT NULL,
    -- Ownership edges (GLEIF RR-CDF aligned):
    --   'OWNS'                         → direct shareholding
    --   'CONTROLS'                     → control without majority ownership
    --   'CONSOLIDATES'                 → IS_DIRECTLY_CONSOLIDATED_BY
    --   'ULTIMATELY_CONSOLIDATES'      → IS_ULTIMATELY_CONSOLIDATED_BY
    --   'IS_BRANCH_OF'                 → IS_INTERNATIONAL_BRANCH_OF
    --   'IS_FUND_MANAGED_BY'           → fund management relationship
    --
    -- Officer edges (ISO 5009 aligned):
    --   'SERVES_AS'                    → officer appointment (Director, Secretary, etc.)
    --
    -- Beneficial ownership edges (BODS aligned):
    --   'BENEFITS_FROM'                → beneficial ownership interest
    --
    -- Governance edges:
    --   'REPORTS_TO'                   → committee reports to board
    --   'APPOINTED_BY'                 → resolution appointed this officer
    --   'AUTHORIZED_BY'               → transaction authorized by resolution
    --
    -- Document edges:
    --   'EVIDENCED_BY'                → relationship evidenced by document
    
    -- Direction: source → target
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    
    -- Temporal bounds (GLEIF model)
    valid_from      DATE NOT NULL,
    valid_to        DATE,                                   -- NULL = current
    
    -- Edge properties (JSONB)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- For OWNS edges:
    -- {
    --   "ownership_percentage": 75.5,
    --   "voting_percentage": 80.0,
    --   "accounting_standard": "IFRS",
    --   "control_mechanism": "voting_rights",
    --   "share_class": "Class A Common",
    --   "shares_held": 7550000,
    --   "source": "annual_return"
    -- }
    -- For SERVES_AS edges (ISO 5009):
    -- {
    --   "role_oor_code": "GBLTDI",
    --   "role_name": "Director",
    --   "appointed_on": "2020-03-15",
    --   "is_executive": false,
    --   "is_independent": true,
    --   "committees": ["AUDIT", "REMUNERATION"],
    --   "resolution_id": "uuid"
    -- }
    -- For BENEFITS_FROM edges (BODS):
    -- {
    --   "interest_type": "shareholding",
    --   "interest_level": "direct",
    --   "share_percentage_min": 75,
    --   "share_percentage_max": 100,
    --   "psc_nature_of_control": ["ownership-of-shares-75-to-100-percent"],
    --   "boi_report_filed": true,
    --   "boi_filing_date": "2026-02-01"
    -- }
    
    -- Reference to operational table (e.g. officer_appointment, entity_relationship)
    ref_table       VARCHAR(50),
    ref_id          UUID,
    
    -- Edge weight (for graph algorithms)
    weight          DECIMAL(10,4) NOT NULL DEFAULT 1.0,
    
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    CONSTRAINT no_self_loop CHECK (source_node_id != target_node_id)
);

CREATE INDEX idx_gedge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_gedge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_source_type ON graph_edge(tenant_id, source_node_id, edge_type);
CREATE INDEX idx_gedge_target_type ON graph_edge(tenant_id, target_node_id, edge_type);
CREATE INDEX idx_gedge_active ON graph_edge(tenant_id, edge_type, is_active) WHERE is_active = TRUE;
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties);
CREATE INDEX idx_gedge_temporal ON graph_edge(valid_from, valid_to);
```

---

## Operational Tables (Relational — Standard CRUD)

```sql
-- ============================================================
-- OPERATIONAL TABLES (Relational core for CRUD operations)
-- ============================================================

-- These tables handle day-to-day operations. The graph layer
-- is synced from these via CDC (Change Data Capture).

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

CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- Legal Entity (operational record — synced to graph_node)
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
    fiscal_year_end_month INTEGER,
    fiscal_year_end_day   INTEGER,
    
    -- Graph node reference (denormalised for convenience)
    graph_node_id       UUID REFERENCES graph_node(id),
    
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_jurisdiction ON legal_entity(tenant_id, jurisdiction_code);

-- Person (operational record — synced to graph_node)
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
    
    -- Graph node reference
    graph_node_id       UUID REFERENCES graph_node(id),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_tenant ON person(tenant_id);
CREATE INDEX idx_person_name ON person(tenant_id, family_name, given_name);

-- Entity relationships (operational — synced to graph_edge)
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
    
    -- Graph edge reference
    graph_edge_id       UUID REFERENCES graph_edge(id),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_self_ref CHECK (child_entity_id != parent_entity_id)
);

-- Officer appointments (operational — synced to graph_edge SERVES_AS)
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
    resolution_id       UUID REFERENCES resolution(id),
    appointment_data    JSONB NOT NULL DEFAULT '{}',
    
    -- Graph edge reference
    graph_edge_id       UUID REFERENCES graph_edge(id),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appt_entity ON officer_appointment(tenant_id, entity_id);
CREATE INDEX idx_appt_person ON officer_appointment(tenant_id, person_id);
CREATE INDEX idx_appt_active ON officer_appointment(tenant_id, entity_id) WHERE status = 'ACTIVE';

-- Beneficial ownership (operational — synced to graph_edge BENEFITS_FROM)
CREATE TABLE beneficial_ownership (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    entity_id           UUID NOT NULL REFERENCES legal_entity(id),
    person_id           UUID REFERENCES person(id),
    interested_entity_id UUID REFERENCES legal_entity(id),
    interest_type       VARCHAR(50) NOT NULL,
    interest_level      VARCHAR(20),
    share_percentage    DECIMAL(5,2),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    bo_data             JSONB NOT NULL DEFAULT '{}',
    
    -- Graph edge reference
    graph_edge_id       UUID REFERENCES graph_edge(id),
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Meetings, resolutions, documents, compliance — same as Model 3 (hybrid)
-- These are relational-only; not projected into the graph.

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
    UNIQUE(committee_id, person_id, appointed_on)
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(meeting_id, person_id)
);

CREATE TABLE action_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    meeting_id      UUID REFERENCES meeting(id),
    agenda_item_id  UUID REFERENCES agenda_item(id),
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES person(id),
    due_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    ai_data         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(resolution_id, person_id)
);

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
    expires_at      TIMESTAMPTZ
);

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

-- Audit log
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
    status          VARCHAR(10) NOT NULL DEFAULT 'SUCCESS',
    severity        VARCHAR(10) NOT NULL DEFAULT 'INFO',
    event_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, event_time DESC);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
```

---

## Graph Queries (The Power of This Model)

```sql
-- ============================================================
-- GRAPH QUERIES: Ownership Chain Traversal
-- ============================================================

-- 1. Full ownership tree from ultimate parent (using ltree)
-- This is dramatically faster than recursive CTE for deep hierarchies
SELECT gn.display_name, gn.node_type,
       nlevel(gn.ownership_path) - 1 AS depth,
       gn.properties->>'jurisdiction_code' AS jurisdiction,
       gn.properties->>'entity_status' AS status
FROM graph_node gn
WHERE gn.tenant_id = :tenant_id
  AND gn.ownership_path <@ (
      SELECT ownership_path FROM graph_node
      WHERE id = :ultimate_parent_node_id
  )
  AND gn.is_active = TRUE
ORDER BY gn.ownership_path;

-- 2. All ancestors of an entity (path to ultimate parent)
SELECT gn.display_name,
       nlevel(gn.ownership_path) AS level,
       gn.properties->>'jurisdiction_code' AS jurisdiction
FROM graph_node gn
WHERE gn.tenant_id = :tenant_id
  AND (
      SELECT ownership_path FROM graph_node WHERE id = :child_node_id
  ) <@ gn.ownership_path
ORDER BY nlevel(gn.ownership_path);

-- 3. Ownership chain with percentages (recursive CTE + graph_edge)
WITH RECURSIVE ownership_chain AS (
    -- Start from a subsidiary
    SELECT
        ge.source_node_id,
        ge.target_node_id,
        gn_source.display_name AS child_name,
        gn_target.display_name AS parent_name,
        (ge.properties->>'ownership_percentage')::DECIMAL AS direct_ownership,
        (ge.properties->>'ownership_percentage')::DECIMAL AS effective_ownership,
        1 AS depth,
        ARRAY[ge.source_node_id] AS path
    FROM graph_edge ge
    JOIN graph_node gn_source ON gn_source.id = ge.source_node_id
    JOIN graph_node gn_target ON gn_target.id = ge.target_node_id
    WHERE ge.tenant_id = :tenant_id
      AND ge.source_node_id = :subsidiary_node_id
      AND ge.edge_type = 'OWNS'
      AND ge.is_active = TRUE
    
    UNION ALL
    
    SELECT
        ge.source_node_id,
        ge.target_node_id,
        gn_source.display_name,
        gn_target.display_name,
        (ge.properties->>'ownership_percentage')::DECIMAL,
        oc.effective_ownership * (ge.properties->>'ownership_percentage')::DECIMAL / 100,
        oc.depth + 1,
        oc.path || ge.source_node_id
    FROM graph_edge ge
    JOIN graph_node gn_source ON gn_source.id = ge.source_node_id
    JOIN graph_node gn_target ON gn_target.id = ge.target_node_id
    JOIN ownership_chain oc ON oc.target_node_id = ge.source_node_id
    WHERE ge.tenant_id = :tenant_id
      AND ge.edge_type = 'OWNS'
      AND ge.is_active = TRUE
      AND NOT ge.source_node_id = ANY(oc.path)  -- Prevent cycles
      AND oc.depth < 20                          -- Safety limit
)
SELECT * FROM ownership_chain ORDER BY depth;

-- ============================================================
-- GRAPH QUERIES: Director Network Analysis
-- ============================================================

-- 4. Directors who serve on multiple boards (overlap analysis)
SELECT
    gn_person.display_name AS director_name,
    COUNT(DISTINCT ge.target_node_id) AS boards_served,
    ARRAY_AGG(DISTINCT gn_entity.display_name) AS entity_names,
    ARRAY_AGG(DISTINCT gn_entity.properties->>'jurisdiction_code') AS jurisdictions
FROM graph_edge ge
JOIN graph_node gn_person ON gn_person.id = ge.source_node_id
JOIN graph_node gn_entity ON gn_entity.id = ge.target_node_id
WHERE ge.tenant_id = :tenant_id
  AND ge.edge_type = 'SERVES_AS'
  AND ge.is_active = TRUE
  AND gn_person.node_type = 'PERSON'
  AND gn_entity.node_type = 'ENTITY'
GROUP BY gn_person.id, gn_person.display_name
HAVING COUNT(DISTINCT ge.target_node_id) > 1
ORDER BY boards_served DESC;

-- 5. Conflict of interest detection: find directors who serve on
--    boards of entities that have a business relationship
SELECT
    gn_person.display_name AS director,
    gn_entity1.display_name AS entity_1,
    gn_entity2.display_name AS entity_2,
    ge_rel.edge_type AS relationship_type,
    ge_rel.properties->>'ownership_percentage' AS ownership_pct
FROM graph_edge ge1
JOIN graph_edge ge2 ON ge1.source_node_id = ge2.source_node_id  -- Same person
    AND ge1.target_node_id != ge2.target_node_id                 -- Different entities
JOIN graph_edge ge_rel ON (
    (ge_rel.source_node_id = ge1.target_node_id AND ge_rel.target_node_id = ge2.target_node_id)
    OR
    (ge_rel.source_node_id = ge2.target_node_id AND ge_rel.target_node_id = ge1.target_node_id)
)
JOIN graph_node gn_person ON gn_person.id = ge1.source_node_id
JOIN graph_node gn_entity1 ON gn_entity1.id = ge1.target_node_id
JOIN graph_node gn_entity2 ON gn_entity2.id = ge2.target_node_id
WHERE ge1.tenant_id = :tenant_id
  AND ge1.edge_type = 'SERVES_AS' AND ge1.is_active = TRUE
  AND ge2.edge_type = 'SERVES_AS' AND ge2.is_active = TRUE
  AND ge_rel.edge_type IN ('OWNS', 'CONTROLS', 'CONSOLIDATES')
  AND ge_rel.is_active = TRUE;

-- 6. Board independence analysis
SELECT
    gn_entity.display_name AS entity_name,
    COUNT(*) AS total_directors,
    COUNT(*) FILTER (WHERE (ge.properties->>'is_independent')::BOOLEAN = TRUE) AS independent_directors,
    COUNT(*) FILTER (WHERE (ge.properties->>'is_executive')::BOOLEAN = TRUE) AS executive_directors,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE (ge.properties->>'is_independent')::BOOLEAN = TRUE) / COUNT(*),
        1
    ) AS independence_percentage
FROM graph_edge ge
JOIN graph_node gn_entity ON gn_entity.id = ge.target_node_id
WHERE ge.tenant_id = :tenant_id
  AND ge.edge_type = 'SERVES_AS'
  AND ge.is_active = TRUE
  AND ge.properties->>'role_name' = 'Director'
GROUP BY gn_entity.id, gn_entity.display_name;

-- ============================================================
-- GRAPH QUERIES: Beneficial Ownership Tracing
-- ============================================================

-- 7. Trace ultimate beneficial owners through intermediate entities
WITH RECURSIVE bo_chain AS (
    SELECT
        ge.source_node_id,
        ge.target_node_id,
        gn_source.display_name AS beneficiary_name,
        gn_source.node_type AS beneficiary_type,
        gn_target.display_name AS subject_name,
        (ge.properties->>'share_percentage_min')::DECIMAL AS ownership_min,
        (ge.properties->>'share_percentage_max')::DECIMAL AS ownership_max,
        1 AS depth,
        ARRAY[ge.source_node_id] AS path
    FROM graph_edge ge
    JOIN graph_node gn_source ON gn_source.id = ge.source_node_id
    JOIN graph_node gn_target ON gn_target.id = ge.target_node_id
    WHERE ge.tenant_id = :tenant_id
      AND ge.target_node_id = :target_entity_node_id
      AND ge.edge_type IN ('BENEFITS_FROM', 'OWNS', 'CONTROLS')
      AND ge.is_active = TRUE
    
    UNION ALL
    
    SELECT
        ge.source_node_id,
        ge.target_node_id,
        gn_source.display_name,
        gn_source.node_type,
        gn_target.display_name,
        (ge.properties->>'share_percentage_min')::DECIMAL,
        (ge.properties->>'share_percentage_max')::DECIMAL,
        bc.depth + 1,
        bc.path || ge.source_node_id
    FROM graph_edge ge
    JOIN graph_node gn_source ON gn_source.id = ge.source_node_id
    JOIN graph_node gn_target ON gn_target.id = ge.target_node_id
    JOIN bo_chain bc ON bc.source_node_id = ge.target_node_id
    WHERE ge.tenant_id = :tenant_id
      AND ge.edge_type IN ('BENEFITS_FROM', 'OWNS', 'CONTROLS')
      AND ge.is_active = TRUE
      AND NOT ge.source_node_id = ANY(bc.path)
      AND bc.depth < 10
)
SELECT * FROM bo_chain
WHERE beneficiary_type = 'PERSON'  -- Only natural persons as UBOs
ORDER BY depth;

-- 8. Entities in a jurisdiction with no identified beneficial owner
SELECT gn.display_name, gn.properties->>'jurisdiction_code' AS jurisdiction,
       gn.properties->>'registration_number' AS reg_number
FROM graph_node gn
WHERE gn.tenant_id = :tenant_id
  AND gn.node_type = 'ENTITY'
  AND gn.is_active = TRUE
  AND gn.properties->>'jurisdiction_code' = 'GB'
  AND NOT EXISTS (
      SELECT 1 FROM graph_edge ge
      WHERE ge.target_node_id = gn.id
        AND ge.edge_type = 'BENEFITS_FROM'
        AND ge.is_active = TRUE
  );

-- ============================================================
-- GRAPH QUERIES: AI / Analytics
-- ============================================================

-- 9. Governance risk scoring inputs (for AI model)
SELECT
    gn.id AS entity_node_id,
    gn.display_name,
    gn.properties->>'jurisdiction_code' AS jurisdiction,
    
    -- Board composition metrics
    (SELECT COUNT(*) FROM graph_edge ge 
     WHERE ge.target_node_id = gn.id AND ge.edge_type = 'SERVES_AS' 
     AND ge.is_active = TRUE) AS total_board_members,
    
    (SELECT COUNT(*) FROM graph_edge ge 
     WHERE ge.target_node_id = gn.id AND ge.edge_type = 'SERVES_AS' 
     AND ge.is_active = TRUE 
     AND (ge.properties->>'is_independent')::BOOLEAN = TRUE) AS independent_members,
    
    -- Ownership complexity
    (SELECT COUNT(*) FROM graph_edge ge 
     WHERE ge.target_node_id = gn.id AND ge.edge_type = 'OWNS' 
     AND ge.is_active = TRUE) AS direct_owners,
    
    -- Beneficial ownership completeness
    (SELECT COUNT(*) FROM graph_edge ge 
     WHERE ge.target_node_id = gn.id AND ge.edge_type = 'BENEFITS_FROM' 
     AND ge.is_active = TRUE) AS identified_ubos,
    
    -- Network centrality (pre-computed)
    gn.centrality_score,
    gn.cluster_id,
    
    -- Subsidiary count
    (SELECT COUNT(*) FROM graph_edge ge 
     WHERE ge.source_node_id = gn.id AND ge.edge_type IN ('OWNS', 'CONSOLIDATES') 
     AND ge.is_active = TRUE) AS subsidiary_count
    
FROM graph_node gn
WHERE gn.tenant_id = :tenant_id
  AND gn.node_type = 'ENTITY'
  AND gn.is_active = TRUE;

-- 10. Find entities within N hops of a given entity
--     (for due diligence / related party identification)
WITH RECURSIVE related_entities AS (
    SELECT
        gn.id AS node_id,
        gn.display_name,
        gn.node_type,
        0 AS hops,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.id = :start_node_id
    
    UNION ALL
    
    SELECT
        CASE WHEN ge.source_node_id = re.node_id THEN ge.target_node_id
             ELSE ge.source_node_id END AS node_id,
        gn.display_name,
        gn.node_type,
        re.hops + 1,
        re.path || CASE WHEN ge.source_node_id = re.node_id THEN ge.target_node_id
                        ELSE ge.source_node_id END
    FROM graph_edge ge
    JOIN related_entities re ON (ge.source_node_id = re.node_id OR ge.target_node_id = re.node_id)
    JOIN graph_node gn ON gn.id = CASE WHEN ge.source_node_id = re.node_id THEN ge.target_node_id
                                       ELSE ge.source_node_id END
    WHERE ge.tenant_id = :tenant_id
      AND ge.is_active = TRUE
      AND re.hops < :max_hops  -- Typically 2-3
      AND NOT (CASE WHEN ge.source_node_id = re.node_id THEN ge.target_node_id
                    ELSE ge.source_node_id END) = ANY(re.path)
)
SELECT DISTINCT node_id, display_name, node_type, MIN(hops) AS shortest_path
FROM related_entities
WHERE node_id != :start_node_id
GROUP BY node_id, display_name, node_type
ORDER BY shortest_path, display_name;
```

---

## CDC Sync Process

```sql
-- ============================================================
-- CHANGE DATA CAPTURE: Sync operational tables → graph layer
-- ============================================================

-- Trigger function: when a legal_entity is inserted or updated,
-- sync the corresponding graph_node

CREATE OR REPLACE FUNCTION sync_entity_to_graph() RETURNS TRIGGER AS $$
DECLARE
    v_node_id UUID;
BEGIN
    -- Check if node exists
    SELECT id INTO v_node_id FROM graph_node
    WHERE tenant_id = NEW.tenant_id AND node_type = 'ENTITY' AND ref_id = NEW.id;
    
    IF v_node_id IS NULL THEN
        -- Create new node
        INSERT INTO graph_node (tenant_id, node_type, ref_id, display_name, properties)
        VALUES (
            NEW.tenant_id,
            'ENTITY',
            NEW.id,
            NEW.legal_name,
            jsonb_build_object(
                'legal_name', NEW.legal_name,
                'lei', NEW.lei,
                'jurisdiction_code', NEW.jurisdiction_code,
                'legal_form_code', NEW.legal_form_code,
                'entity_status', NEW.entity_status,
                'registration_number', NEW.registration_number,
                'incorporation_date', NEW.registration_date
            )
        )
        RETURNING id INTO v_node_id;
        
        -- Update back-reference
        UPDATE legal_entity SET graph_node_id = v_node_id WHERE id = NEW.id;
    ELSE
        -- Update existing node
        UPDATE graph_node SET
            display_name = NEW.legal_name,
            properties = jsonb_build_object(
                'legal_name', NEW.legal_name,
                'lei', NEW.lei,
                'jurisdiction_code', NEW.jurisdiction_code,
                'legal_form_code', NEW.legal_form_code,
                'entity_status', NEW.entity_status,
                'registration_number', NEW.registration_number,
                'incorporation_date', NEW.registration_date
            ),
            is_active = (NEW.entity_status = 'ACTIVE'),
            updated_at = now()
        WHERE id = v_node_id;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_entity_to_graph
    AFTER INSERT OR UPDATE ON legal_entity
    FOR EACH ROW EXECUTE FUNCTION sync_entity_to_graph();

-- Similar triggers for person, entity_relationship, officer_appointment, beneficial_ownership
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Multi-Tenant Foundation | 2 | tenant, app_user |
| Entity Management | 3 | legal_entity, entity_relationship, beneficial_ownership |
| Person & Officers | 2 | person, officer_appointment |
| Meeting Management | 5 | committee, committee_membership, meeting, agenda_item, meeting_attendance |
| Action Items | 1 | action_item |
| Resolutions & Voting | 2 | resolution, resolution_vote |
| E-Signature | 1 | signing_envelope |
| Document Management | 2 | document, document_permission |
| Compliance | 1 | compliance_obligation |
| Share Register | 2 | share_class, share_transaction |
| Audit Log | 1 | audit_log (partitioned) |
| **Total** | **24** | (+ 2 graph tables = 26 effective) |

---

## Key Design Decisions

1. **Graph layer as a projection, not the source of truth** — the operational relational tables remain the source of truth for CRUD operations. The graph layer (graph_node + graph_edge) is populated via CDC triggers and can be rebuilt at any time. This avoids the complexity of making graph queries transactional.

2. **PostgreSQL ltree for ownership hierarchies** — the `ownership_path` column on graph_node uses PostgreSQL's ltree extension for fast ancestor/descendant queries without recursive CTEs. The path is maintained by a trigger when entity_relationship records change.

3. **Typed edge labels aligned with international standards** — edge types directly map to GLEIF RR-CDF relationship types (OWNS, CONSOLIDATES, IS_BRANCH_OF), BODS interest types (BENEFITS_FROM), and ISO 5009 officer roles (SERVES_AS). This ensures the graph can be exported in formats compatible with regulatory systems.

4. **Temporal edges** — every graph_edge has valid_from/valid_to dates, enabling time-travel graph queries ("what was the ownership structure on 1 January 2025?"). This is critical for regulatory compliance where historical ownership must be provable.

5. **Pre-computed centrality and clustering** — graph_node includes `centrality_score` and `cluster_id` fields that are updated by batch graph algorithms (betweenness centrality, Louvain community detection). These power AI governance risk scoring without requiring real-time graph computation.

6. **JSONB properties on nodes and edges** — graph properties are stored in JSONB for flexibility. Different edge types carry different properties (ownership_percentage for OWNS, role_name for SERVES_AS, share_percentage for BENEFITS_FROM) without requiring separate edge tables.

7. **Optional Neo4j/Neptune upgrade path** — the graph_node/graph_edge schema is designed to be exportable to a dedicated graph database. For very large corporate structures (10,000+ entities), a Neo4j or Amazon Neptune deployment would provide Cypher/Gremlin query capabilities that outperform PostgreSQL recursive CTEs for deep traversals.

8. **Director network conflict detection** — the graph model makes conflict-of-interest detection a natural query: "find persons who SERVES_AS at two entities that also have an OWNS or CONTROLS edge between them." This query is intractable in a pure relational model without graph joins.

9. **Beneficial ownership chain resolution** — tracing ultimate beneficial owners through intermediate holding companies is a recursive graph traversal that the graph layer handles efficiently. The BODS-aligned edge properties carry the interest type and percentage data needed for regulatory reporting.

10. **Graph analytics for AI governance risk scoring** — the graph structure is a rich input for AI models: board composition diversity, ownership complexity, network centrality of key persons, and cluster analysis of entity groups all provide features that pure relational queries cannot efficiently compute.
