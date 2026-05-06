# Standards & API Reference

> Project: Corporate Secretary Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 15489-1:2016 — Records Management**
Defines principles and requirements for managing records in any format or media. Directly applicable to board minutes, resolutions, and governance document retention schedules.

**ISO/IEC 27001:2022 — Information Security Management Systems**
Mandatory benchmark for any platform holding sensitive board materials, director PII, or M&A documents. The 2022 update adds cloud-native and supply chain security controls. SOC 2 Type II certification is commonly required alongside this.

**ISO 19600 / ISO 37301 — Compliance Management Systems**
ISO 37301 (2021) replaces ISO 19600 and provides a framework for establishing, developing, and maintaining an effective compliance management system — core to what a corporate secretary platform must demonstrate.

**ISO 27701 — Privacy Information Management**
Extension to ISO 27001 covering GDPR and CCPA-aligned privacy controls. Required for platforms processing director PII and board correspondence across jurisdictions.

### W3C & IETF Standards

**XBRL 2.1 (eXtensible Business Reporting Language)**
Open standard maintained by XBRL International for structured financial and regulatory reporting. Mandated by the SEC for all public company filings; relevant to any module generating 10-K, 10-Q, or proxy reports.
Reference: https://www.xbrl.org/specifications/

**Inline XBRL (iXBRL)**
SEC-mandated format that embeds XBRL tags within human-readable HTML documents. Required for annual and quarterly reports (10-K, 10-Q) filed via EDGAR.
Reference: https://www.sec.gov/data-research/structured-data/inline-xbrl

**RFC 6749 — OAuth 2.0 Authorization Framework**
Standard token-based authorization used by Computershare GEMS, Diligent APIs, and EDGAR APIs.

**RFC 7519 — JSON Web Tokens (JWT)**
Standard for securely transmitting claims between platform services; used in governance platform SSO flows.

**W3C DID / Verifiable Credentials**
Emerging W3C standards for decentralised identity and document provenance — relevant for next-generation board resolution signing and director identity verification.

### Data Model & API Specifications

**OpenAPI 3.1**
De facto standard for REST API documentation. Diligent's Developer Portal publishes OpenAPI-compatible docs for the Entities Reports API.

**GraphQL**
Diligent Entities uses GraphQL for flexible querying of entity and officer data. Relevant as a schema design consideration for a competing platform.

**SEC EDGAR REST API (data.sec.gov)**
Free RESTful API delivering JSON submissions history, XBRL financial data, and company facts. Enables automated retrieval of public entity filings for subsidiary monitoring and due diligence.
Reference: https://www.sec.gov/search-filings/edgar-application-programming-interfaces

**XBRL US Data Quality Committee Rules**
Industry-maintained validation rules for SEC XBRL filings; a platform generating regulatory output must pass DQC validation before submission.

### Security & Authentication Standards

**eIDAS Regulation (EU No 910/2014)**
EU regulation establishing three tiers of electronic signatures (SES, AdES, QES) for corporate governance use cases: shareholder resolutions, board minutes, cross-border M&A. QES requires QTSP-certified hardware signing; AdES satisfies most corporate governance needs.
Reference: https://www.docusign.com/products/electronic-signature/learn/eidas

**ESIGN Act (USA, 2000) / UETA**
US federal and state legislation validating electronic signatures on corporate consents, resolutions, and board approvals. All major e-signature providers (DocuSign, Adobe Sign) are compliant.

**GDPR (EU) / CCPA (California)**
Data protection regulations governing the storage and processing of director PII, board correspondence, and governance records. Platform must support data subject access requests, retention limits, and cross-border transfer controls.

**DORA — Digital Operational Resilience Act (EU, effective Jan 2026)**
Applies to financial institutions and their IT service providers operating in the EU. Corporate secretary platforms serving regulated financial entities must demonstrate ICT risk management, incident reporting, and third-party oversight capabilities.

**NIST Cybersecurity Framework 2.0**
The 2026 revision adds stronger governance, supply chain risk, and AI risk controls. Increasingly referenced in enterprise procurement requirements for governance software.

**SOC 2 Type II**
Industry-standard audit report (AICPA) for SaaS platforms. Required by enterprise buyers evaluating board portal and entity management vendors.

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant for AI-augmented corporate secretary workflows. An MCP server exposing governance data (entity records, meeting schedules, resolution status) would allow AI assistants to surface information contextually, draft minutes, or prepare filing checklists without requiring full platform integration.
Reference: https://modelcontextprotocol.io/

---

## Similar Products — Developer Documentation & APIs

### Diligent Entities (Entities Reports API)

- **Description:** World's leading entity management platform (formerly Blueprint OneWorld). Manages subsidiary hierarchies, officer registers, compliance calendars, and corporate records across 180+ jurisdictions.
- **API Documentation:** https://developer.diligent.com/api/entities
- **Schema:** GraphQL — supports flexible querying of entity trees, officer data, and compliance events
- **Authentication:** OAuth 2.0 (Bearer token)
- **Standards:** REST/GraphQL, OpenAPI-compatible documentation
- **SDKs/Libraries:** No public SDK; GraphQL clients available in any language

### Computershare GEMS (Global Entity Management System)

- **Description:** Enterprise entity management software used by large law firms and corporate legal teams worldwide. Manages registered agent services, compliance calendars, and officer records.
- **API Documentation:** Available via enterprise agreement (contact vendor)
- **Developer Guide:** Web API over standard HTTP; OAuth 2.0 token-based authorization
- **Authentication:** OAuth 2.0
- **Standards:** REST/JSON, HTTP
- **SDKs/Libraries:** Not publicly documented

### SEC EDGAR API (data.sec.gov)

- **Description:** Free public API from the US Securities and Exchange Commission providing access to company submissions, financial facts, and XBRL-tagged data for all public companies.
- **API Documentation:** https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- **Developer Guide:** https://www.sec.gov/about/developer-resources
- **Standards:** REST/JSON; no authentication required for read access
- **Authentication:** None (public); rate-limited to 10 requests/second
- **SDKs/Libraries:** sec-api.io provides a commercial wrapper SDK

### DocuSign eSignature API

- **Description:** Leading e-signature platform with extensive API for embedding signing workflows into governance platforms. Supports eIDAS AdES and QES, ESIGN Act, and UETA compliance.
- **API Documentation:** https://developers.docusign.com/docs/esign-rest-api/
- **SDKs/Libraries:** Official SDKs for Python, Node.js, Java, C#, PHP, Ruby, Go
- **Developer Guide:** https://developers.docusign.com/docs/esign-rest-api/how-to/
- **Standards:** REST/JSON, OpenAPI 3.0, eIDAS, ESIGN Act, UETA
- **Authentication:** OAuth 2.0 (Authorization Code Grant, JWT Grant)

### Adobe Acrobat Sign API

- **Description:** Enterprise e-signature and document workflow API; strong competitor to DocuSign with native PDF handling. Relevant for board packs, resolutions, and consent workflows.
- **API Documentation:** https://developer.adobe.com/document-services/docs/overview/sign-api/
- **SDKs/Libraries:** Java, .NET, Node.js, PHP, Python, Ruby
- **Standards:** REST/JSON, OpenAPI, eIDAS, ESIGN Act
- **Authentication:** OAuth 2.0

### Companies House API (UK)

- **Description:** Official UK government API for company registration data — filings, officer lists, registered charges, and confirmation statements. Essential for UK entity management modules.
- **API Documentation:** https://developer.company-information.service.gov.uk/
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** API Key

### EDGAR Full-Text Search API (Efts)

- **Description:** SEC API for full-text searching of EDGAR filings. Useful for competitive intelligence, due diligence, and monitoring peer company disclosures.
- **API Documentation:** https://efts.sec.gov/LATEST/search-index?q=%22...%22&dateRange=custom&startdt=...&enddt=...
- **Standards:** REST/JSON, public
- **Authentication:** None

---

## Notes

- **Emerging standard:** The UK Companies House is transitioning to mandatory digital filing (XBRL-based) for annual accounts from 2026 onward, mirroring the EU iXBRL mandate — platforms must prepare for multi-jurisdiction structured filing support.
- **AI governance gap:** No established standard exists yet for AI-generated board minutes or AI-assisted resolution drafting; NIST AI RMF and emerging ISO/IEC 42001 (AI Management Systems, 2023) are the closest references and will likely be referenced in enterprise procurement requirements by 2027.
- **MCP opportunity:** No major corporate secretary platform currently publishes an MCP server — this represents a differentiation opportunity for an AI-native entrant.
