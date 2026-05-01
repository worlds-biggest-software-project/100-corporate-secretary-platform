# Corporate Secretary Platform — Feature & Functionality Survey

> Candidate #100 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Diligent Boards (Diligent One) | Enterprise board portal + GRC suite | Proprietary SaaS | https://www.diligent.com |
| Nasdaq Boardvantage | Board portal + meeting management | Proprietary SaaS | https://www.nasdaq.com/solutions/governance/boardvantage |
| BoardEffect | Board portal (mid-market / nonprofits) | Proprietary SaaS | https://www.boardeffect.com |
| Azeus Convene | Board portal (regulated industries) | Proprietary SaaS | https://www.azeusconvene.com |
| Govenda (now part of OnBoard) | Board portal (SME / mid-market) | Proprietary SaaS | https://www.govenda.com |
| OnBoard | Board portal + meeting builder | Proprietary SaaS | https://www.onboardmeetings.com |
| BoardPro | Board management (SME / nonprofits) | Proprietary SaaS | https://www.boardpro.com |
| Diligent Entities (formerly Blueprint OneWorld) | Entity / subsidiary management | Proprietary SaaS | https://www.blueprintoneworld.com |
| Computershare Entity Management (GEMS / CT Corp) | Entity management database | Proprietary SaaS / Hosted | https://www.computershare.com |
| Legalinc | Registered agent + entity compliance automation | Proprietary SaaS | https://www.legalinc.com |

## Feature Analysis by Solution

### Diligent Boards (Diligent One)

**Core features**
- Board book assembly: drag-and-drop agenda builder, automatic pagination, and PDF/HTML board pack generation
- Secure document repository with granular permission levels per committee and director
- E-signatures and written resolutions workflow
- Meeting calendar, RSVPs, and attendance tracking
- Director questionnaires and conflict-of-interest declarations
- Mobile apps (iOS, Android) with offline access and annotation

**Differentiating features**
- Smart Minutes: AI-generated first-draft minutes from typed notes, transcripts, or board materials; maps decisions and action items to agenda items
- AI Board Member: secure AI assistant for directors, launched at Elevate 2026; provides contextual briefings on board materials
- Subsidiary Governance Agent: agentic AI that prepares board packs, minutes, and filings across dozens of subsidiary entities automatically
- GRC module integration: audit, risk, ethics case management, and ESG rolled into a single "Diligent One" platform
- 75% Fortune 500 market penetration; 31.5% mindshare among board-portal buyers

**UX patterns**
- Presenter mode for meeting facilitation on screen
- Annotation and private note-taking within board books
- Action item tracker visible to individual directors and the company secretary
- Templated agenda structures editable by administrators

**Integration points**
- Diligent Entities (entity management) connected for unified governance data
- SSO via SAML 2.0 and Azure AD
- eSignature providers (DocuSign, internal workflow)
- CSC registered-agent data feed into Diligent Entities

**Known gaps**
- Very high cost; smaller organizations and nonprofits frequently excluded
- Deep feature set produces steep learning curve for occasional users
- Entity management and board portal sold as separate modules, requiring cross-module configuration
- Agentic AI features (AI Board Member, Subsidiary Agent) still in limited / fall 2026 GA as of research date

**Licence / IP notes**
- Fully proprietary SaaS; no open-source components exposed
- Data residency options available (US, EU, AU)
- Responsible AI commitments published; no customer data used to train external models

---

### Nasdaq Boardvantage

**Core features**
- Structured meeting workflow: agenda creation, board book compilation, and distribution
- Secure document management with version control and audit trail
- Meeting minutes automation: AI-assisted generation, 91–97% reported accuracy in internal testing
- E-voting and resolutions with audit-ready records
- Mobile app (iOS, Android) with offline mode
- Multi-language support (17+ languages including Arabic, Chinese, Japanese, Thai)

**Differentiating features**
- AI-powered document summarization condensing lengthy board materials to actionable briefs
- Built on Microsoft Azure / Azure OpenAI (Foundry Models); no customer data shared with model training
- 25% reported reduction in board preparation time; up to 10–30 hours monthly manual work saved
- Used by 4,400+ organizations globally; particularly strong in public-company and financial-services segments

**UX patterns**
- Single-workflow meeting lifecycle from invite to signed minutes
- Drag-and-drop board book assembly
- In-app notifications and deadline reminders

**Integration points**
- Azure AD / SSO
- Microsoft 365 (document authoring pipeline)
- Nasdaq governance analytics and reporting dashboards

**Known gaps**
- Entity management not included; separate tooling required
- GRC (risk, audit, ESG) not native to Boardvantage; requires Nasdaq partner ecosystem
- Fewer advanced workflow automation features than Diligent at the enterprise tier
- AI features reported as solid but less agentic than Diligent's 2026 roadmap

**Licence / IP notes**
- Proprietary SaaS; owned by Nasdaq, Inc.
- Azure-hosted; meets ISO 27001, SOC 2 Type II

---

### BoardEffect

**Core features**
- Board book creation and secure distribution
- Committee management and membership tracking
- Meeting agenda and minutes workflows
- Voting and approval workflows
- Document library with version control
- Orientation resources for new board members

**Differentiating features**
- Purpose-built for nonprofits, healthcare organizations, credit unions, and associations
- Board effectiveness surveys and self-assessment tools
- Governance library and policy management
- 16.1% mindshare; strong in nonprofit and association segments

**UX patterns**
- Simplified admin interface designed for non-technical company secretaries
- Onboarding checklists and training materials for new directors

**Integration points**
- SSO and LDAP
- Limited native ERP/CRM integrations; primarily standalone

**Known gaps**
- AI features limited compared to Diligent and Boardvantage
- Entity management not included
- Less suitable for multi-entity or public-company use cases
- Weaker analytics and reporting depth

**Licence / IP notes**
- Proprietary SaaS; BoardEffect is a Diligent subsidiary (acquired 2019)
- SOC 2 Type II certified

---

### Azeus Convene

**Core features**
- Secure board book distribution with encrypted PDF delivery
- Meeting agenda builder and minute-taking tools
- Resolution and approval workflows
- E-signature support
- iOS, Android, Windows, and Mac apps with offline access
- Whiteboard and annotation tools for collaborative review

**Differentiating features**
- Strong compliance posture for regulated industries (banking, government, listed companies)
- Deployed in 100+ countries; multi-jurisdiction statutory compliance focus
- Award-winning security architecture; ISO 27001 certified
- Granular access controls per document, per user, per meeting

**UX patterns**
- Director-focused mobile-first interface
- Secure messaging and chat within the platform
- Presenter mode and live polling during meetings

**Integration points**
- Active Directory / SSO
- Limited third-party integrations; security-first, closed architecture

**Known gaps**
- Limited AI / automation features as of 2026
- Entity management not included
- Primarily meeting-centric; weaker on broader governance workflows (ESG, risk, audit)
- Higher friction for US-style XBRL/SEC filing needs

**Licence / IP notes**
- Proprietary SaaS; Azeus Systems Limited (Hong Kong-headquartered)
- ISO 27001, CSA STAR certified

---

### Govenda (acquired by OnBoard, 2024)

**Core features**
- Agenda creation, board book assembly, and secure distribution
- E-voting and approval workflows
- iOS / Android mobile apps
- Annotation and private notes on board materials
- Audit trail and access logs

**Differentiating features**
- Positioned as a mid-market alternative to Diligent; strong G2 and Capterra ratings
- Post-acquisition by OnBoard (2024), accelerating investment in AI analytics and integrated governance
- ISO 27001 and SOC 2 certified
- Pricing more accessible than Diligent for organizations with 5–25 board seats

**UX patterns**
- Administrator dashboard for meeting prep and director management
- Director self-service portal for RSVPs and document access

**Integration points**
- SSO / Active Directory
- Integration roadmap expanding under OnBoard ownership

**Known gaps**
- Post-acquisition, some feature-set overlap with OnBoard causing product consolidation uncertainty
- Entity management not native
- AI features still developing (roadmap inherited from OnBoard)

**Licence / IP notes**
- Proprietary SaaS; acquired by OnBoard (Passageways) in 2024

---

### OnBoard

**Core features**
- Meeting builder with agenda templates and board book generation
- Voting, resolutions, and e-signature workflows
- Minutes builder with action item tracking
- Director portal with document access and annotation
- Assessments: board effectiveness surveys
- Reporting and analytics dashboards

**Differentiating features**
- Consistently top-rated on G2, Capterra, and SoftwareReviews for user satisfaction
- Govenda acquisition (2024) expanding portfolio and geographic reach
- AI-assisted meeting minutes drafting (announced 2025)
- Meeting analytics showing participation and engagement data

**UX patterns**
- Clean, modern UI with low learning curve reported by administrators
- Roles-based interface (administrator vs. director views differ significantly)

**Integration points**
- Zoom and Microsoft Teams integration for hybrid meeting support
- SSO (Okta, Azure AD)
- Zapier connectors for lightweight workflow automation

**Known gaps**
- Entity and subsidiary management not available
- GRC/audit/ESG modules not native
- Less deeply embedded in large public-company segment than Diligent or Boardvantage
- AI features launched but not at parity with Diligent's agentic capabilities

**Licence / IP notes**
- Proprietary SaaS; OnBoard by Passageways
- SOC 2 Type II, ISO 27001

---

### BoardPro

**Core features**
- Meeting pack assembly and distribution
- Agenda builder with time allocation
- Digital minutes with action item tracking
- Entity registers: director register, shareholder register, interests register
- Task management and follow-up tracking

**Differentiating features**
- One of the few platforms with native statutory entity registers at the SME tier
- Pricing accessible for small boards and nonprofits (from ~$350/month)
- Strong traction in Australia, New Zealand, and UK markets
- Simple enough for boards where the administrator is not an IT professional

**UX patterns**
- Wizard-driven meeting setup for non-technical administrators
- Action items automatically extracted from minutes
- Director summary emails after each meeting

**Integration points**
- Limited; primarily standalone
- Xero integration for New Zealand/Australia market

**Known gaps**
- Limited AI features
- No GRC, ESG, or audit modules
- Not suitable for large public companies or complex multi-entity structures
- Reporting depth limited compared to enterprise platforms

**Licence / IP notes**
- Proprietary SaaS; New Zealand-headquartered startup

---

### Diligent Entities (formerly Blueprint OneWorld)

**Core features**
- Centralized corporate record for all entities and subsidiaries
- Officer and director register with appointment and resignation tracking
- Automated compliance reminders by jurisdiction (annual filings, registered agent renewals)
- Dynamic corporate structure charts and org-chart visualization
- Direct electronic filings from within the platform
- Document management for constitutional documents, resolutions, and share certificates

**Differentiating features**
- AI-driven document summarization and data extraction from constitutional documents
- 318% ROI, 90% cost reduction, 70% time savings on entity data tasks (Diligent case study)
- CSC registered-agent integration for automatic compliance data feeds
- Subsidiary Governance Agent (agentic AI) manages subsidiary board operations at scale
- Multi-jurisdiction compliance rule library across 100+ countries

**UX patterns**
- Legal-team-first design; structured data entry for corporate facts
- Compliance dashboard surfacing overdue and upcoming filings
- Export to Excel / PDF for regulatory reporting

**Integration points**
- Diligent Boards (meeting management) for unified governance workflow
- CSC registered agent services (automated compliance data)
- Workiva / XBRL tools for regulatory reporting

**Known gaps**
- Pricing at enterprise tier; not accessible for single-entity companies
- Significant implementation effort for large entity portfolios
- Some jurisdiction coverage gaps for emerging markets

**Licence / IP notes**
- Proprietary SaaS; Diligent subsidiary
- Acquired Blueprint OneWorld in 2017

---

### Computershare Entity Management (GEMS / CT Corporation – Wolters Kluwer)

**Core features**
- Entity database with officer, director, and ownership records
- Compliance calendar and filing deadline management
- Registered agent integration
- Share ledger and capitalization table management
- Document vault for corporate records

**Differentiating features**
- Long-established; deeply embedded in US legal-department workflows at Fortune 500 companies
- CT Corporation registered-agent network integrated for automatic compliance data updates
- Breadth of US jurisdiction coverage

**UX patterns**
- Legacy enterprise application; powerful but not modern UX
- Heavy reliance on client services team for complex tasks

**Integration points**
- Registered agent data feeds
- Export to Excel; some API access for enterprise clients

**Known gaps**
- Aging UX; dated compared to Diligent Entities
- Limited AI or automation features
- Board portal not included; entity management only
- Minimal innovation investment in recent years

**Licence / IP notes**
- Proprietary hosted / SaaS; Wolters Kluwer / CT Corporation brand
- No open-source components

---

### Legalinc

**Core features**
- Registered agent services in all 50 US states
- Automated state filing compliance tracking
- Entity formation and dissolution workflows
- Annual report and franchise tax filing automation
- Officer / director record management

**Differentiating features**
- Automation-first approach: high-volume compliance task automation for law firms and legal tech platforms
- API-first architecture enabling legal software vendors to embed entity management
- Flat-fee transparent pricing model (significant savings vs. legacy registered agent providers)
- Designed for legal tech and law firm partnership integrations

**UX patterns**
- Clean self-service dashboard for entity portfolio management
- Email/SMS alerts for upcoming compliance deadlines
- Bulk upload and management for large entity portfolios

**Integration points**
- REST API for embedding in legal software platforms
- Integration with practice management and document automation tools
- Secretary of State data feeds for automated record updates

**Known gaps**
- US-only jurisdiction coverage
- No board portal or meeting management features
- Limited audit trail / governance documentation features
- Not suited for public companies with complex governance requirements

**Licence / IP notes**
- Proprietary SaaS; Legalinc Corporate Services Inc.
- No open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Secure board book assembly and distribution (encrypted, permission-gated)
- Meeting agenda management and minute-taking workflows
- E-voting and written resolution support
- Mobile apps (iOS and Android) with offline access
- Audit trail and access logging for regulatory compliance
- E-signature support (ESIGN / eIDAS compliant)
- Single sign-on (SAML, OAuth, Active Directory)
- Role-based access control (administrator vs. director vs. observer)

### Differentiating Features
- AI-generated meeting minutes with decision and action item mapping (Diligent, Boardvantage, OnBoard)
- Agentic AI for subsidiary governance automation at scale (Diligent — leading edge, 2026)
- AI-powered board material summarization (Diligent, Boardvantage)
- Native entity and subsidiary management integrated with board portal (Diligent One)
- Full GRC suite integration: audit, risk, ESG, ethics (Diligent One)
- API-first registered-agent automation with transparent pricing (Legalinc)
- Native statutory entity registers for SMEs (BoardPro)
- Multi-jurisdiction compliance automation across 100+ countries (Diligent Entities)

### Underserved Areas / Opportunities
- Affordable entity management integrated with a board portal for mid-market companies (gap between BoardPro and Diligent pricing tiers)
- AI-powered regulatory change monitoring across jurisdictions (no vendor has fully addressed this)
- Conversational interface for directors to query board materials in plain language (early stage with Diligent AI Board Member)
- Integrated XBRL/iXBRL generation from board-approved financials for SEC filers
- Conflict-of-interest detection using AI across director networks and company data
- Automated shareholder register and cap table management for private companies

### AI-Augmentation Candidates
- Minutes drafting from transcription (high ROI, several vendors doing it; AI-native version can be far superior)
- Resolution and written consent generation from structured precedents
- Compliance deadline monitoring and cross-jurisdiction regulatory change tracking
- Entity data extraction from constitutional documents (already proven at Diligent: 70% time saving)
- Director onboarding briefing packages generated from historical board materials
- Governance risk scoring from board composition, attendance, and committee activity data

## Legal & IP Summary

- All analysed solutions are **proprietary SaaS**. There are no meaningful open-source corporate secretary or board portal platforms with active communities.
- No GPL or AGPL concerns. No embedding risks from copyleft licences.
- **CPT / AMA analogy**: The underlying code sets relevant to corporate law (ICD equivalents would be statutory forms) are public-domain government documents in most jurisdictions.
- **Key IP risks for a new platform**: (1) Reproducing vendor-specific UI flows closely enough to trigger trade dress claims; (2) ingesting and reproducing copyrighted precedent clause libraries from legal publishers (e.g., Practical Law, Lexis) without licence; (3) automated filing integrations relying on Secretary of State screen-scraping rather than official APIs may violate terms of service.
- Data residency and sovereignty obligations vary by jurisdiction (GDPR, Australian Privacy Act, PDPA) and must be designed in from the start.

## Recommended Feature Scope

**Must-have (MVP)**:
- Secure board book assembly, distribution, and version-controlled document library with granular role-based access
- Meeting lifecycle management: agenda creation, digital minutes with action item extraction, and attendance recording
- E-voting, written resolutions, and e-signature workflow (ESIGN / eIDAS compliant)
- Entity register: officer and director register with appointment tracking and basic compliance deadline calendar
- Mobile-responsive access (iOS, Android) with offline capability and annotation tools
- Audit trail and access logging meeting ISO 15489 and SOC 2 requirements

**Should-have (v1.1)**:
- AI-generated draft minutes from transcript or notes, with decision and action item mapping to agenda items
- Multi-entity / subsidiary portfolio view with jurisdiction-specific compliance calendar
- AI-powered board material summarization delivered to directors before meetings
- Committee management: separate books, memberships, and committee-level permissions
- E-signatures integrated with a qualified provider (DocuSign or equivalent) for board resolutions

**Nice-to-have (backlog)**:
- Agentic AI for subsidiary board operations: automated pack preparation, resolution generation, and filing across entity portfolio
- Governance risk scoring from board composition, attendance, and ESG data
- Regulatory change monitoring across jurisdictions with entity-level impact flagging
- XBRL / iXBRL output for SEC-filing public companies
- Conversational AI director briefing interface (query board materials in natural language)
